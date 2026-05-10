# TradingAgents — AGENTS.md

## Entrypoints

- **CLI**: `tradingagents` (installed) or `python -m cli.main` — interactive questionary prompts
- **Programmatic**: `main.py` or `TradingAgentsGraph(config).propagate("NVDA", "2026-01-15")`

## Quick start

```powershell
pip install .                    # or: pip install -e .
cp .env.example .env             # fill in your LLM provider API key
tradingagents                    # interactive CLI
```

## Two-tier LLM architecture

Every graph creates two LLM clients from `default_config.py`:

| Tier | Config key | Agents using it |
|------|-----------|----------------|
| Deep thinking | `deep_think_llm` | Research Manager, Portfolio Manager |
| Quick thinking | `quick_think_llm` | All analysts, researchers, trader, risk debators, reflector |

Override per-run:
```python
config["deep_think_llm"] = "llama3.2:3b"
config["quick_think_llm"] = "llama3.2:3b"
```

## Provider structure

All providers dispatched through `tradingagents/llm_clients/factory.py`:

| Provider(s) | Client class | SDK |
|-------------|-------------|-----|
| openai, xai, deepseek, qwen, glm, **ollama**, openrouter | `OpenAIClient` | `langchain-openai` |
| anthropic | `AnthropicClient` | `langchain-anthropic` |
| google | `GoogleClient` | `langchain-google-genai` |
| azure | `AzureOpenAIClient` | `langchain-openai` |

**Ollama specifics** (`openai_client.py:119`):
- Base URL: `http://localhost:11434/v1`, no API key needed
- Model validation is bypassed (any model name accepted)
- Provider default set in `_PROVIDER_CONFIG`; override with `backend_url` in config

## Model catalog

`tradingagents/llm_clients/model_catalog.py` — controls CLI dropdown options.
Two modes per provider: `"quick"` and `"deep"`. Add new models here.

## Graph flow (execution order)

```
Analysts (market → social → news → fundamentals, in order)
  → Bull Researcher ↔ Bear Researcher (debate, up to max_debate_rounds)
  → Research Manager (deep_llm, structured output)
  → Trader (structured output)
  → Aggressive ↔ Conservative ↔ Neutral (risk debate, up to max_risk_discuss_rounds)
  → Portfolio Manager (deep_llm, structured output)
```

## Structured-output agents

Research Manager, Trader, Portfolio Manager use `with_structured_output` via `tradingagents/agents/utils/structured.py`. If the provider doesn't support it (`NotImplementedError`), they silently fall back to free-text generation.

## Testing

```
pytest                           # all tests
pytest -m unit                   # fast isolated tests only
pytest -m integration            # tests requiring external services
pytest -m smoke                  # quick sanity checks
```

- Conftest auto-sets placeholder API keys for all providers — tests don't need real credentials
- `mock_llm_client` fixture mocks `create_llm_client` for graph-level tests
- Smoke script: `OPENAI_API_KEY=... python scripts/smoke_structured_output.py openai`

## Persistence

- **Memory log** (always on): `~/.tradingagents/memory/trading_memory.md` — stores decisions, resolved with returns on next same-ticker run
- **Checkpoints** (opt-in via `--checkpoint` or `config["checkpoint_enabled"]=True`): `~/.tradingagents/cache/checkpoints/<TICKER>.db` — per-ticker SQLite, cleared on successful completion
- Override paths with `TRADINGAGENTS_MEMORY_LOG_PATH` / `TRADINGAGENTS_CACHE_DIR` env vars
- Use `--clear-checkpoints` to reset before a run

## 5-tier rating vocabulary

`tradingagents/agents/utils/rating.py` — canonical list: `Buy > Overweight > Hold > Underweight > Sell`. Shared by Research Manager, Portfolio Manager, signal processor, and memory log. Trader uses 3-tier (Buy/Hold/Sell).

## Data vendors

yfinance (default, no API key needed) or Alpha Vantage. Configured in `default_config.py` under `data_vendors` (category-level) and `tool_vendors` (tool-level overrides).

## Env files

- `.env.example` — template for LLM provider API keys
- `.env.enterprise.example` — Azure-specific settings
- Both loaded via `python-dotenv` at CLI startup (`.env` first, then `.env.enterprise` overrides)
- `.env` is gitignored

## Docker

```powershell
docker compose run --rm tradingagents           # default (needs .env with API keys)
docker compose --profile ollama run --rm tradingagents-ollama  # local Ollama
```

## Project conventions

- `pyproject.toml` is the source of truth for dependencies (not `requirements.txt`, which just contains `.`)
- Source: `tradingagents/` package + `cli/` package
- No CI pipelines, no formatter/linter config found (no ruff, mypy, black configs)
- Ticker values are sanitized via `safe_ticker_component()` before use in file paths (path traversal prevention)
