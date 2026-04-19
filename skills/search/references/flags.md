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

### SERP JSON output shape

When invoked with `--json`, `bdata search` returns an envelope with top-level keys:

- `general` — query metadata (result count, timing, spell-correction, …)
- `organic` — the main result array; each entry has `title`, `link`, `description`, `rank`
- `navigation`, `pagination`, `related` — auxiliary UI elements

To extract just the links from the main results: `jq -r '.organic[].link'`.

News/images/shopping verticals use different top-level keys (`news`, `images`, `shopping`) but the same record shape.

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

### Discover JSON output shape

- `results` — array of result objects, each with `title`, `link`, `description`, `relevance_score` (and `content` when `--include-content` is set)
- `status`, `timestamp`, `duration_seconds` — job metadata

To extract links: `jq -r '.results[].link'`. To pull `content` bodies: `jq -r '.results[].content'`.

## When to use `search` vs `discover`

| You want | Use |
|---|---|
| "What Google ranks right now for keyword X" | `search` |
| "Pages that match this meaning/intent, ranked by relevance" | `discover` |
| Vertical SERP (news/images/shopping) | `search --type` |
| Dedup + semantic ranking across many queries | `discover` |
| Time-bounded content discovery | `discover --start-date --end-date` |
| Get result list AND page bodies in one call | `discover --include-content` |
