> [!NOTE]
> **Portfolio Mirror**: This repository contains the public-facing documentation, architecture, and visual evidence for the RSI-MACD Trading Bot. The core execution engine and proprietary trading strategies are maintained in a secure, private repository.

# RSI-MACD Trading Bot (Phase 3)

A fully autonomous, machine-learning-enhanced trading system designed for consistent profitability in US and Global markets. This system leverages technical analysis (RSI/MACD), XGBoost confidence gating, and a VIX-based Survival Mode.

## 🚀 Key Features (Phase 3)
- **3-Tier Orchestration**: Modular separation of Daily Prep (T1), Intraday Execution (T2), and AI Intelligence (T3).
- **Survival Mode**: Automated macro circuit breaker (VIX > 40) that halts buys and shifts to defensive postures.
- **Fee-Aware Sizing**: Minimum Viable Capital (MVC) logic to ensure trades remain profitable after commissions.
- **Cloud Archiving**: Automated monthly data shipping to Google Drive via `rclone`.
- **Management Insight**: High-fidelity JSONL decision logs displayed in a premium Streamlit dashboard.
- **Multi-Broker**: Unified routing for Alpaca (US), IBKR (Global), and Crypto.

## 📂 Documentation Hub
For detailed guides on the architecture and product vision, please refer to the `docs/` folder:

| Section | Link |
| :--- | :--- |
| **Start Here** | [Master Design Document](docs/index.md) |
| **Vision & Goals** | [Product Requirements (PRD)](docs/prd.md) |
| **Architecture** | [3-Tier System Architecture](docs/system_architecture.md) |
| **Deployment** | [DevOps & Infrastructure Guide](docs/devops_guide.md) |
| **Operations** | [Support & Maintenance Guide](docs/maintenance_guide.md) |
| **Portfolio** | [Visual Showcase & AI Residency](#-visual-portfolio--ai-residency) |

## 🖼️ Visual Portfolio & AI Residency

This project is a flagship demonstration of **AI Residency (AIR)** — the next-generation skill of architecting and orchestrating multiple AI agents to build and maintain production-grade systems. 

The following evidence highlights the technical depth and operational transparency achieved through this human-agent partnership.

````carousel
![01 - Global Strategy & Exposure](docs/portfolio_images/01_dashboard_overview.png)
**Control Center**: The multi-broker dashboard provides a high-level view of portfolio intelligence, equity curves, and real-time P&L tracking across Alpaca and IBKR.
<!-- slide -->
![02 - Intraday Execution Map](docs/portfolio_images/02_intraday_analysis.png)
**Performance Audit**: Detailed performance analysis comparing **Actual vs. Potential P&L**. This section measures the "Capture Gap" — proving execution efficiency against a feasible best-case simulation.
<!-- slide -->
![03 - Strategic Holdings & Multi-Day Thesis](docs/portfolio_images/03_strategic_holdings_cards.png)
**Strategic Logic**: The bot manages multi-day positions with auto-generated "Theses," ensuring every trade has a documented rationale for long-term optimization.
<!-- slide -->
![04 - Strategic Efficiency & Sector Architecture](docs/portfolio_images/04_strategic_efficiency.png)
**Risk Management**: Portfolio concentration and sector-level performance metrics ensure the bot maintains a balanced, diversified exposure automatically.
<!-- slide -->
![05 - Global Intelligence & Thematic Research](docs/portfolio_images/05_thematic_research.png)
**AI Research Layer**: Powered by LLMs, this tier scans macro sentiments and global news to identify emerging themes, allowing the system to pivot its bias based on live research.
<!-- slide -->
![06 - Management Insight Decision Logs](docs/portfolio_images/06_decision_logs.png)
**Decision Transparency**: Full auditability. Every decision to BUY, SELL, or SKIP is logged with a detailed rationale and confidence score for 100% accountability.
<!-- slide -->
![07 - Sector Rotation Architecture](docs/portfolio_images/07_sector_rotation.png)
**Tactical Edge**: Visualizing relative strength across major market sectors to guide the bot's tactical rotation and allocation strategy.
<!-- slide -->
![08 - Control Audit & Self-Correction Trails](docs/portfolio_images/08_control_audit_logs.png)
**Autonomous Evolution**: Evidence of nightly self-tuning. The bot logs its own parameter adjustments (margins, buffers) based on simulated market gaps.
<!-- slide -->
![09 - Alpha Optimizer](docs/portfolio_images/09_alpha_optimizer.png)
**Closed-Loop Learning**: The Alpha Optimizer screen showing the bot's ability to learn from recent performance and adjust its operational plan for maximum ROCE.
````

### 🚀 The "AI Residency" Approach
Unlike traditional development, this bot was built using **Agentic Orchestration**. As the Human Operator, I acted as the "Digital Architect," guiding multiple AI agents to solve complex engineering challenges:
- **Collaborative Architecture**: Designing the 3-Tier separation of concerns through iterative AI dialogue.
- **Bug-Squashing & Evolution**: Rapidly identifying edge cases (like multi-index data mapping) and implementing fixes in minutes.
- **Systemic Integration**: Managing the connection between XGBoost models, Flask APIs, and Streamlit front-ends as a unified product vision.

*This project is evidence of my ability to lead AI-augmented workflows in any high-stakes environment.*

## 📈 Operational Status
The bot currently operates in a daily cycle:
1. **08:00 EST**: Data ingestion, retraining of XGBoost confidence gates.
2. **09:30 EST**: Deployment of the Intraday trading loop.
3. **16:00 EST**: Automated P&L settlement and risk reporting.
4. **17:00 EST**: Cloud archiving of high-fidelity decision logs.

---
*Developed for Advanced Agentic Trading.*
