# Support & Maintenance Guide

## 0. Daily Broker Checklist (Pre-Market)
To ensure the bot executes correctly, perform these steps **30 minutes before market open**:
1.  **Disk space (VM)**:
    -   `df -h /` — alert if **≥ 90%** used (see §2.5, §4.9).
    -   Automated: `scripts/vm_disk_guard.sh` runs daily at **06:00 UTC** via cron (`logs/disk_guard.log`).
2.  **IBKR Connectivity**:
    -   **Auto-Login**: The `ib-gateway` container is configured to log in automatically using **IBC**.
    -   **Monitoring**: You can monitor the Gateway screen via your browser:
        -   **URL**: `http://<VM_HOST>:6080` (production VM removed; use local Docker Compose if testing)
        -   **Password**: Set via `VNC_PASSWORD` in `.env` / `docker-compose.yml` — never commit real values.
    -   **Manual Intervention**: Only required if 2FA is triggered during the daily restart.
    -   **API ports**: Gateway exposes `4002` internally; the **trading-bot** connects to **`ib-gateway:8888`** (socat bridge) per `IBKR_PORT` in `.env` / `docker-compose.yml`.
    -   **UK (LSE)**: Confirm **LSE Equities** market-data subscription is active in IBKR Client Portal (typically £1/mo). Live `.L` bars come from IBKR, not Yahoo.
3.  **Alpaca Status**:
    -   Log into the Alpaca web dashboard.
    -   Verify your **Paper Trading** buying power is sufficient.
4.  **Bot Health**:
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
2.  **Verify GHA-VM Handover**: If harvesting completes on GHA but the VM doesn't start training, check `logs/orchestrator_auto.log` on the VM for argument errors or connection failures.
3.  **Verify Candidates**: Ensure `data/top_20_day_trading_candidates.csv` exists and has a current timestamp.
4.  **Check Bot Connectivity**: `docker logs --tail 50 trading-bot`. Look for "Connection Error" or "401 Unauthorized" from Alpaca/IBKR.
5.  **Database Integrity**: Check if `ml_db/ml_data.db` is locked or corrupted.

## 2. Common Failure Modes

### 2.1 API Rate Limiting (429)
-   **Symptom**: "Too Many Requests" in logs.
-   **Resolution**: The bot will auto-retry with exponential backoff. If persistent, reduce `data_refresh_interval_hours` in `config/trading_config.json`.

### 2.2 Stale Market Data (Staleness Guard)
-   **Symptom**: Logs show `🚨 … EXTREMELY STALE` or `❌ … Skipping BUY EVALUATION`; RSI/MACD unchanged for multiple cycles.
-   **Cause**: `trading_bot.py` blocks **entries** when the last bar is older than a threshold (5 min for live IBKR; 20 min for YFinance/Alpaca/delayed). **Exits** may still run on stale data (by design).
-   **Data priority** (`trading_bot/core/data_factory.py`):
    1.  **IBKR** — all symbols including UK (`.L`), via main connection (see §2.15).
    2.  **Alpaca** — US only (no `.L` / international suffixes).
    3.  **YFinance** — last resort (delayed); often fails when disk is full.
-   **Resolution**:
    1.  Confirm IBKR connected: `docker logs trading-bot --tail 30 | grep "Successfully connected"`.
    2.  Confirm live UK bars: `grep "Using IBKR real-time" logs/trading_bot.log | grep "\.L" | tail`.
    3.  If IBKR fetch fails, fix §2.15 before blaming the LSE subscription.
    4.  Restart `trading-bot` after gateway login or code deploy.

### 2.2.1 Trading regions (LSE focus / US live scanning)
-   **Purpose**: When US intraday data is stale or unavailable, run **LSE (`.L`) only** for ADV candidates and live buy scans. US open positions still receive **exit** evaluation.
-   **Dashboard**: Control Center → **US live scanning (Alpaca)** toggle. Off = LSE-only mode; Alpaca (US Operations) card is hidden; hero metrics show IBKR only.
-   **Config** (`trading_config.json` → `day_trading.trading_regions`):
    ```json
    "trading_regions": {
      "active": ["L"],
      "us_live_scanning_enabled": false
    }
    ```
-   **Env overrides** (optional on VM `.env`):
    -   `US_LIVE_SCANNING_ENABLED=false` — same as toggle off.
    -   `TRADING_REGIONS=L` — restrict active regions (when US scanning is on).
-   **Code**: `trading_bot/core/trading_regions.py`; filters in `get_symbols_to_trade()` and `weekly_analysis/find_top_day_trading_candidates.py`.
-   **Logs**: On bot start: `Trading regions: LSE (.L) only — US live scanning off`. During session: `Buy path: N regional candidates (LSE (.L) only — US live scanning off)`.
-   **Re-enable US**: Turn toggle **on** in Control Center (or set `us_live_scanning_enabled: true` and `active: ["US", "L"]`) after IBKR/Alpaca US bars are reliably live (no routine `Skipping BUY EVALUATION` on stale data).

### 2.3 PDT Violations
-   **Symptom**: "Order rejected: Pattern Day Trader" in Alpaca.
-   **Resolution**: Check `pdt_state.json`. If it's out of sync with your broker, manually update the `day_trade_count` to match reality. Note: The bot now allows reversals on profits >3% even with 1 day trade remaining.

### 2.4 ML Schema Drift (Strict Gating)
-   **Symptom**: "MLIntegration: STRICT_SCHEMA active. Missing feature... Blocking trade."
-   **Resolution**: This occurs when the daily data fetcher produces features that don't match the `feature_schema.json` the model was trained on. 
    2. Force a model retrain: `python ml_pipeline/run_pipeline.py --force`.
    3. Verify `ml_db/models/feature_schema.json` matches the training output.

### 2.6 Float Object Errors ('float' object has no attribute 'get')
-   **Symptom**: "Error in main loop: 'float' object has no attribute 'get'" in logs.
-   **Cause**: Occurs when the bot expects a dictionary (like a `sector_report` score or a `news_sentiment` tuple) but receives a raw float value instead.
-   **Resolution**: Ensure consistent return types in `_get_news_sentiment` and add robust type checking when accessing `sector_report` in `trading_bot.py`.

### 2.7 MLDataCollector 'has no attribute log_decision' (Buy Path Crash)
-   **Symptom**: `'MLDataCollector' object has no attribute 'log_decision'` spamming in `trading_bot.log`; `enhanced_decision_log.jsonl` only contains a rotation header and no real decisions.
-   **Cause**: `trading_bot.py` was mistakenly calling `self.ml_data_collector_instance.log_decision()`. The `MLDataCollector` class does not have a `log_decision` method — that belongs to `EnhancedDecisionLogger`.  This crashes the buy path for every symbol.
-   **Resolution**: Remove all `ml_data_collector_instance.log_decision(...)` calls. Decisions are already logged via the `decision_logger` (`EnhancedDecisionLogger`) object which is the correct API.

### 2.8 IBKR Error 322 — Account Summary Subscription Leak
-   **Symptom**: Persistent `Error 322: Maximum number of account summary requests exceeded`.
-   **Cause**: Repeatedly calling `reqAccountSummary()` in a loop without proper cancellation, leading to a session leak in TWS/Gateway.
-   **Resolution**: The `IBKRBroker` now initiates a single, **long-lived account summary subscription** (using `reqId=9001`) during the connection phase. The `get_account_summary()` method reads from this cached state, preventing request saturation.


### 2.9 Alpaca Position Loading (0 Positions Sync)
-   **Symptom**: `Alpaca: loaded 0 positions` even when positions exist in the web dashboard.
-   **Cause**: Version mismatch between the `alpaca-py` SDK (which uses `get_all_positions`) and legacy SDKs or paper trading setups (which use `list_positions`).
-   **Resolution**: The `BrokerRouter` and `TradingBot` now use a duck-typing approach to try both methods sequentially.

### 2.10 rclone Archive Upload — Config Is a Directory
-   **Symptom**: `Failed to load config file ".../rclone.conf": read ... is a directory` during Data Archiver step.
-   **Cause**: Often Docker: bind-mounting a **single file** that did not exist on the host makes Docker create **`rclone.conf` as a directory** on the host. A real `rclone.conf` file is then impossible at that path until the bad directory is removed.
-   **Resolution**:
    1.  **Compose (recommended)**: Use `RCLONE_CONFIG_DIR` in `.env` — the repo mounts the **whole** `~/.config/rclone` directory into the orchestrator, so `rclone.conf` stays a normal file inside it.
    2.  **If you still have a directory named `rclone.conf`**: On the VM, `rm -rf ~/.config/rclone/rclone.conf` (only if it is the mistaken empty directory), then run `rclone config` or copy a valid `rclone.conf` **file** into `~/.config/rclone/`.

### 2.11 Healer Circuit Breaker (Auto-Execution Skipped)
-   **Symptom**: Logs show `Circuit breaker OPEN: last 3 consecutive directives all failed. Skipping auto-execution.`
-   **Cause**: The Healer skips auto-execution after the last three consecutive directive runs failed (e.g. search/replace mismatches) to avoid repeated failures.
-   **Resolution**: Use the **Run Healer Directives** button in the app sidebar (under **🔧 Healer**). It is passcode-protected (Authorize Session first). The button runs **only** Healer directive execution (no full orchestrator), bypasses the circuit breaker, and is intended for retrying after code or directive fixes.
-   **Where to see Healer log output**:
    -   **Sidebar**: After a run, use the **View last run log** expander under 🔧 Healer for the execution log of that run.
    -   **File**: `logs/healer_history.jsonl` (or `/app/logs/healer_history.jsonl` in Docker). Each line is a JSON record with `execution_log`, `status`, `symbol`, etc.
    -   **Container stdout**: When the app runs in Docker, Healer `logger.info`/`logger.error` lines go to the container’s stdout. Use `docker logs <container_name>` (e.g. `docker logs trading-bot` or `docker logs streamlit`) to see them.

### 2.12 Why the bot sometimes closes positions at a loss (Profit-first options)
-   **What’s going on**: The bot can realise a loss in several ways:
    1. **Bracket stop-loss (IBKR/Alpaca)**  
       When you place a bracket order, a stop-loss order is sent to the exchange. If price hits that level, the **exchange** executes the stop; the bot is not “deciding” to sell at that moment. So volatility can trigger the SL and close the position at a loss.
    2. **Same-day emergency stop**  
       In `exit_elevator`, if a same-day position is down by the configured percentage (e.g. 2%), the bot **does** decide to sell to cap the loss.
    3. **Overnight hard stop**  
       For positions held overnight, a larger loss (e.g. ~3% or 1.5× ADR) triggers a hard stop and the bot sells.
    4. **StagedExit time stop**  
       After a position has been underwater for many days (e.g. 10/15), the Staged Exit protocol can **auto-sell** (CRITICAL/HIGH urgency) to free capital.
-   **Safe Exit is not the cause**  
   If you see “Activating Safe Exit” in logs, the bot **wanted** to sell (e.g. take profit) but the broker said shares were “held” by bracket legs. It then cancels those legs and retries the sell. So we’re not selling *because* we can’t cancel; we’re cancelling so we *can* sell (e.g. at a profit).
-   **Profit-first behaviour (prefer to hold and wait)**  
   To reduce realising small losses on **small LSE positions** (notional in GBP below a threshold), you can use these options under `config.trading_config.json` → `day_trading`:
    - **`profit_first_bracket_sl_min_notional_gbp`** (default `0` = off)  
      If the **order** notional (for LSE: price in pence × 0.01 × qty) is **below** this value in GBP, the bot uses a **wide “catastrophe only”** stop instead of the normal tight stop (so you’re less likely to be stopped out by normal volatility).
    - **`profit_first_catastrophe_stop_pct`** (default `0.25`)  
      When the above is active, stop is placed at this % below entry (e.g. 0.25 = 25% down). Only used for LSE when the order notional is below the min.
    - **`emergency_sl_min_notional_gbp`** (default `0` = off)  
      Same-day **emergency** stop-loss is **skipped** when the **position** notional (LSE, in GBP) is below this. So small same-day positions are not sold at a small loss; the bot holds and waits.
    - **`profit_first_staged_exit_min_notional_gbp`** (default `0` = off)  
      StagedExit **auto-sell** (time-based CRITICAL/HIGH) is **skipped** when the position notional (LSE, GBP) is below this; the recommendation is still persisted for the dashboard.
-   **Example**  
  Set `profit_first_bracket_sl_min_notional_gbp` to e.g. `500` and `emergency_sl_min_notional_gbp` to `500`. Then any LSE order/position with notional < £500 will: use a 25% catastrophe stop on the bracket (instead of ~2%), and will not be sold by the same-day emergency stop. You can still close manually or via TP/trailing logic.
-   **Homework (actual vs could-have-done)**  
  The bot's "checking its own homework" is done in `PerformanceAnalyzer.analyze_efficiency()`, which compares actual P&L to theoretical max (perfect long/short day trade) for a symbol/date. Use that to review "what we did vs what we could have done."

### 2.13 Volume regime and regime-based profit-first (US + UK)
-   **Volume regime**  
  The bot classifies each symbol’s current volume vs its 20-day average as **low** (ratio < threshold), **normal**, or **high**. This is used for profit-first gating and optional size dampening.
-   **Config** (under `day_trading`):
    - **`volume_regime_low_threshold`** (default `0.8`): Ratio below this vs 20d avg ⇒ "low".
    - **`volume_regime_high_threshold`** (default `1.2`): Ratio above this ⇒ "high".
    - **`volume_regime_low_size_mult`** (default `0.8`): When volume regime is "low", order size is multiplied by this (e.g. 0.8 = 80% size).
-   **Regime-based profit-first (US + UK)**  
  When **`profit_first_in_volatile_regime`** is `true`, the bot applies profit-first (catastrophe stop, skip emergency SL, skip StagedExit auto-sell for small positions) in **both** US and UK when any of:
  - Macro regime is **VOLATILE_CAUTION**, or
  - VIX ≥ **`profit_first_vix_threshold`** (default `20`), or
  - Volume regime is **low**.
  Notional thresholds:
  - **`profit_first_bracket_sl_min_notional_usd`** (default `500`): US order notional below this ⇒ catastrophe stop on bracket.
  - **`emergency_sl_min_notional_usd`** (default `0`): US position notional below this ⇒ skip same-day emergency SL.
  - **`profit_first_staged_exit_min_notional_usd`** (default `0`): US position notional below this ⇒ skip StagedExit auto-sell (recommendation still persisted).
  So you can compare US vs UK with profit-first only in volatile/low-volume conditions by setting `profit_first_in_volatile_regime: true` and the USD thresholds (e.g. `500`).

### 2.5 Disk Space Exhaustion (Hetzner VM)
-   **Symptom**: `No space left on device`, `disk I/O error` (YFinance), `docker exec` failures, unhealthy containers.
-   **Typical culprits**: Docker container JSON logs under `/var/lib/docker/containers/` (requires **sudo** to truncate), unused Docker images, `~/.cache/ms-playwright`, large `logs/*.jsonl`.
-   **Automated guard (deploy user, no sudo):**
    -   Script: `scripts/vm_disk_guard.sh`
    -   **Cron (production VM):** `0 6 * * * /home/deploy/trading-bot/scripts/vm_disk_guard.sh >> /home/deploy/trading-bot/logs/disk_guard.log 2>&1`
    -   Runs cleanup only when disk use ≥ **92%** (`DISK_GUARD_THRESHOLD_PCT` to override).
-   **Resolution:**
    1.  **Inspect:** `bash scripts/disk_audit_vm.sh` (or `DEPLOY_DIR=/home/deploy/trading-bot bash scripts/disk_audit_vm.sh`).
    2.  **No-sudo quick fix:** `bash scripts/vm_disk_guard.sh` (or lower threshold: `DISK_GUARD_THRESHOLD_PCT=80 bash scripts/vm_disk_guard.sh`).
    3.  **Sudo (Docker logs — most effective):**
       ```bash
       sudo journalctl --vacuum-size=200M
       sudo bash -c 'truncate -s 0 /var/lib/docker/containers/*/*-json.log'
       df -h /
       ```
    4.  **Emergency:** `sudo bash scripts/disk_free_emergency_vm.sh`
    5.  **Prevention:** `docker-compose.yml` sets `logging.options.max-size` / `max-file` on `trading-bot` and `ib-gateway`. Recreate containers after compose changes: `docker compose up -d --force-recreate trading-bot ib-gateway`.
    6.  **Deploy workflow:** Also prunes archives, old CSVs, and large logs when **Deploy to VM** runs from GitHub Actions.
-   **Incident reference:** [2026-05-18 VM Recovery](investigations/2026-05-18_vm_recovery.md)

### 2.14 Healer Wiped `trading_agent.py` (Orchestrator Down)
-   **Symptom**: `ImportError: cannot import name 'TradingAgent'` in `logs/orchestrator_auto.log`; `trading_bot/core/trading_agent.py` is **0 bytes** on the VM.
-   **Cause**: Healer auto-fix failed on 2026-05-10 (`BRBY.L`, ambiguous search/replace) and left an empty file. Orchestrator cannot import; `latest_intraday_handoff.csv` goes stale.
-   **Resolution**:
    1.  Restore from git: `git fetch origin main && git reset --hard origin/main` in `/home/deploy/trading-bot`.
    2.  Set **`HEALER_AUTO_MERGE=false`** in VM `.env` (production default after May 2026 recovery).
    3.  Re-run orchestrator: `./scripts/run_orchestrator.sh` or wait for cron.
    4.  Do **not** enable Healer auto-merge on production without review.

### 2.15 IBKR Historical Fetch Timeouts (UK + US)
-   **Symptom**: `Threaded IBKR fetch failed for *.L`, then `Using YFinance fallback (delayed)`; no UK trades despite active LSE subscription.
-   **Cause (fixed May 2026):** Each bar request opened a **new** IBKR connection with a random `clientId`, overwhelming the gateway (`TimeoutError` after connect/disconnect).
-   **Resolution (current code):** `IBKRBroker.get_historical_data()` uses the **main** session on the trading-bot thread (`IBKR_HISTORICAL_USE_MAIN=true`, default). Verify:
    ```bash
    grep "Using IBKR real-time" logs/trading_bot.log | grep "\.L" | tail -5
    ```
-   **Streamlit/UI only:** Set `IBKR_HISTORICAL_ALLOW_EPHEMERAL=true` if a secondary thread must fetch bars (not recommended on production VM).

### 2.16 Buy Path Crash — `mock_support_resistance` / `TradingAgent`
-   **Symptom**: `Error in _process_symbol_buy_path: name 'mock_support_resistance' is not defined` for every symbol.
-   **Cause:** `trading_agent.py` called a helper defined only in `trading_bot.py` (import cycle). Broke all new entries until May 2026 fix.
-   **Resolution:** Ensure `trading_agent.py` defines `_support_resistance()` locally (commit `5713c568+`). Deploy and `docker compose restart trading-bot`.

## 3. Monitoring Dashboards
-   **Grafana (Port 3000)**:
    -   **Trading Engine Health**: CPU/RAM usage and loop latency.
    -   **P&L Pulse**: Real-time equity curve and drawdown monitoring.
    -   **ML Performance**: Distribution of prediction confidence vs. trade outcomes.
-   **Streamlit (Port 8501)**:
    -   **Individual Symbol View**: Drill down into specific decision reasons/logic.

## 4. Maintenance Runbooks

### 4.1 Cleaning up Docker Junk (Weekly & Emergency)
```bash
# Emergency reclamation (clears cache and unused images)
docker system prune -a -f

# Specific project cleanup
./scripts/cleanup_docker.sh
```
The Hetzner production VM has a **38GB** root volume (`/dev/sda1`). Docker images alone can use ~20GB; **container JSON logs** often fill the rest until truncated with sudo (see §2.5).

### 4.9 Automated Disk Guard (Cron)
Installed on the production VM (`deploy` user crontab):
```cron
0 6 * * * /home/deploy/trading-bot/scripts/vm_disk_guard.sh >> /home/deploy/trading-bot/logs/disk_guard.log 2>&1
```
- **Manual run:** `bash /home/deploy/trading-bot/scripts/vm_disk_guard.sh`
- **Log:** `logs/disk_guard.log`
- **Note:** Does not replace periodic **sudo** Docker log truncation when disk is critically full; use §2.5 step 3.

### 4.2 Log Management & Rotation
The system uses a two-tier log management strategy:
1.  **Application-Level Rotation**: `DailyOrchestrator` runs `run_log_rotation()` which uses `log_rotator.py` to archive `trading_bot.log` and `orchestrator.log` when they reach size limits.
2.  **JSONL Decision Logs**: Found in `logs/enhanced_decision_log.jsonl`. These are append-only audit trails. If they grow too large (>500MB), consider manual archiving or clearing:
    -   `cp logs/enhanced_decision_log.jsonl data/archives/`
    -   `truncate -s 0 logs/enhanced_decision_log.jsonl`

### 4.3 Cloud Archive Management (rclone)
Automated cloud offloading to Google Drive keeps the VM lean.
-   **Config**: `rclone` remote `gdrive:TradingBotArchives`. On the Hetzner VM, add to `.env`:
    `RCLONE_CONFIG_DIR=/home/deploy/.config/rclone` (directory containing `rclone.conf`; **not** the path to the file alone — see §2.10). Without this, the default `/home/gary/.config/rclone` is used, which may not exist on the VM and causes "Data Archiver sequence finished with issues."
-   **Automation**: The `DailyOrchestrator` runs `run_data_archiver()` daily. Retention is **3 days** (files older than 3 days are zipped, uploaded, then deleted). When free disk &lt;2GB the archiver uses **1-day** retention and deletes any old zips in `archives/`. Archived: `data/historical`, `data/trades_exports`, `weekly_analysis` CSVs, rotated logs, sentinel reports.
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

### 4.6 Sentinel SRE & Reality-Sync
Phase 3.5 introduced the Sentinel Agent for autonomous maintenance and **Reality-Sync** auditing:
- **Automation**: Runs nightly via `daily_orchestrator.py` and via the standalone `scripts/sentinel_health_check.py`.
- **Reality-Sync**: Automatically detects if critical logs are stale (e.g., bot hasn't updated in 3h) or if the broker's positions drift from the bot's internal tracking.
- **Activity Log**: `logs/sentinel_activity.jsonl` tracks all agent findings and actions.
- **Health Report**: Available in the Dashboard under Health & Audit → Sentinel Audits.
- **Manual Audit**:
    ```bash
    # Run deep health check (throttled to 12h)
    python scripts/sentinel_health_check.py

    # Force immediate health check
    python scripts/sentinel_health_check.py --force
    ```
- **Direct Feedback**: The agent communicates via `data/sentinel_feedback.json`. If the bot is behaving unexpectedly (e.g., ignoring entries), check if the Sentinel has issued a "Strategic Reset" directive.
- **Strategy Auditing**: If the Sentinel flags "Strategic Stagnation," it means the bot's parameter tuning is not producing trades. Review `logs/control_audit_log.jsonl` for details.

### 4.7 Healer Autonomous Repair
The Healer Agent can apply code fixes based on Sentinel findings. **On the production VM, auto-merge is disabled** (`HEALER_AUTO_MERGE=false` in `.env`) after a May 2026 incident where a failed fix **zeroed `trading_agent.py`** and stopped the orchestrator for days.

**How It Works** (when enabled):
1. **Detection**: Sentinel identifies a logic gap or performance issue.
2. **Fix Generation**: Healer generates a specific code patch.
3. **Auto-Application**: The patch is applied to a new branch, syntax-checked, and committed.
4. **Auto-Merge & Push** (optional): If `HEALER_AUTO_MERGE=true` and validation passes, merged to `main` and pushed.
5. **Deployment**: The VM pulls the updated code (via standard git operations) for the next run.

**Production recommendation:** Keep `HEALER_AUTO_MERGE=false`. Review directives in the dashboard; merge fixes via PR or manual `git pull` after inspection.

**Monitoring Execution**:
- **Dashboard**: "Health & Audit" → "Healer Directives" tab updates in real-time.
- **Git History**: You will see commits authored by `healer-bot` in the repository history.
- **Activity Logs**: `logs/healer_history.jsonl` tracks every step (Generations -> Execution -> Git Push).

**Human Oversight (The "Kill Switch")**:
While the system is autonomous, you retain control:
- **Overrides**: If the Healer applies a fix you disagree with, simply `git revert` the commit locally and push.
- **Strict Mode**: To disable auto-merging for a specific directive type, edit `agent/healer_config.yaml` (if applicable) or set `can_auto_apply: false` in the dashboard for pending directives.

**Rollback Protocols**:
- **Automatic**: If the Healer's fix fails syntax validation or unit tests (if configured), it self-reverts and logs an error.
- **Manual**: Use standard git commands to revert bad commits.
    ```bash
    git log --author="healer-bot"
    git revert <commit_hash>
    ```

**Troubleshooting**:
- **"Ghost Directives"**: If a fix is applied but the issue persists, check if the Healer is stuck in a loop trying to apply the same fix. The system has deduplication logic to prevent this, but check `logs/healer_history.jsonl`.
- **Git Conflicts**: In rare cases, the Healer might hit a merge conflict. It will abort and alert via the dashboard. You must manually resolve these conflicts.
- **Vertex AI errors**: Check service account permissions for Gemini API access

### 4.8 Agent Skills (New Lifecycle)
To enhance efficiency, the bot now leverages an **Agent Skills** system ([agent_skills.md](file:///home/gary/rsi-macd-bot/.agent/agent_skills.md)).

**Data-Harvester Skill**:
- **Purpose**: Automates the pulling of production runtime context (databases, logs, AI intents, ML data) from the VM to the local repository.
- **Usage**: Run `./scripts/pull_vm_data.sh` to synchronize local datasets with production.

**Healer-Sync Skill**:
- **Purpose**: Automatically pulls overnight code fixes generated by the Healer Agent on the VM.
- **Usage**: At the start of a session, run `./scripts/sync_healer.sh` or ask Antigravity to *"Sync with VM fixes."*

**Docs-Sync Skill**:
- **Purpose**: Automates synchronization of documentation to the public `portfolio-mirror` repository.
- **Usage**: After any documentation change, run `./scripts/sync_portfolio.sh` or ask Antigravity to *"Mirror these doc updates."*
