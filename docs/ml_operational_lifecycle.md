# ML Operational Lifecycle & Accelerated Learning Universe (ALU)

This document explains the dynamic lifecycle of the Machine Learning (ML) pipeline, specifically how the bot "learns" from market volatility in real-time between global market sessions.

## 1. The Core Philosophy: "What Could Have Been"
The bot doesn't just learn from static historical data; it learns from **reconstructing its own opportunities**. Every time the orchestrator runs (7:00 AM UTC for UK/EU, 1:30 PM UTC for US, 11:00 PM UTC for Asia), it performs a three-step cycle to calibrate the ML model to the current "Market Pulse."

### Step 1: Scenario Reconstruction (find_top_day_trading)
The [candidate discovery script](file:///home/gary/rsi-macd-bot/weekly_analysis/find_top_day_trading_candidates.py) identifies the most volatile symbols in the current session and fetches the last 21 days of 5-minute data for them.
- **Hindsight Labeling**: For every 5-minute bar, it asks: *"If I had bought here, would the price have hit $X target within 30 minutes?"*
- **The Output**: A dataset labeled with `1` (Success) and `0` (Failure) based on actual realized price action.

### Step 2: Hindsight Training (ALU)
The [ML training script](file:///home/gary/rsi-macd-bot/ml_pipeline/training/train_mi_model.py) then takes these labeled scenarios and trains the XGBoost model.
- **Calibration**: The model "learns" which technical patterns (RSI, MACD, Volume spikes) successfully led to profit *in this specific group of stocks* over the last 3 weeks.
- **ALU (Accelerated Learning Universe)**: This allows the bot to adapt its sensitivity to the current regime before the market even opens.

### Step 3: Global Handoff (The Orchestrator)
The [Daily Orchestrator](file:///home/gary/rsi-macd-bot/daily_orchestrator.py) ensures this intelligence flows correctly between time zones.

| Time (UTC) | Session | Action | Context Handoff |
| :--- | :--- | :--- | :--- |
| **07:00** | **UK/EU** | Initial Discovery | Creates fresh global watchlist. |
| **13:30** | **US Open** | Pre-market Refresh | Picks up 7:00 AM symbols, adds US gaps, and re-trains. |
| **23:00** | **Asia/OC** | Clean Slate | Discards US data (as stale) and restarts for the new day. |

---

## 2. Technical Safeguards

### Data Freshness (TTL)
To prevent the bot from trading US sessions based on stale UK news (or vice-versa after too much time has passed), the orchestrator enforces an **8-hour Time-To-Live (TTL)**.
- If the "Success/Failure" file is < 8 hours old, the orchestrator uses it as "Institutional Memory" to jumpstart analysis.
- If the file is > 8 hours old, it is marked as `STALE`, and the bot generates a new worldview from scratch.

### Dynamic Handoff
The orchestrator explicitly passes symbols and data paths to the ML initialization scripts. This ensures that the **Learning** (Step 2) is always perfectly aligned with the **Discovery** (Step 1) performed in that specific run.

---

## 3. Why this Matters
Traditional models are trained once and then used for weeks. This "Operational Lifecycle" approach means your bot is **re-learning the market** at every major shift in global time zones. It doesn't just know "Technical Analysis"; it knows "Technical Analysis for the specific stocks trending *right now*."
