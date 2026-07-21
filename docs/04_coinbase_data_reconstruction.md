# Reconstructing Volume Footprints from Coinbase Data
**Document ID:** SOL-CB-004  
**Classification:** Technical Blueprint  
**Date:** July 21, 2026  

---

## 1. Coinbase SOL-USD Ingestion Channels

For the US fiat-onshore market, Coinbase is the benchmark exchange. Constructing volume footprints from Coinbase requires understanding their specific developer architecture, which differs significantly from Binance's S3 bulk-file paradigm.

Coinbase offers three distinct methods for capturing tick-level trade data:
1. **Advanced Trade REST API (Historical/Backfill)**: Standard HTTP endpoints with cursor-based pagination.
2. **Advanced Trade WebSocket API (Real-Time)**: Low-latency streaming channels (`matches` and `ticker`).
3. **Coinbase Data Marketplace (Bulk Archival)**: A premium institutional service providing full historical tick data via SFTP storage.

---

## 2. API Endpoint Mechanics & Schema

Unlike Binance, Coinbase does not provide an open, free, unthrottled S3 bucket for years of tick-level trade downloads. For non-institutional researchers, the main path is to use their REST API or stream and record real-time data over WebSockets.

### 2.1 REST API: Get Market Trades
To fetch historical trades for `SOL-USD`, use the following endpoint:
`GET https://api.coinbase.com/api/v3/brokerage/products/SOL-USD/ticker?limit=1000` or the advanced trading trades history endpoint:
`GET https://api.coinbase.com/api/v3/brokerage/products/SOL-USD/candles` (for OHLCV) and `/api/v3/brokerage/products/SOL-USD/ticker` (or historical trades equivalents).

In Coinbase Exchange (legacy Pro) or Advanced Trade REST APIs, the standard historical trades endpoint is structured as:
`GET /api/v3/brokerage/products/{product_id}/ticker` or `GET /products/{product_id}/trades`.

#### Response Schema (JSON):
```json
{
  "trades": [
    {
      "trade_id": "100818246",
      "product_id": "SOL-USD",
      "price": "124.13",
      "size": "9.412",
      "time": "2026-07-21T00:01:02.828723Z",
      "side": "SELL"
    }
  ]
}
```

### 2.2 Cursor-Based Pagination
Coinbase implements strict **cursor pagination** to handle heavy datasets. To download historical tick trades sequentially, the client must use the custom headers returned by each response.

* **Headers returned**:
  * `CB-BEFORE`: A cursor representing the start of the current page. Passing this value to the `before` query parameter fetches **newer** trades.
  * `CB-AFTER`: A cursor representing the end of the current page. Passing this value to the `after` query parameter fetches **older** trades.
* **Pagination Loop**: To reconstruct historical footprints over multiple days, your script must execute a synchronized backward loop, passing the `CB-AFTER` value of response $N$ as the `after` parameter of request $N+1$.

---

## 3. Real-Time Streaming: WebSocket Match Channel

For real-time footprint reconstruction, subscribing to Coinbase's WebSocket feed is superior to REST polling. The critical channel is **`matches`** (or `heartbeat` + `ticker` under some API iterations).

### 3.1 WebSocket Subscription Payload
```json
{
  "type": "subscribe",
  "product_ids": ["SOL-USD"],
  "channels": ["matches"]
}
```

### 3.2 Incoming WebSocket Message Schema
```json
{
  "type": "match",
  "trade_id": 48291048,
  "sequence": 9582910482,
  "product_id": "SOL-USD",
  "price": "124.15",
  "size": "4.521",
  "time": "2026-07-21T00:34:25.102834Z",
  "side": "buy"
}
```

---

## 4. Ingesting via Coinbase Data Marketplace (Institutional SFTP)

For institutions requiring clean, multi-year backtesting datasets without rate-limit constraints, Coinbase provides the **Coinbase Data Marketplace**.

* **Access Method**: SFTP (Secure File Transfer Protocol).
* **Directory Structure**:
  `/marketplace/trades/coinbase/coinbase/sol/usd/YYYY/MM/DD/`
* **File Format**: Gzipped CSV files.
  `trades.coinbase.coinbase.sol.usd.YYYY-MM-DD.1735689600.csv.gz`
* **CSV Schema**:
  ```csv
  trade_id,product_id,price,size,side,time
  100818246,"SOL-USD",124.13000000,9.41200000,"sell","2026-07-21T00:01:02.828723Z"
  ```

---

## 5. Reconstructing Footprints: The Coinbase Simplification

One highly favorable characteristic of Coinbase data compared to Binance is the **absence of taker-maker flag inversion**.

* **Direct Aggressor Side**: Coinbase explicitly labels the trade `side` from the perspective of the **taker (aggressor)**.
  * If `side = "buy"` (or `"BUY"`): The taker purchased asset by lifting the ask. This is classified as **Ask/Buy Volume ($V_{\text{ask}}$)**.
  * If `side = "sell"` (or `"SELL"`): The taker sold asset by hitting the bid. This is classified as **Bid/Sell Volume ($V_{\text{bid}}$)**.

This direct labeling prevents common scripting bugs associated with Maker/Taker inversion:

$$V_{\text{direction}} = \begin{cases} 
V_{\text{ask}} & \text{if } \text{side} = \text{"buy"} \\
V_{\text{bid}} & \text{if } \text{side} = \text{"sell"} 
\end{cases}$$

---

## 6. Python Implementation: Coinbase REST Historical Backfill

This script demonstrates how to paginate backwards through Coinbase's historical trade endpoint to retrieve trade logs, parse timestamps into discrete bins, and build a localized Volume Footprint.

```python
import requests
import pandas as pd
import time
from datetime import datetime

class CoinbaseFootprintBuilder:
    def __init__(self, product_id: str = "SOL-USD"):
        self.base_url = "https://api.exchange.coinbase.com" # Public Exchange REST Endpoint
        self.product_id = product_id
        
    def fetch_historical_trades(self, limit: int = 100, after_cursor: str = None):
        """
        Fetches one page of trades. Uses Exchange API which is open for public trade history.
        """
        url = f"{self.base_url}/products/{self.product_id}/trades"
        params = {"limit": limit}
        if after_cursor:
            params["after"] = after_cursor
            
        response = requests.get(url, params=params)
        
        if response.status_code == 429:
            time.sleep(1.0) # Simple backoff
            return self.fetch_historical_trades(limit, after_cursor)
            
        if response.status_code != 200:
            raise Exception(f"API Error: {response.status_code} - {response.text}")
            
        # Coinbase returns cursors in the custom response headers
        cb_after = response.headers.get("CB-AFTER")
        trades_data = response.json()
        
        return trades_data, cb_after

    def backfill_and_reconstruct(self, pages_to_pull: int = 5, tick_step: float = 0.05, bin_size_sec: int = 300):
        """
        Iteratively pulls historical pages and aggregates them into a volume footprint.
        """
        all_trades = []
        cursor = None
        
        print(f"Beginning ingestion of {pages_to_pull} pages from Coinbase...")
        for page in range(pages_to_pull):
            trades, cursor = self.fetch_historical_trades(limit=100, after_cursor=cursor)
            all_trades.extend(trades)
            if not cursor:
                break
            time.sleep(0.3) # Respect rate limits
            
        # Convert to DataFrame
        df = pd.DataFrame(all_trades)
        if df.empty:
            print("No trades retrieved.")
            return None
            
        # Parse and type-cast
        df['price'] = df['price'].astype(float)
        df['size'] = df['size'].astype(float)
        df['time'] = pd.to_datetime(df['time'])
        
        # Discretize (Binning)
        df['time_bin'] = df['time'].dt.floor(f'{bin_size_sec}S')
        df['price_bin'] = (df['price'] / tick_step).apply(lambda x: int(x)) * tick_step
        
        # Map Side to Bid/Ask volumes
        df['bid_vol'] = df.apply(lambda row: row['size'] if row['side'] == 'sell' else 0.0, axis=1)
        df['ask_vol'] = df.apply(lambda row: row['size'] if row['side'] == 'buy' else 0.0, axis=1)
        
        # Group to create footprint
        footprint = df.groupby(['time_bin', 'price_bin']).agg(
            bid_volume=('bid_vol', 'sum'),
            ask_volume=('ask_vol', 'sum'),
            total_volume=('size', 'sum')
        ).reset_index()
        
        footprint['delta'] = footprint['ask_volume'] - footprint['bid_volume']
        return footprint

# Usage Example:
# builder = CoinbaseFootprintBuilder(product_id="SOL-USD")
# footprint = builder.backfill_and_reconstruct(pages_to_pull=10, tick_step=0.01)
# print(footprint.head())
```

---

## 7. Comparison: Binance vs. Coinbase Order Flow Ingestion

| Dimension | Binance (`SOLUSDT`) | Coinbase (`SOL-USD`) |
| :--- | :--- | :--- |
| **Data Cost** | 100% Free | Free REST/WS; Paid premium historical files |
| **Historical Range** | Multi-year archive available since launch | REST limited to recent thousands; SFTP paid |
| **Ingestion Pipeline** | S3 bulk downloads (offline backtesting) | REST cursor paginate or active WS recorder |
| **Schema Complexity** | Maker-Taker flag requires logic inversion | `side` is explicit (buy/sell), very clean |
| **Liquidity & Volume** | High volume, ideal for micro-pattern detection | Regulated flow, lower volume, excellent retail check |

---

## 8. References & Official Links

1. **Coinbase Developer Documentation**: [Exchange REST API Pagination Guide](https://docs.cdp.coinbase.com/exchange/rest-api/pagination/)
2. **Coinbase WebSockets Advanced Trade Feed**: [WS Channels Reference](https://docs.cdp.coinbase.com/advanced-trade/docs/ws-channels/)
3. **Coinbase Data Marketplace Catalog**: [Marketplace Products](https://data.coinbase.com/)
