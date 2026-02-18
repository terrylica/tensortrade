# examples/ — Training Scripts & Notebooks

## Training Scripts (`training/`)

| Script                  | Approach                                             |
| ----------------------- | ---------------------------------------------------- |
| `train_simple.py`       | Basic demo: fetch data, create env, run random agent |
| `train_ray_long.py`     | Distributed training with Ray/RLlib PPO              |
| `train_optuna.py`       | Hyperparameter optimization with Optuna              |
| `train_best.py`         | Configuration from best experiment results           |
| `train_historical.py`   | Walk-forward validation                              |
| `train_walkforward.py`  | Walk-forward with rolling windows                    |
| `train_robust.py`       | Robustness testing across conditions                 |
| `train_trend.py`        | Trend-following strategies                           |
| `train_profit.py`       | Profit-focused reward shaping                        |
| `train_advanced.py`     | Advanced techniques                                  |
| `run_ray_simulation.py` | Ray cluster simulation runner                        |

## Standard Pattern (from train_simple.py)

```python
# 1. Fetch data
cdd = CryptoDataDownload()
data = cdd.fetch("Bitfinex", "USD", "BTC", "1h")

# 2. Create streams from OHLCV columns
price = Stream.source(data["close"], dtype="float").rename("USD-BTC")

# 3. Setup exchange with simulated execution
exchange = Exchange("exchange", service=execute_order, options=ExchangeOptions(commission=0.003))
exchange(price)  # register price stream

# 4. Create wallets and portfolio
portfolio = Portfolio(USD, [Wallet(exchange, 10000 * USD), Wallet(exchange, 0 * BTC)])

# 5. Build environment
env = default.create(feed=feed, portfolio=portfolio, action_scheme=BSH(cash, asset),
                     reward_scheme=PBR(price), window_size=5)

# 6. Train loop
obs, info = env.reset()
while not done:
    action = policy(obs)
    obs, reward, terminated, truncated, info = env.step(action)
```

## Dependencies

Training examples require additional packages: `pip install -r examples/requirements.txt` (Ray, ccxt, optuna, torch, scikit-learn, etc.)
