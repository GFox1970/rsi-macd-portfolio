# API & Integration Specification

## 1. External Broker Integrations

### 1.1 Alpaca Trade API (Primary)
-   **Version**: V2 (Trading & Market Data)
-   **Authentication**: API Key ID and Secret Key via `APCA-API-KEY-ID` and `APCA-API-SECRET-KEY` headers.
-   **Key Endpoints**:
    -   `POST /v2/orders`: Place bracket orders (Take Profit/Stop Loss).
    -   `GET /v2/positions`: Retrieve current active holdings.
    -   `GET /v2/account`: Monitor buying power and equity.
-   **Websockets**: `wss://paper-api.alpaca.markets/stream` for real-time order fill updates and account changes.

### 1.2 Interactive Brokers (IBKR)
-   **Integration Method**: IBKR Client Portal API (Headless Gateway).
-   **Host/Port**: `localhost:4002` (within Docker network).
-   **Authentication**: Automated via `TWS_USERID` and `TWS_PASSWORD` injected into the Gateway container.
-   **Flow**: The bot sends REST requests to the gateway, which forwards them to IBKR's FIX/REST bridge.

## 2. Market Data Providers

### 2.1 Polygon.io
-   **Authentication**: Bearer token via `apiKey` query parameter.
-   **Usage**:
    -   **Sector Analysis**: Primary source for fetching daily bars for the `SectorAnalyzer`.
    -   **Intraday Outliers**: Used via `fetch_polygon_intraday_data.py` to identify high-volatility candidates during weekly analysis.
-   **Persistence**: Data is cached in `./data/sector_report.json`.

### 2.2 yfinance (Primary Prep)
-   **Usage**: The primary data source for the **ML Preparation Pipeline** and **Historical Ranking**.
-   **Protocol**: HTTPS (Scraping/REST fallback).

## 3. Internal Event Flows

### 3.1 Decision Logger (JSONL)
-   **Pattern**: Append-only log stream.
-   **Payload**: Structured JSON containing `market_context`, `agent_reasoning`, and `order_id`.
-   **Consumer**: `app.py` (Streamlit Dashboard) which tails the file for real-time UI updates.

## 4. Error Handling & Resilience
-   **Rate Limiting**: Implementation of exponential backoff for Alpaca (429 errors).
-   **Circuit Breakers**: If 3 consecutive API failures occur with a broker, the bot enters "Graceful Shutdown" mode to prevent stale-data orders.
-   **Payload Validation**: All outgoing order payloads are validated against the `OrderSchema` to prevent rejection due to malformed metadata.
