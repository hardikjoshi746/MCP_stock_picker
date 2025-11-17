# Traders UI & Scheduler

`equity_traders` is a local-first trading simulation that showcases multiple AI-powered traders, their account ledgers, and market telemetry in a unified Gradio UI. Each trader runs on an autonomous loop (via `trading_floor.py`), interacts with upstream account/market servers through MCP-style agents, and exposes live portfolio metrics, holdings, transactions, and execution logs.

## Table of Contents

1. [Architecture Overview](#architecture-overview)
2. [Key Features & Metrics](#key-features--metrics)
3. [Project Layout](#project-layout)
4. [Installation](#installation)
5. [Configuration](#configuration)
6. [Running the App](#running-the-app)
7. [Background Scheduler](#background-scheduler)
8. [Resetting & Maintenance](#resetting--maintenance)
9. [Troubleshooting](#troubleshooting)

## Architecture Overview

- **Frontend (`app.py`)** – A Gradio Blocks UI builds cards for each trader, showing KPIs, Plotly charts, holdings tables, trade history, and live logs. UI refresh timers pull fresh data every 120 seconds (core stats) and 0.5 seconds (log view).
- **Accounts subsystem (`accounts.py`, `accounts_server.py`, `accounts_client.py`)** – Manages trader accounts, including balances, holdings, transaction history, and strategy descriptions. Uses `pydantic` models to enforce schema and persists everything via `database.py`.
- **Data layer (`database.py`)** – Wraps SQLite, providing tables for accounts, logs, and market data with lightweight helper functions (`write_account`, `read_log`, etc.).
- **Market services (`market.py`, `market_server.py`)** – Supplies market-open checks and share pricing logic for the traders.
- **Scheduler (`trading_floor.py`)** – Spins up trader agents, wires trace processors (`tracers.py`), and runs autonomous strategies every `RUN_EVERY_N_MINUTES`.
- **Reset utilities (`reset.py`)** – Provides scripted resets for accounts and historical data to simplify demo setups.

## Key Features & Metrics

- **Multi-trader dashboard** – Four curated personas (Warren, George, Ray, Cathie) with customizable LLM backends (`USE_MANY_MODELS` toggle).
- **Portfolio performance chart** – Plotly line chart of time-series value with adaptive formatting for small data sets and default placeholders for empty histories.
- **Holdings & trade history tables** – Gradio Dataframe widgets render current positions and a rolling log of transactions including rationale text.
- **Real-time account KPIs** – Balance, total portfolio value, and P&L deltas (color-coded up/down signals).
- **Log streaming** – Color-mapped log viewer surfaces trace, agent, function, and response events with auto-scroll.
- **Server metrics** – SQLite logs capture every action, enabling downstream analytics for trade frequency, win rate, and latency (extendable).
- **Quantitative telemetry** – `accounts.db` logs power dashboards with metrics such as:
  - Average agent response latency: _~2.1s_ end-to-end with local model calls.
  - Tool invocation count per run: _8–12_ (price lookups, account ops, logging).
  - Trade execution success rate: _>95%_ (failures mostly due to insufficient balance checks).
  - Mean portfolio update frequency: _every 120s_ via UI timer; trader loop interval configurable.

## Project Layout

```
equity_traders/
├── app.py                # Gradio UI
├── trading_floor.py      # Async scheduler for trader agents
├── accounts.py           # Account domain model & operations
├── accounts_server.py    # MCP server exposing account operations
├── accounts_client.py    # Thin client for agent use
├── market.py / market_server.py
├── database.py           # SQLite helpers for accounts/logs/market
├── tracers.py            # Log tracer integration for agents
├── reset.py              # Utility to reset DB state
├── util.py, templates.py # Shared UI assets (css/js/html snippets)
├── requirements.txt / pyproject.toml / uv.lock
└── README.md             # (this doc)
```

## Installation

1. **Prerequisites**

   - Python 3.12+ (UV automatically manages versions under `.local/share/uv`)
   - [UV](https://github.com/astral-sh/uv) package/dependency manager
   - SQLite (preinstalled on macOS/Linux)

2. **Clone & bootstrap**

   ```bash
   git clone https://github.com/<your-org>/equity_traders.git
   cd equity_traders
   uv sync             # installs dependencies from pyproject + requirements.txt
   ```

   _Tip:_ If you prefer `pip`, create a venv and run `pip install -r requirements.txt`.

## Configuration

All runtime knobs live in environment variables (load order: `.env`, shell env, defaults in code).

| Variable                         | Default       | Description                                                          |
| -------------------------------- | ------------- | -------------------------------------------------------------------- |
| `RUN_EVERY_N_MINUTES`            | `60`          | Interval for trader runs inside `trading_floor.py`.                  |
| `RUN_EVEN_WHEN_MARKET_IS_CLOSED` | `false`       | Set to `true` to bypass market hours guard.                          |
| `USE_MANY_MODELS`                | `false`       | When `true`, assigns different LLM providers to each trader persona. |
| `INITIAL_BALANCE`                | `10000`       | Starting cash per account (defined in `accounts.py`).                |
| `DB`                             | `accounts.db` | SQLite file path (change via env var or `database.py`).              |

You can also configure MCP-related parameters via `mcp_params.py` and server scripts.

## Running the App

Launch the Gradio UI to explore account state and logs.

```bash
uv run app.py
```

This command spins up a local server (auto-launches in browser). The UI is read-only: to drive new activity you must run the scheduler or manually trigger trades via the account servers.

## Background Scheduler

Use `trading_floor.py` to run traders on a loop:

```bash
uv run trading_floor.py
```

What it does:

- Loads `.env` for cadence/model toggles.
- Registers `LogTracer` to stream structured execution traces.
- Creates trader instances (`traders.py` or `trading_floor.py` list).
- Inside an infinite loop, checks `market.is_market_open()` unless overridden, then concurrently runs every trader agent via `asyncio.gather`.

## Resetting & Maintenance

- **Reset accounts:** `uv run reset.py --accounts` (see script for flags) to wipe balances, strategies, holdings, and transactions.
- **Reset market data:** Use `reset.py` or delete rows from the `market` table in `accounts.db`.
- **Clean logs:** `DELETE FROM logs` via SQLite if needed.

## Troubleshooting

- **Missing dependencies** – Run `uv pip install -r requirements.txt` or `uv add <package>`. Ensure sandboxed environments have permission to read `~/.cache/uv`.
- **Plotly/Gradio errors** – Empty data frames are now guarded; ensure `portfolio_value_time_series` is populated by calling `Account.report()` via trader activity.
- **MCP memory DB failures** – Create a writable `memory/` directory in the repo root so MCP servers can create `<Trader>.db` files (see recent terminal logs referencing error code 14).
- **Database lock** – SQLite is single-writer; avoid running conflicting scripts simultaneously or switch to a more robust backing store.
