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
