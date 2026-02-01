# Infrastructure & DevOps Guide

## 1. Environments
- **Local Development**: Used for feature development, unit testing, and initial dry-runs. Managed via Docker Compose.
- **Paper Trading (Staging)**: Deployed to a dedicated Hetzner VM. Executes trades on Alpaca/IBKR Paper accounts.
- **Live Trading (Production)**: Future phase. Dedicated, hardened environment with restricted access.

## 2. Infrastructure as Code (IaC)
The system's infrastructure is defined primarily through **Docker Compose**.
- `docker-compose.yml`: Defines the core services (Orchestrators, Trading Bot, Brokers, Observability).
- `supervisord.conf`: Manages process lifecycle and recovery within the main trading-bot container.

## 3. CI/CD Pipelines
Automated workflows are managed via **GitHub Actions**:
- **Continuous Integration (ci.yml)**: Runs unit and integration tests on every push and PR.
- **Scheduled Orchestration (scheduled-orchestrator.yml)**: Triggers the daily pre-market cycle.
- **Deployment (deploy-to-vm.yml)**: Uses `rsync` and `ssh` to sync code to the Hetzner VM and restart services.
- **Docker Auto-Push (docker-autopush.yml)**: Builds and pushes images to Docker Hub (`gfox1970/trading-bot`).

## 4. Secrets Management
- **GitHub Secrets**: Stores critical API keys (`ALPACA_API_KEY_ID`, `IBKR_USER`, `GITHUB_TOKEN`, etc.).
- **Environment Variables**: Injected via `.env` file (processed by Docker Compose).
- **Hardening**: `.env` files and logs are explicitly ignored by Git. VM file permissions are restricted to the `1000:1000` user.

## 5. Observability Stack
A comprehensive monitoring suite is built into the system:
- **Prometheus**: Scrapes metrics from the bot (PnL, decision confidence, system health).
- **Loki**: Centralizes logs from all containers for high-performance querying.
- **Grafana**: The primary visualization layer for real-time dashboards and alerting.
- **Healthchecks**: Docker-native healthchecks verify that services (Webserver, Streamlit) are responsive.

## 6. Disaster Recovery
- **Persistence**: All critical data (ML models, decision logs, positions) is stored in host-mounted volumes (`./logs`, `./ml_db`, `./data`).
- **Backups**: Automated cron jobs on the VM take periodic snapshots of the `./ml_db` and `./logs` directories.
- **Rollback**: Triggered by redeploying the previous stable Git commit via the `deploy-to-vm.yml` workflow.
