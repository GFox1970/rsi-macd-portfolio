# Developer Onboarding Guide

## 1. Local Environment Setup
To get started with development, follow these steps:
1.  **Clone the Repo**: `git clone https://github.com/GFox1970/rsi-macd-bot.git`
2.  **Virtual Environment**:
    ```bash
    python3 -m venv venv
    source venv/bin/activate
    pip install -r requirements.txt
    ```
3.  **Environment Variables**: Copy `.env.example` to `.env` and populate your developer API keys:
    - **Trading APIs**: Alpaca (paper trading), IBKR credentials
    - **Vertex AI**: Follow the [Vertex AI Setup Guide](vertex_ai_setup.md) to configure Google Cloud credentials for AI features
    
4.  **Vertex AI Setup (Required for AI Features)**:
    - Complete the [Vertex AI Setup Guide](vertex_ai_setup.md) to enable:
      - Sentinel agent (autonomous monitoring)
      - News analysis
      - Strategic AI recommendations
      - Thematic research
    
5.  **Launch Stack**: Use Docker Compose to bring up dependencies.
    ```bash
    docker-compose up -d prometheus loki grafana
    ```
6.  **Broker Prerequisites**:
    -   **Alpaca**: Paper trading API keys generated.
    -   **IBKR**: **IB Gateway** or **TWS** installed and logged in on the same network.
        -   Enable "ActiveX and Socket Clients" in API Settings.
        -   Ensure Trusted IPs include `127.0.0.1` (or your Docker network range).

## 2. Coding Standards
-   **Style**: Adhere to **PEP 8**.
-   **Formatting**: Use `black` or `ruff format`.
-   **Linting**: Run `ruff check .` before committing.
-   **Documentation**: All new functions should have Google-style docstrings.
-   **Type Hinting**: Required for all public-facing method signatures.

## 3. Project Structure
-   `/trading_bot`: The core engine. Logic changes go here.
-   `/ml_pipeline`: Retraining and inference logic.
-   `/tests`: All test suites. Coverage must not drop after a PR.
-   `/app`: Streamlit dashboard and webserver code.

## 4. Branching Strategy
We use a simplified **Gitflow** model:
-   `main`: Production-ready code only.
-   `develop`: The stability branch for the next release.
-   `feature/*`: Branch off `develop` for new features.
-   `fix/*`: Branch off `develop` or `main` (for hotfixes).

## 5. Contribution Workflow
1.  **Sync**: Ensure your `develop` branch is up to date.
2.  **Branch**: `git checkout -b feature/awesome-new-indicator`.
3.  **Test**: Write a corresponding test in `/tests`.
4.  **Lint**: Run `ruff check`.
5.  **Verify**: Ensure `./venv/bin/pytest` passes 100%.
6.  **PR**: Open a Pull Request against the `develop` branch. Tag @Gary for review.

## 6. Resources
-   **Vertex AI Setup Guide**: [Vertex AI Setup](vertex_ai_setup.md)
-   **Alpaca API Documentation**: [https://alpaca.markets/docs/](https://alpaca.markets/docs/)
-   **XGBoost Documentation**: [https://xgboost.readthedocs.io/](https://xgboost.readthedocs.io/)
-   **Google Cloud Vertex AI**: [https://cloud.google.com/vertex-ai/docs](https://cloud.google.com/vertex-ai/docs)
-   **Mermaid Live Editor**: For updating architecture diagrams in documentation.
