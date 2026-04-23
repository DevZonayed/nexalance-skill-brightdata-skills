---
name: python-sdk-best-practices
description: |
  Web data extraction and discovery using the Bright Data Python SDK.
  Use when user asks to "scrape", "get data from", "extract", "search for",
  or "find" information from websites. Also use when user mentions specific
  platforms like Amazon, LinkedIn, Instagram, Facebook, TikTok, YouTube,
  Reddit, Pinterest, Zillow, Crunchbase, or DigiKey, or asks for "bulk data",
  "historical data", or "dataset". Covers scraping, searching, datasets,
  and browser automation.
metadata:
  author: brightdata
  version: "1.0"
---

# Bright Data SDK

Access web data through a unified Python SDK. One client, eight service categories: platform scraping, platform search, web search (SERP), AI-powered discovery, datasets, web unlocking, browser automation, and scraper studio.

Always use the client as a context manager. In synchronous environments (scripts, notebooks, Claude Code), use `SyncBrightDataClient`. In async environments, use `BrightDataClient`. Both use the same method names — the sync client wraps calls automatically. Note: the sync client currently has limited platform coverage — see the sync compatibility note in `references/scrapers.md` for details. For unsupported platforms or the datasets API, use the async client (`BrightDataClient`).

## How to Handle Requests

### Exploring capabilities

If the user wants to know what's available or asks "what can this do?", describe these categories:

1. **Platform scraping** — extract structured data from 11 supported platforms (Amazon, LinkedIn, Facebook, Instagram, YouTube, TikTok, Reddit, ChatGPT, Perplexity, Pinterest, DigiKey)
2. **Platform search** — search within platforms (Amazon products, LinkedIn profiles/jobs, Instagram posts, TikTok videos, YouTube channels, Pinterest pins)
3. **Web search (SERP)** — search Google, Bing, or Yandex and get structured results
4. **AI-powered discovery** — find entities (companies, people, products) matching natural language intent
5. **Datasets** — access 310+ pre-built datasets with historical/bulk data across e-commerce, social media, business, real estate, reviews, and more
6. **Web unlocking** — scrape any URL with anti-bot bypass for sites without a dedicated scraper
7. **Browser automation** — connect via CDP for login flows, JavaScript-heavy pages, click/scroll/fill interactions
8. **Scraper Studio** — run pre-built or custom scraping templates via collector IDs

Offer to load the relevant reference file for details on any category.

### Data extraction from a specific URL

The user has a URL and wants structured data from it.

If the URL is from a supported platform (Amazon, LinkedIn, Facebook, Instagram, YouTube, TikTok, Reddit, ChatGPT, Perplexity, Pinterest, DigiKey — see `references/scrapers.md` for the full list and available methods):
- Use `client.scrape.<platform>.<method>(url=...)`
- Read `references/scrapers.md` for available methods per platform

If the URL is from an unsupported platform or a generic website:
- Use `client.scrape_url(url=...)` for raw page data with anti-bot bypass
- Read `references/advanced.md` for web unlocker options

If the user has MULTIPLE URLs (batch):
- Use BrightDataClient and trigger methods (`_trigger` suffix) to avoid sequential blocking
- Fire all triggers first, then collect results with `job.wait()` and `job.to_result()`
- Read `references/advanced.md` for batch execution patterns

### Research or discovery without a specific URL

The user wants to find information but doesn't have a starting URL.

For web search results (links, snippets, rankings):
- Use `client.search.google(query=...)`, `client.search.bing(query=...)`, or `client.search.yandex(query=...)`
- Read `references/search.md` for available search engines and parameters

For platform-specific search (find products on Amazon, profiles on LinkedIn, videos on YouTube, etc.):
- Use `client.search.<platform>.<method>(...)`
- Read `references/scrapers.md` — search methods are listed under each platform

For deeper discovery (find companies, people, or entities matching criteria):
- Use `client.discover(query=..., intent=...)`
- The Discover API requires an intent phrase, not just keywords
- Read `references/search.md` for discover API details

### Bulk or historical data needs

The user asks for "bulk data", "historical data", "database", "list of", or wants data at scale without scraping individual pages.

- Use `client.datasets.list()` at runtime to discover available datasets
- Read `references/datasets-overview.md` for dataset categories and usage patterns
- Create a filtered snapshot: `snapshot_id = client.datasets.<name>(filter={...})`
- Download data: `data = client.datasets.<name>.download(snapshot_id)` (default format is jsonl; also supports json, csv)
- Snapshots take time to build — download blocks until ready (up to 5 minutes)

### Multi-step research workflow

The user has a broad research goal (e.g., "research competitors in Berlin").

Step 1: Find sources
- `client.discover(query=..., intent=...)` for entity-level discovery
- OR `client.search.google(query=...)` for web search results

Step 2: Extract data from discovered sources
- `client.scrape.<platform>.<method>(url=...)` on each discovered URL
- Use trigger methods for batch processing if many URLs

Step 3: Optionally enrich with bulk data
- Check `client.datasets` for historical context on the entities found

### Interactive web tasks

The user needs login, clicking, scrolling, form filling, or JavaScript execution.

- Use `client.browser.get_connect_url()` to get a CDP WebSocket URL
- Connect with Playwright, Puppeteer, or another CDP client
- This is the most expensive option — only use when simpler methods cannot accomplish the task
- Read `references/advanced.md` for browser API details

### Scraper Studio templates

The user wants to use a pre-built or custom scraping template.

- Use `client.scraper_studio.run(collector="c_xxx", input={...})`
- Requires a collector ID — the user must provide this or know which template to use
- Read `references/advanced.md` for scraper studio details

## Gotchas

- Do NOT use browser automation for simple product/profile scraping — platform scrapers are 10x cheaper and faster. Browser API is a last resort for interactive tasks only.
- Datasets return HISTORICAL data, not live/real-time data. If the user needs current data, use platform scrapers or web unlocker instead.
- The Discover API requires an INTENT (natural language description of what you're looking for), not just a keyword. Rephrase bare keywords like "restaurants" into intent phrases like "find Italian restaurants with outdoor seating in downtown Austin."
- When a scraper returns 403 or is blocked, try `client.scrape_url()` (web unlocker) as fallback — it handles anti-bot protections.
- Always prefer the cheapest service that satisfies the request. Cost hierarchy (cheapest first): datasets → SERP → platform scrapers → web unlocker → discover → scraper studio → browser API.
- Always use the client as a context manager. Never create multiple client instances — reuse one client across all operations in a session.
- Each scraper supports 3 execution patterns: quick (blocks until result), trigger (returns job immediately), manual (trigger + status + fetch). Default to quick unless the user needs batch processing or non-blocking execution.
- Quick methods block for several minutes depending on platform and page complexity. Do NOT set short timeouts — the SDK defaults are calibrated per platform. Expect 2-10 minutes for most operations.
- The SDK auto-retries network errors and timeouts (3 retries, exponential backoff). Do NOT add your own retry logic on top — it will double-retry and waste API credits.
- Default rate limit: 10 requests/second. When processing multiple items, call sequentially or use trigger methods. Do NOT fire parallel quick calls for batch operations — you will hit the rate limiter.
- Dataset operations return a `snapshot_id`, not data directly. Snapshots go through a lifecycle: scheduled → building → ready. Use `.download(snapshot_id)` which blocks until the snapshot is ready. Supported download formats: json, jsonl, csv.
- For batch work or notebook environments, use trigger methods (`_trigger` suffix) for non-blocking execution, then `job.wait()` to poll and `job.to_result()` to collect results. This avoids sequential blocking on long-running operations.
- In synchronous environments (scripts, notebooks, Claude Code), use `SyncBrightDataClient`. In async environments, use `BrightDataClient`. Both use the SAME method names — the only difference is that async calls need `await`. Do NOT use `_sync` suffix methods with `SyncBrightDataClient`. Note: the sync client has limited platform coverage. Sync scraping supports: Amazon, LinkedIn, Instagram, Facebook, ChatGPT, Pinterest. Sync search supports: Google, Bing, Yandex, Amazon, LinkedIn, Instagram, ChatGPT, Pinterest. For TikTok, YouTube, Reddit, Perplexity, DigiKey scrapers/search and the datasets API, use the async client.
- Platform search methods (e.g., `client.search.amazon.products()`) are different from platform scrapers (e.g., `client.scrape.amazon.products()`). Search finds items by keyword. Scrape extracts data from a specific URL.

## Examples

### "Get me reviews for this Amazon product"

Use `client.scrape.amazon.reviews(url="<the_url>")`.
Returns structured review data: rating, text, date, reviewer name.
Quick method — blocks until complete (up to ~4 minutes).

### "Find AI startups in Berlin"

Step 1: `client.discover(query="AI startups in Berlin", intent="find technology companies")`
Returns a list of matching entities with URLs and metadata.

Step 2: For each result with a URL, optionally scrape deeper data:
`client.scrape.linkedin.companies(url=...)` or `client.scrape_url(url=...)`.

### "I need historical pricing data for electronics"

Step 1: `client.datasets.list()` to find relevant datasets.

Step 2: Create a filtered snapshot:
`snapshot_id = client.datasets.amazon_products(filter={"name": "category", "operator": "=", "value": "Electronics"}, records_limit=1000)`

Step 3: Download the data:
`data = client.datasets.amazon_products.download(snapshot_id)`

Note: Download blocks while the snapshot builds (up to 5 minutes). Default format is jsonl (also supports json, csv). This is historical/bulk data, not live prices. Returns a list of records.

## Troubleshooting

- **401 Unauthorized**: API token is invalid or expired. Check the token passed to the client constructor.
- **403 Forbidden / Blocked**: The target site blocked the request. Try `client.scrape_url()` (web unlocker) as fallback, or use a different scraper method.
- **Timeout**: Do not lower the timeout — increase it. Some operations take several minutes. Platform-specific defaults are already optimized.
- **"Dataset not found"**: Use `client.datasets.list()` to see available datasets. Dataset attribute names are snake_case (e.g., `amazon_products`, `linkedin_profiles`).
- **SSL/Proxy errors in sandboxed environments**: Pass `ssl_verify=False` to the client constructor to skip SSL verification, or use `ssl_ca_cert='/path/to/cert.pem'` for custom certificate handling.
- **Rate limit errors**: Reduce concurrency. Default limit is 10 requests/second. Use sequential calls or trigger methods for batch work.

## When to Load References

- Read `references/scrapers.md` when the user mentions **Amazon, LinkedIn, Facebook, Instagram, YouTube, TikTok, Reddit, ChatGPT, Perplexity, Pinterest, DigiKey** or other specific platforms — to see available scraper methods, search methods, and parameters for that platform.
- Read `references/search.md` when the user asks to **"find", "search", "discover", "research", "look up"** something without mentioning a specific platform — to see SERP engines and Discover API options.
- Read `references/datasets-overview.md` when the user asks for **"bulk data", "historical data", "database", "list of", "dataset"** or wants data at scale — to see dataset categories and how to discover specific datasets at runtime.
- Read `references/advanced.md` when the user needs **batch processing of multiple URLs, non-blocking execution, browser automation, JavaScript execution, login/session handling, custom scraping templates, or when simpler methods have failed** — to see execution patterns, batch workflows, Web Unlocker, Browser API, and Scraper Studio details.
