# ML Strategy & Performance Analysis

This document provides a comprehensive overview of the Machine Learning (ML) strategy and performance tracking mechanisms implemented in the bot.

## 1. ML Strategy Overview

The bot employs a **multi-layered gating strategy** where Machine Learning (XGBoost) provides the primary technical conviction, which is then refined by specialized analyzers.

### 🧠 Core Model Architecture
- **Model Type**: [XGBoost Classifier](file:///home/gary/rsi-macd-bot/ml_pipeline/training/train_mi_model.py) (`XGBClassifier`).
- **Input Features**: A 12-dimensional vector including:
  - **Lagged Indicators**: RSI, MACD (line, signal, hist), ATR, and Volatility (Standard Deviation).
  - **Momentum**: Returns over 1, 5, and 15-minute windows.
  - **Market Dynamics**: Volume spikes.
  - **Temporal Context**: Sin/Cos encoding of the minute-of-day ([feature engineering](file:///home/gary/rsi-macd-bot/trading_bot/core/ml_integration.py#L400-L493)).
- **Target Variable**: Historically trained to predict price movement within a $N$-bar lookahead window. It is increasingly being shifted toward **Hindsight-based labeling** using "Perfect Trades."

### 🛡️ The Scaling Gate (AITradingAgent)
ML is not the sole decision-maker. It is one of several factors processed by the [AITradingAgent](file:///home/gary/rsi-macd-bot/trading_bot/core/trading_bot.py#L1458):
1. **Technical Score**: Derived directly from the ML model's prediction probability.
2. **Sentiment Score**: News-based sentiment analysis.
3. **Sector Score**: Relative strength of the symbol's sector ETF (e.g., XLK for Tech).
4. **Macro Score**: Global market regime (Bull/Bear/Volatile) determined by VIX and other macro data.

### ⚙️ Adaptive Sensitivity
The bot uses **Adaptive Thresholding**. Instead of a static $0.5$ threshold, it adjusts based on the [Macro Regime](file:///home/gary/rsi-macd-bot/trading_bot/core/trading_bot.py#L1583):
- **Bull Confident**: Decreases threshold (more aggressive).
- **Bear Defensive**: Increases threshold (more selective).

---

## 2. Performance Tracking & Gap Analysis

The bot features a sophisticated "Closed-Loop" performance analysis system designed for continuous improvement.

### 📊 Key Metric: Efficiency
The primary performance indicator is **Trading Efficiency**, calculated as:
$$\text{Efficiency} = \frac{\text{Actual Return \%}}{\text{Theoretical Max (Perfect) Return \%}}$$

### 🔍 Gap Classification
The [PerformanceAnalyzer](file:///home/gary/rsi-macd-bot/trading_bot/core/performance_analyzer.py) automatically categorizes why the bot missed profit:
- **No Trades Executed**: Signal was missed or the ML gate was too restrictive.
- **Loss Incurred**: Entry was good but stop-loss was hit or the market reversed.
- **Early Exit**: Profit target was hit too early, missing a larger move.
- **Optimal Capture**: The bot successfully captured a significant portion of the move.

### 🔄 Hindsight Learning (Continuous Training)
The bot generates [perfect_trades.csv](file:///home/gary/rsi-macd-bot/data/ml/perfect_trades.csv) after each trading day. These represent the "gold standard" entries and exits that *could* have happened. 
- **Missed Opportunities**: If the ML gate blocked a trade that would have been "perfect," the system can flag this as a false negative and use it to retrain the model.

---

## 3. Findings & Recommendations

### 💡 Strengths
- **Robust Feature Set**: The inclusion of temporal and volatility data makes the model aware of market context beyond simple price action.
- **Defensive Design**: The "fail-open" logic ensures that ML failures (model missing, loading errors) don't crash the bot, though they may reduce performance.

### ⚠️ Identified Gaps (Potential Development Areas)
- **Feature Schema Drift**: If the model is retrained with new features but the bot is not updated, it defaults to zero-filling. We should implement an automated schema validation check.
- **Commission Sensitivity**: Small trades on low-priced stocks can be eaten by commissions. The current MVC (Minimum Viable Capital) logic should be tightly coupled with ML confidence (e.g., only boost to MVC on high-confidence signals).
