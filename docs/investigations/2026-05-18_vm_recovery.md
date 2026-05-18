# VM Recovery — May 2026 (Disk, Healer, IBKR Data)

**Date:** 2026-05-18  
**Commit:** `5713c568` — *Fix IBKR data reliability and restore trading agent after Healer incident.*

## Summary

The bot stopped UK (and eventually all) trading due to **stacked operational failures**, not a missing LSE market-data subscription. Recovery restored live IBKR bars for `.L` symbols, fixed the daily orchestrator, and added disk/Healer safeguards.

## Root causes

| Issue | Impact | Fix |
| :--- | :--- | :--- |
| **Disk 100% full** | Logging failed; YFinance `disk I/O error`; Docker exec failed | Freed ~3GB (caches, logs); user ran `sudo` Docker log truncate; `vm_disk_guard.sh` cron |
| **`trading_agent.py` wiped (0 bytes)** | Orchestrator `ImportError` since ~14 May; no candidate refresh | Restored from git; Healer auto-merge disabled on VM |
| **Ephemeral IBKR connections** | `TimeoutError` on every historical fetch; fallback to delayed YFinance | `IBKRBroker.get_historical_data()` uses **main session** (commit `5713c568`) |
| **`mock_support_resistance` undefined** | Every buy evaluation crashed on UK/US symbols | `_support_resistance()` in `trading_agent.py` |
| **Stale `latest_intraday_handoff.csv`** | 8+ days old targets | Orchestrator re-run after import fix |

## What was *not* the cause

- **LSE subscription inactive** — User confirmed subscription active; logs showed `✅ IAG.L: Using IBKR real-time data` after fix (May 2026).
- **Staleness guard alone** — Guard is working as designed; entries were blocked earlier because data never arrived (IBKR timeout + full disk).

## Code & ops changes (permanent)

### `trading_bot/core/ibkr_executor.py`

- Historical bars use the **persistent** `self.ib` connection (`reqHistoricalData` on main thread).
- Ephemeral per-symbol connections (random `clientId`) disabled by default; set `IBKR_HISTORICAL_ALLOW_EPHEMERAL=true` only for Streamlit/UI threads if needed.

### `trading_bot/core/trading_agent.py`

- Restored full module after Healer incident (10 May 2026, `BRBY.L` failed auto-fix).
- Local `_support_resistance()` — do not call `mock_support_resistance` from `trading_bot.py` (avoids circular imports).

### `scripts/vm_disk_guard.sh`

- Runs via **cron** on VM: `0 6 * * *` → `logs/disk_guard.log`.
- Cleans only when disk ≥ **92%** (configurable via `DISK_GUARD_THRESHOLD_PCT`).
- No sudo: truncates large app logs, removes Playwright/pip caches, `git gc`.

### `docker-compose.yml`

- `json-file` log rotation: `trading-bot` (50m×3), `ib-gateway` (30m×3).

### Production `.env` (VM)

- `HEALER_AUTO_MERGE=false` — prevent Healer from merging destructive patches without review.

## Manual recovery commands (reference)

```bash
# SSH to VM
ssh hetzner
cd /home/deploy/trading-bot

# Align with main
git fetch origin main && git reset --hard origin/main

# Disk (requires sudo password on VM)
sudo journalctl --vacuum-size=200M
sudo bash -c 'truncate -s 0 /var/lib/docker/containers/*/*-json.log'
df -h /

# Restart core services
docker compose up -d --force-recreate trading-bot ib-gateway

# Verify
docker logs trading-bot --tail 50 | grep -E "IBKR|Using IBKR"
crontab -l | grep vm_disk_guard
```

## Monitoring

- **Disk:** `df -h /` and `tail logs/disk_guard.log` after 06:00 UTC.
- **UK data:** `grep "Using IBKR real-time" logs/trading_bot.log | grep "\.L" | tail`.
- **Orchestrator:** `tail logs/orchestrator_auto.log` — should not show `TradingAgent` import errors.
- **Healer:** `logs/healer_history.jsonl` — failed `BRBY.L` run on 2026-05-10 correlates with empty `trading_agent.py`.

## Related docs

- [Support & Maintenance Guide](../maintenance_guide.md) — §2.5, §2.14–2.16, §4.9
- [Deployment Guide](../deployment_guide.md) — VM cron and disk
- [API Spec](../api_spec.md) — IBKR market data priority
