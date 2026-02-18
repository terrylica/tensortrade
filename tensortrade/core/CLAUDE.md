# core/ — Component System & Foundation

Foundation layer providing dependency injection, identity, time tracking, and event observation for all TensorTrade components.

## Key Pattern: Context Injection via Metaclass

`Component` uses `InitContextMeta` (metaclass) to inject config from `TradingContext` **before** `__init__` runs. Every `Component` subclass must declare `registered_name` — the key used for context lookup:

```python
class MyReward(Component):
    registered_name = "rewards"  # matches TradingContext config key

    def __init__(self, scale=1.0):
        self.scale = self.default('scale', scale)  # context value wins over default

with TradingContext({"rewards": {"scale": 2.5}}):
    r = MyReward()  # r.scale == 2.5 (from context, not default)
```

**Execution order**: `__new__` → `context` injected → `__init__` called. So `self.context` is available inside `__init__`.

## Modules

| File            | Exports                                                                          | Purpose                                                                                             |
| --------------- | -------------------------------------------------------------------------------- | --------------------------------------------------------------------------------------------------- |
| `base.py`       | `Identifiable`, `TimeIndexed`, `TimedIdentifiable`, `Observable`, `global_clock` | Mixins: lazy UUID, clock binding, listener pattern                                                  |
| `clock.py`      | `Clock`                                                                          | Tracks simulation time (`step` counter) and real time (`now()`) independently                       |
| `component.py`  | `Component`, `InitContextMeta`, `ContextualizedMixin`                            | Base class + metaclass for context-injected components                                              |
| `context.py`    | `TradingContext`, `Context`                                                      | Thread-local config stack (`with` statement); `Context` is the injected dict-like object            |
| `registry.py`   | `registry()`, `register()`, `MAJOR_COMPONENTS`                                   | Maps Component classes → registered names; auto-populated via `__init_subclass__`                   |
| `exceptions.py` | 9 exception types                                                                | Domain errors: `InsufficientFunds`, `InvalidOrderQuantity`, `IncompatibleInstrumentOperation`, etc. |

## Gotchas

- **Clock.now() vs Clock.step**: `now()` returns real `datetime.now()`, completely independent of `step` counter. They are two unrelated time concepts.
- **registered_name = None**: Forgetting to set this on a subclass causes `KeyError` at instantiation, not at class definition.
- **Context merge order**: `{**shared_config, **component_config}` — component-specific keys override shared keys.
- **Thread-local stack**: Each thread gets its own `TradingContext` stack. An empty default context is auto-created on first access.
- **Observable.detach()**: Raises `ValueError` if listener not found (no guard).
- **MAJOR_COMPONENTS list**: `["actions", "rewards", "observer", "monitor", "stopper", "renderer"]` — note `"monitor"` not `"informer"`.
