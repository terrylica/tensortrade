# env/ — Trading Environments

Gymnasium-compatible trading environments with pluggable components.

## Structure: generic/ vs default/

- **`generic/`**: Abstract interfaces — `TradingEnv` + 6 abstract component base classes
- **`default/`**: Concrete implementations + `create()` factory

## TradingEnv Step/Reset Cycle

**`step(action)`** executes in this order:

1. `action_scheme.perform(env, action)` → creates/submits orders
2. `observer.observe(env)` → numpy observation array
3. `reward_scheme.reward(env)` → scalar reward
4. `stopper.stop(env)` → terminated bool
5. Check `max_episode_steps` → truncated bool
6. `informer.info(env)` → metadata dict
7. `clock.increment()`

**`reset()`**: Generates new `episode_id`, resets clock + all components, returns initial observation. Clock starts at step=1 after reset (incremented before returning).

## Six Pluggable Components

| Abstract Class | `registered_name` | Abstract Methods                 | Default Impl                                                                                  |
| -------------- | ----------------- | -------------------------------- | --------------------------------------------------------------------------------------------- |
| `ActionScheme` | `"actions"`       | `action_space`, `perform()`      | `BSH`, `SimpleOrders`, `ManagedRiskOrders`                                                    |
| `RewardScheme` | `"rewards"`       | `reward()`                       | `SimpleProfit`, `RiskAdjustedReturns`, `PBR`, `AdvancedPBR`                                   |
| `Observer`     | `"observer"`      | `observation_space`, `observe()` | `TensorTradeObserver`, `IntradayObserver`                                                     |
| `Stopper`      | `"stopper"`       | `stop()`                         | `MaxLossStopper`                                                                              |
| `Informer`     | `"monitor"`       | `info()`                         | `TensorTradeInformer`                                                                         |
| `Renderer`     | `"renderer"`      | `render()`                       | `EmptyRenderer`, `ScreenLogger`, `FileLogger`, `PlotlyTradingChart`, `MatplotlibTradingChart` |

## Factory: default.create()

```python
env = default.create(
    portfolio=portfolio,
    action_scheme='bsh',           # string or instance
    reward_scheme='risk-adjusted', # string or instance
    feed=feed,
    window_size=20,
    max_allowed_loss=0.5,          # MaxLossStopper threshold
    renderer='plotly',             # string, instance, or list
)
```

Accepts string identifiers resolved via per-module `_registry` dicts (e.g., `actions.get('bsh')` → `BSH`).

## Gotchas

- **Informer registered_name is `"monitor"`**, not `"informer"` — mismatches class name.
- **BSH action_space is `Discrete(2)`**: 0=Hold, 1=Switch position (not Buy/Sell/Hold despite the name).
- **PBR requires manual callback**: `pbr.on_action(action)` must be called to update position tracking.
- **SimpleOrders action space explosion**: Cartesian product of `criteria × trade_sizes × durations × sides` can produce hundreds of discrete actions.
- **Observer.reset()** takes `random_start` parameter; other components take no args.
- **PlotlyTradingChart** requires timestamp data in the feed; will `KeyError` if missing.
- **First observation at step=1**: Reset increments clock before returning, so step 0 is never observed.
