# stochastic/ — Synthetic Price Generation

Mathematical models for generating synthetic OHLCV data for backtesting without real market data.

## Available Models

| Model                      | File                              | Behavior                            | Key Use                         |
| -------------------------- | --------------------------------- | ----------------------------------- | ------------------------------- |
| Geometric Brownian Motion  | `processes/gbm.py`                | Log-normal random walk              | Standard price simulation       |
| Brownian Motion            | `processes/brownian_motion.py`    | Pure Wiener process                 | Foundation for GBM              |
| Heston                     | `processes/heston.py`             | GBM + stochastic volatility (CIR)   | Volatility clustering           |
| Merton Jump-Diffusion      | `processes/merton.py`             | GBM + Poisson jumps                 | Market shocks/crashes           |
| Cox-Ingersoll-Ross         | `processes/cox.py`                | Mean-reverting, level-dependent vol | Interest rates                  |
| Ornstein-Uhlenbeck         | `processes/ornstein_uhlenbeck.py` | Mean-reverting                      | Spread/pairs trading            |
| Fractional Brownian Motion | `processes/fbm.py`                | Memory via Hurst exponent           | Trending/mean-reverting regimes |

## Usage

All models share the same interface, returning a DataFrame with OHLCV columns:

```python
from tensortrade.stochastic.processes.gbm import gbm

df = gbm(base_price=100, base_volume=50, start_date="2020-01-01",
         times_to_generate=1000, time_frame="1h")
```

## Configuration

`ModelParameters` (`utils/parameters.py`) centralizes all process parameters (18 total). Helpers:

- `default(base_price, t_gen, delta)` — sensible defaults (mu=0.058, sigma=0.125)
- `random(base_price, t_gen, delta)` — randomized for robustness testing

`utils/helpers.py`: `get_delta(time_frame)` converts timeframe strings to fractions of a year (e.g., "1h" → 1/(252x24)).

## Note

Heston model has a known issue: code comment says "This method is dodgy! Need to debug!" in the correlated path construction.
