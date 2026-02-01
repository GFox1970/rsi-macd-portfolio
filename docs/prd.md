# Product Vision & Requirements Document (PRD)

## 1. Product Vision
To build a fully autonomous, self-optimizing trading system that leverages technical analysis (RSI/MACD), Machine Learning, and Macro Regime Detection to achieve consistent profitability in US and Global markets, with a focus on risk management and transparency.

## 2. Goals
- **Autonomous Execution**: The bot runs without manual intervention during market hours across Alpaca and IBKR.
- **Strategic Judgement Layer**: Combine technical indicators, macro regime detection (VIX/FRED), news sentiment (LLM-based), and ML-based gating (XGBoost).
- **Self-Tuning Architecture**: Daily feedback loop where performance post-mortems and adaptive tuning retrain the system.
- **Risk First (Survival Mode)**: Enforce VIX-based circuit breakers, PDT compliance, and symbol weight caps (10%).
- **Tactical Agility**: Replaced hard profit targets with trailing stops and ML-driven hold overrides to maximize winners.
- **Fail-Safe Gating**: Implement strict schema enforcement to prevent trading on missing or corrupted ML feature data.
- **Efficiency**: Fee-aware position sizing to ensure profitability against commission structures.
- **Data Longevity**: Automated cloud archiving to Google Drive for perpetual historical context.
- **Transparency**: High-fidelity Streamlit dashboard featuring "Management Insight" via JSONL audit logs.

## 3. Non-Goals
- **Manual Trade Execution**: Not a terminal for manual entries.
- **Sub-Millisecond Latency**: Operates on 1-min to 5-min intervals.
- **High-Frequency Trading (HFT)**: The system is a systematic swing/day trader, not HFT.

## 4. User Stories
- **As an Operator (Gary)**, I want the bot to deploy automatically to my VM and archive its own logs/data to Google Drive daily.
- **As an Analyst**, I want to see a full audit trail ("Management Insight") of why every trade was placed, including ML scores and VIX context.
- **As a Risk Manager**, I want the bot to enter "Survival Mode" (block all buys) if market volatility spikes (VIX > 40).
- **As a Developer**, I want a unified `BrokerRouter` that handles US stocks (Alpaca), Global Stocks (IBKR), and Crypto seamlessly.

## 5. Functional Requirements
### 5.1 Trading Engine
- **Multi-Broker Routing**: Support Alpaca (US), IBKR (Global), and Crypto via `BrokerRouter`.
- **Intraday Order Management**: Bracket orders (Limit, TP, SL) with automated sync.
- **Fee-Aware Sizing**: MVC (Minimum Viable Capital) calculation to boost small allocations above profitable fee thresholds, gated by ML confidence.
- **Agile Exit Strategy**: Dynamic exit logic featuring trailing stops (activating at +5%) and parabolic holds using "peak" tracking.

### 5.2 Intelligence Layer (Phase 3)
- **Survival Mode**: Macro circuit breaker triggered by VIX > 40 or SPY drop, shifting to defensive/short tracks.
- **News/Thematic Sentiment**: Agentic analysis of news headlines and thematic rotation.
- **ML Gating**: XGBoost confidence scoring with adaptive thresholds and **Strict Schema Enforcement** to block invalid feature sets.
- **Decision Agility**: Reversal-aware PDT management (allowing <5% exits on detected reversals).

### 5.3 Orchestration & Maintenance
- **Daily Orchestrator**: Sequence of Post-Mortem → Auto-Tuning → Strategic Analysis → ML Pipeline → Archiving.
- **Cloud Archiving**: Monthly ZIP bundles of data and logs shipped to `gdrive:TradingBotArchives` via `rclone`.
- **Log Rotation**: Automatic rotation of system and decision logs to keep VM disk space lean.

## 6. Non-Functional Requirements
- **Reliability**: 99.9% uptime during market hours; systemd-monitored.
- **Security**: Environment variable secret management and GitHub Action CI/CD.
- **Observability**: Real-time Streamlit monitoring with deep JSONL decision inspection.
- **Scalability**: Capable of monitoring 100+ symbols and managing cross-broker inventory.
