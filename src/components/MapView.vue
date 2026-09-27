<template>
  <PlaygroundLayout
    title="Map"
    subtitle="Discover all URLs on a website"
    :tabs="[
      { key: 'links', label: 'Links' },
      { key: 'json', label: 'JSON' },
    ]"
    :running="loading"
    :error="error || null"
    :has-result="hasResult"
    :status="statusText"
    :status-type="statusType"
    :duration="durationMs"
    empty-hint="Enter a URL and click Find URLs to discover all links on the site."
  >
    <!-- ── Request pane ─────────────────────────────────────────────────── -->
    <template #request>
      <form @submit.prevent="handleSubmit" class="map-form">
        <div class="form-group">
          <label for="baseUrl">Base URL</label>
          <input
            id="baseUrl"
            v-model="baseUrl"
            type="url"
            placeholder="https://example.com"
            required
          />
        </div>

        <div class="form-group">
          <label for="search">Search Query</label>
          <input id="search" v-model="search" type="text" placeholder="Optional search terms" />
        </div>

        <div class="form-group">
          <label for="sitemapMode">Sitemap Mode</label>
          <select id="sitemapMode" v-model="sitemapMode">
            <option value="include">Include sitemap</option>
            <option value="skip">Skip sitemap</option>
            <option value="only">Only sitemap URLs</option>
          </select>
        </div>

        <div class="form-group form-group--checkbox">
          <label class="checkbox-label">
            <input type="checkbox" v-model="includeSubdomains" />
            Include Subdomains
          </label>
        </div>

        <div class="form-group">
          <label for="limit">Limit</label>
          <input id="limit" v-model.number="limit" type="number" min="1" max="30000" />
        </div>

        <div class="form-group">
          <label for="timeout">Timeout (ms)</label>
          <input id="timeout" v-model.number="timeout" type="number" min="0" />
        </div>

        <button type="submit" class="submit-btn" :disabled="loading">
          <span v-if="loading" class="btn-spinner" aria-hidden="true"></span>
          {{ loading ? 'Mapping…' : 'Find URLs' }}
        </button>
      </form>
    </template>

    <!-- ── Response pane ───────────────────────────────────────────────── -->
    <template #response="{ activeTab }">
      <!-- Single notice banner: fallback disclaimer with optional API warning -->
      <div
        v-if="notice"
        class="notice-banner"
        :class="noticeType === 'warning' ? 'notice-banner--warning' : 'notice-banner--info'"
      >
        <svg viewBox="0 0 24 24" width="18" height="18" aria-hidden="true">
          <path d="M12 22a10 10 0 1 0 0-20 10 10 0 0 0 0 20z" />
          <path d="M12 16v-4" />
          <path d="M12 8h.01" />
        </svg>
        <span>{{ notice }}</span>
      </div>

      <!-- Links tab -->
      <div v-if="activeTab === 'links'" class="links-pane">
        <!-- Zero-links state -->
        <div v-if="showZero" class="zero-state">
          <svg viewBox="0 0 24 24" width="32" height="32" aria-hidden="true">
            <path d="M4 7V5a1 1 0 0 1 1-1h14a1 1 0 0 1 1 1v2" />
            <path d="M4 7h16v12a1 1 0 0 1-1 1H5a1 1 0 0 1-1-1z" />
            <path d="M8 12h8M8 16h5" />
          </svg>
          <p>No links found.</p>
        </div>

        <ul v-else-if="urls.length > 0" class="url-list">
          <li v-for="(url, index) in urls" :key="index" class="url-item">
            <span class="url-text" :title="url">{{ url }}</span>
            <div class="url-actions">
              <CopyButton :text="url" label="Copy URL" />
              <a
                :href="url"
                target="_blank"
                rel="noopener noreferrer"
                class="open-link"
                :aria-label="`Open ${url} in a new tab`"
              >
                <svg viewBox="0 0 24 24" width="13" height="13" aria-hidden="true">
                  <path d="M18 13v6a2 2 0 0 1-2 2H5a2 2 0 0 1-2-2V8a2 2 0 0 1 2-2h6" />
                  <polyline points="15 3 21 3 21 9" />
                  <line x1="10" y1="14" x2="21" y2="3" />
                </svg>
                Open
              </a>
            </div>
          </li>
        </ul>
      </div>

      <!-- JSON tab -->
      <CodeBlock v-else-if="activeTab === 'json'" :json="urls" label="JSON" />
    </template>

    <!-- ── Response actions: copy + download buttons in tab bar ───────── -->
    <template #response-actions>
      <CopyButton v-if="urls.length > 0" :text="urlListText" label="Copy URL list" />
      <button v-if="urls.length > 0" type="button" class="download-btn" @click="downloadJson">
        Download JSON
      </button>
    </template>
  </PlaygroundLayout>
</template>

<script setup lang="ts">
import { ref, computed, inject, onMounted } from 'vue';
import { useRoute } from 'vue-router';
import type { FirecrawlMappingApi, FirecrawlScrapingApi } from '@/services/firecrawl';
import PlaygroundLayout from './playground/PlaygroundLayout.vue';
import CodeBlock from './playground/CodeBlock.vue';
import CopyButton from './playground/CopyButton.vue';

/**
 * MapView Component
 *
 * Allows users to map URLs starting from a specified base URL, with optional
 * search, sitemap handling, subdomain inclusion, and limits. If /v2/map returns
 * zero links, a single bounded /v2/scrape fallback discovers homepage links.
 */

/**
 * Injects the API instance provided by the API plugin.
 * Expected to expose `mapping` and `scraping` clients.
 */
const api = inject('api') as
  { mapping?: FirecrawlMappingApi; scraping?: FirecrawlScrapingApi } | undefined;
if (!api?.mapping) {
  throw new Error(
    'Mapping API is not available. Ensure the API plugin provides the mapping service.',
  );
}
if (!api?.scraping) {
  throw new Error(
    'Scraping API is not available. Ensure the API plugin provides the scraping service.',
  );
}

const baseUrl = ref('');
const search = ref('');
const sitemapMode = ref<'include' | 'skip' | 'only'>('include');
const includeSubdomains = ref(false);
const limit = ref(5000);
const timeout = ref<number | null>(null);

const route = useRoute();
onMounted(() => {
  const q = route.query.url;
  const initialUrl = Array.isArray(q) ? q[0] : q;
  if (initialUrl) {
    baseUrl.value = String(initialUrl);
  }
});

const urls = ref<string[]>([]);
const loading = ref(false);
const error = ref('');
const completed = ref(false);
const durationMs = ref<number | null>(null);

const apiWarning = ref('');
const fallbackNote = ref('');
const fallbackWarning = ref('');

// ── Computed helpers ───────────────────────────────────────────────────────

/** True once a Map request has succeeded, even when it returns zero links. */
const hasResult = computed<boolean>(() => completed.value);

/** Plain-text URL list for the copy action. */
const urlListText = computed<string>(() => urls.value.join('\n'));

/** Status bar text: count on completion (including zero), 'Error' on failure. */
const statusText = computed<string | null>(() => {
  if (error.value) return 'Error';
  if (!completed.value) return null;
  return `${urls.value.length} link${urls.value.length === 1 ? '' : 's'}`;
});

/** Semantic status color. */
const statusType = computed<'success' | 'error' | 'idle'>(() => {
  if (error.value) return 'error';
  if (completed.value) return 'success';
  return 'idle';
});

/**
 * Single notice banner: always surfaces a successful fallback disclaimer or a
 * fallback failure reason, and appends any API warning so both remain visible
 * in one banner.
 */
const notice = computed<string | null>(() => {
  const parts: string[] = [];
  if (fallbackNote.value) {
    parts.push(fallbackNote.value);
  } else if (fallbackWarning.value) {
    parts.push(fallbackWarning.value);
  }
  if (apiWarning.value) {
    parts.push(apiWarning.value);
  }
  return parts.length > 0 ? parts.join(' ') : null;
});

/** Whether the notice is a warning (yellow) or an info note (blue). */
const noticeType = computed<'warning' | 'info'>(() => {
  if (apiWarning.value || fallbackWarning.value) return 'warning';
  return 'info';
});

/** True when a request completed successfully but produced no URLs. */
const showZero = computed<boolean>(
  () => completed.value && !loading.value && !error.value && urls.value.length === 0,
);

// ── Request handling ───────────────────────────────────────────────────────

/**
 * Submits the map request and, when applicable, runs a single scrape fallback
 * for empty map results.
 */
async function handleSubmit(): Promise<void> {
  // Reset per-request state so stale results, errors, and warnings do not persist.
  urls.value = [];
  error.value = '';
  completed.value = false;
  apiWarning.value = '';
  fallbackNote.value = '';
  fallbackWarning.value = '';

  const payload = {
    url: baseUrl.value,
    search: search.value || undefined,
    sitemap: sitemapMode.value,
    includeSubdomains: includeSubdomains.value,
    limit: limit.value || undefined,
    timeout: timeout.value ?? undefined,
  };

  loading.value = true;
  const t0 = performance.now();

  try {
    const response = await api.mapping.mapUrls(payload);

    // Mark completed immediately on a successful Map HTTP response.
    completed.value = true;

    const rawLinks = response.data.links;
    urls.value = normalizeUrls(Array.isArray(rawLinks) ? rawLinks : [], baseUrl.value, {
      includeSubdomains: includeSubdomains.value,
      limit: limit.value,
    });

    apiWarning.value = typeof response.data.warning === 'string' ? response.data.warning : '';

    // Single bounded fallback: only when map returned zero links, no search,
    // sitemap is not "only", and formats/links is requested with onlyMainContent false.
    if (urls.value.length === 0 && sitemapMode.value !== 'only' && !search.value) {
      const scrapeTimeout = Math.min(
        typeof timeout.value === 'number' && timeout.value > 0 ? timeout.value : 30000,
        30000,
      );

      try {
        const scrapeResponse = await api.scraping.scrapeAndExtractFromUrl({
          url: baseUrl.value,
          formats: ['links'],
          onlyMainContent: false,
          timeout: scrapeTimeout,
        });

        const rawScrapeLinks = scrapeResponse.data.data?.links;
        const fallbackLinks = normalizeUrls(
          Array.isArray(rawScrapeLinks) ? rawScrapeLinks : [],
          baseUrl.value,
          {
            includeSubdomains: includeSubdomains.value,
            limit: limit.value,
          },
        );

        if (fallbackLinks.length > 0) {
          urls.value = fallbackLinks;
          fallbackNote.value = 'Homepage only, not full site.';
        } else {
          fallbackWarning.value =
            'Homepage-only discovery returned no links; not a complete site map.';
        }
      } catch (scrapeErr: unknown) {
        const message =
          scrapeErr instanceof Error ? scrapeErr.message : 'Homepage-only discovery failed';
        fallbackWarning.value = `${message}; not a complete site map.`;
      }
    }
  } catch (err: unknown) {
    error.value =
      err instanceof Error
        ? err.message
        : 'Failed to map URLs. Please check the URL and try again.';
    urls.value = [];
    completed.value = false;
  } finally {
    durationMs.value = performance.now() - t0;
    loading.value = false;
  }
}

/**
 * Normalizes discovered URLs: only http(s), filters to the base host (with
 * optional subdomain matching), treats www and non-www as the same canonical
 * host, removes duplicates, and caps the result count.
 *
 * @param rawLinks - Raw link values of unknown type from the API.
 * @param baseUrlStr - The base URL used to resolve relative links.
 * @param options - Normalization options.
 * @returns A deduplicated, capped array of normalized absolute URLs.
 */
function normalizeUrls(
  rawLinks: unknown[],
  baseUrlStr: string,
  options: { includeSubdomains?: boolean; limit?: number },
): string[] {
  let baseHost: string;
  try {
    baseHost = new URL(baseUrlStr).hostname.toLowerCase();
  } catch {
    return [];
  }

  const baseCanonical = baseHost.startsWith('www.') ? baseHost.slice(4) : baseHost;
  const rawLimit = typeof options.limit === 'number' && options.limit > 0 ? options.limit : 30000;
  const cap = Math.min(rawLimit, 30000);
  const seen = new Set<string>();
  const result: string[] = [];

  for (const raw of rawLinks) {
    if (typeof raw !== 'string' || raw.trim() === '') {
      continue;
    }

    try {
      const url = new URL(raw.trim(), baseUrlStr);
      const protocol = url.protocol.toLowerCase();
      if (protocol !== 'http:' && protocol !== 'https:') {
        continue;
      }

      const host = url.hostname.toLowerCase();
      const canonical = host.startsWith('www.') ? host.slice(4) : host;

      const exactOrAlias = canonical === baseCanonical;
      const subdomain = host.endsWith('.' + baseHost) || host.endsWith('.' + baseCanonical);
      const hostnameMatch = exactOrAlias || (options.includeSubdomains === true && subdomain);

      if (!hostnameMatch) {
        continue;
      }

      const href = url.href;
      if (seen.has(href)) {
        continue;
      }
      seen.add(href);
      result.push(href);

      if (result.length >= cap) {
        break;
      }
    } catch {
      // Ignore malformed URLs.
    }
  }

  return result;
}

/**
 * Downloads the URL list as a JSON file.
 */
function downloadJson(): void {
  const blob = new Blob([JSON.stringify(urls.value, null, 2)], {
    type: 'application/json',
  });
  const link = document.createElement('a');
  link.href = URL.createObjectURL(blob);
  link.download = 'urls.json';
  document.body.appendChild(link);
  link.click();
  document.body.removeChild(link);
  URL.revokeObjectURL(link.href);
}
</script>

<style scoped>
/* ── Request pane form ───────────────────────────────────────────────────── */

.map-form {
  display: flex;
  flex-direction: column;
  gap: 1rem;
}

.form-group {
  display: flex;
  flex-direction: column;
  gap: 0.35rem;
}

.form-group label {
  font-size: 0.83rem;
  font-weight: 600;
  color: var(--color-text-soft);
}

.form-group input[type='url'],
.form-group input[type='text'],
.form-group input[type='number'],
.form-group select {
  padding: 0.45rem 0.7rem;
  font-size: 0.93rem;
  font-family: var(--font-sans);
  color: var(--color-text);
  background: var(--color-background-soft);
  border: 1px solid var(--color-border);
  border-radius: var(--radius-sm);
  outline: none;
  -webkit-backdrop-filter: blur(6px);
  backdrop-filter: blur(6px);
  transition:
    border-color var(--transition-fast),
    box-shadow var(--transition-fast);
}

.form-group input:focus,
.form-group select:focus {
  border-color: var(--violet-500);
  box-shadow: var(--shadow-ring);
}

.form-group--checkbox {
  flex-direction: row;
  align-items: center;
}

.checkbox-label {
  display: flex;
  align-items: center;
  gap: 0.5rem;
  font-size: 0.88rem;
  font-weight: 500;
  color: var(--color-text);
  cursor: pointer;
}

.submit-btn {
  display: inline-flex;
  align-items: center;
  justify-content: center;
  gap: 0.5rem;
  align-self: flex-start;
  padding: 0.5rem 1.1rem;
  font-size: 0.93rem;
  font-weight: 600;
  color: #fff;
  background: var(--gradient-violet);
  border: none;
  border-radius: var(--radius-sm);
  cursor: pointer;
  box-shadow: var(--box-shadow-button);
  transition:
    background var(--transition-fast),
    transform var(--transition-fast),
    box-shadow var(--transition-fast);
}

.submit-btn:hover:not(:disabled) {
  background: var(--gradient-violet-hover);
  transform: translateY(-1px);
  box-shadow: 0 10px 26px -6px rgba(124, 92, 255, 0.6);
}

.submit-btn:disabled {
  opacity: 0.6;
  cursor: not-allowed;
}

.btn-spinner {
  width: 14px;
  height: 14px;
  border-radius: 50%;
  border: 2px solid rgba(255, 255, 255, 0.4);
  border-top-color: #fff;
  animation: spin 0.65s linear infinite;
  flex-shrink: 0;
}

@keyframes spin {
  to {
    transform: rotate(360deg);
  }
}

/* ── Response notices ────────────────────────────────────────────────────── */

.notice-banner {
  display: flex;
  align-items: center;
  gap: 0.5rem;
  padding: 0.65rem 0.9rem;
  border-radius: var(--radius-sm);
  font-size: 0.88rem;
  font-weight: 500;
  margin-bottom: 0.75rem;
}

.notice-banner--info {
  color: var(--hue-info);
  background: var(--hue-info-soft);
}

.notice-banner--warning {
  color: var(--hue-warning);
  background: var(--hue-warning-soft);
}

.notice-banner svg {
  flex-shrink: 0;
  fill: none;
  stroke: currentColor;
  stroke-width: 2;
  stroke-linecap: round;
  stroke-linejoin: round;
}

.zero-state {
  display: flex;
  flex-direction: column;
  align-items: center;
  justify-content: center;
  gap: 0.7rem;
  text-align: center;
  padding: 2rem;
  color: var(--color-text-mute);
}

.zero-state svg {
  fill: none;
  stroke: currentColor;
  stroke-width: 1.7;
  stroke-linecap: round;
  stroke-linejoin: round;
  opacity: 0.75;
}

/* ── Links tab ───────────────────────────────────────────────────────────── */

.links-pane {
  height: 100%;
  overflow-y: auto;
  padding: 0.5rem 0;
}

.url-list {
  list-style: none;
  margin: 0;
  padding: 0;
  display: flex;
  flex-direction: column;
}

.url-item {
  display: flex;
  align-items: center;
  justify-content: space-between;
  gap: 0.6rem;
  padding: 0.5rem 1rem;
  border-bottom: 1px solid var(--color-border);
  min-width: 0;
  transition:
    background var(--transition-fast),
    border-color var(--transition-fast);
}

.url-item:hover {
  background: var(--glass-fill);
  border-color: var(--color-border-hover);
}

.url-item:last-child {
  border-bottom: none;
}

.url-text {
  flex: 1;
  min-width: 0;
  font-size: 0.84rem;
  font-family: var(--font-mono);
  color: var(--color-text);
  white-space: nowrap;
  overflow: hidden;
  text-overflow: ellipsis;
}

.url-actions {
  display: flex;
  align-items: center;
  gap: 0.35rem;
  flex-shrink: 0;
}

.open-link {
  display: inline-flex;
  align-items: center;
  gap: 0.3rem;
  font-size: 0.78rem;
  font-weight: 600;
  color: var(--color-text-soft);
  background: var(--color-background-mute);
  border: 1px solid var(--color-border);
  border-radius: var(--radius-sm);
  padding: 0.3rem 0.55rem;
  text-decoration: none;
  transition:
    color var(--transition-fast),
    border-color var(--transition-fast),
    background var(--transition-fast);
}

.open-link:hover {
  color: var(--brand-strong);
  border-color: var(--violet-500);
  background: var(--brand-soft);
}

.open-link svg {
  fill: none;
  stroke: currentColor;
  stroke-width: 2;
  stroke-linecap: round;
  stroke-linejoin: round;
}

/* ── Download button ─────────────────────────────────────────────────────── */

.download-btn {
  padding: 0.35rem 0.8rem;
  font-size: 0.8rem;
  font-weight: 600;
  color: #fff;
  background: var(--gradient-violet);
  border: none;
  border-radius: var(--radius-sm);
  cursor: pointer;
  box-shadow: var(--box-shadow-button);
  transition:
    background var(--transition-fast),
    transform var(--transition-fast),
    box-shadow var(--transition-fast);
}

.download-btn:hover {
  background: var(--gradient-violet-hover);
  transform: translateY(-1px);
  box-shadow: 0 8px 22px -6px rgba(124, 92, 255, 0.6);
}
</style>
