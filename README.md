# Rithmic API Health Monitor

![Python](https://img.shields.io/badge/python-3.10%2B-blue)
![License](https://img.shields.io/badge/license-MIT-green)
![Checks](https://img.shields.io/badge/checks-10_automated-orange)
![Architecture](https://img.shields.io/badge/architecture-file--based-purple)

**File-based health monitoring for Rithmic futures trading API.**

Monitors your Rithmic connection health without opening a second session. Reads status files written by your live scanner, checks processes, and computes a Trust Score from 0-100.

## Why File-Based?

Rithmic allows **only ONE session per user**. If a health monitor opened its own connection, it would either disconnect your live trading scanner or get rejected.

This scanner solves the problem by reading the status JSON that your live scanner already writes:

```
┌─────────────────┐                ┌──────────────────┐
│ Your Live       │  writes JSON   │ Health Scanner    │
│ Trading Scanner │ ──────────────→│ (reads JSON only) │
│ (Rithmic conn)  │   /tmp/*.json  │ (NO connection!)  │
└─────────────────┘                └──────┬───────────┘
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

**Zero interference. Full visibility.**

## The 10 Checks

| # | Check | What it catches | Source |
|---|-------|----------------|--------|
| 1 | **Reachability** | Scanner not running? Rithmic disconnected? | Status JSON + pgrep |
| 2 | **Latency** | Status file too old? Scanner might be stuck | Status JSON age |
| 3 | **Data Freshness** | Last bar older than 5 minutes? Trading blind | Status JSON |
| 4 | **Contract** | Wrong symbol? Missing NQ/MNQ? | Status JSON |
| 5 | **Auth** | FCM/IB IDs missing? Auth failed? | Status JSON |
| 6 | **canTrade** | Process down? Connection lost? | Status JSON + pgrep |
| 7 | **Data Quality** | Tick price NaN? Outside NQ range? | Status JSON |
| 8 | **Balance** | Auto-pass in scan mode (no trading) | Config |
| 9 | **Loop Continuity** | Scanner crashed? Log gaps > 3 min? | Log files |
| 10 | **Truth Layer** | Cross-validation layer blocked/erroring? | State JSON |

## Trust Score

```
 80-100  HEALTHY    All systems go.
 50-79   DEGRADED   Investigate before relying on data.
  0-49   CRITICAL   Do NOT use. Fix issues first.
```

## Quick Start

```bash
git clone https://github.com/nessos666/rithmic-api-health-monitor.git
cd rithmic-api-health-monitor

# No dependencies beyond Python 3.10+ stdlib!
# Optional: pip install orjson  (faster JSON)

cp .env.example .env
# Edit .env with your paths

python rithmic_health_scanner.py
```

## Configuration

All via environment variables or `.env`:

| Variable | Default | Description |
|----------|---------|-------------|
| `RITHMIC_STATUS_FILE` | `/tmp/rithmic_scanner_status.json` | Path to scanner status JSON |
| `HEALTH_OUTPUT_FILE` | `/tmp/rithmic_health.json` | Where to write health results |
| `HEALTH_CHECK_INTERVAL` | `300` | Seconds between checks |
| `RITHMIC_ACCOUNT_ID` | *(empty)* | Account ID for display |
| `RITHMIC_SCANNER_PROCESSES` | *(empty)* | Comma-separated process names to check via pgrep |
| `SCANNER_LOG_DIR` | `.` | Directory containing scanner log files |
| `TRUTH_STATE_FILE` | `/tmp/truth_state.json` | Path to truth layer state file |

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

## Market Hours

Automatically handles NQ futures schedule:
- **Trading**: Sunday 18:00 - Friday 17:00 ET
- **Daily pause**: 17:00 - 18:00 ET
- Data-dependent checks auto-pass when market is closed

## Dependencies

- **Python 3.10+** (stdlib only)
- **orjson** (optional, faster JSON)
- **pgrep** (for process detection, available on Linux/macOS)

## License

MIT License. See [LICENSE](LICENSE).
