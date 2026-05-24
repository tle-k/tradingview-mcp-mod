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

## [2026-05-15] — Fix `volume_breakout_scan` Alphabet Bias on Large Exchanges

### `src/tradingview_mcp/core/services/scanner_service.py`

#### Change 6 — Full-universe batching in `volume_breakout_scan` (commits `709e03d`, `ba295d5`)

Removed the 500-symbol hard cap on the batched scan loop and doubled `batch_size` from 100 to 200 to match the established `fetch_bollinger_analysis` pattern.

```python
# Before
batch_size = 100

for i in range(0, min(len(symbols), 500), batch_size):
    batch = symbols[i : i + batch_size]

# After
batch_size = 200

for i in range(0, len(symbols), batch_size):
    batch = symbols[i : i + batch_size]
```

**Why:** Same alphabet-bias pathology as Change 5 — capping the iteration at 500 symbols meant the scan only ever covered roughly A–AT on NASDAQ (~5,700 symbols) and A–AT on NYSE (~2,800 symbols). Sort ties on `volume_strength` (capped at 10) were broken by alphabetical scan order, producing result sets clustered entirely in early-alphabet tickers regardless of actual volume strength. `smart_volume_scan` inherits the fix automatically since it calls `volume_breakout_scan` internally. The `batch_size` bump halves the number of `get_multiple_analysis` round trips per scan (NYSE ~14 batches, NASDAQ ~29 vs. prior 28/57), reducing rate-limit pressure. Tested on NASDAQ 15m — results now span the full alphabet.

---

## [2026-05-15] — Route Stock-Exchange Batch Scans Through Webshare Proxy

### `src/tradingview_mcp/core/services/scanner_service.py` and `screener_service.py`

#### Change 7 — Conditional proxy routing for stock-exchange batch scans (commits `01c352d`, `08d537e`)

Added a `_proxies_for(exchange)` helper to both files and threaded a `proxies=` parameter into `get_multiple_analysis` calls in `volume_breakout_scan`, `fetch_bollinger_analysis`, and `fetch_trending_analysis`.

```python
# New helper (mirrored in both files)
def _proxies_for(exchange: str) -> Optional[dict]:
    """Return Webshare proxy dict for batch scans on stock exchanges; None otherwise."""
    if is_stock_exchange(exchange) and is_proxy_configured():
        return get_proxy()
    return None

# Before (in each batch loop)
analysis = get_multiple_analysis(screener=screener, interval=timeframe, symbols=batch)

# After
proxies = _proxies_for(exchange)  # computed once before the loop
analysis = get_multiple_analysis(
    screener=screener, interval=timeframe, symbols=batch, proxies=proxies
)
```

**Why:** After Change 6 unbounded the universe traversal, full NASDAQ/NYSE scans fired 14–29 batches from a single Cloud Run egress IP in roughly 3 seconds. TradingView rate-limited the IP aggressively, returning empty/HTML bodies for most batches that failed `json.loads()` with `JSONDecodeError: Expecting value: line 1 column 1 (char 0)` — silently swallowed by the existing `except Exception: continue` and producing empty result sets. Diagnostic instrumentation confirmed every batch returned non-JSON after the first few. Webshare's 10-IP rotating residential proxy pool distributes batches across multiple egress points, avoiding the per-IP throttle. The fork already had `proxy_manager.py` plumbed in via `sentiment_service.py` for Reddit calls; this change extends the same pattern to the TradingView batch scans. Guards:

- `is_stock_exchange(exchange)` — only proxies NYSE/NASDAQ/AMEX/EGX where the rate limit actually hits. Crypto scans (~4 batches max) skip the proxy to preserve the 1 GB/month bandwidth budget.
- `is_proxy_configured()` — falls back to direct connection if Webshare env vars aren't set, so the code still works in dev/test environments.

Single-symbol calls (`analyze_coin`, `volume_confirmation_analyze`, `run_multi_timeframe_analysis`) and small bounded scans (`scan_consecutive_candles` capped at 200 symbols) skip the proxy by design — they don't trigger rate limits and proxying would waste bandwidth. Tested on NASDAQ 15m, NYSE 15m/1D for both `volume_breakout_scan` and `bollinger_scan`: all return full-alphabet result sets on first call after container warm-up. Webshare dashboard confirmed +120 requests / +10 MB bandwidth across the test session.

---

## [2026-05-19] — Fix `volume_ratio` / `average_volume` in Volume Tools

### `src/tradingview_mcp/core/services/scanner_service.py`

#### Change 8 — Request and read `average_volume_30d_calc` for volume_ratio (commits `c9fdf56`, `dc7818d`)

Two coordinated changes to make the volume_ratio calculation in `volume_breakout_scan` and `volume_confirmation_analyze` actually work:

1. Changed read key at both sites from `ind.get("volume.SMA20", 0)` to `ind.get("average_volume_30d_calc", 0)` (commit `c9fdf56`).

2. Added `additional_indicators=["average_volume_30d_calc"]` to both `get_multiple_analysis` call sites so the field is included in the response (commit `dc7818d`).

```python
# Before — volume_breakout_scan
analysis = get_multiple_analysis(
    screener=screener, interval=timeframe, symbols=batch, proxies=proxies
)
# ...
sma20_volume = ind.get("volume.SMA20", 0)

# After
analysis = get_multiple_analysis(
    screener=screener,
    interval=timeframe,
    symbols=batch,
    proxies=proxies,
    additional_indicators=["average_volume_30d_calc"],
)
# ...
sma20_volume = ind.get("average_volume_30d_calc", 0)
```

```python
# Before — volume_confirmation_analyze
analysis = get_multiple_analysis(screener=screener, interval=timeframe, symbols=[full_symbol])
# ...
sma20_volume = ind.get("volume.SMA20", 0)

# After
analysis = get_multiple_analysis(
    screener=screener,
    interval=timeframe,
    symbols=[full_symbol],
    additional_indicators=["average_volume_30d_calc"],
)
# ...
sma20_volume = ind.get("average_volume_30d_calc", 0)
```

**Why:** The dict key `volume.SMA20` is not a valid TradingView scanner column and is not in `tradingview_ta`'s documented indicator list ([pastebin 1DjWv2Hd](https://pastebin.com/1DjWv2Hd)). It was never present in `get_multiple_analysis` response dicts, so `ind.get("volume.SMA20", 0)` always returned `0`. Downstream effects:

- `volume_confirmation_analyze`: `volume_ratio` collapsed to the `else 1` branch, `average_volume` reported `0`, `signals[]` always empty.
- `volume_breakout_scan`: fell through to `avg_estimate = volume/2`, producing uniform `volume_ratio = 2.0` for every symbol — trivially clearing the default `volume_multiplier=2.0` filter and reducing ranking to `abs(changePercent)` alone.

`average_volume_30d_calc` is the correct field: present in TradingView's scanner field list (Average Volume, 30 day) and in `tradingview_ta`'s `additional_indicators` whitelist (pastebin line 63). It's not in the library's hardcoded default request set, so it must be explicitly requested via `additional_indicators` for the column to appear in the response.

**A/B verified** against `tradingview-mcp` (upstream control). Examples after deploy:

- `volume_confirmation_analysis(symbol="BINANCE:BTCUSDT", timeframe="1h")` — mod: `average_volume: 664.56`, `volume_ratio: 0.08`; upstream: `average_volume: 0`, `volume_ratio: 1`.
- `volume_confirmation_analysis(symbol="NASDAQ:AAPL", timeframe="1d")` — mod: `average_volume: 46.8M`, `volume_ratio: 0.74`; upstream: `average_volume: 0`, `volume_ratio: 1`.
- `volume_breakout_scanner(exchange="BINANCE", timeframe="15m", volume_multiplier=1.5, price_change_min=0.5)` — mod: 1 result with real `volume_ratio: 9.01`; upstream: 7 results all uniform `volume_ratio: 2.0`.

Confirmed working for both `crypto` and `america` screeners. Local variable name `sma20_volume` retained to keep diff minimal; semantically it now holds a 30-day calendar average, not a 20-bar SMA. Bug existed identically in upstream `atilaahmettaner/tradingview-mcp`.

---

## [2026-05-19] — Fix `volume_breakout_scan` Sort Key Using Capped Field

### `src/tradingview_mcp/core/services/scanner_service.py`

#### Change 9 — Sort by raw `volume_ratio` instead of capped `volume_strength` (commit `b37fdc8`)

Changed the final sort key in `volume_breakout_scan` from the display-capped `volume_strength` (min 10) to the underlying raw `volume_ratio`.

```python
# Before
volume_breakouts.sort(
    key=lambda x: (x["volume_strength"], abs(x["changePercent"])),
    reverse=True,
)

# After
volume_breakouts.sort(
    key=lambda x: (x["volume_ratio"], abs(x["changePercent"])),
    reverse=True,
)
```

**Why:** `volume_strength = min(10, volume_ratio)` is a display cap that mirrors the categorical 0-10 strength labels used in `volume_confirmation_analyze`. Using it as the sort key collapses every symbol with `volume_ratio >= 10` into a single priority tier, breaking ties on `abs(changePercent)` alone. A stock at 10.2x volume with a 5% move incorrectly outranks one at 47x volume with a 3% move — the opposite of what a "volume breakout scanner" should report.

The bug was previously invisible: before Change 8 (commits `c9fdf56` / `dc7818d`, same-day), the `volume_ratio` calculation itself was broken — the `ind.get("volume.SMA20", 0)` lookup always returned `0`, falling through to `avg_estimate = volume / 2` and producing uniform `volume_ratio = 2.0` for every symbol. With a constant first sort key, the sort effectively ran on `abs(changePercent)` alone, masking the capped-sort defect. Once Change 8 made `volume_ratio` carry real signal, the cap began actively distorting ranking on any symbol exceeding 10x.

Output schema unchanged: `volume_strength` remains in the returned dict, still 0-10. Only the internal ordering changes. Downstream consumers (`smart_volume_scan`, Memory.md, the HTML reference) require no updates. `smart_volume_scan` inherits the fix automatically since it calls `volume_breakout_scan` and only post-filters by RSI.

---

## [2026-05-20] — Route `multi_timeframe_analysis` Through Webshare Proxy

### `src/tradingview_mcp/core/services/screener_service.py`

#### Change 10 — Proxy the per-timeframe loop in `run_multi_timeframe_analysis` (commit `72b3a921`)

Added `proxies = _proxies_for(exchange)` before the timeframe loop and threaded `proxies=proxies` into each per-TF `get_multiple_analysis` call.

```python
# Before
    tf_results: dict = {}
    alignment_scores: list[int] = []

    for tf in timeframes:
        try:
            analysis = get_multiple_analysis(screener=screener, interval=tf, symbols=[symbol])
            if symbol not in analysis or analysis[symbol] is None:

# After
    tf_results: dict = {}
    alignment_scores: list[int] = []
    proxies = _proxies_for(exchange)

    for tf in timeframes:
        try:
            analysis = get_multiple_analysis(
                screener=screener, interval=tf, symbols=[symbol], proxies=proxies
            )
            if symbol not in analysis or analysis[symbol] is None:
```

**Why:** The function fires 5 sequential `get_multiple_analysis` calls (1W/1D/4h/1h/15m) for a single symbol. Although each call is single-symbol, the 5-request burst from one Cloud Run egress IP trips TradingView's per-IP rate limit on stock exchanges — the same pathology that Change 7 addressed for batch scans. The rate limit was intermittent but consistent enough to fail multi-timeframe calls on NASDAQ/NYSE during repeated queries in quick succession, returning `JSONDecodeError: Expecting value: line 1 column 1 (char 0)` on all 5 timeframes.

The `_proxies_for(exchange)` helper, already in place for Change 7, preserves the bandwidth budget by returning `None` for crypto exchanges (single symbols, no bulk scanning) and returning the Webshare pool only for stock exchanges (where the burst is long enough to trigger the limit). Webshare's rotating residential IPs distribute the 5 sequential calls across different egress points, avoiding the per-IP threshold.

**A/B verified** with upstream control (`tradingview-mcp` on `tradingview-mcp-61567401482.us-central1.run.app`):
- Upstream (unproxied): Calls 1–3 (NASDAQ:AAPL, NASDAQ:NVDA, NYSE:JPM) returned fully populated timeframes. Call 4 (NYSE:BAC) and Call 5 (NASDAQ:MSFT) failed: all 5 timeframes returned `"error": "Expecting value: line 1 column 1 (char 0)"` — rate limit signature.
- Mod (proxied): All 10 calls (original 5 + 5 fresh symbols: TSLA, AMD, GE, EGX:ETEL, and a bad EGX symbol) completed successfully with 5/5 populated timeframes each, except the bad symbol which correctly returned "No data" errors.

The fix uses the same `_proxies_for` guard and pattern established in Change 7; Change 7's documented bandwidth-budget concern applies equally here (Webshare $19/mo plan = 1 GB/month; each proxied multi-timeframe call ~40 KB, supporting ~25k stock multi-timeframe analyses per month within budget).

---

## [2026-05-23] — Gate Walk-Forward Verdict on OOS Trade Count + Adaptive Fold Widening

### `src/tradingview_mcp/core/services/backtest_service.py`

#### Change 11 — Statistical-significance gate + adaptive fold reduction in `walk_forward_backtest` (commit `5350008`)

Four coordinated edits so the robustness score is no longer treated as a hard pass/fail when it is computed from too few out-of-sample trades to be meaningful. The per-fold loop was extracted into a reusable `_run_wf_folds()` helper, and `walk_forward_backtest` was rebuilt around it.

1. Extended `_VALID_PERIODS` with `5y` and `10y`.

```python
# Before
_VALID_PERIODS   = {"1mo", "3mo", "6mo", "1y", "2y"}

# After
_VALID_PERIODS   = {"1mo", "3mo", "6mo", "1y", "2y", "5y", "10y"}
```

2. Lowered the `n_splits` floor from 2 to 1 (a single labeled holdout is now a valid last-resort fallback).

```python
# Before
    if not (2 <= n_splits <= 10):
        return {"error": "n_splits must be between 2 and 10"}

# After
    if not (1 <= n_splits <= 10):
        return {"error": "n_splits must be between 1 and 10"}
```

3. Added two parameters — `min_oos_trades: int = 8` and `auto_widen: bool = True` — and replaced the single-pass fold loop with an adaptive loop that retries at progressively fewer folds (larger test windows) until the out-of-sample trade count clears the gate or a single holdout is reached. The recompute reuses the already-fetched candles — no extra network calls.

```python
# Before (single inline pass)
    fn        = _STRATEGY_MAP[strategy]
    fold_size = len(candles) // n_splits
    folds, all_test_trades = [], []
    for fold_i in range(n_splits):
        ...   # fold built inline

# After (adaptive, via _run_wf_folds helper)
    fn = _STRATEGY_MAP[strategy]
    widen_trail: list[dict] = []
    used_splits = n_splits
    trial       = n_splits
    folds, all_test_trades = [], []
    while trial >= 1:
        f, t = _run_wf_folds(candles, fn, trial, train_ratio,
                             commission_pct, slippage_pct, initial_capital, interval)
        widen_trail.append({"n_splits": trial, "valid_folds": len(f), "oos_trades": len(t)})
        folds, all_test_trades, used_splits = f, t, trial
        if f and len(t) >= min_oos_trades:
            break
        if not auto_widen:
            break
        trial -= 1
```

4. Added a statistical-significance verdict tier. When total OOS trades fall below `min_oos_trades`, the verdict is `INSUFFICIENT DATA` and the score must not be used as a hard reject. The four robustness bands now sit behind this gate.

```python
# Before
    if avg_robust >= 0.8:
        verdict = "ROBUST — ..."
    elif avg_robust >= 0.5:
        verdict = "MODERATE — ..."
    elif avg_robust >= 0.2:
        verdict = "WEAK — ..."
    else:
        verdict = "OVERFITTED — ..."

# After
    data_sufficient = oos_trades >= min_oos_trades
    if not data_sufficient:
        verdict = ("INSUFFICIENT DATA — only N out-of-sample trade(s) ...; "
                   "Treat as UNTESTED: size down or pass. Do NOT hard-reject ...")
    elif avg_robust >= 0.8:
        verdict = "ROBUST — ..."
    elif avg_robust >= 0.5:
        verdict = "MODERATE — ..."
    elif avg_robust >= 0.2:
        verdict = "WEAK — ..."
    else:
        verdict = "OVERFITTED — ..."
```

New output fields: `effective_n_splits`, `min_oos_trades`, `auto_widen`, `data_sufficient`, `widen_trail`.

**Why:** The robustness score is `avg(per-fold test_return / train_return)`. On a 2y daily history split into 3 folds with a 70/30 train/test ratio, each test window is ~50 trading days. Slow signals (Bollinger mean-reversion, MACD crossover) fire 0–2 times in a window that short, so the score was driven by one or two trades — pure noise — and the degenerate branches (`train_return == 0` → `1.0` or `0.0`; a single losing OOS trade flooring the ratio at `−1.0`) let that noise masquerade as a confident verdict. Observed live: NSC Bollinger scored `−0.33` and RBA Bollinger `−0.03` across folds with 0–2 OOS trades, both hard-rejected at Stage 6 despite intact weekly structure — the gate was firing on sample-size artifacts, not genuine lack of edge. The fix separates "no edge" from "not enough data to tell": a genuine fail *with* an adequate trade count still returns OVERFITTED/WEAK, but a thin sample now returns INSUFFICIENT DATA (size down or pass). `auto_widen` and the new `5y`/`10y` periods are the levers for accumulating more OOS trades — a longer period is preferred because it adds trades while preserving fold count (cross-regime coverage), whereas collapsing folds trades that coverage away.

The per-fold ratio math in `_run_wf_folds` is intentionally unchanged — the gate sits in front of it. Backward-compatible: the two new parameters have safe defaults, so existing callers and the `server.py` tool surface are unaffected (exposing them per-call would be a separate edit). Merged to `main` (branch `fix/wf-insufficient-data-gate` deleted post-merge); takes effect on the live MCP server after Cloud Run redeploy. Not present in upstream `atilaahmettaner/tradingview-mcp`.

---

## Summary

| # | File | Commit | Change | Purpose |
|---|------|--------|--------|---------|
| 1 | `server.py` | `ccd149b` | `default="sse"` in argparse | Default to SSE transport for Cloud Run |
| 2 | `server.py` | `ccd149b` | `transport="sse"` in `mcp.run()` | Replace streamable-http with SSE |
| 3 | `server.py` | `4778237` | `host="0.0.0.0"` keyword arg in `FastMCP` | Bind to all interfaces for Cloud Run |
| 4 | `Dockerfile` | `748c8b4`, `928ecb2` | `sse`, `0.0.0.0`, port `8080` in `CMD` | Fix transport, host, and port for Cloud Run |
| 5 | `screener_service.py` | `6acc410` | Full-universe batching in `fetch_bollinger_analysis` | Fix alphabet bias on large exchanges (NYSE, NASDAQ) |
| 6 | `scanner_service.py` | `709e03d`, `ba295d5` | Full-universe batching + `batch_size=200` in `volume_breakout_scan` | Fix alphabet bias on large exchanges (NASDAQ, NYSE) |
| 7 | `scanner_service.py`, `screener_service.py` | `01c352d`, `08d537e` | Route stock-exchange batch scans through Webshare proxy | Avoid TradingView per-IP rate limiting on full-universe scans |
| 8 | `scanner_service.py` | `c9fdf56`, `dc7818d` | Request and read `average_volume_30d_calc` in volume tools | Fix broken `volume_ratio` / `average_volume` in `volume_confirmation_analysis`, `volume_breakout_scanner`, `smart_volume_scanner` |
| 9 | `scanner_service.py` | `b37fdc8` | Sort `volume_breakout_scan` by raw `volume_ratio` not capped `volume_strength` | Fix ranking distortion at the 10x ceiling on `volume_breakout_scanner` and `smart_volume_scanner` |
| 10 | `screener_service.py` | `72b3a921` | Route `multi_timeframe_analysis` per-TF calls through Webshare proxy | Avoid TradingView per-IP rate limiting on 5-timeframe burst calls |
| 11 | `backtest_service.py` | `5350008` | `min_oos_trades` gate + `auto_widen` adaptive folds + `5y`/`10y` periods in `walk_forward_backtest` | Stop hard-rejecting candidates on statistically meaningless robustness scores from thin OOS samples |

All other code — imports, tool handlers, resource routing, and business logic — is identical to upstream.
