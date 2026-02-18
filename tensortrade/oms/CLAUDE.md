# oms/ — Order Management System

Simulated trading infrastructure: instruments, wallets, order lifecycle, exchange execution, and portfolio tracking.

## Submodule Map

| Submodule             | Key Classes                                             | Purpose                                                                   |
| --------------------- | ------------------------------------------------------- | ------------------------------------------------------------------------- |
| `instruments/`        | `Instrument`, `Quantity`, `TradingPair`, `ExchangePair` | Asset definitions, operator-overloaded amounts, market pairs              |
| `wallets/`            | `Wallet`, `Portfolio`, `Ledger`                         | Balance tracking (free/locked), multi-wallet aggregation, transaction log |
| `orders/`             | `Order`, `Broker`, `OrderSpec`, `Trade`, criteria       | Order lifecycle, execution routing, chaining, composable conditions       |
| `exchanges/`          | `Exchange`, `ExchangeOptions`                           | Price streams, commission/slippage config, order dispatch                 |
| `services/execution/` | `execute_order` (simulated)                             | FIFO fill logic for backtesting                                           |
| `services/slippage/`  | `SlippageModel`, `RandomUniformSlippageModel`           | Post-fill price/size adjustment                                           |

## Order Execution Flow

```
ActionScheme.get_orders(action)
    → Broker.submit(order)         # queued as unexecuted
    → Broker.update()              # each step: checks is_executable
        → Order.execute()          # status: PENDING → OPEN
            → Exchange.execute_order()
                → execute_order()  # simulated FIFO fill
                    → Wallet.transfer()  # moves funds, records in Ledger
                → Order.fill(trade)    # status: PARTIALLY_FILLED
        → Broker.on_fill()         # if complete: Order.complete()
            → OrderSpec.create_order()  # chained follow-up order (stop-loss, take-profit)
```

## Instrument & Quantity System

33 pre-defined instruments (BTC, ETH, USD, AAPL, etc.). Operator overloading:

- `5 * BTC` → `Quantity(BTC, 5)`
- `BTC / USD` → `TradingPair(BTC, USD)`
- `Quantity + Quantity` → validated same-instrument addition

`Quantity` uses `Decimal` internally (not float). Supports locking via `path_id` for order fund reservation.

## Wallet Fund Locking

```python
wallet = Wallet(exchange, 10000 * USD)
locked = wallet.lock(Quantity(USD, 5000), order, "LOCK")  # free: 5000, locked: 5000
wallet.unlock(locked, "CANCEL")                            # free: 10000, locked: 0
```

`Wallet.transfer()` validates fund conservation: `(source_before - source_after) == (quantity + commission)`.

## Order Lifecycle

`PENDING` → `OPEN` → `PARTIALLY_FILLED` → `FILLED` (or `CANCELLED` from any active state)

Orders support chaining via `OrderSpec`: after fill, automatically creates follow-up orders (e.g., buy → stop-loss sell) sharing the same `path_id`.

## Composable Order Criteria

```python
from tensortrade.oms.orders.criteria import Stop, Limit, Timed, StopDirection

# Operators: & (AND), | (OR), ^ (XOR), ~ (NOT)
risk_exit = Stop(StopDirection.DOWN, 0.05) ^ Stop(StopDirection.UP, 0.10)  # stop-loss XOR take-profit
timed_limit = Limit(55000) & Timed(100)  # limit order valid for 100 steps
```

## Order Creation Helpers

- `proportion_order(portfolio, source_wallet, target_wallet, proportion)` — rebalance by proportion
- `risk_managed_order(side, type, pair, price, qty, down_pct, up_pct, portfolio)` — entry + auto stop-loss/take-profit
- `market_order()`, `limit_order()`, `hidden_limit_order()`

## Gotchas

- **ExchangeOptions.commission** default is 0.003 (0.3%), applied to fill quantity.
- **Order quantity locked at creation**: Funds are reserved in the wallet immediately, not at execution time.
- **Broker.update() runs every step**: Checks both execution readiness and expiration for all orders.
- **Portfolio.profit_loss**: Calculated as `1.0 - (net_worth / initial_net_worth)`, so positive means loss.
