# Reconstructing Volume Footprints from Binance Data
**Document ID:** SOL-BI-003  
**Classification:** Technical Blueprint  
**Date:** July 21, 2026  

---

## 1. Data Ingestion & Source Directories

Binance provides comprehensive historical market data for free through its public data archive hosted on AWS S3. This archive is accessible directly via HTTPS or CLI.

### 1.1 S3 URL Structure and Path Formats
The base endpoint is: `https://data.binance.vision/`

For Solana, historical trade data is divided into monthly and daily ZIP archives under two distinct folders:
* **Spot Market (`SOLUSDT`)**:
  * Monthly Aggregated Trades:
    `data/spot/monthly/aggTrades/SOLUSDT/SOLUSDT-aggTrades-YYYY-MM.zip`
  * Daily Aggregated Trades:
    `data/spot/daily/aggTrades/SOLUSDT/SOLUSDT-aggTrades-YYYY-MM-DD.zip`
  * Monthly Raw Trades:
    `data/spot/monthly/trades/SOLUSDT/SOLUSDT-trades-YYYY-MM.zip`
  * Daily Raw Trades:
    `data/spot/daily/trades/SOLUSDT/SOLUSDT-trades-YYYY-MM-DD.zip`
* **USDT-Margined Perpetual Market (`SOLUSDT` - UM Futures)**:
  * Monthly Aggregated Trades:
    `data/futures/um/monthly/aggTrades/SOLUSDT/SOLUSDT-aggTrades-YYYY-MM.zip`
  * Daily Aggregated Trades:
    `data/futures/um/daily/aggTrades/SOLUSDT/SOLUSDT-aggTrades-YYYY-MM-DD.zip`

---

## 2. Dataset Selection: Raw `trades` vs. `aggTrades`

When building a high-resolution footprint system, selecting the correct dataset involves a tradeoff between precision and processing overhead.

### 2.1 Raw Trades (`trades`)
* **Characteristics**: Represents the exact execution messages emitted by the matching engine. Each row maps to a single atomic fill between one maker and one taker.
* **Pros**: Max precision. Essential for studying trade size distributions, detecting automated execution slice signatures (e.g., TWAP/VWAP iceberg slicing), and microsecond-level latency studies.
* **Cons**: Massive storage and memory footprint. A high-volume day for SOLUSDT can easily contain millions of rows.

### 2.2 Aggregated Trades (`aggTrades`)
* **Characteristics**: Binance aggregates consecutive trade executions that occur:
  1. At the exact same price,
  2. Under the exact same trade sign (aggressor side), and
  3. Within the same millisecond.
  These are compressed into a single "aggTrade" entry.
* **Pros**: Reduces data file size by **60% to 80%**, drastically speeding up ingestion, storage, and footprint computation.
* **Cons**: Blurs the individual order-size distribution (e.g., a single market order of 1,000 SOL filled across 5 limit orders is aggregated into one row, but 5 separate retail market orders of 200 SOL in the same millisecond are also aggregated into one row).
* **Quant Recommendation**: For timeframes like 5m, 15m, and 1h volume footprint charts, **`aggTrades` is highly recommended and standard**. It provides identical directional volume-by-price totals as raw `trades` while saving millions of compute cycles.

---

## 3. CSV Schema Definitions

The historical CSV files downloaded from the S3 bucket do not contain header rows. The columns must be mapped according to the following schemas:

### 3.1 Raw Spot Trades (`trades`)
```
trade_id, price, quantity, quote_quantity, timestamp, is_buyer_maker, is_best_match
```
* **Example**:
  `124958102, 124.13000000, 15.42000000, 1914.0846, 1735689600010, False, True`

### 3.2 Aggregated Spot Trades (`aggTrades`)
```
aggregate_trade_id, price, quantity, first_trade_id, last_trade_id, timestamp, is_buyer_maker, is_best_match
```
* **Example**:
  `5810294, 124.13000000, 45.10000000, 124958102, 124958105, 1735689600012, False, True`

### 3.3 Futures Aggregated Trades (`aggTrades` - Futures)
*Note: Futures datasets omit the `is_best_match` column.*
```
aggregate_trade_id, price, quantity, first_trade_id, last_trade_id, timestamp, is_buyer_maker
```

---

## 4. The Taker-Maker Flag Inversion Trap

The single biggest source of error when retail programmers construct volume delta or footprint indicators is the interpretation of Binance's **`is_buyer_maker`** flag.

* **Definition of `is_buyer_maker`**: A boolean flag indicating whether the limit order (maker) on the buy side was filled.
* **Microstructure Inversion**:
  * If `is_buyer_maker = True`: The buyer was the maker (resting liquidity provider). This means a **seller** was the taker (aggressor) who initiated the trade by hitting the bid. Therefore, this trade is a **Sell (Hit Bid)**.
  * If `is_buyer_maker = False`: The buyer was the taker (aggressor) who initiated the trade by hitting the ask. Therefore, this trade is a **Buy (Lift Ask)**.

### Directional Mapping Table:
| `is_buyer_maker` Value | Maker Side | Taker (Aggressor) Side | Volume Footprint Classification |
| :--- | :--- | :--- | :--- |
| **`True`** | Buyer | Seller | **Sell / Bid Volume ($V_{\text{bid}}$)** |
| **`False`** | Seller | Buyer | **Buy / Ask Volume ($V_{\text{ask}}$)** |

**Quant Rule**:
$$V_{\text{direction}} = \begin{cases} 
V_{\text{bid}} & \text{if } \text{is\_buyer\_maker} = \text{True} \\
V_{\text{ask}} & \text{if } \text{is\_buyer\_maker} = \text{False} 
\end{cases}$$

---

## 5. Footprint Reconstruction Algorithm (Step-by-Step)

To construct an institutional-grade $5\text{m}$ volume footprint from Binance Spot or Futures `aggTrades` data, follow this step-by-step pipeline.

### Step 5.1: Grid Parameter Initialization
Define the temporal bin size $T$ (e.g., $300\text{ seconds}$ for 5m candles) and the price consolidation step $P_{\text{step}}$ (e.g., if Solana's tick size is $\$0.001$, consolidate to $\$0.01$ or $\$0.05$ to keep the price ladder readable).

### Step 5.2: Ingestion & Record Mapping
For each CSV row, parse:
* Price: $p_{\text{raw}}$ (float)
* Quantity: $q_{\text{raw}}$ (float)
* Timestamp: $t_{\text{ms}}$ (integer millisecond epoch)
* Buyer Maker Flag: $bm$ (boolean)

### Step 5.3: Discretization (Binning)
Map the raw micro-values onto the footprint grid:
* **Time Bin**:
  $$t_{\text{bin}} = \left\lfloor \frac{t_{\text{ms}}}{1000 \cdot T} \right\rfloor \cdot T$$
* **Price Bin**:
  $$p_{\text{bin}} = \left\lfloor \frac{p_{\text{raw}}}{P_{\text{step}}} \right\rfloor \cdot P_{\text{step}}$$

### Step 5.4: Directional Accumulation
For each coordinate tuple $(p_{\text{bin}}, t_{\text{bin}})$, update the cell's volume:

$$\begin{aligned}
\text{If } bm = \text{True}: \quad & V_{\text{bid}}(p_{\text{bin}}, t_{\text{bin}}) \leftarrow V_{\text{bid}}(p_{\text{bin}}, t_{\text{bin}}) + q_{\text{raw}} \\
\text{If } bm = \text{False}: \quad & V_{\text{ask}}(p_{\text{bin}}, t_{\text{bin}}) \leftarrow V_{\text{ask}}(p_{\text{bin}}, t_{\text{bin}}) + q_{\text{raw}}
\end{aligned}$$

### Step 5.5: Derived Metric Extraction
For each completed time interval $t_{\text{bin}}$:
1. **Total Volume**: $V_{\text{total}}(t_{\text{bin}}) = \sum_{p} [V_{\text{bid}}(p, t_{\text{bin}}) + V_{\text{ask}}(p, t_{\text{bin}})]$
2. **Point of Control (POC)**:
   $$\text{POC}(t_{\text{bin}}) = \arg\max_{p} \left[ V_{\text{bid}}(p, t_{\text{bin}}) + V_{\text{ask}}(p, t_{\text{bin}}) \right]$$
3. **Value Area (70%) Calculation**:
   - Find the POC price level.
   - Sum the volume of POC.
   - Iteratively add the volume of adjacent price rows (above and below) that have the highest combined volume, until the accumulated volume is $\ge 70\%$ of the total bar volume.
   - The highest boundary is **Value Area High (VAH)**, and the lowest is **Value Area Low (VAL)**.

---

## 6. Python Reference Implementation (High-Performance Vectorized)

The following production-grade script illustrates how to ingest a raw Binance CSV file and reconstruct a highly granular volume footprint and Cumulative Volume Delta (CVD) series using `pandas` and `numpy`.

```python
import pandas as pd
import numpy as np

def reconstruct_volume_footprint(csv_path: str, timeframe_sec: int = 300, tick_step: float = 0.05):
    """
    Ingests raw Binance spot aggTrades data and outputs a multi-indexed 
    DataFrame containing Bid and Ask volume for each price-time cell.
    """
    # Define columns for Spot aggTrades (No headers in Binance files)
    cols = ['agg_trade_id', 'price', 'quantity', 'first_id', 'last_id', 'timestamp', 'is_buyer_maker', 'is_best_match']
    
    # Load efficiently
    df = pd.read_csv(csv_path, names=cols, usecols=['price', 'quantity', 'timestamp', 'is_buyer_maker'], dtype={
        'price': np.float64,
        'quantity': np.float64,
        'timestamp': np.int64,
        'is_buyer_maker': bool
    })
    
    # 1. Convert millisecond timestamp to datetime
    df['datetime'] = pd.to_datetime(df['timestamp'], unit='ms')
    
    # 2. Time-binning (e.g., 5m intervals)
    df['time_bin'] = df['datetime'].dt.floor(f'{timeframe_sec}S')
    
    # 3. Price-binning (Consolidation)
    df['price_bin'] = (df['price'] / tick_step).apply(np.floor) * tick_step
    
    # 4. Map Taker-Maker Flag to Bid/Ask volumes
    # Inversion: if is_buyer_maker is True -> Sell (Hit Bid) -> bid_vol
    # If is_buyer_maker is False -> Buy (Lift Ask) -> ask_vol
    df['bid_vol'] = np.where(df['is_buyer_maker'] == True, df['quantity'], 0.0)
    df['ask_vol'] = np.where(df['is_buyer_maker'] == False, df['quantity'], 0.0)
    
    # 5. Group by price_bin and time_bin to construct footprint
    footprint = df.groupby(['time_bin', 'price_bin']).agg(
        bid_volume=('bid_vol', 'sum'),
        ask_volume=('ask_vol', 'sum')
    ).reset_index()
    
    # Calculate cell-level metrics
    footprint['total_volume'] = footprint['bid_volume'] + footprint['ask_volume']
    footprint['delta'] = footprint['ask_volume'] - footprint['bid_volume']
    
    return footprint

def compute_bar_cvd(footprint_df):
    """
    Computes bar-level delta and Cumulative Volume Delta (CVD)
    """
    bar_metrics = footprint_df.groupby('time_bin').agg(
        bar_volume=('total_volume', 'sum'),
        bar_delta=('delta', 'sum')
    ).reset_index()
    
    # Compute running cumulative sum of delta
    bar_metrics['cvd'] = bar_metrics['bar_delta'].cumsum()
    return bar_metrics
```

---

## 7. References & Official Links

1. **Binance Developer Documentation**: [Aggregate Trades Endpoint](https://binance-docs.github.io/apidocs/spot/en/#compressed-aggregate-trades-list)
2. **Binance Public Data GitHub Archive**: [binance/binance-public-data](https://github.com/binance/binance-public-data)
3. **AWS S3 Browser Mirror**: [data.binance.vision](https://data.binance.vision/)
