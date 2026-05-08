# Market Simulator

The built-in market simulator generates realistic-looking stock price movements without requiring an external API key. It runs as an in-process asyncio background task and implements the same `MarketDataSource` interface as the Massive client, so all downstream code is agnostic to the source.

---

## Overview

The simulator is split into two classes:

- **`GBMSimulator`** — pure math engine. Advances all tracked tickers by one time step using Geometric Brownian Motion with correlated moves. Has no asyncio concerns.
- **`SimulatorDataSource`** — wraps `GBMSimulator` and implements the `MarketDataSource` ABC. Runs the GBM loop as an asyncio task, writes results to `PriceCache`.

---

## Mathematical Model

### Geometric Brownian Motion (GBM)

GBM is the standard model for equity price simulation. Each tick advances every price by:

```
S(t+dt) = S(t) · exp((μ − σ²/2) · dt + σ · √dt · Z)
```

Where:
- `S(t)` — current price
- `μ` (mu) — annualized drift (expected return)
- `σ` (sigma) — annualized volatility
- `dt` — time step as a fraction of a trading year
- `Z` — standard normal random variable (correlated across tickers)

### Time Step (dt)

The simulator ticks every 500 ms. Expressed as a fraction of a trading year:

```
TRADING_SECONDS_PER_YEAR = 252 days × 6.5 hours/day × 3600 s/hour = 5,896,800 s
dt = 0.5 / 5,896,800 ≈ 8.48 × 10⁻⁸
```

This tiny `dt` produces sub-cent moves per tick that accumulate naturally over time — realistic drift without jarring jumps on every update.

### Why GBM

GBM is a natural choice for a simulator because:
- It preserves the lognormal property of prices (prices can never go negative)
- It produces continuous paths that look like real intraday price charts
- The two parameters (μ, σ) are intuitive and independently controllable per ticker
- It's the model underlying Black-Scholes, so students learning finance will recognize it

### Drift vs. Volatility Intuition

For a 500 ms tick:
- **Drift contribution**: `(μ − σ²/2) · dt` — at μ=0.05, σ=0.25, this is ~1.7 × 10⁻⁹ per tick (negligible per tick, meaningful over hours)
- **Diffusion contribution**: `σ · √dt · Z` — at σ=0.25, `√dt ≈ 2.91 × 10⁻⁴`, so a 1-sigma move is 0.025% per tick (~$0.05 on a $200 stock)

---

## Seed Prices and Per-Ticker Parameters

Defined in `seed_prices.py`. These are the GBM starting prices and annualized parameters for the 10 default watchlist tickers:

```python
SEED_PRICES = {
    "AAPL": 190.00,
    "GOOGL": 175.00,
    "MSFT": 420.00,
    "AMZN": 185.00,
    "TSLA": 250.00,
    "NVDA": 800.00,
    "META": 500.00,
    "JPM": 195.00,
    "V":    280.00,
    "NFLX": 600.00,
}

TICKER_PARAMS = {
    "AAPL":  {"sigma": 0.22, "mu": 0.05},
    "GOOGL": {"sigma": 0.25, "mu": 0.05},
    "MSFT":  {"sigma": 0.20, "mu": 0.05},
    "AMZN":  {"sigma": 0.28, "mu": 0.05},
    "TSLA":  {"sigma": 0.50, "mu": 0.03},  # volatile, lower drift
    "NVDA":  {"sigma": 0.40, "mu": 0.08},  # volatile, strong drift
    "META":  {"sigma": 0.30, "mu": 0.05},
    "JPM":   {"sigma": 0.18, "mu": 0.04},  # low vol (bank)
    "V":     {"sigma": 0.17, "mu": 0.04},  # low vol (payments)
    "NFLX":  {"sigma": 0.35, "mu": 0.05},
}

DEFAULT_PARAMS = {"sigma": 0.25, "mu": 0.05}  # for dynamically added tickers
```

**Volatility guide:**
- `sigma < 0.20` — stable, slow-moving (JPM, V)
- `sigma 0.20–0.30` — typical large-cap tech (AAPL, MSFT)
- `sigma 0.30–0.40` — high-growth tech (META, AMZN, NVDA)
- `sigma > 0.40` — high volatility (TSLA, NVDA at 0.40–0.50)

---

## Correlation Model

Real stocks don't move independently — tech stocks tend to rise and fall together, as do financials. The simulator models this using a **Cholesky decomposition** of a sector-based correlation matrix.

### Correlation Groups

```python
CORRELATION_GROUPS = {
    "tech":    {"AAPL", "GOOGL", "MSFT", "AMZN", "META", "NVDA", "NFLX"},
    "finance": {"JPM", "V"},
}

INTRA_TECH_CORR    = 0.6   # Tech stocks move together
INTRA_FINANCE_CORR = 0.5   # Finance stocks move together
CROSS_GROUP_CORR   = 0.3   # Between sectors / unknown tickers
TSLA_CORR          = 0.3   # TSLA does its own thing (despite being in tech)
```

### Pairwise Correlation Logic

```python
def _pairwise_correlation(t1: str, t2: str) -> float:
    if t1 == "TSLA" or t2 == "TSLA":
        return TSLA_CORR          # 0.3

    tech = CORRELATION_GROUPS["tech"]
    finance = CORRELATION_GROUPS["finance"]

    if t1 in tech and t2 in tech:
        return INTRA_TECH_CORR    # 0.6
    if t1 in finance and t2 in finance:
        return INTRA_FINANCE_CORR # 0.5

    return CROSS_GROUP_CORR       # 0.3
```

Dynamically added tickers (not in any group) get 0.3 correlation with everyone.

### Cholesky Decomposition

To generate correlated normal draws, the simulator:

1. Builds an `n×n` correlation matrix `C` from `_pairwise_correlation()`
2. Computes `L = cholesky(C)` (lower triangular)
3. Each tick: draws `n` independent standard normals `z`, computes `Lz` to get correlated normals

```python
# Build correlation matrix
n = len(tickers)
C = np.eye(n)
for i in range(n):
    for j in range(i+1, n):
        rho = _pairwise_correlation(tickers[i], tickers[j])
        C[i, j] = C[j, i] = rho

# Cholesky factor (rebuilt whenever tickers are added/removed)
L = np.linalg.cholesky(C)

# Each tick:
z_independent = np.random.standard_normal(n)
z_correlated = L @ z_independent
```

The Cholesky matrix is rebuilt whenever tickers are added or removed. At `n < 50` tickers this is fast enough to do synchronously.

---

## Random Shock Events

Beyond normal GBM diffusion, the simulator introduces occasional sudden large moves:

```python
EVENT_PROBABILITY = 0.001  # ~0.1% chance per tick per ticker

if random.random() < EVENT_PROBABILITY:
    shock_magnitude = random.uniform(0.02, 0.05)   # 2–5%
    shock_sign = random.choice([-1, 1])
    price *= 1 + shock_magnitude * shock_sign
```

With 10 tickers at 2 ticks/second, expect roughly one shock event every ~50 seconds. These create the kind of sudden moves that make a trading terminal feel alive.

---

## GBMSimulator Class

Lives in `simulator.py`. Pure math engine with no asyncio.

```python
class GBMSimulator:
    TRADING_SECONDS_PER_YEAR = 252 * 6.5 * 3600   # 5,896,800
    DEFAULT_DT = 0.5 / TRADING_SECONDS_PER_YEAR   # ~8.48e-8

    def __init__(
        self,
        tickers: list[str],
        dt: float = DEFAULT_DT,
        event_probability: float = 0.001,
    ) -> None: ...

    def step(self) -> dict[str, float]:
        """Advance all tickers by one dt. Returns {ticker: new_price}.
        Called every 500 ms — this is the hot path."""

    def add_ticker(self, ticker: str) -> None:
        """Add a ticker. Uses SEED_PRICES or random $50–$300. Rebuilds Cholesky."""

    def remove_ticker(self, ticker: str) -> None:
        """Remove a ticker. Rebuilds Cholesky."""

    def get_price(self, ticker: str) -> float | None:
        """Current simulated price, or None."""

    def get_tickers(self) -> list[str]:
        """Currently tracked tickers."""
```

### step() in detail

```python
def step(self) -> dict[str, float]:
    n = len(self._tickers)
    if n == 0:
        return {}

    # Draw correlated normals
    z_independent = np.random.standard_normal(n)
    z_correlated = self._cholesky @ z_independent if self._cholesky else z_independent

    result = {}
    for i, ticker in enumerate(self._tickers):
        mu, sigma = self._params[ticker]["mu"], self._params[ticker]["sigma"]

        # GBM update
        drift = (mu - 0.5 * sigma**2) * self._dt
        diffusion = sigma * math.sqrt(self._dt) * z_correlated[i]
        self._prices[ticker] *= math.exp(drift + diffusion)

        # Shock event
        if random.random() < self._event_prob:
            shock = random.uniform(0.02, 0.05) * random.choice([-1, 1])
            self._prices[ticker] *= 1 + shock

        result[ticker] = round(self._prices[ticker], 2)

    return result
```

---

## SimulatorDataSource Class

Wraps `GBMSimulator` and implements `MarketDataSource`. Manages the asyncio event loop integration.

```python
class SimulatorDataSource(MarketDataSource):
    def __init__(
        self,
        price_cache: PriceCache,
        update_interval: float = 0.5,        # seconds between ticks
        event_probability: float = 0.001,
    ) -> None: ...
```

### Lifecycle

```python
async def start(self, tickers: list[str]) -> None:
    # 1. Create the GBM simulator with starting tickers
    self._sim = GBMSimulator(tickers=tickers, event_probability=self._event_prob)

    # 2. Seed the cache immediately — SSE has data before the first tick
    for ticker in tickers:
        price = self._sim.get_price(ticker)
        if price is not None:
            self._cache.update(ticker=ticker, price=price)

    # 3. Start the background loop
    self._task = asyncio.create_task(self._run_loop(), name="simulator-loop")

async def stop(self) -> None:
    if self._task and not self._task.done():
        self._task.cancel()
        try:
            await self._task
        except asyncio.CancelledError:
            pass

async def add_ticker(self, ticker: str) -> None:
    if self._sim:
        self._sim.add_ticker(ticker)
        # Seed cache immediately — no wait for next tick
        price = self._sim.get_price(ticker)
        if price is not None:
            self._cache.update(ticker=ticker, price=price)

async def remove_ticker(self, ticker: str) -> None:
    if self._sim:
        self._sim.remove_ticker(ticker)
    self._cache.remove(ticker)
```

### Core Loop

```python
async def _run_loop(self) -> None:
    while True:
        try:
            if self._sim:
                prices = self._sim.step()
                for ticker, price in prices.items():
                    self._cache.update(ticker=ticker, price=price)
        except Exception:
            logger.exception("Simulator step failed")
        await asyncio.sleep(self._interval)
```

The `step()` call is synchronous and fast (microseconds for ≤50 tickers), so it runs directly on the event loop without a thread pool. The `asyncio.sleep()` yields control to other coroutines between ticks.

---

## Configuration

All configurable parameters have sensible defaults. Pass overrides when constructing `SimulatorDataSource` (or indirectly through `create_market_data_source()`):

| Parameter | Default | Description |
|-----------|---------|-------------|
| `update_interval` | `0.5` s | Seconds between GBM ticks |
| `event_probability` | `0.001` | Per-tick probability of a shock event |
| `dt` | `~8.48e-8` | Time step (fraction of trading year per tick) |

For testing, pass `update_interval=0.0` and call `step()` directly on `GBMSimulator` to advance time deterministically.

---

## Tuning the Simulator

### Increase volatility for drama
Set higher `sigma` values in `TICKER_PARAMS` or `DEFAULT_PARAMS`:
```python
# Makes TSLA move ±8% per day on average instead of ±5%
TICKER_PARAMS["TSLA"]["sigma"] = 0.80
```

### Add a new sector correlation group
```python
CORRELATION_GROUPS["energy"] = {"XOM", "CVX", "COP"}
INTRA_ENERGY_CORR = 0.55

# Then update _pairwise_correlation to check for this group
```

### Faster updates (no API key needed)
```python
# 200 ms ticks instead of 500 ms
source = SimulatorDataSource(price_cache=cache, update_interval=0.2)
```

### Disable shock events entirely
```python
source = SimulatorDataSource(price_cache=cache, event_probability=0.0)
```

---

## Adding New Tickers (Runtime)

When a user adds a ticker not in `SEED_PRICES`, the simulator assigns:
- **Initial price**: random uniform between $50 and $300
- **Parameters**: `DEFAULT_PARAMS = {"sigma": 0.25, "mu": 0.05}`
- **Correlation**: 0.3 with all existing tickers (cross-sector)

The price is seeded into the cache immediately via `SimulatorDataSource.add_ticker()`, so the ticker has a price for the SSE stream before the next tick.

---

## Testing

The GBMSimulator is deterministic when seeded with `numpy.random.seed()`:

```python
import numpy as np
from app.market.simulator import GBMSimulator

np.random.seed(42)
sim = GBMSimulator(tickers=["AAPL", "MSFT"])

prices_t1 = sim.step()  # deterministic
prices_t2 = sim.step()  # deterministic

assert prices_t1["AAPL"] != prices_t2["AAPL"]  # prices move
assert prices_t1["AAPL"] > 0                    # never negative
```

For `SimulatorDataSource` integration tests, use `update_interval=0.0` and advance time manually to avoid real-time waits.
