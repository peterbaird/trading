# Market Data Interface

## Design Philosophy

The market data subsystem uses the **Strategy pattern**: all downstream code (SSE streaming, portfolio valuation, trade execution) reads from a single `PriceCache` and is completely agnostic to whether prices come from the Massive API or the built-in simulator. Swapping data sources is a startup-time decision driven by an environment variable.

```
MarketDataSource (ABC)
├── SimulatorDataSource   ← default (no API key needed)
└── MassiveDataSource     ← when MASSIVE_API_KEY is set
        │
        ▼
   PriceCache (thread-safe, in-memory)
        │
        ├──→ SSE /api/stream/prices
        ├──→ Portfolio valuation (GET /api/portfolio)
        └──→ Trade execution (POST /api/portfolio/trade)
```

---

## Module Location

```
backend/app/market/
├── __init__.py        # Public re-exports
├── models.py          # PriceUpdate dataclass
├── cache.py           # PriceCache
├── interface.py       # MarketDataSource ABC
├── simulator.py       # GBMSimulator + SimulatorDataSource
├── massive_client.py  # MassiveDataSource (Polygon.io REST poller)
├── factory.py         # create_market_data_source()
├── seed_prices.py     # Seed prices and per-ticker GBM params
└── stream.py          # FastAPI SSE router factory
```

---

## Core Types

### PriceUpdate

An immutable frozen dataclass representing one ticker's price at a point in time.

```python
@dataclass(frozen=True, slots=True)
class PriceUpdate:
    ticker: str
    price: float
    previous_price: float
    timestamp: float          # Unix seconds

    # Computed properties (no storage):
    change: float             # price - previous_price
    change_percent: float     # % change
    direction: str            # "up", "down", or "flat"

    def to_dict(self) -> dict:  # for JSON / SSE serialization
```

**Example**

```python
update = PriceUpdate(
    ticker="AAPL",
    price=191.50,
    previous_price=191.20,
    timestamp=1714500000.456,
)
update.direction       # "up"
update.change          # 0.3
update.change_percent  # 0.157
update.to_dict()
# {
#   "ticker": "AAPL",
#   "price": 191.5,
#   "previous_price": 191.2,
#   "timestamp": 1714500000.456,
#   "change": 0.3,
#   "change_percent": 0.157,
#   "direction": "up"
# }
```

---

### PriceCache

Thread-safe in-memory store. One writer (the active `MarketDataSource`), multiple concurrent readers.

```python
class PriceCache:
    def update(self, ticker: str, price: float, timestamp: float | None = None) -> PriceUpdate
    def get(self, ticker: str) -> PriceUpdate | None
    def get_price(self, ticker: str) -> float | None
    def get_all(self) -> dict[str, PriceUpdate]
    def remove(self, ticker: str) -> None
    version: int   # monotonically increasing; bumped on every update
```

**`version`** is used by the SSE endpoint for change detection: the endpoint sleeps when `cache.version` hasn't changed since the last push, avoiding redundant events.

**Example**

```python
cache = PriceCache()

# Writer (called by the data source background task):
update = cache.update("AAPL", 191.50)

# Reader (called by API endpoints):
update = cache.get("AAPL")          # PriceUpdate or None
price = cache.get_price("AAPL")     # float or None
all_prices = cache.get_all()        # dict[str, PriceUpdate]
```

---

### MarketDataSource (ABC)

The abstract contract that both implementations fulfill.

```python
class MarketDataSource(ABC):
    async def start(self, tickers: list[str]) -> None: ...
    async def stop(self) -> None: ...
    async def add_ticker(self, ticker: str) -> None: ...
    async def remove_ticker(self, ticker: str) -> None: ...
    def get_tickers(self) -> list[str]: ...
```

**Contract rules:**

- `start()` must be called exactly once before any other method
- `stop()` is safe to call multiple times; after it returns the source will not write to the cache again
- `add_ticker()` / `remove_ticker()` may be called concurrently with the background polling loop; implementations must be safe
- `remove_ticker()` must also remove the ticker from the `PriceCache`

---

## Implementations

### SimulatorDataSource

Backed by `GBMSimulator`. Runs as an asyncio background task that calls `GBMSimulator.step()` every 500 ms and writes results to the cache. No external dependencies.

Key behaviors:
- Seeds the cache immediately on `start()` so SSE has data before the first tick
- `add_ticker()` seeds the cache immediately so the new ticker has a price right away (no wait for next tick)
- GBM math produces sub-cent moves per tick that accumulate naturally into realistic-looking drift
- Correlated moves across tickers (tech stocks move together, finance stocks move together)
- ~0.1% chance per tick per ticker of a sudden 2–5% shock event for visual drama

See `MARKET_SIMULATOR.md` for implementation details.

### MassiveDataSource

Backed by the Massive (Polygon.io) REST API. Runs as an asyncio task that polls `GET /v2/snapshot/locale/us/markets/stocks/tickers` for all watched tickers in a single API call, then writes results to the cache.

Key behaviors:
- Does an immediate poll on `start()` so the cache has real data before the first SSE event
- `add_ticker()` is additive — the new ticker appears on the next poll cycle (up to 15 s on the free tier)
- The synchronous `RESTClient` is called via `asyncio.to_thread()` to avoid blocking the event loop
- Errors are logged but not re-raised; the loop retries on the next interval

See `MASSIVE.md` for API endpoint details.

---

## Factory

`create_market_data_source()` selects the implementation based on the environment:

```python
from app.market import PriceCache, create_market_data_source

cache = PriceCache()
source = create_market_data_source(cache)
# → MassiveDataSource if os.environ["MASSIVE_API_KEY"] is set and non-empty
# → SimulatorDataSource otherwise
```

`factory.py`:

```python
def create_market_data_source(price_cache: PriceCache) -> MarketDataSource:
    api_key = os.environ.get("MASSIVE_API_KEY", "").strip()
    if api_key:
        return MassiveDataSource(api_key=api_key, price_cache=price_cache)
    else:
        return SimulatorDataSource(price_cache=price_cache)
```

---

## Lifecycle

### Application Startup

```python
from app.market import PriceCache, create_market_data_source

# 1. Create the cache (shared by all readers)
cache = PriceCache()

# 2. Create and start the source (begins pushing to cache)
source = create_market_data_source(cache)
await source.start(["AAPL", "GOOGL", "MSFT", "AMZN", "TSLA",
                    "NVDA", "META", "JPM", "V", "NFLX"])
```

### Runtime — Watchlist Changes

```python
# User adds a ticker via POST /api/watchlist
await source.add_ticker("PYPL")

# User removes a ticker via DELETE /api/watchlist/NFLX
await source.remove_ticker("NFLX")
# cache.get("NFLX") returns None from this point on
```

### Reading Prices

```python
# Single ticker (e.g., for trade execution)
update = cache.get("AAPL")
if update is None:
    raise HTTPException(422, "Price not available for AAPL")
price = update.price

# All tickers (e.g., for portfolio valuation)
all_prices = cache.get_all()
for ticker, update in all_prices.items():
    ...

# Just the float (e.g., quick check)
price = cache.get_price("AAPL")   # float | None
```

### Application Shutdown

```python
await source.stop()
```

---

## SSE Streaming

`stream.py` provides a factory that returns a FastAPI `APIRouter` wired to the cache:

```python
from app.market import create_stream_router

router = create_stream_router(cache)
# Mounts: GET /api/stream/prices  (text/event-stream)
```

The SSE endpoint uses `cache.version` to detect changes between polls (sleeping 50 ms between checks). When the version increments, it pushes all current prices as a batch of `data:` events — one per ticker.

Each SSE event payload:

```json
{
  "ticker": "AAPL",
  "price": 191.50,
  "previous_price": 191.20,
  "timestamp": 1714500000.456,
  "change": 0.3,
  "change_percent": 0.157,
  "direction": "up"
}
```

The frontend connects via the native `EventSource` API, which handles reconnection automatically.

---

## Public Imports

All consumer code imports from the package root — never from sub-modules directly:

```python
from app.market import (
    PriceUpdate,
    PriceCache,
    MarketDataSource,
    create_market_data_source,
    create_stream_router,
)
```

---

## Daily Change %

The PLAN specifies daily change % as `(current_price − seed_price) / seed_price × 100`, where `seed_price` is the GBM reference price at process startup. The backend includes `seed_price` in the `GET /api/watchlist` response so the frontend can compute the percentage.

In simulator mode, `seed_price` = the value from `seed_prices.py` at startup.

In Massive mode, `seed_price` is captured from the first successful poll for each ticker (the price returned by `lastTrade.p` on startup).

The `PriceCache` does not store `seed_price`; it is managed by the data source and included in the watchlist API response separately.

---

## Comparison: Simulator vs. Massive

| Aspect | SimulatorDataSource | MassiveDataSource |
|--------|---------------------|-------------------|
| Data source | GBM math (in-process) | Polygon.io REST API |
| Update frequency | 500 ms | 2–15 s (tier dependent) |
| Requires API key | No | Yes (`MASSIVE_API_KEY`) |
| Supports any ticker | Yes (uses default params) | Yes (price may be `null` until next poll) |
| Outside market hours | Always active | Stale prices (no new trades) |
| Cost | Free | $0–$199+/mo |
| Determinism | Stochastic (random seed) | Real market movement |
