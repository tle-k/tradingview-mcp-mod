# Modification Log

This file tracks changes made to the `tle-k/tradingview-mcp-mod` fork that diverge from the upstream [`atilaahmettaner/tradingview-mcp`](https://github.com/atilaahmettaner/tradingview-mcp) project.

---

## [2026-05-10] — Google Cloud Run Deployment Configuration

**File modified:** `src/tradingview_mcp/server.py`

### Change 1 — Host binding (commit `ccd149b`)

Converted the server from a local-only tool to a public-facing web service by binding to all network interfaces.

```python
# Before
mcp = FastMCP(
    name="TradingView Multi-Market Screener",
    ...

# After
mcp = FastMCP(
    "0.0.0.0",
    name="TradingView Multi-Market Screener",
    ...
```

**Why:** Cloud Run routes inbound HTTPS traffic to the container's internal port. Without `host="0.0.0.0"`, the server binds only to `127.0.0.1` (loopback) and Cloud Run's proxy cannot reach it, causing all requests to fail with a connection-refused error.

---

### Change 2 — Default transport mode (commit `ccd149b`)

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

### Change 3 — Transport protocol (commit `ccd149b`)

Switched the HTTP transport from `streamable-http` to `sse`.

```python
# Before
mcp.run(transport="streamable-http")

# After
mcp.run(transport="sse")
```

**Why:** `streamable-http` requires HTTP/2 bidirectional streaming, which Cloud Run's frontend proxy does not fully support for MCP use. The `sse` (Server-Sent Events) transport uses standard HTTP/1.1 long-polling, which is compatible with Cloud Run and with the Claude.ai remote MCP connector.

---

### Change 4 — Fix host as keyword argument (commit `4778237`)

Corrected the `FastMCP` initializer to pass the host as a proper keyword argument instead of a positional argument.

```python
# Before (introduced in Change 1 — incorrect)
mcp = FastMCP(
    "0.0.0.0",
    name="TradingView Multi-Market Screener",
    instructions=( ...

# After (corrected)
mcp = FastMCP(
    name="TradingView Multi-Market Screener",
    host="0.0.0.0",
    instructions=( ...
```

**Why:** `FastMCP`'s first positional parameter is `name`, not `host`. Passing `"0.0.0.0"` positionally silently overwrote the server name, left the host unset (still defaulting to `127.0.0.1`), and caused the named `name=` kwarg to conflict. Using `host="0.0.0.0"` as a keyword argument correctly binds the interface while preserving the server name.

---

## Summary

| Commit | Change | Purpose |
|--------|--------|---------|
| `ccd149b` | `"0.0.0.0"` as first arg to `FastMCP` | Bind to all interfaces (later corrected) |
| `ccd149b` | `default="sse"` in argparse | Default to HTTP transport for Cloud Run |
| `ccd149b` | `transport="sse"` in `mcp.run()` | Use SSE instead of streamable-http |
| `4778237` | `host="0.0.0.0"` as keyword arg | Fix incorrect positional arg for host binding |

All other code — imports, tool handlers, resource routing, and business logic — is identical to upstream.
