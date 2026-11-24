# Dynamic Market Exchange — Simulator & Hackathon Reference

This document preserves the full **Dynamic Market Exchange (DME)** simulator mechanics, CLI reference, UI controls, scenario configurations, and hackathon submission rules from the upstream starter guide.

For an overview of this repository and our submitted agents, see the [main README](../README.md) and the dedicated strategy guides:
- [`444_agent` Strategy Guide](444_agent.md)
- [`rotating_forks_agent` Strategy Guide](rotating_forks_agent.md)
- [Agent Interface Reference](agent_interface.md)

---

- [1. Workspace & Agent Setup](#1-workspace--agent-setup)
  - [1.1 Workspace Setup](#11-workspace-setup)
  - [1.2 Create an Agent Bundle](#12-create-an-agent-bundle)
  - [1.3 Check That It Runs](#13-check-that-it-runs)
  - [1.4 Submit Your Agent](#14-submit-your-agent)
- [2. Simulation Mechanics](#2-simulation-mechanics)
- [3. Testing Reference](#3-testing-reference)
  - [3.1 Options and Configurations](#31-options-and-configurations)
  - [3.2 Using the UI](#32-using-the-ui)
  - [3.3 Comparing Seeds and Scenarios](#33-comparing-seeds-and-scenarios)
  - [3.4 Results and Recordings](#34-results-and-recordings)
- [4. Troubleshooting](#4-troubleshooting)

---

## 1. Workspace & Agent Setup

### 1.1 Workspace Setup

Python 3.11–3.13 is supported; `uv` manages the locked environment:

```bash
uv sync --locked --python 3.12
```

Native Windows, Linux, and macOS are supported. On Windows, decision deadlines are checked while executing Python code; blocking I/O and native-library calls are checked when they return to Python. This adds tracing overhead, so local timing can differ from the Unix evaluator. WSL, Codespaces, and Linux containers remain alternatives.

Run all terminal commands from the repository root (`pmds-pricing-hackathon/` containing `pyproject.toml`, `config/`, and `src/`).

### 1.2 Create an Agent Bundle

Create a subfolder for each agent under **`src/dme_simulator/agents/`** containing `agent.py`, optional Python helpers, and optional models:

```text
pmds-pricing-hackathon/
  README.md
  pyproject.toml
  config/
  src/
    dme_simulator/
      agents/
        base.py                       # simulator interface; leave unchanged
        loader.py
        builtin_bundles/              # supplied baseline opponents
        your-agent-name/              # one folder per agent bundle
          agent.py
          strategy.py                 # optional
          models/                     # optional
```

Example minimal agent in `agent.py`:

```python
from dme_simulator.agents import (
    AssemblyOrder, BaseAgent, Bid, Decision, Listing, Observation,
)
from dme_simulator.config import GoodsConfig


class StarterAgent(BaseAgent):
    def decide(self, state: Observation, goods_config: GoodsConfig) -> Decision:
        return Decision(
            bids=[
                Bid(lot.good, 1, lot.reserve_price * 1.1)
                for lot in state.auction_lots
            ],
            assembly=[
                AssemblyOrder(name, 1) for name in goods_config.recipes
            ],
            listings=[
                Listing(good, 1_000, 250.0)
                for good in goods_config.finished_goods
            ],
        )
```

The folder name is the agent ID. Define exactly one concrete `BaseAgent` subclass in `agent.py`, constructible without required arguments. Use only the supplied public inputs, your own memory, and bundled models; treat inputs as read-only. See [Agent Interface Reference](agent_interface.md).

### 1.3 Check That It Runs

```bash
uv run python -m compileall -q src/dme_simulator/agents/your-agent-name
uv run dme-sim --list-agents
uv run dme-sim --turns 5
```

To watch the simulation live:

```bash
uv run dme-sim --turns 5 --ui terminal --turn-delay 0.5
uv run dme-sim --turns 5 --ui window --turn-delay 0.5
```

### 1.4 Submit Your Agent

**Official scoring:** the competition evaluates agents across multiple matches and private market scenarios. Each match ranks agents by final capital (cash), highest first. **Inventory adds zero value to the score.** Overall rankings use the arithmetic mean of each agent's ranks across matches (lower is better).

Portal bundle constraints:
- **Bundle layout:** exactly one `agent.py` with one concrete `BaseAgent` class callable without arguments; optional helpers/models stay inside the same folder. Every `.py` file must be valid UTF-8 Python.
- **Size limits:** at most **64 files** and **20 MiB** extracted contents; ZIPs must also be at most **20 MiB** (stored or deflate, unencrypted).
- **Allowed extensions:** `.py` and model files (`.bin`, `.pt`, `.pth`, `.pkl`, `.pickle`, `.joblib`, `.onnx`, `.safetensors`, `.npy`, `.npz`, `.h5`, `.hdf5`). Exclude `__pycache__/`, `.pyc`, docs, plots, `.git/`, and `.venv/`.
- **Execution limits:** **100 ms** decision deadline per turn; **10 s** wall-clock validation smoke check on three default-scenario turns. Errors, invalid actions, or timeouts fall back to empty decisions (or reject the upload during validation).
- **Model loading:** resolve paths via `Path(__file__).parent / ...` and load once in `__init__`.

---

## 2. Simulation Mechanics

There are **no workforce costs** and **no storage costs**. Capital only leaves your company through material purchases at auction and only returns through sales. Unsold raw materials and finished goods carry over to the next turn for free.

### 2.1 The Turn Cycle

At the start of each turn, every solvent agent receives an `Observation` and has `agents.decide_timeout_ms` ms (default `100.0`) to return a single `Decision` containing bids, assembly orders, and sales listings. All three are chosen simultaneously *before* the auction clears:

1. **Auction** — published raw-material lots are sold to the highest sealed bids; won materials enter inventory immediately.
2. **Production** — teams convert raw materials (including newly won materials) into finished goods by recipe.
3. **Sales** — teams list finished goods at chosen prices and spawned customers buy them; revenue is paid immediately.

Because later phases use materials and goods actually held when they run, planned assembly or listings may be partially fulfilled if an upstream auction bid or assembly order fell short.

### 2.2 Phase 1 — Raw-Material Auction

- Every turn, suppliers replenish stock from `[production_min, production_max]`. Unsold stock carries over.
- Each lot in `state.auction_lots` publishes available `quantity` and dynamic `reserve_price` (starts at `base_price` and falls as supplier stock accumulates).
- Each `suppliers.<good>` entry configures `price_stock_factor`, `price_floor_multiplier`, and `adaptation` (whether persistent empty/full stock scales replenishment and capacity).
- **Sealed, pay-your-bid:** Bids (`Bid(good, quantity, price_per_unit)`) are sealed, sorted highest-first per material, and filled until the lot runs out. Bids below `reserve_price` are discarded. Winners pay **their own bid price per unit**, not a clearing price.
- **Ties:** Equal-price bids share remaining units via round-robin in random team order.
- **Budget cap:** Winnings are capped by remaining capital. Materials clear in random order each turn.

### 2.3 Phase 2 — Production

Recipes convert raw materials into finished goods (`goods_config.recipes`). `AssemblyOrder(recipe_name, quantity)` orders execute sequentially in the order returned, limited only by held materials (including auction wins from Phase 1) and the optional `production.capacity_per_turn` cap. Produced units are immediately available in Phase 3.

### 2.4 Phase 3 — Sales

- `Listing(good, quantity, price)` offers finished goods for sale. You may submit multiple listings for the same good at different prices; listings are capped by held inventory and require `price > 0`.
- Unsold and unlisted stock carries over to the next turn at zero cost.
- Customers spawn each turn in fluctuating volume (`base_customers`, `amplitude`, `period_divisor`). Each belongs to a **segment** (e.g., `BUDGET`, `MAINSTREAM`, `PREMIUM`) with preference weights and private `willingness_to_pay` ranges per good.
- Each customer picks a good, draws a private valuation, and buys from eligible listings (`price <= valuation`). Lower prices have higher selection probability (governed by `price_sensitivity_beta`), though customers do not always pick the cheapest eligible offer.

### 2.5 Public Information

- `state.public_inventories` and `state.public_capital` expose every team's holdings and cash at the start of each turn.
- `state.auction_transactions` and `state.sales_transactions` accumulate completed trades across turns. By default (`observe_all_transactions: false`), only trades involving your team are visible; when `true`, all market trades are exposed.
- Current-turn bids and listings are sealed until the turn resolves.

### 2.6 Bankruptcy

If capital drops below zero at the end of a turn, the company is liquidated (inventory cleared, no further turns).

### 2.7 Built-in Scenarios (`config/`)

- `config/tutorial_pricing.yaml`: single good, abundant/near-free materials — isolates pricing.
- `config/tutorial_auction.yaml`: single good, fixed-like sales prices — isolates bidding.
- `config/default.yaml`: 40 turns, 4 materials (`CHIP`, `DISPLAY`, `BATTERY`, `CASING`), 5 goods (`PHONE`, `LAPTOP`, `TABLET`, `SMARTWATCH`, `CONSOLE`), 3 customer segments (`BUDGET`, `MAINSTREAM`, `PREMIUM`).
- `config/scaled_market.yaml`: 30 turns, 5x capital (`60,000`), larger lots, 5x customer volume.
- `config/ultralong.yaml`: 240 turns, same market parameters as `default.yaml` to test long-horizon capital compounding and supplier adaptation.

---

## 3. Testing Reference

### 3.1 Options and Configurations

Run via `uv run dme-sim [OPTIONS]`:

| Option | Purpose |
| --- | --- |
| `--config PATH` | Choose a YAML scenario; defaults to `config/default.yaml`. Repeat for several scenarios. |
| `--turns N` | Override duration with a positive turn count. |
| `--seed N` | Set a reproducible seed; otherwise use the scenario's seed. |
| `--ui window \| terminal \| none` | Choose a pygame window, live console dashboard, or plain progress. |
| `--turn-delay SECONDS` | Pace a live display; nonnegative seconds per turn. |
| `--output-dir PATH` | Choose the directory for PNG charts. |
| `--simulations N` | Run N consecutive seeds per scenario; defaults to 1. |
| `--max-workers N` | Limit parallel batch workers; defaults to CPU count. |
| `--list-agents` | Print discovered bundle IDs without constructing agents or running turns. |
| `--agents-dir PATH` | Override the bundle search root (repeatable). |
| `--extra-agent PATH` | Append a bundle folder stored elsewhere (repeatable). |
| `--profile` | Print decision and phase timings for one simulation. |
| `--guide` | Print the dashboard guide and exit. |
| `--record [PATH]` | Record a single run to JSONL (`outputs/recordings/<timestamp>/simulation.jsonl` if omitted). |
| `--replay PATH` | Replay a recording without loading agents or running a simulation. |
| `--help` | Show complete command-line help. |

### 3.2 Using the UI

| Window control | Action |
| --- | --- |
| Space / Play | Pause or resume. |
| Right / N / Next | View or execute the next turn. |
| Left / P / Back | Pause and view the previous turn. |
| Home / Restart | Return the viewer to turn 0 and pause; preserves simulation state. |
| Up / Down | Speed up / slow down. |
| C / chart tabs | Switch between Cash, Cash flow, and Sales charts. |
| H / F1 / Guide | Open the guide with metric definitions and controls. |
| Ctrl + plus/minus; Ctrl + 0 | Zoom; reset zoom. |
| Mouse wheel / Page Up / Page Down | Scroll the panel under the pointer or the guide. |
| Esc | Close the guide if open; otherwise stop the run. |
| Q / close window | Stop the run; close the viewer after completion. |

### 3.3 Comparing Seeds and Scenarios

```bash
uv run dme-sim --config config/default.yaml --config config/scaled_market.yaml --simulations 3 --turns 30 --seed 100 --max-workers 3 --output-dir outputs/comparison
```

### 3.4 Results and Recordings

With `render_plots: true`, outputs include:
- `*_dashboard_overview.png`: capital, cash movements, market share, captured/missed demand.
- `*_dashboard_auction.png`: reserve and winning prices, offered materials, fill rates, units won.
- `*_dashboard_sales.png`: units sold, listed/realized prices, unserved customers, unsold stock.

Record and replay a run:

```bash
uv run dme-sim --turns 40 --record
uv run dme-sim --replay outputs/recordings/<timestamp>/simulation.jsonl
```

---

## 4. Troubleshooting

| Problem | Check |
| --- | --- |
| Tool not found | Reopen the terminal after installing `git` / `uv`, then verify `git --version` and `uv --version`. |
| Unsupported Python | Run `uv sync --locked --python 3.12`. |
| Agent fails to load | Bundle folder must contain `agent.py` with one concrete `BaseAgent` subclass constructible with no required arguments. |
| Duplicate ID | Give each agent bundle folder a unique name. |
| Agent takes no actions | Run with `--ui none` to inspect warnings/tracebacks. Exceptions, invalid decisions, or timeouts (>100 ms) trigger empty fallback decisions. |
