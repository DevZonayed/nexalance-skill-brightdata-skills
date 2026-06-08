# Search & Discover reference — `@brightdata/sdk`

Two distinct services for "find things without a URL":

- **SERP** (`client.search.*`) — structured search-engine results (links, titles, snippets, rankings).
- **Discover** (`client.discover`) — AI-ranked discovery of pages/entities by natural-language intent.

> The JS SDK has **no platform search router** (no `client.search.amazon`). To
> search within a platform, use that platform's `discover*` scraper methods — see
> `references/scrapers.md`.

---

## SERP — `client.search.<engine>(query, options?)`

Engines: **`google`**, **`bing`**, **`yandex`**. `query` is a string or `string[]`
(array → parallel batch). Returns `Promise<object[]>`.

| Option | Values | Notes |
|---|---|---|
| `country` | two-letter code | geo-target the SERP |
| `format` | `'json'` (and engine-native formats) | structured output |

```javascript
// single
const r = await client.search.google('best mechanical keyboards');

// batch (parallel)
const rs = await client.search.google(['pizza near me', 'sushi near me']);

// with options
const gb = await client.search.google('vat registration', { country: 'gb', format: 'json' });

// other engines
const b = await client.search.bing('quantum computing news');
const y = await client.search.yandex('погода москва', { country: 'ru' });
```

Use SERP when the user wants **web pages / links / rankings**, not entities.

---

## Discover — `client.discover(query, options?)`

AI-powered discovery with intent-based relevance ranking. Returns
`Promise<object[]>` where each item is roughly `{ link, title, description, relevance_score }`.

| Option | Type | Notes |
|---|---|---|
| `intent` | string | natural-language description of what you want — **strongly recommended**; ranks results semantically |
| `filterKeywords` | `string[]` | keep only results matching these keywords |
| `country` | string | two-letter code |
| `numResults` | number | how many to return |
| `includeContent` | boolean | include full page content per result (slower, larger) |

```javascript
// basic
const a = await client.discover('artificial intelligence trends 2026');

// with intent (preferred)
const b = await client.discover('Tesla battery technology', {
  intent: 'recent breakthroughs in EV battery chemistry',
});

// filtered + geo + count
const c = await client.discover('sustainable fashion brands', {
  intent: 'eco-friendly clothing companies',
  filterKeywords: ['sustainability', 'organic'],
  country: 'us',
  numResults: 10,
});

// include full page content
const d = await client.discover('node.js streams tutorial', { includeContent: true, numResults: 3 });
```

**Give `discover` an intent, not a bare keyword.** Rephrase "restaurants" →
`intent: 'find Italian restaurants with outdoor seating in downtown Austin'`.

Use Discover when the user wants **entities / a curated list matching criteria**
("find AI startups in Berlin", "competitors of Acme", "people who worked at X").

### Non-blocking — `client.discoverTrigger(query, options?)`

Same options; returns a `Job` for manual polling.

```javascript
const job = await client.discoverTrigger('SaaS pricing strategies', { intent: 'competitor pricing pages' });
await job.wait({ timeout: 60_000 });
const data = await job.fetch();
```

---

## Choosing SERP vs Discover vs platform discovery

| Want | Use |
|---|---|
| Google/Bing/Yandex result links & rankings | `client.search.google/bing/yandex(query)` |
| A ranked list of entities matching a description | `client.discover(query, { intent })` |
| Products/jobs/posts *within* a platform by keyword | `client.scrape.<platform>.discover*` / `amazon.productSearch` (see scrapers.md) |
