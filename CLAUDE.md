# CLAUDE.md

This file provides guidance to Claude Code (claude.ai/code) when working with code in this repository.

## What this is

TradingAgents is a multi-agent LLM trading-research framework built on **LangGraph**. Specialized agents (analysts → researchers → trader → risk team → portfolio manager) collaborate over a stock/crypto ticker on a given date and emit a trade decision. It is a research scaffold, not a live trading system or financial advice.

## Commands

Use a **virtualenv on Python 3.13**, and prefix commands with `./.venv/bin/`. Two reasons this is not optional on macOS/Homebrew: the system Python is `EXTERNALLY-MANAGED` (a bare `pip install` refuses to write), and Homebrew's current default is Python 3.14, which is past the 3.10–3.13 range CI verifies. Create it with `python3.13 -m venv .venv` (`.venv` is already gitignored).

```bash
pip install -e ".[dev]"        # install with dev extras (ruff, pytest)

pytest -q                       # full test suite
pytest tests/test_memory_log.py # single file
pytest tests/test_vendor_routing.py::test_name   # single test
pytest -m unit                  # by marker: unit | integration | smoke

ruff check .                    # lint (strict; CI runs this on the whole repo)

tradingagents                   # interactive CLI (installed entry point = cli.main:app)
python -m cli.main              # run CLI from source
python main.py                  # minimal programmatic example
```

CI (`.github/workflows/ci.yml`) runs pytest on Python 3.10–3.13, a clean-install import smoke test (catches undeclared runtime deps), and `ruff check .`. The repo is fully clean under the strict ruff select — keep it that way. `ruff format` is **not** yet adopted repo-wide; do not mass-reformat.

Tests never hit real APIs: `tests/conftest.py` autouses fixtures that inject placeholder API keys and block network calls, so the suite runs offline. Preserve that when adding tests.

## Architecture

### Execution flow

Entry point is `TradingAgentsGraph` in [tradingagents/graph/trading_graph.py](tradingagents/graph/trading_graph.py). `.propagate(ticker, date, asset_type)` returns `(final_state, processed_signal)`. Orchestration is split across `tradingagents/graph/`:

- **setup.py** (`GraphSetup`) — builds the LangGraph `StateGraph`. Node sequence: selected analysts (each with a tool-call loop + a message-clear node) → Bull/Bear Researcher debate → Research Manager → Trader → Aggressive/Conservative/Neutral risk debate → Portfolio Manager → END.
- **conditional_logic.py** (`ConditionalLogic`) — routing functions for the analyst tool loops and the debate/risk rounds. Both routers use **complete path maps** (`DEBATE_PATH_MAP`, `RISK_ANALYSIS_PATH_MAP`) so a fall-through speaker label can never crash LangGraph mid-run.
- **analyst_execution.py** — turns `selected_analysts` into an ordered execution plan; also the CLI's wall-time tracker.
- **checkpointer.py** — opt-in per-ticker SQLite checkpoint/resume (`config["checkpoint_enabled"]`). The checkpoint thread ID is keyed on a **run signature** (analysts + debate/risk rounds + asset_type) so a resume under a different graph shape starts fresh instead of silently continuing the old graph.
- **propagation.py / reflection.py / signal_processing.py** — initial-state construction, post-run reflection, and extraction of the final BUY/HOLD/SELL signal.

The shared graph state is `AgentState` (a LangGraph `MessagesState` subclass) in [tradingagents/agents/utils/agent_states.py](tradingagents/agents/utils/agent_states.py). Each analyst writes its own `*_report` field; debate sub-states are nested `TypedDict`s.

### Agents

`tradingagents/agents/` — each agent is a `create_*` factory returning a node callable, re-exported from `agents/__init__.py`. Grouped: `analysts/` (market, sentiment/social, news, fundamentals), `researchers/` (bull, bear), `managers/` (research, portfolio), `trader/`, `risk_mgmt/` (aggressive, conservative, neutral). `agents/utils/` holds the LLM-bound tool definitions (`agent_utils.py` and the `*_tools.py` modules), memory, ratings, and structured-output schemas.

Two things are resolved deterministically **before** any agent runs, to stop LLM hallucination: instrument identity (`resolve_instrument_identity`, cached yfinance lookup, injected as `instrument_context`) and a verified market-data snapshot the market analyst must ground its price/indicator claims in.

### Data layer

`tradingagents/dataflows/` — the interface is **vendor-routed**. `route_to_vendor(method, ...)` in [interface.py](tradingagents/dataflows/interface.py) maps a tool method to a category, looks up the configured vendor chain, and calls implementations in order.

Key contract: **the configured vendor list IS the chain** — requests are never silently routed to a vendor the user did not choose. Configure via `config["data_vendors"]` (per category) or `config["tool_vendors"]` (per tool, takes precedence). List several comma-separated for ordered fallback (`"yfinance,alpha_vantage"`); `"default"` uses all available vendors. Vendors: yfinance, alpha_vantage, fred (macro), polymarket (prediction markets). When all vendors report no data, a single explicit "unavailable" sentinel is returned so agents report unavailability rather than fabricating values.

### LLM clients

`tradingagents/llm_clients/` — `create_llm_client(provider, model, base_url, **kwargs)` factory. Anthropic, Google, Azure, and Bedrock have dedicated client classes (genuinely different APIs). Everything else routes through the **OpenAI-compatible provider registry** `OPENAI_COMPATIBLE_PROVIDERS` in [openai_client.py](tradingagents/llm_clients/openai_client.py) — one declarative `ProviderSpec` row per provider (base URL, key optionality, wire quirks). Adding an OpenAI-compatible provider = one registry row + its API-key env var in `api_key_env.PROVIDER_API_KEY_ENV` (the single source both the client and CLI consult). Dual-region providers (qwen/glm/minimax) keep separate `*-cn` rows because international and China credentials cannot be shared.

Model choices for the CLI live in `model_catalog.py`. `deep_think_llm` handles complex reasoning (Research Manager, Portfolio Manager); `quick_think_llm` handles everything else.

### Configuration

[tradingagents/default_config.py](tradingagents/default_config.py) — `DEFAULT_CONFIG` is the single source of truth. Any `TRADINGAGENTS_*` env var listed in `_ENV_OVERRIDES` overrides its config key, coerced to the existing default's type; an invalid value (e.g. a misspelled bool) **raises at startup** rather than silently misconfiguring an unattended run. When adding an env-overridable option, add a row to `_ENV_OVERRIDES` — no entry-point changes needed.

Persistent state lives under `~/.tradingagents/` (override base with `TRADINGAGENTS_CACHE_DIR`): a decision log at `memory/trading_memory.md` (always on; realized return + reflection injected into the Portfolio Manager on the next same-ticker run) and checkpoint DBs under `cache/checkpoints/`.

### Markets

Works with any Yahoo Finance ticker via its exchange suffix (`.HK`, `.T`, `.SS`, `BTC-USD`, etc.). The alpha benchmark auto-resolves per market via `benchmark_map` (SPY for US); `benchmark_ticker` overrides it globally. Ticker strings are validated with `safe_ticker_component` before use as path components (path-traversal hardening).

## Running an analysis locally

Credentials go in a `.env` at the repo root — `tradingagents/__init__.py` loads it automatically via `load_dotenv(find_dotenv(usecwd=True))` with `override=False`, so a real environment variable always wins over the file. Provider and models are set the same way (`TRADINGAGENTS_LLM_PROVIDER`, `TRADINGAGENTS_DEEP_THINK_LLM`, `TRADINGAGENTS_QUICK_THINK_LLM`).

The `tradingagents` CLI is **questionary-driven and needs a real TTY**; it cannot be run from a non-interactive harness (piping input is fragile). To trigger a run non-interactively, call the graph directly instead — this also lets you cut cost by selecting fewer analysts:

```python
ta = TradingAgentsGraph(debug=True, config=DEFAULT_CONFIG.copy(),
                        selected_analysts=("market", "news"))  # 4 analysts = full pipeline
final_state, decision = ta.propagate("NVDA", "2024-05-10")
```

A full 4-analyst run takes roughly 10 minutes on a reasoning model and produces ~1200 lines of debug output, so run it in the background and poll the log rather than blocking on it. `propagate` returns the graph state plus the extracted signal (a five-tier rating such as `Overweight` / `Hold` / `Underweight`, not a bare BUY/SELL).

Behavior that looks like breakage but is by design — do not "fix" these:

- **`Optional macro_data unavailable … FRED_API_KEY not set`** and **Reddit `HTTP 429` with backoff** are the vendor layer degrading gracefully. Agents report the data as unavailable instead of fabricating it; the run continues.
- **`Portfolio Manager: structured-output invocation failed … retrying once as free text`** fires on every DeepSeek run observed so far. The free-text retry succeeds and the run completes, so it is a provider schema-compatibility weakness, not a graph bug. To investigate cheaply, run `scripts/smoke_structured_output.py <provider>` — it exercises the three structured-output agents (Research/Trader/Portfolio) directly against a real LLM without a full `propagate()`, so you can compare providers before touching the structured-output path.
- Differing decisions across runs are expected. Changing the *analyst selection* moved a NVDA run from `Underweight` to `Overweight`, while a 10-day date shift left an AAPL verdict unchanged at `Hold` — analyst breadth shifts conclusions far more than modest data drift.

## Conventions

- Reference GitHub issues in comments as `(#NNNN)` when a line encodes a specific fix — this codebase does it consistently and the context is valuable; match it when fixing a reported bug.
- `__init__.py` re-exports are intentional (ruff ignores F401 there). Deprecated aliases (e.g. `create_social_media_analyst`) are kept for back-compat — don't remove without a deprecation cycle.
- LLM output is non-deterministic by design; don't treat run-to-run variation as a bug.
