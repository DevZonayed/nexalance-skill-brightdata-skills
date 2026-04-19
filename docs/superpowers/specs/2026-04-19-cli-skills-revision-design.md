# CLI Skills Revision — Design Spec

**Date:** 2026-04-19
**Scope:** Revise three Bright Data skills (`scrape`, `search`, `data-feeds`) from bash+curl implementations to superpowers-style workflows that drive the Bright Data CLI (`bdata`).
**Out of scope:** All other skills (`brd-browser-debug`, `design-mirror`, `competitive-intel`, `scraper-builder`, `brightdata-cli`, `bright-data-best-practices`, `bright-data-mcp`, `python-sdk-best-practices`).

---

## Goals

1. Replace direct `curl` calls to Bright Data API with `bdata` CLI commands.
2. Reshape each skill from a reference document into a **workflow** — trigger, setup gate, decision branching, action, verification, red flags — in the style of superpowers skills.
3. Add proactive setup guidance: detect missing CLI / missing auth and walk the user through `bdata login`.
4. Preserve a documented env-var fallback for legacy users who haven't migrated.
5. Establish cross-skill handoff discipline so agents route work to the right skill (scrape ↔ search ↔ data-feeds).

## Non-goals

- Adding new Bright Data API features.
- Changing CLI behavior. Skills must work with `@brightdata/cli` as it exists today (v0.1.8).
- Touching skills outside the three listed.
- Keeping the existing `scripts/*.sh` files — they are removed; the CLI replaces them.

---

## Ground truth — CLI surface

Verified from the CLI source at `/home/meirk/cli/src/commands/`:

| Command | Args | Key flags |
|---|---|---|
| `bdata scrape <url>` | URL | `-f/--format <markdown\|html\|json>`, `--country <iso>`, `--zone <name>`, `--mobile`, `--async`, `-o/--output <path>`, `--json`, `--pretty`, `--timing` |
| `bdata search <query>` | query string | `--engine <google\|bing\|yandex>`, `--country`, `--language`, `--page <n>`, `--type`, `--zone`, `--device <desktop\|mobile>`, `-o/--output`, `--json`, `--pretty` |
| `bdata discover <query>` | query string | `--intent`, `--country`, `--city`, `--language`, `--num-results <n>`, `--filter-keywords`, `--include-content`, `--no-remove-duplicates`, `--start-date`, `--end-date`, `--timeout`, `-o/--output` |
| `bdata pipelines <type> [params...]` | pipeline type (e.g. `amazon_product`) + params | `--format`, `--timeout <sec>`, `-o/--output`, `--json`, `--pretty`, `--timing` |
| `bdata pipelines list` | — | lists available pipeline types |
| `bdata config` | — | shows current config/auth state |
| `bdata login` / `bdata login --device` / `bdata login --api-key <k>` | — | authenticates and provisions zones |

**Critical facts that shape the design:**

- **No `--batch` flag exists** on any command. Batching is done by the agent via shell loops (`xargs`, `while read`, `for` loops) — and that's OK to recommend; it's the actual idiom.
- **User-facing command is `bdata pipelines`** (the implementation file is `dataset.ts` internally, but the CLI exposes `pipelines`). `bdata pipelines list` returns the available types.
- **`bdata pipelines` polls internally** until the job completes or `--timeout` is hit. No separate `--wait`/`--async` flag on pipelines. Users don't manage jobs manually.
- **`bdata scrape --async`** exists and returns a job ID. Useful for very slow pages; the agent must poll the returned job themselves.
- **`search` has no `--limit`.** Result volume is controlled via `--page` for pagination.
- **Env var `BRIGHTDATA_POLLING_TIMEOUT`** is still honored by the CLI (consulted inside `dataset`). Legacy env-var fallback for that is unchanged.

---

## Architecture

### Per-skill structure

```
skills/<name>/
  SKILL.md              # workflow-shaped
  references/
    flags.md            # full CLI flag reference for this skill's domain
    patterns.md         # batching (shell loops), pagination, retries, error handling, handoff decisions
    examples.md         # 2–4 worked examples per skill
```

The existing `scripts/` directory in each of the three skills is **removed**. Content fetching is done by calling `bdata` directly; no bash wrapper layer.

### SKILL.md workflow skeleton

Every one of the three SKILL.md files follows this shape, in this order:

1. **Frontmatter** — `name`, `description` (trigger), `user-invocable: true` where applicable.
2. **One-paragraph purpose** — what class of tasks this skill owns.
3. **Setup gate** (≤ 10 lines inline) — a check recipe that the agent runs *before* doing anything else:
   - Run `bdata config` — if it errors or shows unauthenticated, route to the install/login flow in [references/cli-setup.md](#shared-setup-reference) (shared, linked from each skill).
   - If the CLI binary is missing: offer `npm install -g @brightdata/cli` (or the curl installer).
   - Env-var fallback: if `bdata` is unavailable but `BRIGHTDATA_API_KEY` (and the skill-specific zone var) are set, fall through to a documented `curl` path in `references/patterns.md`. This path is marked **legacy**.
4. **Pick-your-path** — a short branching block naming the 3–5 shapes this skill handles, plus explicit handoffs to the other two skills.
5. **Action** — the CLI command(s) with required + common flags. Flag-by-flag depth lives in `references/flags.md`.
6. **Verification gate** — a checklist the agent must run before claiming success:
   - Output is non-empty.
   - Output doesn't match block-page signatures (shared list: "Access Denied", "Just a moment", "captcha", "Cloudflare", `<title>Attention Required`).
   - Expected fields/shape present (skill-specific — e.g., `search` results have `title`+`link`; `pipelines amazon_product` has `title`+`price`).
7. **Red flags** — anti-patterns specific to this skill.
8. **References** — pointers to `references/*.md` with one-line summaries.

### Cross-skill handoffs

Each skill names the other two and states when to hand off:

- **scrape → data-feeds** when the URL is on a supported platform (Amazon, LinkedIn, TikTok, Instagram, YouTube, Reddit, Facebook, …). Agent gets cleaner data.
- **scrape → search** when the user has a topic but no URLs.
- **search → scrape** once target URLs are chosen from SERP results.
- **search → data-feeds** when the user wanted structured data from a known platform, not raw SERP.
- **data-feeds → scrape** when the URL is not on a supported platform (check with `bdata pipelines list`).
- **data-feeds → search** when target URLs must be discovered first.

These appear as explicit lines in each skill's Pick-your-path section, not just implicit cross-references.

### Shared setup reference

All three skills share setup guidance. To avoid drift:

- The full install → login → troubleshooting flow lives in `skills/bright-data-best-practices/references/cli-setup.md` (new file).
- Each of the three skills keeps a short inline setup check (≤ 10 lines) and a link to the shared doc for full details.
- `bright-data-best-practices/SKILL.md` gets a one-line pointer to the new reference; no other changes to that skill.

### Shared verification block

The verification checklist shape is identical across the three skills. To keep agents calibrated:

- Each skill's `references/patterns.md` opens with the same "Verification checklist" section (non-empty, not-a-block-page with the shared signature list, expected-shape).
- The skill-specific expected-shape clauses are listed in the skill's own SKILL.md verification gate.

---

## Skill 1 — `scrape`

**Job-to-be-done:** get clean content from one or more URLs that aren't on a supported platform.

**Trigger (`description` frontmatter):** "Scrape web content as clean markdown/HTML/JSON via the Bright Data CLI (`bdata scrape`). Use when the user wants to fetch a page, extract content from a list of URLs, or crawl paginated listings. Hands off to `data-feeds` for supported platforms (Amazon, LinkedIn, TikTok, Instagram, YouTube, Reddit, etc.) and to `search` when URLs must be discovered first. Requires the Bright Data CLI; proactively guides install + login if missing."

### Pick-your-path block

- **Single URL** → `bdata scrape <url> -f markdown`
- **Small list of URLs (≤ ~20)** → shell loop: `while read url; do bdata scrape "$url" -f markdown -o "out/$(echo $url | md5sum | cut -c1-8).md"; done < urls.txt` (CLI has no native batch; looping is correct)
- **Large list (dozens+)** → use `xargs -P` for parallelism, with a capped `-P` (e.g. `-P 4`) to avoid over-provisioning; see `references/patterns.md`
- **Paginated listing** → scrape page 1, extract next-page URL with a simple regex/selector, append to the URL list, repeat; recipe in `examples.md`
- **JS-heavy / login-gated site** → escalate to the `bdata browser` command (covered in the `brightdata-cli` skill — hand off)
- **Supported platform (Amazon, LinkedIn, etc.)** → **stop, hand off to `data-feeds`**

### Action (core commands)

- Clean markdown: `bdata scrape <url> -f markdown`
- Raw HTML: `bdata scrape <url> -f html`
- Structured JSON (when the unlocker returns structured data): `bdata scrape <url> -f json --pretty`
- Geo-targeted: `bdata scrape <url> --country us`
- Mobile user agent: `bdata scrape <url> --mobile`
- Very slow page: `bdata scrape <url> --async` → poll returned job id (recipe in `references/patterns.md`)

### Verification gate

1. Output file is non-empty (`test -s "$out"` or check stdout length > minimum threshold).
2. Content doesn't match block-page signatures.
3. Expected domain-specific markers present (e.g., if user wanted a product page, look for a price pattern).
4. On failure: retry with `--country`, then `--mobile`, then escalate to `bdata browser` (hand off).

### Red flags

- Claiming success without reading the output.
- Silencing errors with `2>/dev/null`.
- Scraping an Amazon/LinkedIn/TikTok URL instead of using `data-feeds` — loses structured data.
- Running `--async` for normal pages (adds latency for no benefit).
- Scraping the same URL repeatedly without caching the first result.
- Using `curl` against `api.brightdata.com` directly — that's the legacy path; only use it when the CLI isn't available.

### References

- `references/flags.md` — full flag reference.
- `references/patterns.md` — shell-loop batching, pagination recipe, `--async` polling, retry/backoff, country rotation, block-page recovery, legacy `curl` fallback, shared verification checklist.
- `references/examples.md` — (1) single page to markdown, (2) crawl a list of URLs with parallelism cap, (3) paginated listing, (4) block-page recovery chain.

### Removed

- `scripts/scrape.sh`
- The required env var `BRIGHTDATA_UNLOCKER_ZONE` becomes optional (the CLI handles zone provisioning via `bdata login`). Still documented as the legacy fallback.

---

## Skill 2 — `search`

**Job-to-be-done:** find things on the web — SERP results, intent-ranked discovery, or URL collection feeding into other skills.

**Trigger (`description` frontmatter):** "Search the web via the Bright Data CLI (`bdata search` for SERP, `bdata discover` for intent-ranked results). Use when the user wants Google/Bing/Yandex results, needs URLs to feed into scraping, or wants semantic web discovery. Hands off to `scrape` once target URLs are chosen, and to `data-feeds` when the user wants structured data from a known platform. Requires the Bright Data CLI; proactively guides install + login if missing."

### Pick-your-path block

- **Single keyword query, just SERP** → `bdata search "<query>" --engine google --json --pretty`
- **Pagination** → loop `--page 1`, `--page 2`, … (CLI has no `--limit`; `--page` is the lever)
- **Multi-query batch** → shell loop over a queries file
- **Intent-ranked / semantic discovery** (e.g., "articles about X written in 2025") → `bdata discover "<query>" --intent "<intent>" --num-results 20`
- **Want the pages, not just links, in one pass** → `bdata discover ... --include-content`
- **Want structured data from a known platform** → **stop, hand off to `data-feeds`**
- **Have URLs, want content** → **hand off to `scrape`**

### Action (core commands)

- Google SERP: `bdata search "<q>" --engine google --json --pretty`
- Localized Bing: `bdata search "<q>" --engine bing --country de --language de`
- Mobile SERP: `bdata search "<q>" --device mobile`
- Intent discovery: `bdata discover "<q>" --intent "product reviews" --num-results 10`
- Intent discovery with content: `bdata discover "<q>" --include-content -o results.json`
- Date-filtered discovery: `bdata discover "<q>" --start-date 2025-01-01 --end-date 2025-12-31`

### Verification gate

1. JSON parses cleanly.
2. Result array is non-empty (or if empty, the query is legitimately zero-result — agent checks by relaxing query).
3. Each result has the expected fields: `search` → `title` + `link`; `discover` → `title` + `url` (+ `content` when `--include-content`).
4. No block-page signatures in content for `discover --include-content`.
5. If SERP looks mis-localized (wrong language, wrong TLD of results) → re-run with `--country` / `--language`.

### Red flags

- Using `search` to fetch content from Amazon/LinkedIn/TikTok/etc. when `data-feeds` returns structured data in one call.
- Scraping every SERP result blindly — filter links first (domain allowlist, keyword relevance).
- Confusing `search` (keyword SERP) with `discover` (intent-ranked semantic) — they answer different questions. `search` for "what ranks for keyword X"; `discover` for "find me pages about topic Y that match intent Z".
- Looping queries without dedup — same URL often appears across queries; dedup before scraping.
- Assuming SERP order is relevance — it's geo/device/account-personalized; always set `--country` and `--device` explicitly for reproducibility.

### References

- `references/flags.md` — full flags for both `search` and `discover`.
- `references/patterns.md` — multi-query dedup, SERP → filter → scrape pipeline, `search` vs `discover` decision matrix, legacy `curl` fallback, shared verification checklist.
- `references/examples.md` — (1) single Google query, (2) localized Bing, (3) batch queries + dedup into URL list, (4) `discover --include-content` end-to-end.

### Removed

- `scripts/search.sh`
- `BRIGHTDATA_UNLOCKER_ZONE` as required (now optional legacy fallback).

---

## Skill 3 — `data-feeds`

**Job-to-be-done:** extract structured data from one of the 42 supported platforms via the CLI's `pipelines` command. Supported types (as of 2026-04-19): `amazon_product`, `amazon_product_reviews`, `amazon_product_search`, `apple_app_store`, `bestbuy_products`, `booking_hotel_listings`, `crunchbase_company`, `ebay_product`, `etsy_products`, `facebook_company_reviews`, `facebook_events`, `facebook_marketplace_listings`, `facebook_posts`, `github_repository_file`, `google_maps_reviews`, `google_play_store`, `google_shopping`, `homedepot_products`, `instagram_comments`, `instagram_posts`, `instagram_profiles`, `instagram_reels`, `linkedin_company_profile`, `linkedin_job_listings`, `linkedin_people_search`, `linkedin_person_profile`, `linkedin_posts`, `reddit_posts`, `reuter_news`, `tiktok_comments`, `tiktok_posts`, `tiktok_profiles`, `tiktok_shop`, `walmart_product`, `walmart_seller`, `x_posts`, `yahoo_finance_business`, `youtube_comments`, `youtube_profiles`, `youtube_videos`, `zara_products`, `zillow_properties_listing`, `zoominfo_company_profile`. Always verify with `bdata pipelines list` before hardcoding names.

**Trigger (`description` frontmatter):** "Extract structured data from 40+ supported platforms (Amazon products/reviews, LinkedIn profiles/companies/jobs, Instagram/TikTok/YouTube/Reddit posts, and more) via the Bright Data CLI (`bdata pipelines`). Use when the user wants clean JSON from a known platform URL rather than raw HTML. Hands off to `scrape` for unsupported URLs and to `search` when target URLs must be discovered first. Requires the Bright Data CLI; proactively guides install + login if missing."

### Pick-your-path block

- **Know the platform + have URL(s)** → `bdata pipelines <type> <url>` (e.g. `bdata pipelines amazon_product https://www.amazon.com/dp/B0...`)
- **Don't know which pipeline type fits** → `bdata pipelines list` first; match by platform + intent (product vs review vs profile)
- **Multiple URLs on the same pipeline type** → shell loop over URL list (no native batch flag); recipe in `examples.md`
- **Long-running job risk** → raise `--timeout` (default 600s) or set `BRIGHTDATA_POLLING_TIMEOUT`; CLI polls internally
- **URL is for an unsupported platform** → **stop, hand off to `scrape`**
- **Need to find target URLs first** → **hand off to `search`**

### Action (core commands)

- List pipeline types: `bdata pipelines list`
- Amazon product: `bdata pipelines amazon_product <url> --format json --pretty`
- Amazon product reviews: `bdata pipelines amazon_product_reviews <url>`
- Amazon product search (keyword-based, not URL-based): `bdata pipelines amazon_product_search <keyword> <domain> <pages_to_search>`
- LinkedIn profile: `bdata pipelines linkedin_person_profile <url>`
- LinkedIn company: `bdata pipelines linkedin_company_profile <url>`
- Instagram posts: `bdata pipelines instagram_posts <url>`
- TikTok profile: `bdata pipelines tiktok_profiles <url>`
- YouTube video: `bdata pipelines youtube_videos <url>`
- Reddit posts: `bdata pipelines reddit_posts <url>`
- With extended timeout: `bdata pipelines <type> <url> --timeout 1200`
- Write to file: `bdata pipelines <type> <url> -o data.json`

### Verification gate

1. Output JSON parses cleanly.
2. Record count matches expected (1 URL = 1 record for most; reviews/posts pipelines return arrays).
3. Per-record error fields are absent (partial failures are silent in pipeline output — explicitly check for a top-level `error`, `warning`, or per-record `error` key).
4. Core fields for the pipeline type are present (examples: `amazon_product` → `title` + `price`; `linkedin_person_profile` → `name` + `headline`; `instagram_posts` → `caption` + `media_url`).
5. On failure: re-run once with `--timeout` doubled; if still failing, check `bdata pipelines list` to confirm the pipeline type name hasn't changed.

### Red flags

- Using `bdata scrape` on Amazon/LinkedIn/etc. when `bdata pipelines <type>` would return clean structured fields in one call.
- Looping `bdata pipelines` in bash for large jobs without rate-limiting (each call can trigger a long-running pipeline; parallelism must be capped).
- Claiming success without a record-count + field-presence check. Partial failures are silent.
- Hardcoding pipeline type names without verifying via `bdata pipelines list` — names evolve. Note especially: `amazon_product` (singular), `amazon_product_reviews`, `linkedin_person_profile` (not `linkedin_profile`), `tiktok_profiles` (plural) — the naming is inconsistent across platforms.
- Short-circuiting polling with a tight `--timeout` on pipelines that legitimately take 5–15 minutes (reviews, company employees).
- Mixing sync `bdata pipelines` with `bdata scrape --async` mentally — they're different mechanisms; pipelines poll internally, scrape's `--async` returns a job ID you own.

### References

- `references/flags.md` — full `pipelines` flags + complete table of all 42 pipeline types (verified 2026-04-19 via `bdata pipelines list`) and their required input shape (URL vs keyword+domain+pages vs multi-arg). Includes note on keyword-shaped pipelines: `amazon_product_search <keyword> <domain> <pages>`, `linkedin_people_search` (multi-arg), `facebook_company_reviews`, `google_maps_reviews`, `youtube_comments`.
- `references/patterns.md` — sync timeout tuning, shell-loop batching with parallelism cap, partial-failure detection, platform-param cheatsheet, legacy `curl` fallback, shared verification checklist.
- `references/examples.md` — (1) single Amazon product, (2) batch LinkedIn companies via shell loop, (3) long-running reviews job with raised timeout, (4) mixed-platform workflow calling `pipelines list` first, (5) keyword-shaped pipeline (`amazon_product_search`).

### Removed

- `scripts/datasets.sh`
- `scripts/fetch.sh`
- `BRIGHTDATA_POLLING_TIMEOUT` env var as required — now optional (CLI still honors it as a default, and the legacy `curl` path documents it).
- `BRIGHTDATA_API_KEY` as required — now only required on the legacy `curl` fallback path.

---

## Shared work (cross-cutting)

### New file: `skills/bright-data-best-practices/references/cli-setup.md`

Contents:
- Install paths (curl installer, npm global, npx one-off).
- `bdata login` options (browser OAuth, `--device` for SSH, `--api-key` for CI).
- `bdata config` diagnostic output interpretation.
- Troubleshooting: "command not found", "not authenticated", "no zones", proxy/firewall issues.
- Env-var fallback table (what each legacy var mapped to, and which CLI command replaces that code path).

### `skills/bright-data-best-practices/SKILL.md`

Add one short section (≤ 5 lines) pointing to `references/cli-setup.md`. No other changes.

### Verification block shared language

Each skill's `references/patterns.md` opens with an identical "Verification checklist" block. This is copy-paste intentional (drift across three files < risk of one drifting from a canonical copy the agent won't find).

Block-page signature list (used in all three skills' verification gates):

```
"Access Denied"
"Just a moment"
"Attention Required"
"Checking your browser"
"captcha" (case-insensitive)
"cf-browser-verification"
"cloudflare" (+ small content size)
```

---

## Error handling and fallbacks

### Tier 1 — CLI present and authenticated

Happy path. Skill runs the `bdata` command, verifies output, done.

### Tier 2 — CLI present, not authenticated

Skill halts, shows `bdata config` output, instructs user to run `bdata login` (or `bdata login --device` in SSH context). Does not attempt to fall back silently.

### Tier 3 — CLI not installed

Skill offers two paths:
- Install globally: `npm install -g @brightdata/cli`
- One-off: `npx --yes --package @brightdata/cli bdata <command>`

Skill asks user which they prefer before acting; doesn't install silently.

### Tier 4 — Legacy fallback

If the user explicitly declines the CLI or is in an environment that can't run Node, skill falls back to the `curl` path documented in `references/patterns.md`. This path requires the old env vars (`BRIGHTDATA_API_KEY` and the skill-specific zone var). It is documented but deprecated; a note at the top of that section says "prefer the CLI path above."

---

## Testing / verification of the revision itself

For each of the three revised skills:

1. Fresh-environment test: no `bdata` installed, no env vars set. Skill should detect Tier 3 and route the user correctly.
2. CLI-but-no-auth test: `bdata` installed, `bdata config` shows unauthenticated. Skill should detect Tier 2 and route to login.
3. Happy path: authenticated CLI, representative URL / query / dataset. Skill produces expected output and passes its own verification gate.
4. Handoff test: feed each skill an input that should route to a sibling (e.g., give `scrape` an Amazon URL, give `search` a request for Amazon product data). Skill should refuse and hand off explicitly.
5. Block-page test: give `scrape` a URL known to Cloudflare-gate. Skill should detect via signature list and escalate (country rotation → mobile → `bdata browser`).

These are manual acceptance tests documented in the plan, not automated. The existing `skills/competitive-intel/evals/` and `skills/scraper-builder/evals/` folders suggest a future eval harness could cover this; out of scope for this revision.

---

## Files touched

**Modified:**
- `skills/scrape/SKILL.md` — full rewrite to workflow shape.
- `skills/search/SKILL.md` — full rewrite to workflow shape.
- `skills/data-feeds/SKILL.md` — full rewrite to workflow shape.
- `skills/bright-data-best-practices/SKILL.md` — one-line pointer to new cli-setup reference.

**New:**
- `skills/scrape/references/flags.md`
- `skills/scrape/references/patterns.md`
- `skills/scrape/references/examples.md`
- `skills/search/references/flags.md`
- `skills/search/references/patterns.md`
- `skills/search/references/examples.md`
- `skills/data-feeds/references/flags.md`
- `skills/data-feeds/references/patterns.md`
- `skills/data-feeds/references/examples.md`
- `skills/bright-data-best-practices/references/cli-setup.md`

**Deleted:**
- `skills/scrape/scripts/scrape.sh`
- `skills/search/scripts/search.sh`
- `skills/data-feeds/scripts/datasets.sh`
- `skills/data-feeds/scripts/fetch.sh`
- The empty `scripts/` directories in all three skills.

---

## Open questions / risks

- **Risk:** pipeline type names may have changed since the existing `scripts/datasets.sh` was written (the naming is also inconsistent: `amazon_product` singular vs `tiktok_profiles` plural). Mitigation: `bdata pipelines list` is the source of truth; skill tells the agent to check before hardcoding.
- **Risk:** `bdata` CLI is v0.1.8 and may evolve. Mitigation: skills pin behavior to commands + flags verified against source on 2026-04-19; reference docs should be re-verified when CLI crosses a minor version.
- **Risk:** the env-var fallback path adds maintenance weight forever. Mitigation: keep it minimal — one `curl` example per skill, in `patterns.md` under a clearly-labeled "Legacy" section.
- **Open:** should the revised skills include any automated eval (like `competitive-intel/evals/`)? Deferred to follow-up; out of scope.
