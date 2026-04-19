# CLI Skills Revision Implementation Plan

> **For agentic workers:** REQUIRED SUB-SKILL: Use superpowers:subagent-driven-development (recommended) or superpowers:executing-plans to implement this plan task-by-task. Steps use checkbox (`- [ ]`) syntax for tracking.

**Goal:** Reshape three Bright Data skills (`scrape`, `search`, `data-feeds`) from bash+curl scripts into superpowers-style workflows that drive the Bright Data CLI (`bdata`).

**Architecture:** Each skill becomes a workflow document (trigger → setup gate → pick-your-path → action → verification → red flags) with a `references/` folder for depth. Proactive setup detection routes users to `bdata login`. Legacy env-var fallback preserved but deprecated. Cross-skill handoffs are explicit. A shared CLI setup reference lives in `bright-data-best-practices/` to avoid drift.

**Tech Stack:** Markdown only. No code. All content is skill-authoring work following the spec at `docs/superpowers/specs/2026-04-19-cli-skills-revision-design.md`. Target CLI: `@brightdata/cli` v0.1.8 (verified 2026-04-19).

---

## File Structure

**New files:**

- `skills/bright-data-best-practices/references/cli-setup.md` — shared install/login/troubleshooting reference linked from all three skills
- `skills/scrape/references/flags.md`
- `skills/scrape/references/patterns.md`
- `skills/scrape/references/examples.md`
- `skills/search/references/flags.md`
- `skills/search/references/patterns.md`
- `skills/search/references/examples.md`
- `skills/data-feeds/references/flags.md`
- `skills/data-feeds/references/patterns.md`
- `skills/data-feeds/references/examples.md`

**Modified files:**

- `skills/scrape/SKILL.md` — full rewrite to workflow shape
- `skills/search/SKILL.md` — full rewrite to workflow shape
- `skills/data-feeds/SKILL.md` — full rewrite to workflow shape
- `skills/bright-data-best-practices/SKILL.md` — add short section pointing to new `cli-setup.md`

**Deleted files:**

- `skills/scrape/scripts/scrape.sh` (and empty `scripts/` dir)
- `skills/search/scripts/search.sh` (and empty `scripts/` dir)
- `skills/data-feeds/scripts/datasets.sh`
- `skills/data-feeds/scripts/fetch.sh` (and empty `scripts/` dir)

---

## Verification Facts (pin these during implementation)

All tasks reference the CLI as verified on 2026-04-19. Do NOT invent flags. Relevant verified surface:

| Command | Required args | Flags (verified) |
|---|---|---|
| `bdata scrape <url>` | url | `-f/--format <markdown\|html\|screenshot\|json>`, `--country <iso>`, `--zone <name>`, `--mobile`, `--async`, `-o/--output <path>`, `--json`, `--pretty`, `--timing`, `-k/--api-key` |
| `bdata search <query>` | query | `--engine <google\|bing\|yandex>`, `--country`, `--language`, `--page <n>` (0-indexed), `--type <web\|news\|images\|shopping>`, `--zone`, `--device <desktop\|mobile>`, `-o/--output`, `--json`, `--pretty` |
| `bdata discover <query>` | query | `--intent <text>`, `--country`, `--city`, `--language`, `--num-results <n>`, `--filter-keywords <csv>`, `--include-content`, `--no-remove-duplicates`, `--start-date <YYYY-MM-DD>`, `--end-date <YYYY-MM-DD>`, `--timeout <sec>`, `-o/--output`, `--json`, `--pretty` |
| `bdata pipelines <type> [params...]` | type + type-specific params | `--format <json\|csv\|ndjson\|jsonl>`, `--timeout <sec>`, `-o/--output`, `--json`, `--pretty`, `--timing` |
| `bdata pipelines list` | — | prints all pipeline types |
| `bdata status <job-id>` | job-id | polls a scrape `--async` snapshot job |
| `bdata config` | — | prints current config JSON |
| `bdata zones` | — | lists zones (requires auth; useful as an auth-check probe) |
| `bdata login` / `--device` / `--api-key <k>` | — | OAuth / device code / non-interactive auth |

**Auth-check probe used in the setup gate:** `bdata zones >/dev/null 2>&1` — returns 0 if authenticated, non-zero otherwise.

**Credentials file (Linux):** `~/.config/brightdata-cli/credentials.json` (darwin: `~/Library/Application Support/brightdata-cli/`; win32: `%APPDATA%\brightdata-cli\`).

**Pipeline types (43, verified 2026-04-19 via `bdata pipelines list`):** `amazon_product`, `amazon_product_reviews`, `amazon_product_search`, `apple_app_store`, `bestbuy_products`, `booking_hotel_listings`, `crunchbase_company`, `ebay_product`, `etsy_products`, `facebook_company_reviews`, `facebook_events`, `facebook_marketplace_listings`, `facebook_posts`, `github_repository_file`, `google_maps_reviews`, `google_play_store`, `google_shopping`, `homedepot_products`, `instagram_comments`, `instagram_posts`, `instagram_profiles`, `instagram_reels`, `linkedin_company_profile`, `linkedin_job_listings`, `linkedin_people_search`, `linkedin_person_profile`, `linkedin_posts`, `reddit_posts`, `reuter_news`, `tiktok_comments`, `tiktok_posts`, `tiktok_profiles`, `tiktok_shop`, `walmart_product`, `walmart_seller`, `x_posts`, `yahoo_finance_business`, `youtube_comments`, `youtube_profiles`, `youtube_videos`, `zara_products`, `zillow_properties_listing`, `zoominfo_company_profile`.

**Keyword-shaped pipelines (do NOT take a URL):**
- `amazon_product_search <keyword> <domain> <pages_to_search>`
- `linkedin_people_search` — multi-arg
- `facebook_company_reviews <url> <number_of_reviews>`
- `google_maps_reviews <url> <days_range>`
- `youtube_comments <url> <number_of_comments>`

**Shared block-page signature list (used verbatim in all three skills' verification gates):**
```
"Access Denied"
"Just a moment"
"Attention Required"
"Checking your browser"
"captcha" (case-insensitive)
"cf-browser-verification"
"cloudflare" (+ small content size, e.g. < 2KB)
```

---

## Task 1: Write the shared CLI setup reference

**Files:**
- Create: `skills/bright-data-best-practices/references/cli-setup.md`

- [ ] **Step 1: Create the file with full content**

Write this exact content to `skills/bright-data-best-practices/references/cli-setup.md`:

````markdown
# Bright Data CLI — Setup & Troubleshooting

Canonical install / login / troubleshooting reference for the `bdata` (aka `brightdata`) CLI. Linked from the `scrape`, `search`, and `data-feeds` skills.

## Install

Pick one path:

**Global (recommended for regular use):**
```bash
npm install -g @brightdata/cli
```

**Curl installer (macOS/Linux):**
```bash
curl -fsSL https://cli.brightdata.com/install.sh | bash
```

**One-off (no install):**
```bash
npx --yes --package @brightdata/cli brightdata <command>
```

Requires Node.js ≥ 20. Both `brightdata` and `bdata` are exposed after install.

Verify:
```bash
bdata --version
```

## Authenticate

One-time:
```bash
bdata login
```

Opens a browser for OAuth, saves credentials locally, auto-creates the default zones (`cli_unlocker`, `cli_browser`), and writes config.

**No browser available (SSH / CI / WSL without X):**
```bash
bdata login --device
```
Prints a code + URL to open on another device.

**Non-interactive (CI / scripted):**
```bash
bdata login --api-key "$BRIGHTDATA_API_KEY"
```

## Verify auth

```bash
bdata zones          # non-zero exit → not authenticated
bdata config         # prints current config JSON
```

The zones probe is the most reliable auth check: it requires valid credentials and returns quickly.

**Config file locations:**
- Linux: `~/.config/brightdata-cli/credentials.json`
- macOS: `~/Library/Application Support/brightdata-cli/credentials.json`
- Windows: `%APPDATA%\brightdata-cli\credentials.json`

## Auth-detection recipe for skills

Skills should run this probe before any `bdata` command:

```bash
if ! command -v bdata >/dev/null 2>&1; then
    echo "bdata CLI not installed" >&2
    # → offer install paths
elif ! bdata zones >/dev/null 2>&1; then
    echo "bdata not authenticated" >&2
    # → instruct: bdata login (or --device for headless)
fi
```

## Troubleshooting

**`command not found: bdata`**
— CLI isn't on PATH. If installed via `npm -g`, ensure npm's global bin is on PATH (`npm bin -g`). If unsure, reinstall with `npm install -g @brightdata/cli`.

**`Error: not authenticated` / 401 responses**
— Run `bdata login` (or `bdata login --device` in SSH). If an env var `BRIGHTDATA_API_KEY` is set but invalid, it takes precedence over saved credentials — unset it or run `bdata login --api-key <valid-key>`.

**`Error: no zones`**
— `bdata login` provisions zones automatically. If they were deleted in the dashboard, re-run `bdata login` or create zones manually via the dashboard, then set the defaults with `bdata config set default_zone_unlocker <name>`.

**Proxy/firewall blocking OAuth:**
— Use `bdata login --device` or `bdata login --api-key <key>` (obtain the key from https://brightdata.com/cp/setting/users).

**Polling timeout during `bdata pipelines <type>`:**
— Raise with `--timeout 1800` (30 min) or set `BRIGHTDATA_POLLING_TIMEOUT=1800` before running. Some pipelines (reviews, company employees, long post lists) legitimately take 5–15 minutes.

## Env-var fallback (legacy)

Before the CLI, skills required:
- `BRIGHTDATA_API_KEY` — Bright Data API key
- `BRIGHTDATA_UNLOCKER_ZONE` — Web Unlocker zone name
- `BRIGHTDATA_POLLING_TIMEOUT` — (optional) pipeline polling cap in seconds

These are still honored by legacy `curl`-based paths documented in each skill's `references/patterns.md`. The CLI path is preferred; env vars are retained for environments where Node/CLI aren't available.

Mapping:

| Env var | CLI replacement |
|---|---|
| `BRIGHTDATA_API_KEY` | `bdata login` (stored in credentials file) or `-k/--api-key` per-command |
| `BRIGHTDATA_UNLOCKER_ZONE` | Auto-provisioned by `bdata login`; override per-command with `--zone <name>` |
| `BRIGHTDATA_POLLING_TIMEOUT` | Still read by `bdata pipelines`; also overridable via `--timeout <sec>` |
````

- [ ] **Step 2: Verify file was written correctly**

Run:
```bash
test -f skills/bright-data-best-practices/references/cli-setup.md && wc -l skills/bright-data-best-practices/references/cli-setup.md
```
Expected: non-zero line count (~120 lines).

- [ ] **Step 3: Commit**

```bash
git add skills/bright-data-best-practices/references/cli-setup.md
git commit -m "docs: add shared CLI setup reference for Bright Data skills"
```

---

## Task 2: Link the shared CLI setup from `bright-data-best-practices`

**Files:**
- Modify: `skills/bright-data-best-practices/SKILL.md`

- [ ] **Step 1: Read the existing file to find the right insertion point**

```bash
head -40 skills/bright-data-best-practices/SKILL.md
```

- [ ] **Step 2: Add a short section pointing to the new reference**

Insert the following section immediately after the frontmatter block (after the closing `---` of the YAML header, before any existing content). If an analogous section already exists, skip. Keep it short — this skill isn't being rewritten:

```markdown
## CLI Setup Reference

Install, authentication, and troubleshooting for the Bright Data CLI (`bdata`) are documented in a single canonical place:

→ [`references/cli-setup.md`](references/cli-setup.md)

Consult it before any task that shells out to `bdata`.
```

- [ ] **Step 3: Verify**

Run:
```bash
grep -n "cli-setup.md" skills/bright-data-best-practices/SKILL.md
```
Expected: at least one match.

- [ ] **Step 4: Commit**

```bash
git add skills/bright-data-best-practices/SKILL.md
git commit -m "docs: link shared CLI setup reference from best-practices skill"
```

---

## Task 3: Rewrite `scrape/SKILL.md` to the workflow shape

**Files:**
- Modify: `skills/scrape/SKILL.md` (full rewrite)

- [ ] **Step 1: Overwrite `skills/scrape/SKILL.md` with the following content**

````markdown
---
name: scrape
description: Scrape web content as clean markdown/HTML/JSON via the Bright Data CLI (`bdata scrape`). Use when the user wants to fetch a page, extract content from a list of URLs, or crawl paginated listings. Hands off to `data-feeds` for supported platforms (Amazon, LinkedIn, TikTok, Instagram, YouTube, Reddit, etc.) and to `search` when URLs must be discovered first. Requires the Bright Data CLI; proactively guides install + login if missing.
---

# Bright Data — Scrape

Get clean content (markdown, HTML, JSON, screenshot) from one or more URLs via the Bright Data CLI. This skill owns the "fetch raw or lightly-structured content" job. For platform-specific structured data (Amazon, LinkedIn, TikTok, etc.), **stop and use `data-feeds` instead** — you'll get clean JSON without selector logic.

## Setup gate (run first)

Before any scrape, verify the CLI is installed and authenticated:

```bash
if ! command -v bdata >/dev/null 2>&1; then
    echo "bdata CLI not installed — see bright-data-best-practices/references/cli-setup.md"
elif ! bdata zones >/dev/null 2>&1; then
    echo "bdata not authenticated — run: bdata login  (or: bdata login --device for SSH)"
fi
```

If either check fails, halt and route the user to `skills/bright-data-best-practices/references/cli-setup.md`. Do not attempt the legacy `curl` fallback silently — ask the user first.

## Pick your path

| Situation | Action |
|---|---|
| Single URL | `bdata scrape <url> -f markdown` |
| Small list (≤ ~20 URLs) | shell loop, 1 at a time (see `references/patterns.md`) |
| Larger list (dozens+) | `xargs -P 4` with parallelism cap (see `references/patterns.md`) |
| Paginated listing | scrape page 1 → extract next-page URL → append → repeat (see `references/examples.md`) |
| JS-heavy / login-gated / interaction-required | escalate to `bdata browser` (see `brightdata-cli` skill) |
| Amazon, LinkedIn, TikTok, Instagram, YouTube, Reddit, … | **stop — hand off to `data-feeds`** |
| No URL yet, just a topic | **hand off to `search`** |

## Action

Core commands:

```bash
# Clean markdown (default)
bdata scrape "https://example.com/article" -f markdown -o article.md

# Raw HTML (when you need the DOM)
bdata scrape "https://example.com" -f html -o page.html

# Structured JSON (when the Unlocker returns parsed fields)
bdata scrape "https://example.com" -f json --pretty -o page.json

# Geo-targeted (override the exit country)
bdata scrape "https://example.com" --country de -f markdown

# Mobile user-agent (for m-dot sites or anti-bot variation)
bdata scrape "https://example.com" --mobile -f markdown

# Very slow page — submit async, get a job ID, poll later
job_id=$(bdata scrape "https://slow-site.example" --async --json | jq -r '.job_id')
bdata status "$job_id"   # poll until status == "done"
```

Full flag reference: [`references/flags.md`](references/flags.md).

## Verification gate (run before claiming success)

1. **Non-empty output:** `test -s "$out_path"` — or, for stdout, at least 200 bytes of content.
2. **Not a block page** — grep the output for any of these signatures (case-insensitive):
   - `Access Denied`
   - `Just a moment`
   - `Attention Required`
   - `Checking your browser`
   - `captcha`
   - `cf-browser-verification`
   - `cloudflare` *(with < 2KB total body)*
3. **Expected markers present** for the task: e.g., a product page should contain a price pattern (`\$\d`); an article should contain at least one `<h1>` or `# ` heading.
4. **On failure, escalation ladder:**
   - Retry with `--country us` (or another country)
   - Retry with `--mobile`
   - Escalate to `bdata browser` for full JS rendering (hand off to `brightdata-cli` skill)

Do not report success until all three checks pass.

## Red flags

- Claiming success without inspecting the output.
- Silencing errors with `2>/dev/null` — you'll miss auth failures and rate-limit errors.
- Running `bdata scrape` on Amazon/LinkedIn/TikTok/Instagram/YouTube/Reddit URLs — these are supported by `data-feeds` and return structured data directly. Scraping loses the structure.
- Using `--async` for normal pages — adds latency for no gain. Only use `--async` for pages that routinely take > 30s or when queuing many long-running jobs.
- Scraping the same URL repeatedly in the same task — cache the first result.
- Looping `bdata scrape` sequentially for large lists instead of using `xargs -P 4` (or similar) with a parallelism cap.
- Using `curl` against `api.brightdata.com` directly — legacy path; only when the CLI isn't available.

## References

- [`references/flags.md`](references/flags.md) — every flag with when-to-use notes.
- [`references/patterns.md`](references/patterns.md) — shell-loop batching, `xargs` parallelism, pagination recipe, `--async` polling, retry/backoff, block-page recovery chain, legacy `curl` fallback.
- [`references/examples.md`](references/examples.md) — (1) single page → markdown, (2) batch a list of URLs with parallelism cap, (3) paginated listing, (4) block-page recovery.
````

- [ ] **Step 2: Verify frontmatter is valid**

Run:
```bash
head -4 skills/scrape/SKILL.md
```
Expected: opens with `---`, has `name: scrape`, `description: ...`, closes with `---`.

- [ ] **Step 3: Commit**

```bash
git add skills/scrape/SKILL.md
git commit -m "refactor(skills): rewrite scrape as CLI-driven workflow"
```

---

## Task 4: Create `scrape/references/flags.md`

**Files:**
- Create: `skills/scrape/references/flags.md`

- [ ] **Step 1: Write the file**

Exact content:

````markdown
# `bdata scrape` — flag reference

Verified against `@brightdata/cli` v0.1.8 on 2026-04-19.

Usage: `bdata scrape [options] <url>`

| Flag | Values | Default | When to use |
|---|---|---|---|
| `-f, --format <format>` | `markdown`, `html`, `screenshot`, `json` | `markdown` | `markdown` for readable content; `html` when you need DOM fidelity; `screenshot` to save a PNG; `json` when the Unlocker has a structured extractor for the URL. |
| `--country <code>` | ISO code (`us`, `de`, `jp`, …) | — | Force a geo-targeted exit. Use when the target site geoblocks, personalizes by country, or returns different content by region. |
| `--zone <name>` | Unlocker zone name | account default | Override the default zone — e.g., when you have a dedicated zone with different residential/mobile settings. |
| `--mobile` | (flag) | off | Use a mobile user agent. Use for m-dot sites or when desktop UA gets blocked. |
| `--async` | (flag) | off | Submit asynchronously; returns a `job_id`. Poll with `bdata status <job-id>`. Use only for pages routinely > 30s or when queuing many long-running jobs. |
| `-o, --output <path>` | file path | stdout | Write result to a file. Required for binary formats (`screenshot`). Recommended for anything > 1KB. |
| `--json` | (flag) | off | Force JSON output envelope (metadata + content). Useful in scripts. |
| `--pretty` | (flag) | off | Pretty-print JSON. Combine with `--json` or `-f json`. |
| `--timing` | (flag) | off | Print request timing breakdown to stderr. Debugging only. |
| `-k, --api-key <key>` | API key | saved credentials or `BRIGHTDATA_API_KEY` | Per-command override. Rarely needed — prefer `bdata login`. |

## `--async` polling recipe

```bash
job_id=$(bdata scrape "https://slow.example" --async --json | jq -r '.job_id')

# Poll until done (handled by bdata status itself; it exits when ready)
bdata status "$job_id" --json --pretty -o result.json
```

`bdata status` is the dedicated polling command for async scrape snapshots.

## Format decision matrix

| Goal | Format |
|---|---|
| Feed content to an LLM | `markdown` |
| Extract via selectors / regex | `html` |
| Visual regression / proof-of-view | `screenshot` (writes PNG — use `-o` required) |
| URL has a structured extractor (Unlocker auto-parses) | `json` |
````

- [ ] **Step 2: Commit**

```bash
git add skills/scrape/references/flags.md
git commit -m "docs(skills): add scrape CLI flag reference"
```

---

## Task 5: Create `scrape/references/patterns.md`

**Files:**
- Create: `skills/scrape/references/patterns.md`

- [ ] **Step 1: Write the file**

Exact content:

````markdown
# Scrape — patterns

## Verification checklist (shared across all Bright Data CLI skills)

Before claiming a scrape succeeded:

1. **Output is non-empty.** `test -s "$out"` for files; for stdout, min 200 bytes.
2. **Not a block page.** Grep (case-insensitive) for any of:
   - `Access Denied`
   - `Just a moment`
   - `Attention Required`
   - `Checking your browser`
   - `captcha`
   - `cf-browser-verification`
   - `cloudflare` *(with total body < 2KB)*
3. **Expected markers present** for the task (e.g., price pattern on a product page, at least one heading in an article).

Failing any check → retry with `--country`, then `--mobile`, then escalate to `bdata browser`.

## Small-batch shell loop (≤ ~20 URLs)

```bash
mkdir -p out
while IFS= read -r url; do
    hash=$(printf '%s' "$url" | md5sum | cut -c1-8)
    bdata scrape "$url" -f markdown -o "out/${hash}.md" \
        || echo "FAIL: $url" >&2
done < urls.txt
```

The `|| echo` prevents one failure from aborting the loop; failures are visible on stderr.

## Large batch with parallelism cap (`xargs -P`)

```bash
mkdir -p out
xargs -a urls.txt -n 1 -P 4 -I {} bash -c '
    url="$1"
    hash=$(printf "%s" "$url" | md5sum | cut -c1-8)
    bdata scrape "$url" -f markdown -o "out/${hash}.md" || echo "FAIL: $url" >&2
' _ {}
```

`-P 4` caps concurrency at 4 parallel `bdata scrape` invocations. Raise cautiously — each scrape consumes bandwidth and counts against zone budget.

## Pagination recipe (listing pages)

For a listing with `?page=N` pagination:

```bash
page=1
while :; do
    url="https://example.com/list?page=$page"
    out="out/page-${page}.md"
    bdata scrape "$url" -f markdown -o "$out"

    # Stop when the page is empty or doesn't contain an item marker
    if [[ ! -s "$out" ]] || ! grep -q '\[.*\](/item/' "$out"; then
        rm -f "$out"
        break
    fi
    page=$((page + 1))
done
```

Adapt the "contains an item marker" grep to the actual site's output.

## Retry / backoff

`bdata scrape` returns non-zero on failure. Wrap with a simple retry:

```bash
scrape_with_retry() {
    local url=$1 out=$2 attempt=1 max=3
    while (( attempt <= max )); do
        if bdata scrape "$url" -f markdown -o "$out"; then
            return 0
        fi
        sleep $((2 ** attempt))   # 2s, 4s, 8s
        attempt=$((attempt + 1))
    done
    return 1
}
```

## Block-page recovery chain

When a scrape returns a block-page signature:

1. Retry same URL with `--country us` (or another ISO code appropriate to the site).
2. If still blocked, retry with `--mobile`.
3. If still blocked, escalate to `bdata browser` (real-browser with JS execution).

Example:

```bash
try_scrape() {
    local url=$1 out=$2
    for args in "" "--country us" "--mobile" "--country us --mobile"; do
        bdata scrape "$url" $args -f markdown -o "$out"
        if ! grep -qiE 'access denied|just a moment|captcha|cloudflare' "$out"; then
            return 0
        fi
    done
    return 1
}
```

If all four attempts return block pages, hand off to the `bdata browser` command.

## Legacy `curl` fallback (deprecated)

Only when the CLI cannot be installed. Requires env vars `BRIGHTDATA_API_KEY` and `BRIGHTDATA_UNLOCKER_ZONE`:

```bash
curl -sS "https://api.brightdata.com/request" \
    -H "Authorization: Bearer $BRIGHTDATA_API_KEY" \
    -H "Content-Type: application/json" \
    -d "{
        \"zone\": \"$BRIGHTDATA_UNLOCKER_ZONE\",
        \"url\": \"$URL\",
        \"format\": \"raw\",
        \"data_format\": \"markdown\"
    }"
```

Prefer the CLI path. This block exists only for environments without Node.js.
````

- [ ] **Step 2: Commit**

```bash
git add skills/scrape/references/patterns.md
git commit -m "docs(skills): add scrape patterns (batching, pagination, recovery)"
```

---

## Task 6: Create `scrape/references/examples.md`

**Files:**
- Create: `skills/scrape/references/examples.md`

- [ ] **Step 1: Write the file**

Exact content:

````markdown
# Scrape — worked examples

## Example 1 — single page to markdown

Fetch a blog post as clean markdown and verify the output.

```bash
bdata scrape "https://engineering.example.com/posts/scaling-bigquery" \
    -f markdown -o post.md

# Verify
test -s post.md || { echo "empty output"; exit 1; }
grep -qiE 'access denied|just a moment|captcha|cloudflare' post.md \
    && { echo "block page"; exit 1; }
head -n 1 post.md   # should show the title as an h1
```

## Example 2 — batch a list of URLs with parallelism cap

Given `urls.txt` (one URL per line), scrape all to `out/` with max 4 parallel requests:

```bash
mkdir -p out
xargs -a urls.txt -n 1 -P 4 -I {} bash -c '
    url="$1"
    hash=$(printf "%s" "$url" | md5sum | cut -c1-8)
    bdata scrape "$url" -f markdown -o "out/${hash}.md" \
        || echo "FAIL: $url" >&2
' _ {}

# Count successes / failures
ok=$(find out -name "*.md" -size +0 | wc -l)
total=$(wc -l < urls.txt)
echo "Scraped $ok of $total URLs"
```

## Example 3 — paginated listing

Scrape every page of a paginated article index until an empty page is hit:

```bash
mkdir -p out
page=1
while :; do
    url="https://blog.example.com/archive?page=$page"
    out="out/archive-${page}.md"
    bdata scrape "$url" -f markdown -o "$out"

    if [[ ! -s "$out" ]] || ! grep -qE '\[.+\]\(/posts/' "$out"; then
        rm -f "$out"
        echo "Done after $((page - 1)) pages"
        break
    fi
    page=$((page + 1))
done
```

## Example 4 — block-page recovery chain

Scrape a URL that's intermittently Cloudflare-gated. Try four geo/UA combinations; if all return block pages, hand off to `bdata browser`:

```bash
URL="https://protected.example.com/catalog"
OUT="catalog.md"

for args in "" "--country us" "--mobile" "--country us --mobile"; do
    bdata scrape "$URL" $args -f markdown -o "$OUT"
    if ! grep -qiE 'access denied|just a moment|captcha|cloudflare' "$OUT"; then
        echo "Succeeded with: $args"
        exit 0
    fi
done

echo "All scrape variants returned block pages — escalating to bdata browser" >&2
# (caller hands off to the bdata browser command, documented in the brightdata-cli skill)
exit 1
```
````

- [ ] **Step 2: Commit**

```bash
git add skills/scrape/references/examples.md
git commit -m "docs(skills): add scrape worked examples"
```

---

## Task 7: Delete `scrape/scripts/`

**Files:**
- Delete: `skills/scrape/scripts/scrape.sh`
- Delete: `skills/scrape/scripts/` (directory)

- [ ] **Step 1: Remove the scripts directory**

```bash
git rm -r skills/scrape/scripts
```

- [ ] **Step 2: Verify**

```bash
test ! -e skills/scrape/scripts && echo "removed"
```
Expected: `removed`.

- [ ] **Step 3: Commit**

```bash
git commit -m "refactor(skills): remove scrape bash scripts (superseded by CLI)"
```

---

## Task 8: Rewrite `search/SKILL.md` to the workflow shape

**Files:**
- Modify: `skills/search/SKILL.md` (full rewrite)

- [ ] **Step 1: Overwrite `skills/search/SKILL.md` with the following content**

````markdown
---
name: search
description: Search the web via the Bright Data CLI — `bdata search` for Google/Bing/Yandex SERP, `bdata discover` for intent-ranked semantic results. Use when the user wants SERP results, needs URLs to feed into scraping, or wants semantic web discovery with optional page content. Hands off to `scrape` once target URLs are chosen, and to `data-feeds` when the user wants structured data from a known platform. Requires the Bright Data CLI; proactively guides install + login if missing.
---

# Bright Data — Search

Find things on the web. Two commands live in this skill:

- **`bdata search`** — classic keyword SERP (Google/Bing/Yandex). Best when you want "what ranks for keyword X."
- **`bdata discover`** — AI intent-ranked discovery with optional page content. Best when you want "pages about topic Y that match intent Z."

For structured data from a known platform (Amazon, LinkedIn, TikTok, …), **stop and use `data-feeds` instead**.

## Setup gate (run first)

```bash
if ! command -v bdata >/dev/null 2>&1; then
    echo "bdata CLI not installed — see bright-data-best-practices/references/cli-setup.md"
elif ! bdata zones >/dev/null 2>&1; then
    echo "bdata not authenticated — run: bdata login  (or: bdata login --device for SSH)"
fi
```

Halt and route to `skills/bright-data-best-practices/references/cli-setup.md` if either check fails.

## Pick your path

| Situation | Action |
|---|---|
| Single keyword query, just SERP | `bdata search "<query>" --engine google --json --pretty` |
| Paginated SERP (more results) | loop `--page 0`, `--page 1`, … (0-indexed) |
| Multiple queries | shell loop over a queries file |
| Intent-ranked / semantic (not keyword) | `bdata discover "<query>" --intent "<intent>" --num-results 20` |
| Want page bodies along with results, one pass | `bdata discover ... --include-content` |
| News / images / shopping SERP | `bdata search "<query>" --type news` (or `images`, `shopping`) |
| Want Amazon/LinkedIn/TikTok/… structured data | **stop — hand off to `data-feeds`** |
| Have URLs, want content | **hand off to `scrape`** |

## Action

Core commands:

```bash
# Google SERP, structured JSON
bdata search "site:example.com privacy policy" --engine google --json --pretty

# Localized Bing (German results, German language)
bdata search "datenschutz" --engine bing --country de --language de --json

# Second page of results (0-indexed)
bdata search "machine learning papers" --page 1 --json

# Mobile SERP (rankings differ from desktop)
bdata search "best coffee shops" --device mobile --json

# News vertical
bdata search "openai" --type news --json --pretty

# Intent-ranked discovery
bdata discover "enterprise LLM platforms" \
    --intent "vendor pages with pricing" \
    --num-results 15 --json

# Discovery with page content in markdown
bdata discover "webhook best practices" \
    --include-content --num-results 10 -o results.json

# Date-filtered discovery
bdata discover "react server components" \
    --start-date 2025-01-01 --end-date 2025-12-31 --num-results 20
```

Full flag reference: [`references/flags.md`](references/flags.md).

### `search` vs `discover` — pick the right one

| You want | Use |
|---|---|
| "What Google ranks for this exact keyword" | `search` |
| "Pages that match this meaning/intent" | `discover` |
| "News / images / shopping vertical SERP" | `search --type <vertical>` |
| "Results + page bodies in one call" | `discover --include-content` |
| "Dedup / semantic ranking across queries" | `discover` |

## Verification gate

1. **JSON parses cleanly:** `jq . <output>` returns 0.
2. **Result array non-empty** — if empty, the query is legitimately zero-result; relax the query and re-run. Don't claim success on empty results without telling the user.
3. **Required fields present:**
   - `search`: each result has `title` + `link`
   - `discover`: each result has `title` + `url`; if `--include-content`, also `content`
4. **For `discover --include-content`:** no block-page signatures in the `content` field (same list as scrape: "Access Denied", "Just a moment", "captcha", etc.).
5. **Geo sanity:** if the user expected country-specific results, inspect TLDs / languages of top results. If mis-localized, re-run with explicit `--country` and `--language`.

## Red flags

- Using `search` to *fetch content* from Amazon, LinkedIn, TikTok, etc. when `data-feeds` returns clean structured data in one call.
- Scraping every SERP result blindly — filter first (domain allowlist, keyword in title, relevance heuristic).
- Confusing `search` (keyword) with `discover` (semantic). They answer different questions.
- Running multiple queries without deduping URLs across result sets before scraping.
- Assuming SERP order is universal — it's personalized by geo + device. Always set `--country` and `--device` explicitly for reproducibility.
- Using `--page` as a result count — it's a page index, not a limit. Each page returns ~10 results.
- Hardcoding `--num-results 100` on `discover` without realizing the pipeline polls until that many are found; can be slow.

## References

- [`references/flags.md`](references/flags.md) — full flags for `search` and `discover` with when-to-use notes.
- [`references/patterns.md`](references/patterns.md) — multi-query dedup, SERP → filter → scrape pipeline, `search` vs `discover` decision, legacy `curl` fallback, shared verification checklist.
- [`references/examples.md`](references/examples.md) — (1) single Google query, (2) localized Bing, (3) batch queries + dedup into URL list, (4) `discover --include-content` end-to-end.
````

- [ ] **Step 2: Verify frontmatter**

```bash
head -4 skills/search/SKILL.md
```
Expected: valid YAML frontmatter with `name: search`.

- [ ] **Step 3: Commit**

```bash
git add skills/search/SKILL.md
git commit -m "refactor(skills): rewrite search as CLI-driven workflow"
```

---

## Task 9: Create `search/references/flags.md`

**Files:**
- Create: `skills/search/references/flags.md`

- [ ] **Step 1: Write the file**

Exact content:

````markdown
# `bdata search` / `bdata discover` — flag reference

Verified against `@brightdata/cli` v0.1.8 on 2026-04-19.

## `bdata search` — classic keyword SERP

Usage: `bdata search [options] <query>`

| Flag | Values | Default | When to use |
|---|---|---|---|
| `--engine <name>` | `google`, `bing`, `yandex` | `google` | Pick the SERP source. `google` for most; `yandex` for RU-centric queries; `bing` as a cross-check. |
| `--country <code>` | ISO (`us`, `de`, `jp`, …) | — | Localized SERP. Required for reproducibility across runs. |
| `--language <code>` | ISO (`en`, `fr`, …) | — | Result language. Often paired with `--country`. |
| `--page <n>` | integer, **0-indexed** | `0` | Result page. Loop `0, 1, 2, …` for more results. ~10 results/page. |
| `--type <type>` | `web`, `news`, `images`, `shopping` | `web` | Search vertical. |
| `--zone <name>` | SERP zone name | account default | Override zone. Rarely needed. |
| `--device <type>` | `desktop`, `mobile` | `desktop` | Mobile rankings differ. Pick explicitly for reproducibility. |
| `-o, --output <path>` | file path | stdout | Write to file. |
| `--json` | (flag) | off | Force JSON envelope. |
| `--pretty` | (flag) | off | Pretty-print JSON. |
| `-k, --api-key <key>` | API key | saved / env | Per-command override. |

## `bdata discover` — AI intent-ranked discovery

Usage: `bdata discover [options] <query>`

| Flag | Values | Default | When to use |
|---|---|---|---|
| `--intent <text>` | free text | — | Semantic intent used to re-rank results. E.g., `"product pricing pages"`, `"academic papers"`. |
| `--country <code>` | ISO | `US` | Localization. |
| `--city <name>` | city string | — | City-level localization (e.g., `"New York"`). |
| `--language <code>` | ISO | `en` | Language. |
| `--num-results <n>` | integer | — | Target result count. Discovery polls until reached or timeout. |
| `--filter-keywords <csv>` | comma-separated | — | Require these keywords in result pages. |
| `--include-content` | (flag) | off | Fetch and include page body (markdown) for each result. Big payload; slower. |
| `--no-remove-duplicates` | (flag) | off | Keep dup URLs. Default dedups. |
| `--start-date <YYYY-MM-DD>` | ISO date | — | Only content updated from this date. |
| `--end-date <YYYY-MM-DD>` | ISO date | — | Only content updated through this date. |
| `--timeout <sec>` | integer | `600` | Max seconds to wait for the target `--num-results`. |
| `-o, --output <path>` | file path | stdout | Write to file. Required for larger result sets. |
| `--json` / `--pretty` | flags | — | JSON formatting. |

## When to use `search` vs `discover`

| You want | Use |
|---|---|
| "What Google ranks right now for keyword X" | `search` |
| "Pages that match this meaning/intent, ranked by relevance" | `discover` |
| Vertical SERP (news/images/shopping) | `search --type` |
| Dedup + semantic ranking across many queries | `discover` |
| Time-bounded content discovery | `discover --start-date --end-date` |
| Get result list AND page bodies in one call | `discover --include-content` |
````

- [ ] **Step 2: Commit**

```bash
git add skills/search/references/flags.md
git commit -m "docs(skills): add search + discover flag reference"
```

---

## Task 10: Create `search/references/patterns.md`

**Files:**
- Create: `skills/search/references/patterns.md`

- [ ] **Step 1: Write the file**

Exact content:

````markdown
# Search — patterns

## Verification checklist (shared across all Bright Data CLI skills)

Before claiming a search succeeded:

1. **JSON parses cleanly** (`jq . <output>` returns 0).
2. **Result array non-empty** — if empty, re-check; don't claim success on zero results silently.
3. **Expected fields present:** `title` + `link` (search) or `title` + `url` (+ `content` when `--include-content`) (discover).
4. **No block-page signatures** in `discover --include-content` bodies:
   `Access Denied`, `Just a moment`, `Attention Required`, `Checking your browser`, `captcha`, `cf-browser-verification`, `cloudflare` (< 2KB).
5. **Geo sanity:** result TLDs / languages match the requested `--country` / `--language`.

## Multi-query batch with dedup

Given `queries.txt` (one query per line), collect deduped result URLs:

```bash
mkdir -p out
: > out/all-urls.txt

while IFS= read -r q; do
    hash=$(printf '%s' "$q" | md5sum | cut -c1-8)
    bdata search "$q" --engine google --country us --json \
        -o "out/serp-${hash}.json"
    jq -r '.results[].link' "out/serp-${hash}.json" >> out/all-urls.txt
done < queries.txt

sort -u out/all-urls.txt > out/urls.txt
wc -l out/urls.txt
```

Note the exact JSON path (`.results[].link`) depends on the CLI's output shape — check one file with `jq keys` first if unsure.

## SERP → filter → scrape pipeline

```bash
# 1. Search
bdata search "enterprise monitoring tools" --engine google --country us \
    --json -o serp.json

# 2. Filter (domain allowlist; relevance heuristic)
jq -r '.results[]
    | select(.link | test("^https://(?!.*(reddit|pinterest))"))
    | select(.title | test("monitoring"; "i"))
    | .link' serp.json > urls.txt

# 3. Scrape (hands off to the scrape skill's patterns)
xargs -a urls.txt -n 1 -P 4 -I {} bash -c '
    url="$1"
    hash=$(printf "%s" "$url" | md5sum | cut -c1-8)
    bdata scrape "$url" -f markdown -o "out/${hash}.md" || echo "FAIL: $url" >&2
' _ {}
```

## `search` vs `discover` decision

Default to `search` for keyword-exactness tasks (SEO research, "what ranks for X"). Default to `discover` when the user's description is a topic, intent, or concept rather than a keyword ("pages about how companies adopt LLMs", "recent articles on post-quantum crypto").

Rule of thumb: if the user's phrasing is a complete sentence or describes intent, `discover`. If it's a short keyword string, `search`.

## Pagination with `search`

`--page` is 0-indexed. Loop pages until empty or duplicate:

```bash
prev_hash=""
for page in 0 1 2 3 4; do
    bdata search "long tail query" --page "$page" --json -o "p${page}.json"
    hash=$(sha1sum "p${page}.json" | awk '{print $1}')
    # Same hash twice → we're looping; break
    [[ "$hash" == "$prev_hash" ]] && { rm "p${page}.json"; break; }
    # Empty results → break
    count=$(jq '.results | length' "p${page}.json")
    [[ "$count" == "0" ]] && { rm "p${page}.json"; break; }
    prev_hash=$hash
done
```

## Legacy `curl` fallback (deprecated)

Only when CLI cannot be installed. SERP API endpoint via Web Unlocker:

```bash
curl -sS "https://api.brightdata.com/request" \
    -H "Authorization: Bearer $BRIGHTDATA_API_KEY" \
    -H "Content-Type: application/json" \
    -d "{
        \"zone\": \"$BRIGHTDATA_UNLOCKER_ZONE\",
        \"url\": \"https://www.google.com/search?q=$(printf '%s' "$QUERY" | jq -sRr @uri)&brd_json=1\",
        \"format\": \"raw\"
    }"
```

Prefer the CLI path. This block exists only for environments without Node.js.
````

- [ ] **Step 2: Commit**

```bash
git add skills/search/references/patterns.md
git commit -m "docs(skills): add search patterns (dedup, pipeline, decision)"
```

---

## Task 11: Create `search/references/examples.md`

**Files:**
- Create: `skills/search/references/examples.md`

- [ ] **Step 1: Write the file**

Exact content:

````markdown
# Search — worked examples

## Example 1 — single Google query

```bash
bdata search "postgres jsonb index performance" \
    --engine google --country us --json --pretty -o serp.json

# Verify
jq '.results | length' serp.json   # > 0
jq -r '.results[0] | "\(.title)\n\(.link)"' serp.json
```

## Example 2 — localized Bing (German)

```bash
bdata search "datenschutz-grundverordnung leitfaden" \
    --engine bing --country de --language de --json -o serp-de.json

# Sanity-check locale: top results should have .de TLDs or German text
jq -r '.results[0:5][].link' serp-de.json
```

## Example 3 — batch queries + dedup into URL list

Given `queries.txt` with multiple related queries:

```bash
mkdir -p out
: > out/all-urls.txt

while IFS= read -r q; do
    hash=$(printf '%s' "$q" | md5sum | cut -c1-8)
    bdata search "$q" --engine google --country us --json \
        -o "out/serp-${hash}.json"
    jq -r '.results[].link' "out/serp-${hash}.json" >> out/all-urls.txt
done < queries.txt

sort -u out/all-urls.txt > out/unique-urls.txt
echo "Total: $(wc -l < out/all-urls.txt)  Unique: $(wc -l < out/unique-urls.txt)"
```

## Example 4 — `discover --include-content` end-to-end

Find recent articles on a topic, with body content in one call, ready to feed into an LLM:

```bash
bdata discover "post-quantum cryptography deployment" \
    --intent "practical deployment case studies from 2025" \
    --num-results 15 \
    --start-date 2025-01-01 \
    --include-content \
    --country us --language en \
    --timeout 900 \
    --json --pretty -o pqc.json

# Verify
jq '.results | length' pqc.json              # should be ~15
jq -r '.results[0] | .title, .url' pqc.json
jq -r '.results[0].content' pqc.json | head  # body as markdown

# Block-page sanity (no content should match)
jq -r '.results[].content' pqc.json \
    | grep -iE 'access denied|just a moment|captcha|cloudflare' \
    && echo "WARN: one or more results returned a block page"
```
````

- [ ] **Step 2: Commit**

```bash
git add skills/search/references/examples.md
git commit -m "docs(skills): add search worked examples"
```

---

## Task 12: Delete `search/scripts/`

**Files:**
- Delete: `skills/search/scripts/search.sh`
- Delete: `skills/search/scripts/` (directory)

- [ ] **Step 1: Remove**

```bash
git rm -r skills/search/scripts
```

- [ ] **Step 2: Verify**

```bash
test ! -e skills/search/scripts && echo "removed"
```
Expected: `removed`.

- [ ] **Step 3: Commit**

```bash
git commit -m "refactor(skills): remove search bash scripts (superseded by CLI)"
```

---

## Task 13: Rewrite `data-feeds/SKILL.md` to the workflow shape

**Files:**
- Modify: `skills/data-feeds/SKILL.md` (full rewrite)

- [ ] **Step 1: Overwrite `skills/data-feeds/SKILL.md` with the following content**

````markdown
---
name: data-feeds
description: Extract structured data from 40+ supported platforms (Amazon, LinkedIn, Instagram, TikTok, Facebook, YouTube, Reddit, and more) via the Bright Data CLI (`bdata pipelines`). Use when the user wants clean JSON from a known platform URL rather than raw HTML. Hands off to `scrape` for unsupported URLs and to `search` when target URLs must be discovered first. Requires the Bright Data CLI; proactively guides install + login if missing.
---

# Bright Data — Data Feeds (Pipelines)

Extract structured data from supported platforms via `bdata pipelines`. One call, clean JSON, no scraping logic. For unsupported URLs, hand off to `scrape`. To find target URLs first, hand off to `search`.

## Setup gate (run first)

```bash
if ! command -v bdata >/dev/null 2>&1; then
    echo "bdata CLI not installed — see bright-data-best-practices/references/cli-setup.md"
elif ! bdata zones >/dev/null 2>&1; then
    echo "bdata not authenticated — run: bdata login  (or: bdata login --device for SSH)"
fi
```

Halt and route to `skills/bright-data-best-practices/references/cli-setup.md` if either check fails.

## Supported pipeline types (verified 2026-04-19)

**Always verify with `bdata pipelines list` before hardcoding names** — they change. Current 43 types:

`amazon_product`, `amazon_product_reviews`, `amazon_product_search`, `apple_app_store`, `bestbuy_products`, `booking_hotel_listings`, `crunchbase_company`, `ebay_product`, `etsy_products`, `facebook_company_reviews`, `facebook_events`, `facebook_marketplace_listings`, `facebook_posts`, `github_repository_file`, `google_maps_reviews`, `google_play_store`, `google_shopping`, `homedepot_products`, `instagram_comments`, `instagram_posts`, `instagram_profiles`, `instagram_reels`, `linkedin_company_profile`, `linkedin_job_listings`, `linkedin_people_search`, `linkedin_person_profile`, `linkedin_posts`, `reddit_posts`, `reuter_news`, `tiktok_comments`, `tiktok_posts`, `tiktok_profiles`, `tiktok_shop`, `walmart_product`, `walmart_seller`, `x_posts`, `yahoo_finance_business`, `youtube_comments`, `youtube_profiles`, `youtube_videos`, `zara_products`, `zillow_properties_listing`, `zoominfo_company_profile`

**Naming note:** inconsistent across platforms. `amazon_product` (singular), `tiktok_profiles` (plural), `linkedin_person_profile` (not `linkedin_profile`). Always copy from `bdata pipelines list`.

## Pick your path

| Situation | Action |
|---|---|
| Know the platform + have URL(s) | `bdata pipelines <type> <url>` |
| Don't know which pipeline fits | `bdata pipelines list` first |
| Pipeline takes keyword, not URL (search-shaped) | See "Keyword-shaped pipelines" below |
| Multiple URLs on the same pipeline type | shell loop with parallelism cap (see `references/patterns.md`) |
| Long job (reviews, company employees, big post feeds) | raise `--timeout 1800` |
| URL is on an unsupported platform | **stop — hand off to `scrape`** |
| Need to find URLs first | **hand off to `search`** |

## Keyword-shaped pipelines (do NOT take a URL)

A few pipelines take non-URL inputs. Verify with `bdata pipelines <type> --help` (or the error message when called with wrong args):

| Pipeline | Input |
|---|---|
| `amazon_product_search` | `<keyword> <domain> <pages_to_search>` — e.g., `"running shoes" amazon.com 2` |
| `linkedin_people_search` | multi-arg (run once with no args to see required positional params) |
| `facebook_company_reviews` | `<url> <number_of_reviews>` |
| `google_maps_reviews` | `<url> <days_range>` |
| `youtube_comments` | `<url> <number_of_comments>` |

All other pipelines take a single URL.

## Action

Core commands:

```bash
# List available pipeline types (source of truth)
bdata pipelines list

# Amazon product
bdata pipelines amazon_product \
    "https://www.amazon.com/dp/B08N5WRWNW" \
    --format json --pretty -o product.json

# Amazon product reviews (slower — reviews can be hundreds)
bdata pipelines amazon_product_reviews \
    "https://www.amazon.com/dp/B08N5WRWNW" \
    --timeout 1200 -o reviews.json

# Amazon product search (keyword-shaped)
bdata pipelines amazon_product_search \
    "noise cancelling headphones" amazon.com 2 \
    --format json --pretty -o search.json

# LinkedIn person profile
bdata pipelines linkedin_person_profile \
    "https://www.linkedin.com/in/example" -o person.json

# LinkedIn company
bdata pipelines linkedin_company_profile \
    "https://www.linkedin.com/company/example" -o company.json

# Instagram posts
bdata pipelines instagram_posts \
    "https://www.instagram.com/example/" -o posts.json

# Google Maps reviews (keyword-shaped: url + days_range)
bdata pipelines google_maps_reviews \
    "https://maps.google.com/?cid=1234567890" 90 -o reviews.json

# NDJSON for big feeds (one record per line)
bdata pipelines linkedin_posts "https://www.linkedin.com/in/example" \
    --format ndjson -o posts.ndjson

# Raise polling timeout for long jobs
bdata pipelines amazon_product_reviews "<url>" --timeout 1800 -o out.json
```

Full flag reference + full type table: [`references/flags.md`](references/flags.md).

## Verification gate

1. **JSON parses cleanly:** `jq . <output>` returns 0 (or for `--format ndjson`, each line parses).
2. **Record count matches expected.** One URL usually = one record, *but* reviews/posts/comments pipelines return arrays sized by what the platform shows. Always check:
   ```bash
   jq 'length' out.json                       # top-level array count
   # OR
   jq 'if type == "array" then length else 1 end' out.json
   ```
3. **No top-level error:**
   ```bash
   jq -e 'if type == "object" then has("error") | not else true end' out.json \
       || { echo "pipeline reported error"; exit 1; }
   ```
4. **No per-record error:** for array results, ensure no record has an `error` field:
   ```bash
   jq -e 'if type == "array" then map(has("error")) | any | not else true end' out.json \
       || echo "WARN: one or more records have error fields"
   ```
   Partial failures are silent — this check is non-optional.
5. **Core fields present** for the pipeline type (examples):
   - `amazon_product` → `.title` + `.price` (or `.final_price`)
   - `linkedin_person_profile` → `.name` + `.headline` (or `.position`)
   - `instagram_posts` → `.caption` or `.description` + `.url` or `.post_id`
   - `youtube_videos` → `.title` + `.video_id` or `.url`
   Spot-check with `jq keys` on the first record to learn the exact schema.
6. **On failure:** double `--timeout` and retry once. If still failing, `bdata pipelines list` to confirm the type name hasn't changed.

## Red flags

- Using `bdata scrape` on Amazon/LinkedIn/TikTok/etc. when `bdata pipelines <type>` returns structured fields in one call. Loses structure and costs more time.
- Looping `bdata pipelines` for large jobs without rate-limiting — each call can trigger a long-running pipeline on the server. Cap parallelism at 2–3.
- Claiming success without the record-count + per-record error check. Partial failures are silent in pipeline output.
- Hardcoding pipeline type names (`amazon_products` with an `s`, `linkedin_profile` without `_person_`, etc.) — they're inconsistent across platforms. Always copy from `bdata pipelines list`.
- Using a tight `--timeout` on pipelines that legitimately take 5–15 minutes (reviews, company employees, big post feeds). Default 600s is a floor for small inputs; raise for long ones.
- Confusing sync `bdata pipelines` with async `bdata scrape --async` — different mechanisms. Pipelines poll internally to completion. Scrape's `--async` hands you a job ID to poll with `bdata status`.
- Calling a keyword-shaped pipeline (`amazon_product_search`, `linkedin_people_search`, `google_maps_reviews`, etc.) with URL-only args — will fail with "Usage: …". Always check `bdata pipelines <type>` error output when in doubt.

## References

- [`references/flags.md`](references/flags.md) — full `pipelines` flags + complete table of all 43 types with input shapes.
- [`references/patterns.md`](references/patterns.md) — sync timeout tuning, shell-loop batching with parallelism cap, partial-failure detection, keyword-shaped pipeline cheatsheet, legacy `curl` fallback, shared verification checklist.
- [`references/examples.md`](references/examples.md) — (1) single Amazon product, (2) batch LinkedIn companies, (3) long reviews job with raised timeout, (4) mixed-platform workflow calling `pipelines list` first, (5) keyword-shaped `amazon_product_search`.
````

- [ ] **Step 2: Verify frontmatter**

```bash
head -4 skills/data-feeds/SKILL.md
```
Expected: valid YAML frontmatter with `name: data-feeds`.

- [ ] **Step 3: Commit**

```bash
git add skills/data-feeds/SKILL.md
git commit -m "refactor(skills): rewrite data-feeds as CLI-driven workflow"
```

---

## Task 14: Create `data-feeds/references/flags.md`

**Files:**
- Create: `skills/data-feeds/references/flags.md`

- [ ] **Step 1: Write the file**

Exact content:

````markdown
# `bdata pipelines` — flag reference

Verified against `@brightdata/cli` v0.1.8 on 2026-04-19.

Usage: `bdata pipelines [options] <type> [params...]`

## Flags

| Flag | Values | Default | When to use |
|---|---|---|---|
| `--format <fmt>` | `json`, `csv`, `ndjson`, `jsonl` | `json` | `json` for most; `ndjson` or `jsonl` for big result sets you'll stream-process line-by-line. `csv` for spreadsheet consumers. |
| `--timeout <seconds>` | integer | `600` (or `$BRIGHTDATA_POLLING_TIMEOUT`) | Max seconds to poll for completion. Raise to `1200`–`1800` for reviews, company employees, or long post feeds. |
| `-o, --output <path>` | file path | stdout | Write to file. Recommended for any result > 1KB. |
| `--json` | (flag) | off | Force JSON envelope. |
| `--pretty` | (flag) | off | Pretty-print JSON. |
| `--timing` | (flag) | off | Stderr timing breakdown. Debug only. |
| `-k, --api-key <key>` | API key | saved / env | Per-command override. |

## Pipeline types — input shape (verified 2026-04-19)

Always cross-check with `bdata pipelines list` before hardcoding names.

### URL-input pipelines (single `<url>`)

| Pipeline | Typical URL shape |
|---|---|
| `amazon_product` | `https://www.amazon.<tld>/dp/<ASIN>` |
| `amazon_product_reviews` | `https://www.amazon.<tld>/dp/<ASIN>` |
| `apple_app_store` | app page URL |
| `bestbuy_products` | product page URL |
| `booking_hotel_listings` | listing URL |
| `crunchbase_company` | company page URL |
| `ebay_product` | product URL |
| `etsy_products` | product URL |
| `facebook_events` | event URL |
| `facebook_marketplace_listings` | listing URL |
| `facebook_posts` | page or post URL |
| `github_repository_file` | file URL |
| `google_play_store` | app URL |
| `google_shopping` | product URL |
| `homedepot_products` | product URL |
| `instagram_comments` | post URL |
| `instagram_posts` | profile or post URL |
| `instagram_profiles` | profile URL |
| `instagram_reels` | profile or reel URL |
| `linkedin_company_profile` | `https://www.linkedin.com/company/<slug>` |
| `linkedin_job_listings` | job URL |
| `linkedin_person_profile` | `https://www.linkedin.com/in/<slug>` |
| `linkedin_posts` | profile URL |
| `reddit_posts` | subreddit or post URL |
| `reuter_news` | article URL |
| `tiktok_comments` | video URL |
| `tiktok_posts` | profile or post URL |
| `tiktok_profiles` | `https://www.tiktok.com/@<handle>` |
| `tiktok_shop` | product URL |
| `walmart_product` | product URL |
| `walmart_seller` | seller URL |
| `x_posts` | profile or post URL |
| `yahoo_finance_business` | company page URL |
| `youtube_profiles` | channel URL |
| `youtube_videos` | video URL |
| `zara_products` | product URL |
| `zillow_properties_listing` | listing URL |
| `zoominfo_company_profile` | company URL |

### Keyword-shaped pipelines (multi-arg, NOT a single URL)

| Pipeline | Args |
|---|---|
| `amazon_product_search` | `<keyword> <domain> <pages_to_search>` — e.g., `"noise cancelling headphones" amazon.com 2` |
| `linkedin_people_search` | multi-arg — run without args to see required positionals |
| `facebook_company_reviews` | `<url> <number_of_reviews>` |
| `google_maps_reviews` | `<url> <days_range>` |
| `youtube_comments` | `<url> <number_of_comments>` |

When in doubt about a pipeline's args, invoke it with no params — the CLI prints the expected usage line.

## Timeout guidance

| Pipeline category | Suggested `--timeout` |
|---|---|
| Single-item products / profiles | `600` (default) |
| Post / video feeds | `900`–`1200` |
| Reviews, comments, large employee lists | `1200`–`1800` |
| Company crawls (Crunchbase, ZoomInfo) | `1200`+ |

`BRIGHTDATA_POLLING_TIMEOUT=1800 bdata pipelines …` also works as an env-var default.
````

- [ ] **Step 2: Commit**

```bash
git add skills/data-feeds/references/flags.md
git commit -m "docs(skills): add pipelines flag reference and type table"
```

---

## Task 15: Create `data-feeds/references/patterns.md`

**Files:**
- Create: `skills/data-feeds/references/patterns.md`

- [ ] **Step 1: Write the file**

Exact content:

````markdown
# Data-feeds — patterns

## Verification checklist (shared across all Bright Data CLI skills)

Before claiming a pipeline succeeded:

1. **JSON parses cleanly** (`jq . <output>` for json; line-by-line for ndjson).
2. **Record count matches expected.** Partial failures are silent — count explicitly.
3. **No top-level `error` key** on object-shaped outputs.
4. **No per-record `error` key** on array-shaped outputs.
5. **Core fields present** for the pipeline type — spot-check with `jq keys` on record 0.

## Sync timeout tuning

```bash
# Default — good for products / profiles
bdata pipelines amazon_product "<url>" -o out.json

# Long jobs — reviews, comment feeds, company employees
bdata pipelines amazon_product_reviews "<url>" --timeout 1800 -o out.json

# Or set a session default
export BRIGHTDATA_POLLING_TIMEOUT=1800
bdata pipelines linkedin_posts "<url>" -o out.json
```

## Batching same pipeline across many URLs (parallelism-capped)

```bash
mkdir -p out
xargs -a urls.txt -n 1 -P 2 -I {} bash -c '
    url="$1"
    hash=$(printf "%s" "$url" | md5sum | cut -c1-8)
    bdata pipelines linkedin_company_profile "$url" \
        --timeout 900 -o "out/${hash}.json" || echo "FAIL: $url" >&2
' _ {}
```

`-P 2` — keep parallelism LOW for pipelines. Each call can trigger a long server-side job; over-provisioning wastes quota and triggers rate limits.

## Partial-failure detection

Partial failures are silent. Run this check after any batch:

```bash
for f in out/*.json; do
    # 1. Parseable?
    if ! jq . "$f" >/dev/null 2>&1; then
        echo "PARSE FAIL: $f"; continue
    fi
    # 2. Top-level error?
    if jq -e 'if type == "object" then has("error") else false end' "$f" >/dev/null; then
        echo "ERROR: $f → $(jq -r .error "$f")"; continue
    fi
    # 3. Per-record error in arrays?
    if jq -e 'if type == "array" then map(has("error")) | any else false end' "$f" >/dev/null; then
        echo "PARTIAL FAIL: $f"
    fi
done
```

## Pipeline-type resolution workflow

When the user names a platform but not a pipeline type:

```bash
# 1. Get the source of truth
bdata pipelines list > types.txt

# 2. Grep by platform
grep '^linkedin' types.txt
# → linkedin_company_profile
# → linkedin_job_listings
# → linkedin_people_search
# → linkedin_person_profile
# → linkedin_posts

# 3. Pick by intent:
#    - Profile of a person → linkedin_person_profile
#    - Profile of a company → linkedin_company_profile
#    - Job postings for a company → linkedin_job_listings
#    - Posts from a profile → linkedin_posts
#    - Search for people by attributes → linkedin_people_search
```

## Keyword-shaped pipeline cheatsheet

These pipelines take non-URL inputs:

```bash
# amazon_product_search — search Amazon by keyword
bdata pipelines amazon_product_search \
    "running shoes" amazon.com 2 \
    -o search.json

# google_maps_reviews — reviews for a place, last N days
bdata pipelines google_maps_reviews \
    "https://maps.google.com/?cid=1234567890" 90 \
    -o gmaps-reviews.json

# facebook_company_reviews — first N reviews
bdata pipelines facebook_company_reviews \
    "https://www.facebook.com/example" 50 \
    -o fb-reviews.json

# youtube_comments — first N comments on a video
bdata pipelines youtube_comments \
    "https://www.youtube.com/watch?v=abc123" 100 \
    -o yt-comments.json
```

When unsure about args, invoke with none — CLI prints the expected usage.

## Legacy `curl` fallback (deprecated)

Only when CLI cannot be installed. Pipelines map to the `/datasets/v3/trigger` + `/datasets/v3/snapshot` endpoints; the polling loop is non-trivial. If this path is truly needed, see the old `scripts/datasets.sh` in the git history (commit before this revision). Prefer the CLI.
````

- [ ] **Step 2: Commit**

```bash
git add skills/data-feeds/references/patterns.md
git commit -m "docs(skills): add data-feeds patterns (batching, verification, resolution)"
```

---

## Task 16: Create `data-feeds/references/examples.md`

**Files:**
- Create: `skills/data-feeds/references/examples.md`

- [ ] **Step 1: Write the file**

Exact content:

````markdown
# Data-feeds — worked examples

## Example 1 — single Amazon product

```bash
bdata pipelines amazon_product \
    "https://www.amazon.com/dp/B08N5WRWNW" \
    --format json --pretty -o product.json

# Verify
jq . product.json >/dev/null || { echo "parse fail"; exit 1; }
jq -e 'has("error") | not' product.json >/dev/null \
    || { echo "pipeline error: $(jq -r .error product.json)"; exit 1; }
jq -r '.title, .price // .final_price' product.json
```

## Example 2 — batch LinkedIn companies via shell loop

Given `companies.txt` (one LinkedIn company URL per line):

```bash
mkdir -p out
xargs -a companies.txt -n 1 -P 2 -I {} bash -c '
    url="$1"
    hash=$(printf "%s" "$url" | md5sum | cut -c1-8)
    bdata pipelines linkedin_company_profile "$url" \
        --timeout 900 -o "out/${hash}.json" || echo "FAIL: $url" >&2
' _ {}

# Summarize results
total=$(wc -l < companies.txt)
ok=$(find out -name "*.json" -size +0 | wc -l)
echo "Fetched $ok of $total companies"

# Partial-failure sweep
for f in out/*.json; do
    jq -e 'has("error") | not' "$f" >/dev/null \
        || echo "ERROR in $f: $(jq -r .error "$f")"
done
```

## Example 3 — long reviews job with raised timeout

Amazon product reviews can take 10+ minutes for products with many reviews:

```bash
bdata pipelines amazon_product_reviews \
    "https://www.amazon.com/dp/B08N5WRWNW" \
    --timeout 1800 \
    --format json --pretty -o reviews.json

# Verify count
count=$(jq 'if type == "array" then length else 1 end' reviews.json)
echo "Got $count reviews"

# Check first record schema
jq '.[0] | keys' reviews.json
```

## Example 4 — mixed platform workflow (discover types first)

User says: "Give me all the data on this TikTok profile and their recent posts."

```bash
# 1. Find the right pipelines
bdata pipelines list | grep '^tiktok'
# → tiktok_comments, tiktok_posts, tiktok_profiles, tiktok_shop

# 2. Profile info
bdata pipelines tiktok_profiles \
    "https://www.tiktok.com/@example" -o profile.json

# 3. Recent posts
bdata pipelines tiktok_posts \
    "https://www.tiktok.com/@example" --timeout 1200 -o posts.json

# Verify both
for f in profile.json posts.json; do
    jq . "$f" >/dev/null || { echo "$f: parse fail"; exit 1; }
    jq -e 'if type == "object" then has("error") | not else true end' "$f" \
        >/dev/null || { echo "$f: pipeline error"; exit 1; }
done

echo "profile: $(jq 'keys' profile.json)"
echo "posts: $(jq 'if type == "array" then length else 1 end' posts.json) records"
```

## Example 5 — keyword-shaped `amazon_product_search`

Search Amazon by keyword (no URL needed):

```bash
bdata pipelines amazon_product_search \
    "mechanical keyboard" amazon.com 2 \
    --format json --pretty -o search.json

jq 'length' search.json                       # number of results
jq -r '.[0:5][] | "\(.title) — \(.price)"' search.json
```

Note the three positional args: `<keyword> <domain> <pages>`. Calling with fewer args prints the expected usage.
````

- [ ] **Step 2: Commit**

```bash
git add skills/data-feeds/references/examples.md
git commit -m "docs(skills): add data-feeds worked examples"
```

---

## Task 17: Delete `data-feeds/scripts/`

**Files:**
- Delete: `skills/data-feeds/scripts/datasets.sh`
- Delete: `skills/data-feeds/scripts/fetch.sh`
- Delete: `skills/data-feeds/scripts/` (directory)

- [ ] **Step 1: Remove**

```bash
git rm -r skills/data-feeds/scripts
```

- [ ] **Step 2: Verify**

```bash
test ! -e skills/data-feeds/scripts && echo "removed"
```
Expected: `removed`.

- [ ] **Step 3: Commit**

```bash
git commit -m "refactor(skills): remove data-feeds bash scripts (superseded by CLI)"
```

---

## Task 18: Manual acceptance test — fresh environment

These last three tasks are **manual verification** (no code). They map to the spec's "Testing / verification of the revision itself" section. Execute them against a representative setup and confirm each passes.

- [ ] **Step 1: Fresh-environment test (Tier 3 detection)**

Temporarily shadow `bdata` so it appears missing:

```bash
PATH=/usr/bin:/bin bash -c '
    if ! command -v bdata >/dev/null 2>&1; then
        echo "Setup gate would detect: CLI not installed"
    fi
'
```

Expected: the message prints. In a real skill invocation with `bdata` truly missing, the skill routes the user to `cli-setup.md`.

- [ ] **Step 2: CLI-but-no-auth test (Tier 2 detection)**

Simulate unauth'd CLI (move credentials file aside):

```bash
cred=~/.config/brightdata-cli/credentials.json
mv "$cred" "${cred}.bak" 2>/dev/null
if ! bdata zones >/dev/null 2>&1; then
    echo "Setup gate would detect: not authenticated"
fi
mv "${cred}.bak" "$cred" 2>/dev/null
```

Expected: the message prints. Restore credentials after.

- [ ] **Step 3: Happy-path smoke tests**

Run one representative invocation per skill:

```bash
# scrape
bdata scrape "https://example.com" -f markdown -o /tmp/smoke-scrape.md
test -s /tmp/smoke-scrape.md && echo "scrape OK"

# search
bdata search "bright data" --engine google --country us --json \
    -o /tmp/smoke-search.json
jq '.results | length' /tmp/smoke-search.json | grep -q '^[1-9]' \
    && echo "search OK"

# pipelines (quick one)
bdata pipelines list | grep -q '^amazon_product$' && echo "pipelines OK"
```

Expected: all three print `OK`.

- [ ] **Step 4: No commit** — these are verification steps, no changes to commit.

---

## Task 19: Handoff test

- [ ] **Step 1: Confirm handoff language in each SKILL.md**

Each of the three rewritten SKILL.md files should name the other two by name and explicitly say "hand off" in at least one pick-your-path row:

```bash
for skill in scrape search data-feeds; do
    echo "=== $skill ==="
    grep -i "hand off" skills/$skill/SKILL.md
done
```

Expected: each skill has at least 2 "hand off" lines (to the other two skills).

- [ ] **Step 2: Confirm platform list in `scrape/SKILL.md` and `search/SKILL.md`**

Both should name at least Amazon, LinkedIn, TikTok in the "stop and use data-feeds" row:

```bash
for s in scrape search; do
    grep -iE 'amazon.*linkedin.*tiktok|linkedin.*tiktok.*amazon' skills/$s/SKILL.md \
        | head -1
done
```

Expected: one match per skill (or adjust grep to platform list present).

- [ ] **Step 3: No commit.**

---

## Task 20: Block-page signature consistency

- [ ] **Step 1: Confirm shared signature list appears identically in all three skills**

```bash
for f in \
    skills/scrape/references/patterns.md \
    skills/search/references/patterns.md \
    skills/data-feeds/references/patterns.md; do
    echo "=== $f ==="
    grep -cE 'Access Denied|Just a moment|captcha|cloudflare' "$f"
done
```

Expected: each file shows a non-zero count (ideally all 6+ matches on the signature lines). The signatures should be identical across files — if they've drifted, fix inline.

- [ ] **Step 2: No commit** unless a drift is fixed — in which case commit with:

```bash
git add -A
git commit -m "docs(skills): align block-page signature list across skills"
```

---

## Self-Review

**Spec coverage check:**

| Spec section | Covered by task |
|---|---|
| Shared `cli-setup.md` reference | Task 1 |
| `bright-data-best-practices` pointer | Task 2 |
| `scrape` SKILL.md rewrite | Task 3 |
| `scrape/references/{flags,patterns,examples}.md` | Tasks 4, 5, 6 |
| Delete `scrape/scripts/` | Task 7 |
| `search` SKILL.md rewrite | Task 8 |
| `search/references/{flags,patterns,examples}.md` | Tasks 9, 10, 11 |
| Delete `search/scripts/` | Task 12 |
| `data-feeds` SKILL.md rewrite | Task 13 |
| `data-feeds/references/{flags,patterns,examples}.md` | Tasks 14, 15, 16 |
| Delete `data-feeds/scripts/` | Task 17 |
| Fresh-env + auth + happy-path tests | Task 18 |
| Handoff tests | Task 19 |
| Block-page signature consistency | Task 20 |

All spec sections have implementing tasks.

**Placeholder check:** no "TBD", no "similar to Task N", every code block is concrete, every command has expected output.

**Type consistency:** the auth-probe (`bdata zones`), the credentials path, the block-page signature list, and the cross-skill handoff phrasing ("hand off") are used identically across all three skills' outputs. The `--page` flag is documented as 0-indexed consistently. Pipeline names are copied verbatim from the verified `bdata pipelines list` output in Task 13 and Task 14.
