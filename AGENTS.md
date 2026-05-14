# TradingAgents — AGENTS.md

## Entrypoints

- **CLI**: `tradingagents` (installed) or `python -m cli.main` — interactive questionary prompts
- **Programmatic**: `TradingAgentsGraph(config).propagate("NVDA", "2026-01-15")` returns `(final_state, decision_string)`

## Quick start

```powershell
pip install -e .
tradingagents                    # interactive CLI
```

Ollama needs no `.env` (no API key). Other providers: `cp .env.example .env`, fill key.

## Two-tier LLM architecture

Every graph creates two clients from `tradingagents/graph/trading_graph.py:86-100`:

| Tier | Config key (`default_config.py`) | Agents using it (`graph/setup.py:50-87`) |
|------|----------------------------------|------------------------------------------|
| Deep thinking | `deep_think_llm` | Research Manager, Portfolio Manager |
| Quick thinking | `quick_think_llm` | All 4 analysts, Bull/Bear researchers, Trader, 3 risk debators, Reflector |

Override per-run:
```python
config["deep_think_llm"] = "gemma-4-26b-a4b-it"
config["quick_think_llm"] = "gemma-4-31b-it"
```

Provider-specific thinking params are dispatched in `trading_graph.py:133-153` via `_get_provider_kwargs()`. Config keys: `google_thinking_level`, `openai_reasoning_effort`, `anthropic_effort`.

## Provider dispatch

`tradingagents/llm_clients/factory.py` routes providers:

| Providers | Client | SDK |
|-----------|--------|-----|
| openai, xai, deepseek, qwen, glm, **ollama**, openrouter | `OpenAIClient` | `langchain-openai` |
| anthropic | `AnthropicClient` | `langchain-anthropic` |
| google | `GoogleClient` | `langchain-google-genai` |
| azure | `AzureOpenAIClient` | `langchain-openai` |

**Ollama** (`openai_client.py:119`): base URL `http://localhost:11434/v1`, no API key, model validation bypassed (any name accepted). Override endpoint via `backend_url` config key.

## Model catalog

`tradingagents/llm_clients/model_catalog.py` — CLI dropdown options in two modes: `"quick"` and `"deep"`. Add new models here.

## Graph execution order

```
Analysts (market → social → news → fundamentals, in order)
  → Bull ↔ Bear Researcher (debate, up to max_debate_rounds)
  → Research Manager (deep_llm, structured output)
  → Trader (structured output)
  → Aggressive ↔ Conservative ↔ Neutral (risk debate, up to max_risk_discuss_rounds)
  → Portfolio Manager (deep_llm, structured output)
```

## Structured-output agents

Research Manager, Trader, Portfolio Manager use `with_structured_output` via `tradingagents/agents/utils/structured.py`. Silent fallback to free-text on `NotImplementedError` (e.g. older Ollama models that don't support function-calling).

## Testing

```
pytest                           # all tests (no API keys needed — conftest mocks them)
pytest -m unit                   # fast isolated tests
pytest -m smoke                  # quick sanity checks
pytest test_structured_agents.py # single file
```

- `mock_llm_client` fixture mocks `create_llm_client` for graph-level tests
- Smoke script: `OPENAI_API_KEY=... python scripts/smoke_structured_output.py openai`

## Persistence

- **Memory log** (always on): `~/.tradingagents/memory/trading_memory.md` — decisions resolved with realised returns on next same-ticker run
- **Checkpoints** (opt-in via `config["checkpoint_enabled"]=True`): `~/.tradingagents/cache/checkpoints/<TICKER>.db` — per-ticker SQLite, cleared on successful completion
- Override with `TRADINGAGENTS_MEMORY_LOG_PATH` / `TRADINGAGENTS_CACHE_DIR`

## Conventions

- `pyproject.toml` is dependency source of truth (`requirements.txt` just contains `.`)
- **5-tier rating**: `Buy > Overweight > Hold > Underweight > Sell` (`agents/utils/rating.py`) — shared by Research Manager, Portfolio Manager, signal processor, memory log. Trader uses 3-tier (Buy/Hold/Sell).
- **Data vendors**: yfinance (default, no key) or Alpha Vantage. Configured in `default_config.py` under `data_vendors` (category) / `tool_vendors` (tool-level override).
- **SignalProcessor** is heuristic, zero LLM calls (`graph/signal_processing.py`).
- `safe_ticker_component()` sanitizes ticker values before path interpolation (path traversal prevention, `dataflows/utils.py`).
- No CI, no formatter/linter config found.
