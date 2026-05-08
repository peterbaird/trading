# Massive API Reference

Massive (formerly Polygon.io, rebranded October 30 2025) provides REST APIs for real-time and historical US stock market data. Existing Polygon.io API keys continue to work unchanged.

## Authentication

Every request requires an API key. The Python client handles authentication automatically:

```python
from massive import RESTClient
client = RESTClient(api_key="YOUR_API_KEY")
```

For raw HTTP, pass the key as a query parameter or header:

```
GET https://api.polygon.io/v2/snapshot/locale/us/markets/stocks/tickers?apiKey=YOUR_KEY
# or
Authorization: Bearer YOUR_API_KEY
```

## Base URL

```
https://api.polygon.io
```

The Python `massive` package targets this URL internally. The Massive.com branding uses `https://api.massive.com` for newer documentation URLs but both resolve correctly.

## Rate Limits

| Plan | Requests / minute | Recommended poll interval |
|------|-------------------|--------------------------|
| Free | 5 | 15 s |
| Starter ($29/mo) | unlimited | 5–10 s |
| Developer ($79/mo) | unlimited | 2–5 s |
| Advanced ($199/mo) | unlimited | 1–2 s |

HTTP 429 is returned when the rate limit is exceeded. The `Retry-After` response header indicates how long to wait.

## Python Client Installation

```bash
pip install massive        # or: uv add massive
```

Requires Python 3.9+. The `massive` package is the official client library (formerly `polygon-api-client`).

---

## Endpoints Relevant to Trading

### 1. Snapshot — All Tickers (v2)

Returns the latest trade, quote, and OHLCV aggregate for a filtered set of tickers in a single call. **This is the primary endpoint for polling multiple tickers.**

```
GET /v2/snapshot/locale/us/markets/stocks/tickers
```

**Query Parameters**

| Parameter | Type | Description |
|-----------|------|-------------|
| `tickers` | string | Comma-separated list of ticker symbols (e.g. `AAPL,MSFT,GOOGL`) |
| `apiKey` | string | Your API key (omit when using Python client) |

**Raw HTTP Example**

```
GET https://api.polygon.io/v2/snapshot/locale/us/markets/stocks/tickers?tickers=AAPL,MSFT,GOOGL&apiKey=YOUR_KEY
```

**Response Schema**

```json
{
  "status": "OK",
  "count": 3,
  "tickers": [
    {
      "ticker": "AAPL",
      "todaysChange": 1.23,
      "todaysChangePerc": 0.65,
      "updated": 1714500000123456789,
      "day": {
        "o": 190.00,
        "h": 192.50,
        "l": 189.50,
        "c": 191.23,
        "v": 45000000,
        "vw": 190.80
      },
      "min": {
        "av": 45000000,
        "t": 1714500000000,
        "o": 191.00,
        "h": 191.50,
        "l": 190.90,
        "c": 191.23,
        "v": 350000,
        "vw": 191.10,
        "n": 1200
      },
      "lastTrade": {
        "p": 191.23,
        "s": 100,
        "t": 1714500000456,
        "c": [14, 41],
        "x": 4
      },
      "lastQuote": {
        "P": 191.24,
        "p": 191.23,
        "S": 300,
        "s": 100,
        "t": 1714500000789
      },
      "prevDay": {
        "o": 188.50,
        "h": 191.00,
        "l": 188.00,
        "c": 190.00,
        "v": 38000000,
        "vw": 189.40
      }
    }
  ]
}
```

**Field Reference — `lastTrade`**

| Field | Type | Description |
|-------|------|-------------|
| `p` | float | Trade price |
| `s` | int | Trade size (shares) |
| `t` | int | Timestamp (Unix **milliseconds**) |
| `c` | array | Trade condition codes |
| `x` | int | Exchange ID |

**Field Reference — `lastQuote`**

| Field | Type | Description |
|-------|------|-------------|
| `P` | float | Ask price |
| `p` | float | Bid price |
| `S` | int | Ask size |
| `s` | int | Bid size |
| `t` | int | Timestamp (Unix milliseconds) |

**Field Reference — `day` / `prevDay` / `min`**

| Field | Type | Description |
|-------|------|-------------|
| `o` | float | Open |
| `h` | float | High |
| `l` | float | Low |
| `c` | float | Close (current for `day`, final for `prevDay`) |
| `v` | float | Volume |
| `vw` | float | VWAP (volume-weighted average price) |

**Python Client**

```python
from massive import RESTClient
from massive.rest.models import SnapshotMarketType

client = RESTClient(api_key="YOUR_KEY")

# Fetch snapshots for specific tickers
snapshots = client.get_snapshot_all(
    market_type=SnapshotMarketType.STOCKS,
    tickers=["AAPL", "MSFT", "GOOGL", "AMZN", "TSLA"],
)

for snap in snapshots:
    price = snap.last_trade.price
    timestamp_ms = snap.last_trade.timestamp   # milliseconds
    timestamp_s = timestamp_ms / 1000.0        # convert to seconds
    prev_close = snap.prev_day.close
    daily_change_pct = snap.todays_change_perc
    print(f"{snap.ticker}: ${price:.2f}  ({daily_change_pct:+.2f}%)")
```

> **Note:** The Python client uses `snake_case` attribute names that map to the abbreviated JSON fields. `lastTrade.p` → `last_trade.price`, `lastTrade.t` → `last_trade.timestamp`, `prevDay.c` → `prev_day.close`, `todaysChangePerc` → `todays_change_perc`.

---

### 2. Single Ticker Snapshot (v2)

Same data as the all-tickers endpoint but for one ticker. Useful for on-demand price checks.

```
GET /v2/snapshot/locale/us/markets/stocks/tickers/{stocksTicker}
```

**Response Schema** — `ticker` object (same fields as individual entries in the all-tickers response above).

**Python Client**

```python
snap = client.get_snapshot_ticker(
    market_type=SnapshotMarketType.STOCKS,
    ticker="AAPL",
)
print(f"AAPL: ${snap.last_trade.price:.2f}")
```

---

### 3. Unified Snapshot (v3)

A newer endpoint covering stocks, options, forex, and crypto in one call. Returns cleaner, more readable field names. Supports filtering up to 250 tickers via `ticker.any_of`.

```
GET /v3/snapshot
```

**Query Parameters**

| Parameter | Type | Max | Description |
|-----------|------|-----|-------------|
| `ticker.any_of` | string | 250 tickers | Comma-separated ticker list |
| `type` | string | — | Asset class filter: `stocks`, `options`, `fx`, `crypto`, `indices` |
| `limit` | int | 250 | Results per page (default: 10) |
| `sort` | string | — | Field to sort by |
| `order` | string | — | `asc` or `desc` |

**Response Schema**

```json
{
  "request_id": "abc123",
  "status": "OK",
  "results": [
    {
      "ticker": "AAPL",
      "name": "Apple Inc.",
      "type": "stocks",
      "market_status": "open",
      "last_trade": {
        "price": 191.23,
        "size": 100,
        "exchange": 4,
        "last_updated": 1714500000456000000
      },
      "last_quote": {
        "bid": 191.23,
        "ask": 191.24,
        "bid_size": 100,
        "ask_size": 300
      },
      "session": {
        "open": 190.00,
        "high": 192.50,
        "low": 189.50,
        "close": 191.23,
        "volume": 45000000,
        "change": 1.23,
        "change_percent": 0.65
      },
      "fmv": 191.20
    }
  ],
  "next_url": "https://api.polygon.io/v3/snapshot?cursor=..."
}
```

> `fmv` (fair market value) is only available on Business-tier plans.

**Python Client**

```python
results = list(client.list_snapshot_unified(
    ticker_any_of="AAPL,MSFT,GOOGL,AMZN,TSLA,NVDA,META,JPM,V,NFLX",
    type="stocks",
    limit=250,
))

for result in results:
    print(f"{result.ticker}: ${result.last_trade.price:.2f}")
```

---

### 4. Previous Day Bar (EOD)

Returns OHLCV data for the most recent completed trading day. Useful for initializing daily change % calculations at startup.

```
GET /v2/aggs/ticker/{stocksTicker}/prev
```

**Response Schema**

```json
{
  "status": "OK",
  "resultsCount": 1,
  "ticker": "AAPL",
  "results": [
    {
      "T": "AAPL",
      "o": 188.50,
      "h": 191.00,
      "l": 188.00,
      "c": 190.00,
      "v": 38000000,
      "vw": 189.40,
      "t": 1714435200000,
      "n": 450000
    }
  ]
}
```

**Python Client**

```python
prev = client.get_previous_close_agg(ticker="AAPL")
if prev and prev.results:
    yesterday_close = prev.results[0].close
    print(f"AAPL previous close: ${yesterday_close:.2f}")
```

---

### 5. Last Trade

Returns the most recent individual trade for a ticker.

```
GET /v2/last/trade/{stocksTicker}
```

**Response Schema**

```json
{
  "status": "OK",
  "results": {
    "T": "AAPL",
    "p": 191.23,
    "s": 100,
    "t": 1714500000456789123,
    "x": 4,
    "c": [14, 41]
  }
}
```

**Python Client**

```python
trade = client.get_last_trade(ticker="AAPL")
print(f"Last AAPL trade: ${trade.price:.2f} @ {trade.timestamp}")
```

---

### 6. Aggregates (OHLCV Bars)

Returns historical OHLCV data over a date range and time resolution. Useful for seeding charts with historical data.

```
GET /v2/aggs/ticker/{stocksTicker}/range/{multiplier}/{timespan}/{from}/{to}
```

**Path Parameters**

| Parameter | Example | Description |
|-----------|---------|-------------|
| `stocksTicker` | `AAPL` | Ticker symbol |
| `multiplier` | `1` | Size of the time window |
| `timespan` | `minute`, `hour`, `day` | Time unit |
| `from` | `2026-04-01` | Start date (YYYY-MM-DD or Unix ms) |
| `to` | `2026-05-01` | End date |

**Python Client**

```python
bars = []
for bar in client.list_aggs(
    ticker="AAPL",
    multiplier=1,
    timespan="day",
    from_="2026-01-01",
    to="2026-05-01",
    limit=50000,
):
    bars.append({
        "open": bar.open,
        "high": bar.high,
        "low": bar.low,
        "close": bar.close,
        "volume": bar.volume,
        "vwap": bar.vwap,
        "timestamp": bar.timestamp,  # Unix milliseconds
    })
```

---

## Polling Multiple Tickers — Recommended Pattern

For Trading's polling loop (fetching prices for the watchlist periodically), use the v2 all-tickers snapshot endpoint via `get_snapshot_all()`. A single API call covers all watched tickers.

```python
import asyncio
from massive import RESTClient
from massive.rest.models import SnapshotMarketType

class MassivePoller:
    def __init__(self, api_key: str, tickers: list[str], interval: float = 15.0):
        self._client = RESTClient(api_key=api_key)
        self._tickers = tickers
        self._interval = interval

    async def poll_loop(self):
        while True:
            try:
                # RESTClient is synchronous — offload to thread pool
                snapshots = await asyncio.to_thread(
                    self._client.get_snapshot_all,
                    market_type=SnapshotMarketType.STOCKS,
                    tickers=self._tickers,
                )
                for snap in snapshots:
                    price = snap.last_trade.price
                    ts = snap.last_trade.timestamp / 1000.0  # ms → s
                    print(f"{snap.ticker}: ${price:.2f}")
            except Exception as e:
                print(f"Poll error: {e}")
            await asyncio.sleep(self._interval)
```

---

## Error Handling

| HTTP Status | Meaning | Action |
|-------------|---------|--------|
| 200 | Success | Parse `tickers` / `results` array |
| 403 | Invalid API key | Check `MASSIVE_API_KEY` env var |
| 404 | Ticker not found | Log and skip; ticker may be delisted |
| 429 | Rate limit exceeded | Back off; check `Retry-After` header |
| 500/503 | Server error | Retry with exponential backoff |

The Python client raises exceptions for 4xx/5xx responses. Wrap calls in `try/except` and log errors without crashing the poll loop.

```python
try:
    snapshots = client.get_snapshot_all(
        market_type=SnapshotMarketType.STOCKS,
        tickers=tickers,
    )
except Exception as e:
    logger.error("Massive poll failed: %s", e)
    # Return stale cache data until next successful poll
```

---

## Market Hours

The Massive snapshot endpoints return data 24/7, but `lastTrade` only updates during:
- Pre-market: 4:00 AM – 9:30 AM ET
- Regular session: 9:30 AM – 4:00 PM ET
- After-hours: 4:00 PM – 8:00 PM ET

Outside these windows, `lastTrade.t` timestamps will be stale. `market_status` in the v3 snapshot indicates the current session state (`open`, `closed`, `early_trading`, `late_trading`).

---

## Related Links

- Official docs: https://massive.com/docs
- REST quickstart: https://massive.com/docs/rest/quickstart
- Stocks overview: https://massive.com/docs/rest/stocks/overview
- Python client (GitHub): https://github.com/massive-com/client-python
- Pricing: https://massive.com/pricing
