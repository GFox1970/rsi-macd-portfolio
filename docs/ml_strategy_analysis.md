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

## 3. Complete Learning Ecosystem

The bot employs a multi-layered, continuous learning architecture that operates in a closed loop:

### 🔄 The Learning Flow

```
LIVE TRADING
    ↓
enhanced_decision_log.jsonl (logs every decision: BUY/SELL/SKIP)
    ↓
    ├─→ ML Pipeline (nightly) → Retrains models based on what worked
    └─→ Sentinel Nightly Pulse (post-market) → Analyzes MISSED opportunities
            ↓
        sentinel_feedback.json → Stores strategic gaps
            ↓
        HealerAgent (nightly) → Generates technical fixes
            ↓
        healer_active_directives.json → Code-level repair instructions
            ↓
        (Applied to trading logic) → Bot improves itself
            ↓
        BACK TO LIVE TRADING (improved)
```

### 📊 Layer 1: Operational Truth (`enhanced_decision_log.jsonl`)
**Purpose**: Real-time log of ALL trading decisions  
**Who Uses It**: MLDataCollector, SentinelAgent, Dashboard  
**Frequency**: Every trading decision  
**Value**: Ground truth for "What did I do and why?"

### 🧠 Layer 2: Tactical Learning (ML Pipeline)
**Purpose**: Retrains XGBoost models based on historical outcomes  
**Data Source**: `enhanced_decision_log.jsonl` + historical price data  
**Frequency**: Nightly via `daily_orchestrator.py`  
**Value**: Learns "What technical patterns lead to profit?"

### 🕵️ Layer 3: Strategic Audit (Sentinel Agent)
**Purpose**: POST-MORTEM analysis of missed opportunities  
**Data Source**: `PerformanceAnalyzer` (actual vs. potential performance)  
**Frequency**: Nightly, after market close  
**Triggers**: HealerAgent  
**Value**: Identifies "What strategic gaps exist?" (e.g., overly conservative VOLATILE mode)

**LLM Integration**:
- **Vertex AI (Gemini)**: Analyzes performance gaps and generates strategic recommendations in natural language
- **Audit Scope**: Reviews `enhanced_decision_log.jsonl`, compares actual P&L vs. "perfect trades," identifies why opportunities were missed

### 🔧 Layer 4: Autonomous Repair (HealerAgent)
**Purpose**: Translates Sentinel findings into code-level fixes  
**Data Source**: `data/sentinel_feedback.json`  
**Frequency**: Nightly, after Sentinel Pulse  
**Output**: `data/healer_active_directives.json`  
**Value**: "How do I fix it?" - Autonomous code repair based on strategic failures

**LLM Integration**:
- **Vertex AI (Gemini)**: Used by HealerAgent to analyze Sentinel findings and generate technical directives (e.g., "Adjust VOLATILE mode weights in `trading_agent.py` to prioritize technical_score > 0.85")
- **Antigravity AI Agent**: Can be invoked (manually or via future automation) to apply directives by editing code, running tests, and committing fixes
- **Token Management**: Vertex AI requests are rate-limited; HealerAgent includes retry logic and fallback to manual directive creation

**Directive Structure**:
Each directive includes:
- `target_symbol`: Symbol the fix applies to (or "GLOBAL")
- `target_file`: File path to modify
- `problem`: Description of strategic gap
- `fix`: Step-by-step repair instructions
- `verification_plan`: How to validate the fix
- `can_auto_apply`: Boolean flag for safe automation

### 🎯 Integration Points
- **Daily Orchestrator**: Runs the complete learning cycle nightly
  1. ML Pipeline retraining
  2. Sentinel Pulse (performance audit)
  3. Healer Pulse (directive generation)
- **Live Trading**: Applies learned improvements in real-time
- **Dashboard**: Displays all decision logs and audit results

### 📋 Deprecated Components
- **monitor-health.yml (4-hour GitHub Actions workflow)**: Archived as redundant. The integrated nightly pulse in `daily_orchestrator.py` provides superior timing and automation.

---

## 4. Findings & Recommendations

### 💡 Strengths
- **Robust Feature Set**: The inclusion of temporal and volatility data makes the model aware of market context beyond simple price action.
- **Defensive Design**: The "fail-open" logic ensures that ML failures (model missing, loading errors) don't crash the bot, though they may reduce performance.
- **Self-Healing Loop**: The Sentinel + Healer integration enables autonomous strategic improvement without manual intervention.

### ⚠️ Identified Gaps (Potential Development Areas)
- **Feature Schema Drift**: If the model is retrained with new features but the bot is not updated, it defaults to zero-filling. We should implement an automated schema validation check.
- **Commission Sensitivity**: Small trades on low-priced stocks can be eaten by commissions. The current MVC (Minimum Viable Capital) logic should be tightly coupled with ML confidence (e.g., only boost to MVC on high-confidence signals).
