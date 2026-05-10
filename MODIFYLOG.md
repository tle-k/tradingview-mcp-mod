# Modification Log

This file tracks changes made to the `tle-k/tradingview-mcp-mod` fork that diverge from the upstream [`atilaahmettaner/tradingview-mcp`](https://github.com/atilaahmettaner/tradingview-mcp) project.

---

## [2026-05-10] — Google Cloud Run Deployment Configuration

**File modified:** `src/tradingview_mcp/server.py`

### Change 1 — Default transport mode (commit `ccd149b`)

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

### Change 2 — Transport protocol (commit `ccd149b`)

Switched the HTTP transport from `streamable-http` to `sse`.

```python
# Before
mcp.run(transport="streamable-http")

# After
mcp.run(transport="sse")
```

**Why:** `streamable-http` requires HTTP/2 bidirectional streaming, which Cloud Run's frontend proxy does not fully support for MCP use. The `sse` (Server-Sent Events) transport uses standard HTTP/1.1 long-polling, which is compatible with Cloud Run and with the Claude.ai remote MCP connector.

---

### Change 3 — Host binding as keyword argument (commit `4778237`)

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

## Summary

| Commit | Change | Purpose |
|--------|--------|---------|
| `ccd149b` | `default="sse"` in argparse | Default to HTTP transport for Cloud Run |
| `ccd149b` | `transport="sse"` in `mcp.run()` | Use SSE instead of streamable-http |
| `4778237` | `host="0.0.0.0"` as keyword arg in `FastMCP` | Bind to all interfaces for Cloud Run |

All other code — imports, tool handlers, resource routing, and business logic — is identical to upstream.
