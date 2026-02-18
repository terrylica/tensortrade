# tests/ — Test Suite

## Running Tests

```bash
pytest tests/ -m "not rllib"                              # all tests, skip RLlib
pytest tests/tensortrade/unit -v                          # unit tests only
pytest tests/tensortrade/unit/oms/wallets/test_portfolio.py -v  # single file
pytest --workers auto tests/                              # parallel execution
```

## Structure

```
tests/
├── tensortrade/
│   ├── unit/
│   │   ├── oms/          # instruments, wallets, orders, exchanges, services
│   │   ├── env/          # default and generic environment tests
│   │   ├── feed/         # DataFeed and stream pipeline tests
│   │   └── stochastic/   # stochastic process shape validation
│   └── integration/
│       └── test_end_to_end.py
├── utils/
│   └── ops.py            # DataFeed assertion helper
└── data/                 # test data fixtures
```

## Markers

| Marker  | Purpose               | Usage                                   |
| ------- | --------------------- | --------------------------------------- |
| `slow`  | Long-running tests    | `pytest -m "not slow"`                  |
| `rllib` | Ray/RLlib integration | `pytest -m "not rllib"` (skipped in CI) |

## Test Style

- Pytest functions (no class-based tests)
- Minimal fixtures; inline setup per test
- Stochastic tests validate output shape (row count, column names), not exact values
- `tests/utils/ops.py` provides helpers for asserting DataFeed outputs
