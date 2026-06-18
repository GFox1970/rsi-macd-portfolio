# Master Design Document

## 1. Executive Summary
The RSI-MACD Trading Bot is a fully autonomous, machine-learning-enhanced trading system designed for consistent profitability in global equities and crypto markets. It utilizes a modular **3-Tier Architecture** to separate daily orchestration (prep/archiving), intraday execution (survival/efficiency), and AI-driven decision intelligence (sentiments/themes).

## 2. Core Vision
Our mission is to eliminate emotional bias from trading by automating the entire lifecycle—from data ingestion and model retraining to order placement and cloud-scale performance preservation.
- **Key Strategy**: RSI/MACD convergence gated by an XGBoost confidence model and refined by the **Strategic Judgement Layer**.
- **Risk Management**: Adaptive survival protocols (VIX-based) and capital allocation guarding.

## 3. High-Level Architecture
The system is divided into three primary tiers:
1.  **Tier 1 (Daily Orchestrator)**: Prepares the battlefield (models, candidates, targets) and scales experience via the **Accelerated Learning Universe (ALU)**.
2.  **Tier 2 (Intraday Engine)**: The continuous market loop evaluating signals, surviving volatility, and executing trades.
3.  **Tier 3 (AI Intelligence Layer)**: Real-time sentiment analysis, thematic rotation, and strategic appetite tuning.

For a detailed breakdown, see the [System Architecture Document](system_architecture.md).

## 4. Key Documentation Links
This document serves as the hub for the following specialized guides:

| Document | Purpose |
| :--- | :--- |
| [Product Vision & Requirements (PRD)](prd.md) | Business goals, user stories, and feature requirements. |
| [Infrastructure & DevOps Guide](devops_guide.md) | Docker setup, CI/CD pipelines, and secret management. |
| [Data Architecture Document](data_architecture.md) | SQLite schema, JSONL logging, and ML feature flows. |
| [API & Integration Specification](api_spec.md) | Alpaca, IBKR, and Polygon.io implementation details. |
| [Testing Strategy Document](testing_strategy.md) | Unit, integration, and E2E testing workflows. |
| [Deployment & Release Guide](deployment_guide.md) | Versioning and rollout/rollback procedures. |
| [Support & Maintenance Guide](maintenance_guide.md) | Troubleshooting, monitoring, disk guard cron, and operational runbooks. |
| [VM Recovery (May 2026)](investigations/2026-05-18_vm_recovery.md) | Disk full, Healer incident, IBKR data fix, production cron. |
| [ML Strategy & Performance Analysis](ml_strategy_analysis.md) | Overview of XGBoost architecture, gated scaling, and performance efficiency metrics. |
| [ML Operational Lifecycle (ALU)](ml_operational_lifecycle.md) | Deep dive into the Accelerated Learning Universe (ALU) and global multi-market handoff logic. |
| [Vertex AI Setup Guide](vertex_ai_setup.md) | Complete guide for configuring Google Cloud Vertex AI authentication and quotas. |
| [Developer Onboarding Guide](onboarding_guide.md) | Local setup, coding standards, and contribution guide. |
| [Intraday profitability checklist](intraday_profitability_checklist.md) | Tracked implementation tasks: net-of-costs sim (`scripts/run_intraday_backtest.py`), execution, intraday risk, universe/regime, live-fill ML loop. |
| [Alpha Optimizer (Top-20 monitor)](dashboard_alpha_optimizer.md) | Streamlit tab: orchestrator ADR bands, strategy board, planned vs actual vs ML log. |
| [LSE scalper operations](lse_scalper_operations.md) | Bot-managed IBKR entries, gated legacy exits, VM cron/archiver. |
| [Broker v2 — LSE Momentum v1](broker_v2_lse_momentum.md) | Active intraday strategy (ORB + momentum), safety guards, go-live gate. |
| [Broker Coach (Phase 1)](broker_coach.md) | Proactive session monitor: docker log parse, momentum refresh, bounded tuning. |
| [Candidate selection approaches](candidate_selection_approaches.md) | Structural vs session-momentum vs catalyst; competition scoring; `meta.approach` tagging. |

## 5. Security & Observability
-   **Security**: Minimalist attack surface with secrets managed via GitHub vault and environment-level isolation.
-   **Observability**: Real-time health monitoring via Streamlit, featuring the "Management Insight" audit trail for full decision transparency.
-   **Self-Healing (Sentinel SRE)**: An autonomous AI agent scans logs every 4 hours to fix technical errors and flag strategic stagnation, ensuring the bot's "thinking" remains sharp.

## 6. Future Roadmap
-   **Enhanced Option Strategies**: Automated hedging using put/call options during Survival Mode.
-   **Autonomous Strategy Switching**: AI-driven selection between Mean Reversion, Trend Following, and Breakout strategies based on detected regime.
-   **Global Multi-Currency P&L**: Unified FX conversion and accounting for all international brokerage accounts.
