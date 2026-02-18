# CLAUDE.md

This file provides guidance to Claude Code (claude.ai/code) when working with code in this repository.

## Build & Test Commands

```bash
# Install (Python 3.12+ required)
pip install -r requirements.txt && pip install -e .

# Run all tests (skip heavy RLlib tests)
pytest tests/ -m "not rllib"

# Unit tests only
pytest tests/tensortrade/unit -v

# Single test file
pytest tests/tensortrade/unit/oms/wallets/test_portfolio.py -v

# Parallel tests
pytest --workers auto tests/

# Doctests
pytest --doctest-modules tensortrade/

# Build docs / package
make docs-build
make package
```

Test markers: `slow` (long-running), `rllib` (Ray/RLlib integration, skipped in CI).

## Architecture Overview

TensorTrade is an RL framework for training algorithmic trading agents, built on Gymnasium. Data flows through composable streams into a trading environment where pluggable components handle actions, rewards, observations, and order execution.

```
Data (CryptoDataDownload / stochastic) → Stream pipeline (feed/)
    → TradingEnv (env/) ←→ OMS (oms/: orders, wallets, exchanges)
        → Agent (agents/ or Ray/RLlib)
```

### Core Design Pattern

All major pieces inherit from `Component` (in `core/`), which uses a metaclass (`InitContextMeta`) to inject configuration from a `TradingContext` at instantiation time. Each subclass declares a `registered_name` used for config lookup:

```python
with TradingContext({"actions": {"param": value}}):
    scheme = BSH(cash, asset)  # BSH.registered_name = "actions" → receives config
```

### TradingEnv Composition

`TradingEnv` (Gymnasium.Env) composes six pluggable components:

| Component      | Role                  | Default implementations                      |
| -------------- | --------------------- | -------------------------------------------- |
| `ActionScheme` | Agent output → Orders | `BSH`, `SimpleOrders`, `ManagedRiskOrders`   |
| `RewardScheme` | Learning signal       | `SimpleProfit`, `RiskAdjustedReturns`, `PBR` |
| `Observer`     | Observations          | Windowed OHLCV feature observer              |
| `Stopper`      | Episode termination   | `MaxLossStopper`                             |
| `Informer`     | Metadata              | `TensorTradeInformer`                        |
| `Renderer`     | Visualization         | Plotly, Matplotlib, file/screen loggers      |

Quick start: `env = default.create(portfolio=..., action_scheme='bsh', reward_scheme='pbr', feed=..., window_size=5)`

## Module Guide (Link Farm)

Each directory has a nested `CLAUDE.md` with full details — pulled in automatically when working in that directory.

| Directory                                                     | Purpose                       | Key Details                                                                                      |
| ------------------------------------------------------------- | ----------------------------- | ------------------------------------------------------------------------------------------------ |
| [`tensortrade/core/`](tensortrade/core/CLAUDE.md)             | Component system & foundation | `InitContextMeta` metaclass, `TradingContext` injection, `Clock`, base mixins, exceptions        |
| [`tensortrade/env/`](tensortrade/env/CLAUDE.md)               | Trading environments          | `generic/` abstracts vs `default/` implementations, step/reset cycle, `default.create()` factory |
| [`tensortrade/feed/`](tensortrade/feed/CLAUDE.md)             | Stream data pipeline          | DAG of `Stream` nodes, `DataFeed` orchestration, rolling/ewm/expanding windows, `NameSpace`      |
| [`tensortrade/oms/`](tensortrade/oms/CLAUDE.md)               | Order management system       | Instruments, wallets (free/locked), order lifecycle, broker, exchange, simulated execution       |
| [`tensortrade/agents/`](tensortrade/agents/CLAUDE.md)         | RL agents (deprecated)        | DQN, A2C, parallel training; prefer Ray/RLlib for new work                                       |
| [`tensortrade/data/`](tensortrade/data/CLAUDE.md)             | Data fetching                 | `CryptoDataDownload` for OHLCV from exchanges                                                    |
| [`tensortrade/stochastic/`](tensortrade/stochastic/CLAUDE.md) | Synthetic price generation    | GBM, Heston, Merton, OU, CIR, FBM models                                                         |
| [`tests/`](tests/CLAUDE.md)                                   | Test suite                    | Unit + integration tests, markers, pytest patterns                                               |
| [`examples/`](examples/CLAUDE.md)                             | Training scripts & notebooks  | End-to-end patterns with Ray/RLlib, Optuna, walk-forward                                         |

## Key Constraints

- **Python 3.12+** enforced in both `setup.py` and `tensortrade/__init__.py`
- **pandas < 3.0** required (breaking `ewm` API changes in 3.0)
- **Ray/RLlib** is optional; tests marked `@pytest.mark.rllib` are excluded from CI
- **TensorFlow >= 2.15.1** is a core dependency
