# DuckDuckGo MCP Server (Stealth)

[![PyPI version](https://img.shields.io/pypi/v/duckduckgo-mcp-server-stealth)](https://pypi.org/project/duckduckgo-mcp-server-stealth/)
[![PyPI downloads](https://img.shields.io/pypi/dm/duckduckgo-mcp-server-stealth)](https://pypi.org/project/duckduckgo-mcp-server-stealth/)
[![Python versions](https://img.shields.io/pypi/pyversions/duckduckgo-mcp-server-stealth)](https://pypi.org/project/duckduckgo-mcp-server-stealth/)

A Model Context Protocol (MCP) server providing DuckDuckGo web search and webpage content fetching, with **browser-based search** that bypasses DuckDuckGo's bot detection.

This is a fork of [nickclyde/duckduckgo-mcp-server](https://github.com/nickclyde/duckduckgo-mcp-server) with a reliable browser-based search backend using [scrapling](https://github.com/D4V1D/scrapling).

## Why this fork?

The original server uses plain `httpx` for DuckDuckGo search, which is frequently blocked by DuckDuckGo's bot detection. The `curl_cffi` TLS impersonation approach also fails with TLS SNI errors under vless VPN for me. This fork uses **scrapling's `StealthyFetcher`** — a real Chromium browser with stealth flags — which reliably bypasses bot detection and returns full search results.

## Quick Start

```bash
uvx duckduckgo-mcp-server-stealth
```

## Installation

```bash
pip install duckduckgo-mcp-server-stealth
```

This installs `scrapling` (which includes `patchright` / Playwright) as a dependency. The first time the server runs, it will automatically download a Chromium browser binary.

## Usage

### Adding to Claude Code (global)

```bash
claude mcp add ddg-search 'duckduckgo-mcp-server' -- --search-backend browser
```

### Adding to Claude Code (project-level)

From the project directory:

```bash
claude mcp add ddg-search 'duckduckgo-mcp-server' -- --search-backend browser
```

### Adding to Claude Code (with custom region)

```bash
claude mcp add ddg-search 'duckduckgo-mcp-server' -- --search-backend browser -- DDG_REGION=us-en
```

### Adding to Claude Desktop

Create or edit your Claude Desktop configuration:
- On Windows: `%APPDATA%\Claude\claude_desktop_config.json`

```json
{
    "mcpServers": {
        "ddg-search": {
            "command": "duckduckgo-mcp-server",
            "args": ["--search-backend", "browser"],
            "env": {
                "DDG_REGION": "us-en"
            }
        }
    }
}
```

### Command line

```bash
# Default — uses browser backend (recommended)
duckduckgo-mcp-server

# Explicitly choose a backend
duckduckgo-mcp-server --search-backend browser   # Chromium browser (default)
duckduckgo-mcp-server --search-backend curl       # curl_cffi TLS impersonation
duckduckgo-mcp-server --search-backend httpx      # Lightweight HTTP client

# With SSE transport
duckduckgo-mcp-server --transport sse --port 8000
```

## Search Backend Options

| Backend | How it works | Reliability against DDG |
|---------|-------------|------------------------|
| `browser` (default) | Real Chromium browser via scrapling StealthyFetcher with ~60 stealth flags | **Reliable** — passes all bot detection |
| `curl` | curl_cffi with Chrome 131 TLS impersonation | May fail with TLS SNI errors |
| `httpx` | Lightweight async HTTP client | Frequently blocked |

## Available Tools

### `search`

Search DuckDuckGo and return results with titles, URLs, and snippets.

```
mcp__ddg-search-stealth__search(query="Python asyncio tutorial", max_results=10, region="us-en")
```

### `fetch_content`

Fetch and extract clean text content from a webpage.

```
mcp__ddg-search-stealth__fetch_content(url="https://example.com/article", max_length=8000)
```

The `fetch_content` tool supports `--fetch-backend curl` for sites with bot detection (Cloudflare, etc.).

## Environment Variables

| Variable | Default | Description |
|----------|---------|-------------|
| `DDG_SAFE_SEARCH` | `MODERATE` | SafeSearch mode: `STRICT`, `MODERATE`, `OFF` |
| `DDG_REGION` | (empty) | Default region code: `us-en`, `cn-zh`, `jp-ja`, `wt-wt`, etc. |

## Dependencies

- `mcp[cli]>=1.26.0` — MCP framework
- `httpx>=0.28.1` — HTTP client (for `fetch_content` and `--search-backend httpx/curl`)
- `beautifulsoup4>=4.13.3` — HTML parsing
- `scrapling>=0.4.0` — Browser-based search (includes `patchright` / Playwright)

## Development

```bash
# Install dependencies
uv sync

# Run with the MCP Inspector
mcp dev src/duckduckgo_mcp_server/server.py

# Run all tests
uv run python -m pytest src/duckduckgo_mcp_server/ -v

# Lint
ruff check src/
ruff format src/
```

## License

MIT — same as the original [duckduckgo-mcp-server](https://github.com/nickclyde/duckduckgo-mcp-server).
