# System Architecture Document

## 1. Overview
The Trading Bot follows a **3-Tier Orchestrated Architecture** designed for high modularity, advanced risk management (Survival Mode), and automated data lifecycle management (Cloud Archiving).

## 2. Component Diagram
```mermaid
graph TB
    subgraph "Tier 1: Daily Orchestrator (Cron)"
        A[Daily Orchestrator] -->|1. Post-Mortem| B[Performance Analyzer]
        A -->|2. Auto-Tuning| C[Adaptive Optimizer]
        A -->|3. Strategic Analysis| D[AI Strategic Agent]
        A -->|4. ML Retraining| E[ML Pipeline]
        A -->|5. Sentinel Pulse| SENT[Sentinel Agent]
        A -->|6. Healer Pulse| HEAL[HealerAgent]
        A -->|7. Targets| F[Candidate Selector]
        A -->|8. Cleanup| G[Log Rotator]
        A -->|9. Cloud Sync| H[Data Archiver / rclone]
        SENT -->|Strategic Gaps| HEAL
    end

    subgraph "Tier 2: Intraday Runtime (Docker/Systemd)"
        I[Intraday Orchestrator] -->|Main Loop| J[Trading Bot Engine]
        J -->|Signals| K[Signal Evaluator]
        J -->|Strategy| L[Strategic Judgement Layer]
        J -->|Risk| M[Capital/PDT Manager]
        J -->|Execute| N[Broker Router]
    end

    subgraph "Tier 3: Intelligence & Analytics"
        O[News Analyst Agent]
        P[XGBoost Gating Model]
        Q[Thematic Manager]
        R[Macro Analyzer]
    end

    subgraph "External Systems"
        S[Alpaca / IBKR / Crypto]
        T[Polygon / yfinance / FRED]
        U[Google Drive]
    end

    J -->|Inference| P
    J -->|Sentiment| O
    J -->|Macro Regime| R
    N -->|Orders| S
    K -->|Market Techs| T
    H -->|ZIP Archives| U
    J -->|Audit Trail| V[JSONL Decision Log]
```

## 3. Data Flows
1.  **Overnight Prep (T1)**:
    - **Post-Mortem**: Analyzes yesterday's trades to find "Efficiency Gaps".
    - **Optimization**: Retunes RSI/MACD parameters based on recent performance.
    - **Strategic Agent**: Generates a high-level `strategic_plan.json` (Risk Multiplier & Bias).
    - **Training**: Retrains XGBoost on new outcomes.
2.  **Market Execution (T2)**:
    - **Signals**: Bot calculates tech indicators and fetches macro regime via `MacroAnalyzer`.
    - **Sizing**: `CapitalAllocationManager` applies fee-aware sizing (MVC). Track A trades are boosted only if ML Score >= 0.75.
    - **Survival**: If `VIX > 40`, the bot enters Survival Mode, blocking new buys.
    - **Exits**: `ExitEvaluator` tracks parabolic moves via High-Water Marks, implementing trailing stops and reversal-detection to maximize capture.
3.  **Archiving & Rotation**:
    - Rotated logs and historical CSVs are zipped and shipped to Google Drive monthly via `rclone`.

## 4. Service Boundaries
-   **Broker Router**: Unified interface for Alpaca (US), IBKR (Global), and CCXT (Crypto).
-   **Strategic Judgement Layer**: Decoupled module that combines ML scores, news sentiment, and macro bias. Includes **Strict Schema Gating** to block invalid data.
-   **Exit Evaluator**: Responsible for same-day and overnight exit logic, featuring trailing profit protection and PDT-optimized reversal exits.
-   **Decision Logger**: Thread-safe JSONL persistence for the "Management Insight" dashboard.
-   **Data Archiver**: Stateless wrapper around `rclone` for cloud synchronization.

## 5. Technology Stack
-   **Language**: Python 3.10+
-   **Infrastructure**: Hetzner Dedicated VM (Linux), Systemd (Process monitoring).
-   **Database**: SQLite (`capital_alloc.db`, `ml_features.db`), JSONL (Decision Audit).
-   **ML Framework**: XGBoost (Gating), OpenAI (News/Thematic Analysis).
-   **Broker APIs**: Alpaca-py, IBKR Gateway (Portal/API), CCXT.
-   **Utilities**: `rclone` (Cloud Sync), `zip` (Log bundling).

## 6. Deployment Topology
-   **Host**: Linux VM.
-   **Persistence**: Local `data/` and `logs/` volumes with monthly cloud offloading.
-   **Monitoring**: Streamlit Dashboard for real-time visibility into the "Brain" (Strategic Judgement).
