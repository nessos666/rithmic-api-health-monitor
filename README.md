<p align="center">
  <h1 align="center">Rithmic API Health Monitor</h1>
  <p align="center">
    <strong>File-based health monitoring for Rithmic futures trading API — 10 checks, Trust Score 0-100, zero interference with your live session.</strong>
  </p>
  <p align="center">
    <a href="#why-file-based">File-Based Approach</a> · <a href="#the-10-checks">The 10 Checks</a> · <a href="#trust-score">Trust Score</a> · <a href="#quick-start">Quick Start</a> · <a href="#configuration">Configuration</a>
  </p>
</p>

<p align="center">
  <img src="https://img.shields.io/badge/python-3.10+-blue?logo=python&logoColor=white" alt="Python 3.10+">
  <img src="https://img.shields.io/badge/license-MIT-green" alt="MIT License">
  <img src="https://img.shields.io/badge/checks-10_automated-orange" alt="10 Checks">
  <img src="https://img.shields.io/badge/architecture-file--based-purple" alt="File-based">
  <img src="https://img.shields.io/badge/dependencies-stdlib_only-success" alt="Stdlib only">
  <img src="https://img.shields.io/github/stars/nessos666/rithmic-api-health-monitor?style=social" alt="Stars">
</p>

---

## Why File-Based?

Rithmic has a hard constraint: **only ONE active session per user account**. If a health monitor opened its own Rithmic connection, it would either:

1. **Disconnect your live trading scanner** — the second session kicks the first one
2. **Get rejected** — Rithmic refuses the second connection, health monitor shows false "DOWN"
3. **Consume a second seat license** — additional cost for monitoring

**This scanner avoids all three problems.** It doesn't connect to Rithmic at all. Instead, it reads the status files that your live scanner *already* writes and checks system-level signals (process tables, log file age, file timestamps).

```
┌─────────────────┐                ┌──────────────────┐
│ Your Live       │  writes JSON   │ Health Scanner    │
│ Trading Scanner │ ──────────────→│                   │
│ (Rithmic conn)  │   status.json  │ Reads status file │
│                 │                │ Checks pgrep      │
│                 │                │ Checks log age    │
│                 │                │ Checks file mtime │
└─────────────────┘                │                   │
                                   │ NO Rithmic conn!  │
                                   └──────┬────────────┘
                                          │
                                   Trust Score 0-100
                                          │
                              ┌───────────┼───────────┐
                              │           │           │
                        ┌─────┴─────┐ ┌───┴───┐ ┌────┴────┐
                        │  JSON     │ │ n8n   │ │ Grafana │
                        │  Export   │ │ Alert │ │ Dash    │
                        └───────────┘ └───────┘ └─────────┘
```

**Zero interference. Full visibility. No extra Rithmic license needed.**

### Comparison: File-based vs Direct Connection

| Aspect | File-based (this) | Direct connection |
|--------|-------------------|-------------------|
| Rithmic session required | **No — reads existing files** | Yes — second session |
| Risk to live scanner | **None** | May disconnect live session |
| Extra license cost | **$0** | May need 2nd seat |
| Detects scanner crashes | **✅ Yes** (via pgrep + log age) | ❌ No (separate connection) |
| Detects Rithmic outages | ✅ Yes (via status file) | ✅ Yes (direct) |
| Complexity | **Low — pure Python stdlib** | Medium — needs Rithmic API client |

---

## The 10 Checks

| # | Check | What it catches | How it checks |
|---|-------|----------------|---------------|
| 1 | **Reachability** | Scanner not running? Rithmic disconnected? | `pgrep` for process + status JSON `connected` field |
| 2 | **Latency** | Status file too old? Scanner might be stuck? | File modification time vs current time |
| 3 | **Data Freshness** | Last bar older than 5 minutes? Trading blind? | `last_bar_time` in status JSON |
| 4 | **Contract** | Wrong symbol configured? Missing NQ/MNQ? | `symbol` field in status JSON |
| 5 | **Auth** | FCM/IB IDs missing? Authentication failed? | `fcm_id` and `ib_id` in status JSON |
| 6 | **canTrade** | Process crashed? Connection lost? | `pgrep` process names + status JSON |
| 7 | **Data Quality** | Tick price NaN? Outside NQ range (±20% of spot)? | `last_tick_price` validation |
| 8 | **Balance** | Auto-pass in scan-only mode (not trading) | Config-based — no balance source needed |
| 9 | **Loop Continuity** | Scanner crashed silently? Log gaps? | Log file mtime gaps > 3 minutes |
| 10 | **Truth Layer** | Cross-validation layer erroring? | `truth_state.json` — separate validation system |

---

## Trust Score

Same weighted composite as the [API Health Trust System](https://github.com/nessos666/api-health-trust-system):

```
trust_score = Σ(score_i × weight_i) / Σ(weight_i) × 100
```

| Score | Status | Action |
|-------|--------|--------|
| **80-100** | HEALTHY | All systems go. |
| **50-79** | DEGRADED | Investigate before relying on data. |
| **0-49** | CRITICAL | Do NOT trade. Fix issues first. |

---

## Quick Start

```bash
git clone https://github.com/nessos666/rithmic-api-health-monitor.git
cd rithmic-api-health-monitor

# No dependencies beyond Python 3.10+ stdlib!
# Optional: pip install orjson for faster JSON

cp .env.example .env
# Edit .env with your paths

python rithmic_health_scanner.py
```

---

## Configuration

All via environment variables or `.env`:

| Variable | Default | Description |
|----------|---------|-------------|
| `RITHMIC_STATUS_FILE` | `/tmp/rithmic_scanner_status.json` | Path to scanner status JSON |
| `HEALTH_OUTPUT_FILE` | `/tmp/rithmic_health.json` | Where to write health results |
| `HEALTH_CHECK_INTERVAL` | `300` | Seconds between checks |
| `RITHMIC_ACCOUNT_ID` | *(empty)* | Account ID for display |
| `RITHMIC_SCANNER_PROCESSES` | *(empty)* | Comma-separated process names for pgrep |
| `SCANNER_LOG_DIR` | `.` | Directory containing scanner log files |
| `TRUTH_STATE_FILE` | `/tmp/truth_state.json` | Path to truth layer state file |

---

## Status JSON Format

Your live scanner should write a JSON file with at least these fields:

```json
{
  "timestamp": "2026-04-07T14:30:00+00:00",
  "connected": true,
  "symbol": "NQM6",
  "account_id": "12345",
  "fcm_id": "FCMID",
  "ib_id": "IBID",
  "last_bar_time": "2026-04-07T14:29:00+00:00",
  "last_tick_price": 19850.25,
  "bars_count": 100
}
```

---

## Output

```json
{
  "timestamp": "2026-04-07T14:30:00+00:00",
  "trust_score": 95.0,
  "status": "HEALTHY",
  "broker": "Rithmic",
  "mode": "file-based (no own connection)",
  "checks": [
    {"name": "reachability", "passed": true, "score": 1.0, "detail": "Rithmic connected | NQM6"}
  ],
  "alerts": []
}
```

---

## Running 24/7 (systemd)

```ini
# ~/.config/systemd/user/rithmic-health.service
[Unit]
Description=Rithmic API Health Monitor

[Service]
ExecStart=/usr/bin/python3 /path/to/rithmic_health_scanner.py
Restart=always
RestartSec=10
EnvironmentFile=/path/to/.env

[Install]
WantedBy=default.target
```

```bash
systemctl --user enable --now rithmic-health
```

---

## Project Structure

```
├── rithmic_health_scanner.py     # Main scanner — 10 checks, Trust Score
├── .env.example                  # Environment variables template
├── requirements.txt
└── README.md
```

---

## Market Hours

Automatically handles NQ futures schedule:
- **Trading**: Sunday 18:00 – Friday 17:00 ET
- **Daily pause**: 17:00 – 18:00 ET
- Data-dependent checks auto-pass when market is closed — no false alerts on weekends.

---

## Dependencies

- **Python 3.10+** — stdlib only (json, os, subprocess, time, pathlib)
- **orjson** — optional, for faster JSON parsing
- **pgrep** — for process detection (available on Linux/macOS by default)

---

## Related

Part of the trading infrastructure ecosystem:

- [api-health-trust-system](https://github.com/nessos666/api-health-trust-system) — Generic Trust Score framework (this tool is built on it)
- [topstepx-api-health-monitor](https://github.com/nessos666/topstepx-api-health-monitor) — Same approach for TopStepX/ProjectX API
- [tv-watch-agent](https://github.com/nessos666/tv-watch-agent) — 24/7 TradingView chart surveillance via CDP

---

## Testing

```bash
# Syntax check
python3 -m py_compile rithmic_health_scanner.py
```

---

## License

MIT — use it, modify it, share it.

<p align="center">
  <small>Zero-interference monitoring for Rithmic algo traders.<br>
  <strong>github.com/nessos666</strong></small>
</p>
