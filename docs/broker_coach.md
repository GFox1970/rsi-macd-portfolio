# Broker Coach — Phase 1

Live session monitor for **LSE Momentum v1**. Replaces the legacy Gemini/Sentinel/Healer loop for strategy tuning during validation.

## What it does

Every coach run (UTC 08:00–16:00, Mon–Fri):

1. Tails **`docker logs trading-bot-scalper`** (not legacy `trading_bot.log`)
2. Parses v1 signals, scalper gate blocks, stale-data blocks, watchlist age
3. **Auto-runs session momentum scan** if CSV is stale (>3.5h) or watchlist <5 symbols
4. Applies **one bounded parameter change** when v1 skip patterns warrant it
5. Writes `data/shared/v1_pilot_overrides.json` and appends `logs/broker_coach.jsonl`

Runs on the **VM host** (needs `docker` CLI). Do not run coach inside the scalper container.

## Auto-actions

| Trigger | Coach action |
| :--- | :--- |
| Momentum CSV age > 3.5h or <5 symbols | Run `session_momentum_scan.py` inside `trading-bot-scalper` via `docker exec` (host has no numpy) |
| ≥8 scalper max-position blocks, 0 v1 eval | `alert_max_positions` |
| ≥8 stale-data blocks in scalper logs | `alert_stale_data` |
| Any `Strategic Decision for *.L: BUY` while v1 active | `alert_legacy_leak` |
| ≥15 volume v1 skips, 0 buys after 10:00 | Loosen `volume_spike_min` −0.1 |

## Tunable parameters (bounds)

| Parameter | Min | Max |
| :--- | :--- | :--- |
| `volume_spike_min` | 1.2 | 2.0 |
| `stop_loss_pct` | 0.006 | 0.012 |
| `take_profit_pct` | 0.010 | 0.020 |
| `top_n_momentum` | 3 | 7 |
| `entry_window_end_london` | — | cap 12:00 (+15m steps) |

## Rules (Phase 1)

| Condition | Action |
| :--- | :--- |
| ≥15 volume skips, 0 buys, after 10:00 London | Loosen `volume_spike_min` by 0.1 |
| ≥10 below-ORB skips, 0 buys, after 11:00 | Extend entry window end +15m (≤12:00) |
| ≥3 buys today | Tighten `volume_spike_min` by 0.1 (quality filter) |
| No v1 lines + ≥5 infra/stale errors | Alert only — fix IBKR/data first |

## VM cron

```cron
*/15 8-16 * * 1-5 /home/deploy/trading-bot/scripts/run_broker_coach.sh
```

Manual (host — uses stdlib only, no venv pandas required):

```bash
cd /home/deploy/trading-bot
python3 agent/broker_coach.py --force --json
```

Or via cron wrapper (prefers `venv`, else scalper container, else host `python3`):

```bash
/home/deploy/trading-bot/scripts/run_broker_coach.sh
```

## Config (`trading_config.json`)

```json
"broker_coach": {
  "enabled": true,
  "auto_apply": true,
  "auto_refresh_momentum": true,
  "momentum_stale_hours": 3.5,
  "min_watchlist_symbols": 5,
  "scalper_container": "trading-bot-scalper"
}
```

## Files

| File | Role |
| :--- | :--- |
| `agent/broker_coach.py` | Coach logic |
| `trading_bot/strategy/v1_pilot_overrides.py` | Override I/O + bounds |
| `trading_bot/core/broker_coach_ui.py` | Streamlit panel |
| `scripts/run_broker_coach.sh` | Cron wrapper |
| `data/shared/v1_pilot_overrides.json` | Live overrides (runtime) |
| `logs/broker_coach.jsonl` | Audit trail |

## Phase 2 (future)

Optional LLM summary of the structured metrics — **shadow mode only**, no auto-apply without explicit flag.
