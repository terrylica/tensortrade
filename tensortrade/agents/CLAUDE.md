# agents/ — RL Agent Implementations

Built-in reinforcement learning agents. **All deprecated** in favor of external frameworks (Ray/RLlib). <!-- SSoT-OK -->

## Agents

| Agent              | Algorithm              | Network                                             | File           |
| ------------------ | ---------------------- | --------------------------------------------------- | -------------- |
| `DQNAgent`         | Deep Q-Network         | Conv1D (9 layers) + GRU (4 layers) + target network | `dqn_agent.py` |
| `A2CAgent`         | Advantage Actor-Critic | Shared Conv1D + separate actor/critic heads         | `a2c_agent.py` |
| `ParallelDQNAgent` | Distributed DQN        | Multi-process trainers + single optimizer           | `parallel/`    |

## Base Agent Interface

```python
class Agent(Identifiable):  # Abstract
    def get_action(state: np.ndarray, **kwargs) -> int
    def train(n_steps, n_episodes, save_every, save_path, callback) -> float
    def save(path) / restore(path)
```

## ReplayMemory

Circular buffer for experience storage. Supports `push(*args)`, `sample(batch_size)`, `head(n)`, `tail(n)`.

## Parallel Architecture (parallel/)

Three-process design: multiple `ParallelDQNTrainer` processes generate transitions → shared memory queue → single `ParallelDQNOptimizer` process updates model → model update queue → trainers sync weights. Uses custom `ParallelQueue` to work around macOS `qsize()` limitation.

## Recommendation

For new work, use Ray/RLlib integration (see `examples/training/train_ray_long.py`). Built-in agents remain functional but unmaintained.
