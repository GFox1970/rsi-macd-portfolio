# Data Architecture Document

## 1. Overview
The system utilizes a hybrid data architecture, combining unstructured **JSONL** logs for high-frequency event auditing and structured **SQLite** storage for Machine Learning feature/outcome pairs.

## 2. Ingestion Layer
Data is ingested from four primary sources:
-   **Market Data (OHLCV)**: Ingested via **yfinance** (Preparation/Training) and **Alpaca** (Live Execution).
-   **Sector Data**: Multi-factor sector relative strength fetched via **Polygon.io**.
-   **Sentiment Data**: Real-time news snippets processed via LLM/NLP.
-   **Broker State**: Positions and equity fetched via Alpaca/IBKR REST APIs.
-   **Macro Indicators**: FRED (Federal Reserve Economic Data) for VIX and interest rate series.

## 3. Data Storage Models

### 3.1 Decision Log (Audit Trail)
-   **Format**: JSONL (JSON Lines)
-   **Location**: `logs/enhanced_decision_log.jsonl`
-   **Primary Key**: `decision_id` (UUID + Timestamp)
-   **Key Fields**:
    -   `market_context`: Indicators (RSI, MACD), OHLCV, Sentiment.
    -   `portfolio_state`: Buying power, current holdings.
    -   `decision`: Action (BUY/SKIP/SELL), Confidence, Reasoning.
    -   `order_outcome`: Fill price, qty, status.

### 3.2 ML Feature DB (Training)
-   **Engine**: SQLite
-   **Location**: `ml_db/ml_data.db`
-   **Table: `ml_rows`**
| Column | Type | Description |
| :--- | :--- | :--- |
| `decision_id` | TEXT (PK) | Unique ID linking to the decision log. |
| `symbol` | TEXT | Stock ticker. |
| `timestamp` | TEXT | ISO8601 creation time. |
| `action` | TEXT | BUY/SELL/SKIP. |
| `json_row` | TEXT (JSON) | Flat map of all features at time of trade. |
| `outcome_profit` | REAL | Raw profit/loss from the trade. |
| `holding_period` | INTEGER | Time in minutes between Open and Close. |

### 3.3 State Management
-   **Positions**: `positions.json` stores current active trades to persist across container restarts.
-   **PDT Cache**: `pdt_state.json` tracks day-trade counts for regulatory compliance.

## 4. Data Flows

### 4.1 Feature Lifecycle
1.  **Extraction**: Indicators calculated by `IndicatorsCalculator`.
2.  **Normalization**: Values scaled/mapped for XGBoost input.
3.  **Logging**: `EnhancedDecisionLogger` writes to both JSONL and SQLite.
4.  **Labeling**: Post-trade "Outcomes" are computed by the `PerformanceAnalyzer` and pushed back into the SQLite `outcome_profit` column.
5.  **Training**: `XGBoostTrainer` queries SQLite `ml_rows` where `outcome_profit` is NOT NULL.

### 4.2 Data Synchronization
To maintain parity between the production environment (Hetzner VM) and the local development environment, data is synchronized via the **Data-Harvester** skill (`scripts/pull_vm_data.sh`). This script pulls down the latest XGBoost models, training datasets, and primary historic database (`ml_trading_data.db`) to enable realistic local backtesting and model iterations.

## 5. Retention Rules
-   **Decision Logs**: Standard retention of 90 days. Older logs are archived to `.gz` format monthly.
-   **ML Data**: Retained indefinitely to support multi-year backtesting and longitudinal model analysis.
-   **Market Data**: Intermediate CSVs in `./data/historical` are refreshed weekly.
