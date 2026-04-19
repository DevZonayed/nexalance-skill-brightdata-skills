# `bdata scrape` — flag reference

Verified against `@brightdata/cli` v0.1.8 on 2026-04-19.

Usage: `bdata scrape [options] <url>`

| Flag | Values | Default | When to use |
|---|---|---|---|
| `-f, --format <format>` | `markdown`, `html`, `screenshot`, `json` | `markdown` | `markdown` for readable content; `html` when you need DOM fidelity; `screenshot` to save a PNG; `json` when the Unlocker has a structured extractor for the URL. |
| `--country <code>` | ISO code (`us`, `de`, `jp`, …) | — | Force a geo-targeted exit. Use when the target site geoblocks, personalizes by country, or returns different content by region. |
| `--zone <name>` | Unlocker zone name | account default | Override the default zone — e.g., when you have a dedicated zone with different residential/mobile settings. |
| `--mobile` | (flag) | off | Use a mobile user agent. Use for m-dot sites or when desktop UA gets blocked. *(Note: in `@brightdata/cli` v0.1.8 this flag is declared but not yet forwarded to the Web Unlocker request — verify behavior before relying on it.)* |
| `--async` | (flag) | off | Submit asynchronously; returns a `response_id`. Poll with `bdata status <response_id> --wait`. Use only for pages routinely > 30s or when queuing many long-running jobs. |
| `-o, --output <path>` | file path | stdout | Write result to a file. Required for binary formats (`screenshot`). Recommended for anything > 1KB. |
| `--json` | (flag) | off | Force JSON output envelope (metadata + content). Useful in scripts. |
| `--pretty` | (flag) | off | Pretty-print JSON. Combine with `--json` or `-f json`. |
| `--timing` | (flag) | off | Print request timing breakdown to stderr. Debugging only. |
| `-k, --api-key <key>` | API key | saved credentials or `BRIGHTDATA_API_KEY` | Per-command override. Rarely needed — prefer `bdata login`. |

## `--async` polling recipe

```bash
response_id=$(bdata scrape "https://slow.example" --async --json | jq -r '.response_id')

# Poll until the snapshot is ready (bdata status --wait handles the loop)
bdata status "$response_id" --wait --json --pretty -o result.json
```

`bdata status` is the dedicated polling command for async scrape snapshots. Without `--wait` it does a single status check; with `--wait` it polls until the job completes.

## Format decision matrix

| Goal | Format |
|---|---|
| Feed content to an LLM | `markdown` |
| Extract via selectors / regex | `html` |
| Visual regression / proof-of-view | `screenshot` (writes PNG — use `-o` required) |
| URL has a structured extractor (Unlocker auto-parses) | `json` |
