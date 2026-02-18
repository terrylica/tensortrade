# feed/ — Stream-Based Data Pipeline

Functional, composable data pipeline built on a DAG (directed acyclic graph) of `Stream` nodes. Drives observations and price data during environment steps.

## Core Concepts

**Stream**: A node in the computation graph. Each stream has `inputs` (upstream dependencies), a `value` (current output), and a `forward()` method that computes the next value.

**DataFeed**: Orchestrator that topologically sorts streams and executes them in order. Must be `compile()`d before use.

**NameSpace**: Context manager that prefixes stream names with `"namespace:/"` separator.

## Usage Pattern

```python
from tensortrade.feed import Stream, DataFeed, NameSpace

with NameSpace("binance"):
    price = Stream.source(price_data, dtype="float").rename("close")
    sma = price.rolling(window=20).mean().rename("sma_20")

feed = DataFeed([price, sma])
feed.compile()  # REQUIRED before next()

while feed.has_next():
    data = feed.next()  # {"binance:/close": 100.5, "binance:/sma_20": 99.8}
```

## Stream Types

| Type             | Created via                   | Purpose                      |
| ---------------- | ----------------------------- | ---------------------------- |
| `IterableStream` | `Stream.source(list)`         | Root node from iterable data |
| `Constant`       | `Stream.constant(value)`      | Emits same value forever     |
| `Sensor`         | `Stream.sensor(object, func)` | Reads live value from object |
| `Placeholder`    | `Stream.placeholder()`        | For push-based (live) feeds  |

## Operators & Methods

Streams support chained transformations via mixin methods registered by dtype:

- **Generic** (all dtypes): `.apply(func)`, `.lag(n)`, `.freeze()`, `.accumulate(func)`, `.warmup(n)`, `.fillna()`, `.ffill()`
- **Float**: `.rolling(n).mean/sum/std/var/min/max()`, `.expanding().mean/sum()`, `.ewm(span=n).mean/var/std()`
- **Float math**: `+`, `-`, `*`, `/`, `**`, `.sqrt()`, `.log()`, `.abs()`, `.pct_change()`, `.diff()`, `.clamp_min/max()`
- **Float cumulative**: `.cumsum()`, `.cumprod()`, `.cummin()`, `.cummax()`
- **Boolean**: `.invert()`
- **String**: `.upper()`, `.lower()`, `.capitalize()`, `.slice(start, end)`, `.startswith()`, `.cat()`

Arithmetic operators auto-promote scalars: `stream + 5` works (wraps 5 in `Constant`).

## Architecture

```
Stream.source([...]) ──→ Lag(1) ──→ BinOp(+) ──→ DataFeed
Stream.source([...]) ──────────────↗               │
                                              compile() → toposort
                                              next()   → run all in order
                                              reset()  → clear stateful operators
```

## Gotchas

- **Must compile before next()**: `feed.compile()` populates execution order. Without it, streams may execute in wrong order.
- **Stream naming format**: `"namespace:/name"` (colon-slash separator). Not `:symbol` or `-symbol`.
- **Stateful operators need reset**: `Lag`, `Rolling`, `Accumulator`, `EWM` maintain internal state. Always call `feed.reset()` between episodes.
- **NaN in early steps**: `Lag(n)` returns `NaN` for first n steps. `Rolling(n)` may return `NaN` before `min_periods` values accumulate.
- **has_next() checks ALL source streams**: Returns `False` if any `IterableStream` is exhausted, even if others have data remaining.
