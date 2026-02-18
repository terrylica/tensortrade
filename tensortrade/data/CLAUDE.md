# data/ — Data Fetching

## CryptoDataDownload (`cdd.py`)

Downloads OHLCV cryptocurrency data from cryptodatadownload.com.

```python
from tensortrade.data import CryptoDataDownload

cdd = CryptoDataDownload()
df = cdd.fetch("Bitfinex", "USD", "BTC", "1h")
# Returns DataFrame: date, open, high, low, close, volume
```

- Handles exchange-specific formats (Gemini has different API)
- Normalizes columns to lowercase standard names
- Optional `include_all_volumes=True` for base + quote volume columns
