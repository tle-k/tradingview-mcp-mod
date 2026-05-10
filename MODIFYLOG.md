# Modification Log

This file tracks changes made to the `tle-k/tradingview-mcp-mod` fork that diverge from the upstream [`atilaahmettaner/tradingview-mcp`](https://github.com/atilaahmettaner/tradingview-mcp) project.

---

## [2026-05-10] — Google Cloud Run Deployment Configuration

### `src/tradingview_mcp/server.py`

#### Change 1 — Default transport mode (commit `ccd149b`)

Changed the CLI argument default from `stdio` to `sse`.

```python
# Before
choices=["stdio", "streamable-http"],
default="stdio",

# After
choices=["stdio", "sse"],
default="sse",
```

**Why:** `stdio` transport communicates via stdin/stdout and is designed for local subprocess use (e.g. Claude Desktop). Cloud Run expects a persistent HTTP server. Defaulting to `sse` means the service starts in the correct mode without requiring explicit CLI flags in the container's startup command.

---

#### Change 2 — Transport protocol (commit `ccd149b`)

Switched the HTTP transport from `streamable-http` to `sse`.

```python
# Before
mcp.run(transport="streamable-http")

# After
mcp.run(transport="sse")
```

**Why:** `streamable-http` requires HTTP/2 bidirectional streaming, which Cloud Run's frontend proxy does not fully support for MCP use. The `sse` (Server-Sent Events) transport uses standard HTTP/1.1 long-polling, which is compatible with Cloud Run and with the Claude.ai remote MCP connector.

---

#### Change 3 — Host binding as keyword argument (commit `4778237`)

Added `host="0.0.0.0"` as a keyword argument to the `FastMCP` initializer to bind the server to all network interfaces.

```python
# Before
mcp = FastMCP(
    name="TradingView Multi-Market Screener",
    instructions=( ...

# After
mcp = FastMCP(
    name="TradingView Multi-Market Screener",
    host="0.0.0.0",
    instructions=( ...
```

**Why:** Cloud Run routes inbound HTTPS traffic to the container's internal port. Without `host="0.0.0.0"`, the server binds only to `127.0.0.1` (loopback) and Cloud Run's proxy cannot reach it, causing all requests to fail with a connection-refused error. Passing it as a keyword argument correctly binds the interface while preserving the server name.

---

### `Dockerfile`

#### Change 4 — Transport, host, and port in CMD (commits `748c8b4`, `928ecb2`)

Updated the container startup command to use the correct transport and binding for Cloud Run.

```dockerfile
# Before
EXPOSE 8000
CMD ["streamable-http", "--host", "0.0.0.0", "--port", "8000"]

# After
ENV PORT=8080
EXPOSE 8080
CMD ["sse", "--host", "0.0.0.0", "--port", "8080"]
```

**Why:** Three issues fixed:
- `streamable-http` is no longer a valid transport choice (removed in Change 1 above); replaced with `sse`.
- Cloud Run expects containers to listen on port `8080` by default; updated `EXPOSE` and `ENV PORT` accordingly.
- The port is hardcoded as `"8080"` in the JSON array `CMD` form because Docker's exec syntax does not expand shell variables — `"${PORT}"` would be passed as a literal string rather than the value of the environment variable.

---

## [2026-05-10] — Fix `bollinger_scan` Alphabet Bias on Large Exchanges

### `src/tradingview_mcp/core/services/screener_service.py`

#### Change 5 — Full-universe batching in `fetch_bollinger_analysis` (commit `6acc410`)

Replaced symbol truncation with the same batched-traversal pattern used by `fetch_trending_analysis`.

```python
# Before
symbols = symbols[: limit * 2]
screener = EXCHANGE_SCREENER.get(exchange, "crypto")

try:
    analysis = get_multiple_analysis(screener=screener, interval=timeframe, symbols=symbols)
except Exception as exc:
    raise RuntimeError(f"Analysis failed: {exc}") from exc

rows: List[Row] = []
for key, value in analysis.items():
    ...

# After
screener = EXCHANGE_SCREENER.get(exchange, "crypto")
batch_size = 200
rows: List[Row] = []

for i in range(0, len(symbols), batch_size):
    batch = symbols[i : i + batch_size]
    try:
        analysis = get_multiple_analysis(screener=screener, interval=timeframe, symbols=batch)
    except Exception:
        continue  # skip failed batch, keep scanning the rest

    for key, value in analysis.items():
        ...
```

**Why:** Coinlist files (NYSE.txt, nasdaq.txt, etc.) are sorted alphabetically. The original `symbols[:limit*2]` truncation meant the scan only ever queried the first `limit*2` symbols — at `limit=100` that's 200 tickers, covering roughly A–AT on NYSE (~2,000 symbols total). Results were therefore always early-alphabet stocks regardless of BBW ranking. `fetch_trending_analysis` already used batched traversal correctly; this change brings `fetch_bollinger_analysis` into alignment. `limit` now correctly means "max results returned post-filter" rather than "fraction of universe scanned". Tested on NYSE 1D — results span full alphabet.

---

## Summary

| # | File | Commit | Change | Purpose |
|---|------|--------|--------|---------|
| 1 | `server.py` | `ccd149b` | `default="sse"` in argparse | Default to SSE transport for Cloud Run |
| 2 | `server.py` | `ccd149b` | `transport="sse"` in `mcp.run()` | Replace streamable-http with SSE |
| 3 | `server.py` | `4778237` | `host="0.0.0.0"` keyword arg in `FastMCP` | Bind to all interfaces for Cloud Run |
| 4 | `Dockerfile` | `748c8b4`, `928ecb2` | `sse`, `0.0.0.0`, port `8080` in `CMD` | Fix transport, host, and port for Cloud Run |
| 5 | `screener_service.py` | `6acc410` | Full-universe batching in `fetch_bollinger_analysis` | Fix alphabet bias on large exchanges (NYSE, NASDAQ) |

All other code — imports, tool handlers, resource routing, and business logic — is identical to upstream.
