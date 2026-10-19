# DuckDuckGo MCP Server (Stealth)

[![GitHub version](https://img.shields.io/github/v/tag/HyperRamzey/duckduckgo-mcp-server-stealth?label=version)](https://github.com/HyperRamzey/duckduckgo-mcp-server-stealth)
[![License: MIT](https://img.shields.io/github/license/HyperRamzey/duckduckgo-mcp-server-stealth)](https://github.com/HyperRamzey/duckduckgo-mcp-server-stealth)

A Model Context Protocol (MCP) server that provides web search capabilities through DuckDuckGo, with additional features for content fetching and parsing, plus **browser-based search** that bypasses DuckDuckGo's bot detection.

This is a fork of [nickclyde/duckduckgo-mcp-server](https://github.com/nickclyde/duckduckgo-mcp-server) with a reliable browser-based search backend (scrapling StealthyFetcher — a real stealth Chromium). All upstream backends (`httpx`, `curl`, `auto`) and features (rate limiting, caching, `ref://` link shortening, `parse_mode` extractors, SSRF guard, TLS/proxy support) are retained; the fork additionally ships `--search-backend browser` as the **default** and targets the MCP SDK 2.x API.

## Quick Start

```bash
# Install (the console command inside the package is still `duckduckgo-mcp-server`)
pip install git+https://github.com/HyperRamzey/duckduckgo-mcp-server-stealth.git
duckduckgo-mcp-server --search-backend browser
```

## Features

- **Web Search**: Search DuckDuckGo with advanced rate limiting and result formatting
- **Content Fetching**: Retrieve and parse webpage content with intelligent text extraction
- **Long URL Shortening**: Over-long result URLs become short `ref://` tokens that `fetch_content` accepts directly, saving context
- **Rate Limiting**: Built-in protection against rate limits for both search and content fetching
- **Error Handling**: Comprehensive error handling and logging
- **LLM-Friendly Output**: Results formatted specifically for large language model consumption

## Installation

Install from this fork's GitHub repository (the fork is not published to PyPI):

```bash
pip install git+https://github.com/HyperRamzey/duckduckgo-mcp-server-stealth.git
```

or as an isolated uv tool (recommended — keeps its MCP SDK dependency out of your other environments):

```bash
uv tool install "duckduckgo-mcp-server-stealth @ git+https://github.com/HyperRamzey/duckduckgo-mcp-server-stealth"
```

The installed console command is still `duckduckgo-mcp-server`.

## Usage

### Running with Claude Desktop

1. Download [Claude Desktop](https://claude.ai/download)
2. Create or edit your Claude Desktop configuration:
   - On macOS: `~/Library/Application Support/Claude/claude_desktop_config.json`
   - On Windows: `%APPDATA%\Claude\claude_desktop_config.json`

Add the following configuration:

**Basic Configuration (No SafeSearch, No Default Region):**
```json
{
    "mcpServers": {
        "ddg-search": {
            "command": "uvx",
            "args": ["--from", "duckduckgo-mcp-server-stealth", "duckduckgo-mcp-server"]
        }
    }
}
```

**With SafeSearch and Region Configuration:**
```json
{
    "mcpServers": {
        "ddg-search": {
            "command": "uvx",
            "args": ["--from", "duckduckgo-mcp-server-stealth", "duckduckgo-mcp-server"],
            "env": {
                "DDG_SAFE_SEARCH": "STRICT",
                "DDG_REGION": "cn-zh"
            }
        }
    }
}
```

**Configuration Options:**
- `DDG_SAFE_SEARCH`: SafeSearch filtering level (optional)
  - `STRICT`: Maximum content filtering (kp=1)
  - `MODERATE`: Balanced filtering (kp=-1, default if not specified)
  - `OFF`: No content filtering (kp=-2)
- `DDG_REGION`: Default region/language code (optional, examples below)
  - `us-en`: United States (English)
  - `cn-zh`: China (Chinese)
  - `jp-ja`: Japan (Japanese)
  - `wt-wt`: No specific region
  - Leave empty for DuckDuckGo's default behavior
- `DDG_CA_CERTS`: Path to a PEM CA bundle used to verify TLS certificates on outbound requests (optional). Needed behind TLS-intercepting proxies — see [Running behind a TLS-intercepting proxy](#running-behind-a-tls-intercepting-proxy).
- `DDG_RATE_LIMIT_STRATEGY`: `sliding` (default, historical 60s window) or `token_bucket` (burst, then smooth).
- `DDG_SEARCH_RPM`: Search requests per minute (default: `30`).
- `DDG_FETCH_RPM`: Global `fetch_content` requests per minute (default: `20`).
- `DDG_FETCH_HOST_RPM`: Optional per-host fetch cap (default: `0`, off). Set a positive number to enable.
- `DDG_CACHE_TTL`: Seconds to keep a parsed page in the in-memory `fetch_content` cache (default: `300`). Paginated reads of the same URL reuse one download. Set `0` to disable.
- `DDG_CACHE_MAX_ENTRIES`: Maximum pages kept in that cache (default: `64`). Least-recently-used eviction. Set `0` to disable.
- `DDG_PARSE_MODE`: Default `fetch_content` extractor (`text`, `main`, or `markdown`). Default is `text` (historical flattened page). Per-call `parse_mode` overrides this.
- `DDG_REF_URL_THRESHOLD`: Search-result URLs longer than this many characters are replaced with short `ref://<id>` tokens (default: `120`). Set `0` to always show full URLs. Also `--ref-url-threshold`.

3. Restart Claude Desktop

### Running with Claude Code

1. Download [Claude Code](https://github.com/anthropics/claude-code)
2. Ensure [`uvenv`](https://github.com/robinvandernoord/uvenv) is installed and the `uvx` command is available
3. Add the MCP server: `claude mcp add ddg-search uvx --from duckduckgo-mcp-server-stealth duckduckgo-mcp-server`

### Running with SSE or Streamable HTTP

The server supports alternative transports for use with other MCP clients:

```bash
# SSE transport
uvx --from duckduckgo-mcp-server-stealth duckduckgo-mcp-server --transport sse

# Streamable HTTP transport
uvx --from duckduckgo-mcp-server-stealth duckduckgo-mcp-server --transport streamable-http
```

The default transport is `stdio`, which is used by Claude Desktop and Claude Code.

When running with `sse` or `streamable-http`, override the default bind address (`127.0.0.1:8000`) with the `--host` and `--port` flags:

```bash
uvx --from duckduckgo-mcp-server-stealth duckduckgo-mcp-server --transport streamable-http --host 0.0.0.0 --port 7070
```

#### Running behind a reverse proxy or in Docker

The MCP SDK enables DNS-rebinding protection for the HTTP transports and, by default, only accepts `Host`/`Origin` headers for `localhost`. Behind a reverse proxy or in a container the client's `Host` header won't match, so requests fail with **`421 Misdirected Request`**.

Fix it by allow-listing the host(s) and origin(s) clients actually use (preferred over disabling protection). Values support `host`, `host:port`, and wildcard-port `host:*`:

```bash
uvx --from duckduckgo-mcp-server-stealth duckduckgo-mcp-server --transport streamable-http --host 0.0.0.0 --port 7070 \
  --allowed-hosts ddg-mcp.example.com "ddg-mcp.example.com:*" \
  --allowed-origins "https://ddg-mcp.example.com"
```

Equivalent environment variables (comma-separated) are also available: `DDG_ALLOWED_HOSTS`, `DDG_ALLOWED_ORIGINS`.

As a last resort you can turn the check off entirely with `--disable-dns-rebinding-protection` (or `DDG_DISABLE_DNS_REBINDING_PROTECTION=1`). Prefer an allow-list — disabling protection removes a defense against DNS-rebinding attacks. When nothing is configured, the secure localhost-only default is preserved.

#### Running behind a TLS-intercepting proxy

Corporate proxies that re-sign HTTPS traffic with their own CA (via `HTTPS_PROXY`) cause outbound requests to fail with certificate verification errors, because the HTTP clients don't trust the proxy's self-signed CA (and httpx no longer reads the `SSL_CERT_FILE` environment variable). Point the server at your proxy's CA bundle:

```bash
uvx --from duckduckgo-mcp-server-stealth duckduckgo-mcp-server --ca-certs /path/to/proxy-ca.pem
```

Or set `DDG_CA_CERTS=/path/to/proxy-ca.pem`. The bundle is used by both the `search` and `fetch_content` tools, on the httpx and curl backends alike.

As a last resort, `--no-ssl-verify` (or `DDG_SSL_VERIFY=0`) disables certificate verification entirely. This exposes traffic to interception by anyone on the network path — prefer `--ca-certs`.

### Backends (bypassing bot detection)

Some sites — and, as of recently, DuckDuckGo's own search endpoint (`html.duckduckgo.com`) — block the default `httpx` client because of its distinctive TLS fingerprint, regardless of User-Agent. Cloudflare Bot Management and similar filters key on the JA3/TLS handshake, not on headers, so `html.duckduckgo.com` may answer `httpx` with an empty **HTTP 202** page (silently yielding "no results"). An opt-in backend, `curl` (implemented via `curl_cffi`), impersonates a real Chrome browser's TLS handshake and passes through those checks.

Both the `search` tool and the `fetch_content` tool support these backends.

**Installation:**

```bash
# Default install (includes the browser backend and httpx)
pip install git+https://github.com/HyperRamzey/duckduckgo-mcp-server-stealth.git

# With the optional curl backend (TLS impersonation)
pip install "duckduckgo-mcp-server-stealth[curl]"
```

**Backend options:**

| Value  | Behavior                                                                                  | Needs `[curl]` |
| ------ | ----------------------------------------------------------------------------------------- | ------------- |
| `httpx` | Lightweight async HTTP. Works on most sites.                                             | no                |
| `curl` | Uses `curl_cffi` with Chrome 131 TLS impersonation. Passes TLS-fingerprint-based filters. | yes               |
| `auto` | Tries `httpx` first; on 403 or a Cloudflare challenge response, retries with `curl`.      | yes               |
| `browser` | Search only (default): real stealth Chromium via scrapling `StealthyFetcher`. Most reliable against DuckDuckGo bot detection. | no (core dependency) |

**Two ways to configure the backend:**

1. **Server-wide default** via the `--fetch-backend` CLI flag (applies to every `fetch_content` call):

   ```bash
   # Default behavior — uses httpx
   uvx --from duckduckgo-mcp-server-stealth duckduckgo-mcp-server

   # Force curl for every fetch (requires the [curl] extra)
   uvx --from "duckduckgo-mcp-server-stealth[curl]" duckduckgo-mcp-server --fetch-backend curl

   # Try httpx first, fall back to curl on 403 / Cloudflare challenge
   uvx --from "duckduckgo-mcp-server-stealth[curl]" duckduckgo-mcp-server --fetch-backend auto
   ```

2. **Per-call override** via the `backend` argument on the `fetch_content` tool (overrides the CLI default for that single call). The tool exposes `backend` in its input schema, so an MCP client can choose `"httpx"`, `"curl"`, or `"auto"` on a fetch-by-fetch basis.

For `fetch_content`, the default stays `httpx` so users who don't need the impersonation don't pay for the extra dependency.

#### Search backend

The `search` tool defaults to **`browser`**: it drives a real stealth Chromium instance (scrapling `StealthyFetcher`) so the request carries a genuine Chrome TLS/JA3 fingerprint and JavaScript environment, passing DuckDuckGo's bot detection reliably. The upstream backends are also available: `auto` (httpx first, curl on block), `curl` (TLS impersonation), and `httpx` (legacy, frequently blocked).

Configure the search backend with the `--search-backend` CLI flag or the `DDG_SEARCH_BACKEND` environment variable (`browser` (default) / `auto` / `curl` / `httpx`):

```bash
# Default: stealth Chromium browser (recommended)
duckduckgo-mcp-server --search-backend browser

# Upstream fallback chain: httpx first, curl on block (requires the [curl] extra)
uvx --from "duckduckgo-mcp-server-stealth[curl]" duckduckgo-mcp-server --search-backend auto

# Force curl for every search (requires the [curl] extra)
uvx --from "duckduckgo-mcp-server-stealth[curl]" duckduckgo-mcp-server --search-backend curl

# Opt out of bot-detection handling (legacy behavior — may return no results while blocked)
duckduckgo-mcp-server --search-backend httpx
```

### Development

For local development:

```bash
# Install dependencies
uv sync

# Run with the MCP Inspector
mcp dev src/duckduckgo_mcp_server/server.py

# Install locally for testing with Claude Desktop
mcp install src/duckduckgo_mcp_server/server.py

# Run all tests
uv run python -m pytest src/duckduckgo_mcp_server/ -v

# Run only unit tests
uv run python -m pytest src/duckduckgo_mcp_server/test_server.py -v

# Run only e2e tests
uv run python -m pytest src/duckduckgo_mcp_server/test_e2e.py -v
```

## Available Tools

### 1. Search Tool

```python
async def search(query: str, max_results: int = 10, region: str = "") -> str
```

Performs a web search on DuckDuckGo and returns formatted results.

**Parameters:**
- `query`: Search query string
- `max_results`: Maximum number of results to return (default: 10)
- `region`: (Optional) Region/language code to override the default. Leave empty to use the configured default region.

**Region Code Examples:**
- `us-en`: United States (English)
- `cn-zh`: China (Chinese)
- `jp-ja`: Japan (Japanese)
- `de-de`: Germany (German)
- `fr-fr`: France (French)
- `wt-wt`: No specific region

**Returns:**
Formatted string containing search results with titles, URLs, and snippets.

**Example Usage:**
- Search with default settings: `search("python tutorial")`
- Search with specific region: `search("latest news", region="jp-ja")` for Japanese news

### 2. Content Fetching Tool

```python
async def fetch_content(
    url: str,
    start_index: int = 0,
    max_length: int = 8000,
    backend: Optional[str] = None,
    parse_mode: Optional[str] = None,
) -> str
```

Fetches and parses content from a webpage.

**Parameters:**
- `url`: The webpage URL to fetch content from, or a `ref://<id>` token from search results
- `start_index`: Character offset to start reading from (for pagination)
- `max_length`: Maximum number of characters to return
- `backend`: Optional per-call override of the default fetch backend (`"httpx"`, `"curl"`, or `"auto"`). When omitted, uses whatever was set via `--fetch-backend` at server startup.
- `parse_mode`: Optional per-call extractor (`"text"`, `"main"`, or `"markdown"`). When omitted, uses `DDG_PARSE_MODE` / `--parse-mode` (default `text`).

**Returns:**
Cleaned and formatted text content from the webpage. The parsed full page is cached in memory (default 5 minutes) so later pages via `start_index` do not re-download. Metadata includes `cache=hit` or `cache=miss` when the cache is enabled.

> **SSRF protection:** By default `fetch_content` refuses URLs that resolve to
> loopback, private (RFC1918), link-local (including the `169.254.169.254` cloud
> metadata endpoint), reserved, multicast, or unspecified addresses, and it
> re-validates every redirect hop. Only `http`/`https` URLs are allowed. For
> trusted local deployments that need to fetch internal hosts, disable the guard
> with `DDG_ALLOW_PRIVATE_URLS=1` or `--allow-private-urls`. See
> [SECURITY.md](SECURITY.md) for details.

### 3. Link Expansion Tool

```python
async def expand_link(token: str) -> str
```

Search results replace URLs longer than `DDG_REF_URL_THRESHOLD` characters (default 120) with short, stable `ref://<id>` tokens so long tracking-laden links do not eat context. `fetch_content` accepts a token in place of a URL, so the model only needs this tool when it has to show or cite the real link.

**Parameters:**
- `token`: A `ref://<id>` token exactly as it appeared in search results (the bare id also works)

**Returns:**
The full original URL, or an error if the token is unknown. Tokens live in memory for the lifetime of the server process (bounded by an LRU cap), so they are forgotten on restart.

## Features in Detail

### Rate Limiting

- Search: 30 requests per minute by default (`DDG_SEARCH_RPM` / `--search-rpm`)
- Content fetching: 20 requests per minute globally (`DDG_FETCH_RPM` / `--fetch-rpm`)
- Optional per-host fetch cap, off by default (`DDG_FETCH_HOST_RPM` / `--fetch-host-rpm`)
- Strategies: `sliding` (default) or `token_bucket` via `DDG_RATE_LIMIT_STRATEGY` / `--rate-limit-strategy`
- HTTP 429 responses honor `Retry-After` (capped at 30s) and retry once
- Cache hits on `fetch_content` skip both the download and the fetch rate limiter

### Content cache

- In-memory TTL cache of the fully parsed page (before pagination)
- Default TTL 300 seconds, 64 entries, least-recently-used eviction
- Errors are never cached
- Configure with `DDG_CACHE_TTL` / `DDG_CACHE_MAX_ENTRIES` or `--cache-ttl` / `--cache-max-entries`
- Set either value to `0` to disable

### Result Processing

- Removes ads and irrelevant content
- Cleans up DuckDuckGo redirect URLs
- Formats results for optimal LLM consumption
- Truncates long content appropriately

### Content parsing modes

`fetch_content` accepts `parse_mode`:

| Mode | Behavior |
| --- | --- |
| `text` | Historical default. Strip chrome, return flattened page text. |
| `main` | Keep the primary `article` / `main` / content container only. |
| `markdown` | Same primary content, rendered as lightweight markdown (headings, lists, links, code). |

### Content Safety

- **SafeSearch Filtering**: Configured at server startup via `DDG_SAFE_SEARCH` environment variable
  - Controlled by administrators, not modifiable by AI assistants
  - Filters inappropriate content based on the selected level
  - Uses DuckDuckGo's official `kp` parameter

- **Region Localization**:
  - Default region set via `DDG_REGION` environment variable
  - Can be overridden per search request by AI assistants
  - Improves result relevance for specific geographic regions

### Error Handling

- Comprehensive error catching and reporting
- Detailed logging through MCP context
- Graceful degradation on rate limits or timeouts

## Contributing

Issues and pull requests are welcome!

## License

This project is licensed under the MIT License.

## Star History

[![Star History Chart](https://api.star-history.com/svg?repos=HyperRamzey/duckduckgo-mcp-server-stealth&type=Date)](https://star-history.com/#HyperRamzey/duckduckgo-mcp-server-stealth&Date)
