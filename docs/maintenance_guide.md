# Support & Maintenance Guide

## 0. Daily Broker Checklist (Pre-Market)
To ensure the bot executes correctly, perform these steps **30 minutes before market open**:
1.  **IBKR Connectivity**:
    -   **Auto-Login**: The `ib-gateway` container is configured to log in automatically using **IBC**.
    -   **Monitoring**: You can monitor the Gateway screen via your browser:
        -   **URL**: `http://<HETZNER_IP>:6080`
        -   **Password**: `secret` (unless changed in `docker-compose.yml`).
    -   **Manual Intervention**: Only required if 2FA is triggered during the daily restart.
    -   Verify the API port is `4002`.
2.  **Alpaca Status**:
    -   Log into the Alpaca web dashboard.
    -   Verify your **Paper Trading** buying power is sufficient.
3.  **Bot Health**:
    -   Check the Streamlit dashboard for a "Connected" status.
    -   Verify that candidates were successfully loaded from the orchestrator.

## 0.1 IP Whitelisting (Hetzner & Docker)
Since everything is running on **Hetzner**, you must ensure the following IPs are whitelisted:

1.  **IBKR Gateway (Internal)**:
    -   The `ib-gateway` container needs to trust the `trading-bot` container.
    -   In the IBKR Gateway settings (**Configuration > API > Trusted IPs**), add:
        -   `127.0.0.1`
        -   `172.16.0.0/12` (Standard Docker internal range) or specifically your bot container's IP.
2.  **IBKR Account Level (External)**:
    -   If you have **IP Restrictions** enabled in your IBKR Client Portal (Account Settings), you must add your **Hetzner Server Public IP**.
3.  **Alpaca**:
    -   Usually does not require IP whitelisting if using standard API keys, but check if you have restricted key access to specific CIDR blocks in the Alpaca dashboard.

## 1. Troubleshooting Checklist
If the bot is not placing trades or the dashboard is stale, follow these steps:
1.  **Check Orchestrator Logs**: `tail -n 100 logs/orchestrator.log`. Verify that the daily sequence completed.
2.  **Verify Candidates**: Ensure `data/top_20_day_trading_candidates.csv` exists and has a current timestamp.
3.  **Check Bot Connectivity**: `docker logs --tail 50 trading-bot`. Look for "Connection Error" or "401 Unauthorized" from Alpaca/IBKR.
4.  **Database Integrity**: Check if `ml_db/ml_data.db` is locked or corrupted.

## 2. Common Failure Modes

### 2.1 API Rate Limiting (429)
-   **Symptom**: "Too Many Requests" in logs.
-   **Resolution**: The bot will auto-retry with exponential backoff. If persistent, reduce `data_refresh_interval_hours` in `config/trading_config.json`.

### 2.2 Stale Market Data
-   **Symptom**: RSI/MACD values are identical for multiple cycles.
-   **Resolution**: Check Polygon.io credit balance. Restart the `trading-bot` container to force a fresh data fetch.

### 2.3 PDT Violations
-   **Symptom**: "Order rejected: Pattern Day Trader" in Alpaca.
-   **Resolution**: Check `pdt_state.json`. If it's out of sync with your broker, manually update the `day_trade_count` to match reality. Note: The bot now allows reversals on profits >3% even with 1 day trade remaining.

### 2.4 ML Schema Drift (Strict Gating)
-   **Symptom**: "MLIntegration: STRICT_SCHEMA active. Missing feature... Blocking trade."
-   **Resolution**: This occurs when the daily data fetcher produces features that don't match the `feature_schema.json` the model was trained on. 
    1. Check `orchestrator.log` for feature engineering errors.
    2. Force a model retrain: `python ml_pipeline/run_pipeline.py --force`.
    3. Verify `ml_db/models/feature_schema.json` matches the training output.

## 3. Monitoring Dashboards
-   **Grafana (Port 3000)**:
    -   **Trading Engine Health**: CPU/RAM usage and loop latency.
    -   **P&L Pulse**: Real-time equity curve and drawdown monitoring.
    -   **ML Performance**: Distribution of prediction confidence vs. trade outcomes.
-   **Streamlit (Port 8501)**:
    -   **Individual Symbol View**: Drill down into specific decision reasons/logic.

## 4. Maintenance Runbooks

### 4.1 Cleaning up Docker Junk (Weekly)
```bash
./scripts/cleanup_docker.sh
```
This removes dangling images and stopped containers to reclaim disk space.

### 4.2 Log Management & Rotation
The system uses a two-tier log management strategy:
1.  **Application-Level Rotation**: `DailyOrchestrator` runs `run_log_rotation()` which uses `log_rotator.py` to archive `trading_bot.log` and `orchestrator.log` when they reach size limits.
2.  **JSONL Decision Logs**: Found in `logs/enhanced_decision_log.jsonl`. These are append-only audit trails. If they grow too large (>500MB), consider manual archiving or clearing:
    -   `cp logs/enhanced_decision_log.jsonl data/archives/`
    -   `truncate -s 0 logs/enhanced_decision_log.jsonl`

### 4.3 Cloud Archive Management (rclone)
Phase 3 introduced automated cloud offloading to Google Drive.
-   **Config**: `rclone` is configured in the project root.
-   **Automation**: The `DailyOrchestrator` runs `run_data_archiver()` daily.
-   **Manual Backup**:
    ```bash
    # Manually trigger a cloud backup of the current month
    ./rclone copy data/archives/ gdrive:TradingBotArchives/Manual -P
    ```
-   **Verification**: Ensure `logs/orchestrator.log` shows "Data Archiver sequence complete."

### 4.4 Updating ML Models
Models are updated automatically by the `daily_orchestrator.py`. If you want to force an update after a period of high volatility:
1.  Manually run the training pipeline: `python ml_pipeline/run_pipeline.py --force`.
2.  Restart the `trading-bot` service/container to load the new model.

### 4.5 Survival Mode Reset
If the bot is stuck in "Survival Mode" because of a stale macro state:
1.  Verify current VIX: `python scripts/check_vix.py`.
2.  If VIX < 40 and the bot still blocks trades, force a macro refresh:
    ```bash
    # Run the macro analyzer manually
    python trading_bot/core/macro_analyzer.py --force
    ```
3.  Restart the `trading-bot` to pick up the `macro.json` update.

### 4.6 Sentinel SRE & Self-Healing
Phase 3.5 introduced the Sentinel Agent for autonomous maintenance:
- **Automation**: Runs every 4 hours via GitHub Actions (`monitor-health.yml`).
- **Activity Log**: `logs/sentinel_activity.jsonl` tracks all agent findings and actions.
- **Health Report**: `weekly_analysis/sentinel_health_report.md` provides a human-readable summary.
- **Manual Run**:
    ```bash
    python agent/sentinel_agent.py
    ```
- **Direct Feedback**: The agent communicates via `config/sentinel_directive.json`. If the bot is behaving unexpectedly (e.g., ignoring entries), check if the Sentinel has issued a "Strategic Reset" directive.
- **Strategy Auditing**: If the Sentinel flags "Strategic Stagnation," it means the bot's parameter tuning is not producing trades. Review `logs/control_audit_log.jsonl` for details.
