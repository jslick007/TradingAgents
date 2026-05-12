# TradingAgents — Prompt Inventory

> All prompts are inline Python strings. There are no YAML/JSON/TOML prompt files.

---

## Phase A (Trade-Time) Prompts

### 1. Market Analyst — Specialized Instruction
| | |
|---|---|
| **File** | `tradingagents/agents/analysts/market_analyst.py:22` |
| **Role** | System message (specialized instruction layer) |
| **Summary** | Instructs the agent to select up to 8 complementary technical indicators (SMAs, EMAs, MACD, RSI, Bollinger Bands, ATR, VWMA), call `get_stock_data` then `get_indicators`, and write a detailed report concluding with a Markdown table. Appends `get_language_instruction()` suffix. |
| **Concatenated with** | "collaboration wrapper" system prompt at `market_analyst.py:54` |

### 2. Market Analyst — Collaboration Wrapper
| | |
|---|---|
| **File** | `tradingagents/agents/analysts/market_analyst.py:54` |
| **Role** | System message (outer layer, `ChatPromptTemplate`) |
| **Summary** | Generic multi-agent collab preamble: describes tool-use protocol, defines the `FINAL TRANSACTION PROPOSAL: **BUY/HOLD/SELL**` handshake, injects `{tool_names}`, `{system_message}` (the specialized instruction above), `{current_date}`, and `{instrument_context}`. |

---

### 3. News Analyst — Specialized Instruction
| | |
|---|---|
| **File** | `tradingagents/agents/analysts/news_analyst.py:21` |
| **Role** | System message (specialized instruction layer) |
| **Summary** | Instructs to analyze recent news & macro trends using `get_news` (company-specific) and `get_global_news` (macro). Must append a Markdown table. Appends `get_language_instruction()` suffix. |

### 4. News Analyst — Collaboration Wrapper
| | |
|---|---|
| **File** | `tradingagents/agents/analysts/news_analyst.py:29` |
| **Role** | System message (outer layer) |
| **Summary** | Identical structure to #2 (same collab preamble with `FINAL TRANSACTION PROPOSAL` handshake). |

---

### 5. Social Media Analyst — Specialized Instruction
| | |
|---|---|
| **File** | `tradingagents/agents/analysts/social_media_analyst.py:15` |
| **Role** | System message (specialized instruction layer) |
| **Summary** | Instructs to analyze social media posts, sentiment, and company-specific news via `get_news`. Must produce a comprehensive long report with Markdown table. Appends `get_language_instruction()` suffix. |

### 6. Social Media Analyst — Collaboration Wrapper
| | |
|---|---|
| **File** | `tradingagents/agents/analysts/social_media_analyst.py:23` |
| **Role** | System message (outer layer) |
| **Summary** | Identical structure to #2. |

---

### 7. Fundamentals Analyst — Specialized Instruction
| | |
|---|---|
| **File** | `tradingagents/agents/analysts/fundamentals_analyst.py:26` |
| **Role** | System message (specialized instruction layer) |
| **Summary** | Instructs to analyze company financials: profile, financial documents, balance sheet, cashflow, income statement via `get_fundamentals`, `get_balance_sheet`, `get_cashflow`, `get_income_statement`. Must append Markdown table. Appends `get_language_instruction()` suffix. |

### 8. Fundamentals Analyst — Collaboration Wrapper
| | |
|---|---|
| **File** | `tradingagents/agents/analysts/fundamentals_analyst.py:35` |
| **Role** | System message (outer layer) |
| **Summary** | Identical structure to #2. |

---

### 9. Bull Researcher
| | |
|---|---|
| **File** | `tradingagents/agents/researchers/bull_researcher.py:15` |
| **Role** | Raw f-string prompt, passed directly to `llm.invoke()` |
| **Summary** | Instructs to advocate for investing: emphasize growth potential, competitive advantages, positive indicators. Must counter the bear's argument with data. Injects all four analyst reports (market, sentiment, news, fundamentals), debate history, and last bear argument. |

### 10. Bear Researcher
| | |
|---|---|
| **File** | `tradingagents/agents/researchers/bear_researcher.py:15` |
| **Role** | Raw f-string prompt, passed directly to `llm.invoke()` |
| **Summary** | Instructs to argue against investing: emphasize risks/challenges, competitive weaknesses, negative indicators. Must counter the bull's claims. Same data injection as Bull Researcher. |

---

### 11. Research Manager
| | |
|---|---|
| **File** | `tradingagents/agents/managers/research_manager.py:22` |
| **Role** | f-string prompt → `with_structured_output` (with free-text fallback) |
| **Summary** | Synthesises the bull/bear debate into a structured `ResearchPlan`. Defines the 5-tier rating scale (Buy > Overweight > Hold > Underweight > Sell). Injects `{instrument_context}` and full `{history}`. Outputs `investment_plan` for the Trader. |

### 12. Trader — System Message
| | |
|---|---|
| **File** | `tradingagents/agents/trader/trader.py:28` |
| **Role** | System message in messages list |
| **Summary** | Sets role: "trading agent analyzing market data to make investment decisions." Instructs to provide a specific buy/sell/hold recommendation anchored in analyst reports and the research plan. |

### 13. Trader — User Message
| | |
|---|---|
| **File** | `tradingagents/agents/trader/trader.py:36` |
| **Role** | User message in messages list |
| **Summary** | Provides company name, instrument context, and the Research Manager's `{investment_plan}`. Instructs to use this foundation to make a strategic trading decision. Outputs a `TraderProposal` via `with_structured_output`. |

---

### 14. Aggressive Risk Analyst
| | |
|---|---|
| **File** | `tradingagents/agents/risk_mgmt/aggressive_debator.py:19` |
| **Role** | Raw f-string prompt, passed directly to `llm.invoke()` |
| **Summary** | Champions high-reward/high-risk opportunities. Must directly rebut Conservative and Neutral analysts with data-driven arguments. Injects trader decision, all four analyst reports, debate history, and last responses from opposing analysts. |

### 15. Conservative Risk Analyst
| | |
|---|---|
| **File** | `tradingagents/agents/risk_mgmt/conservative_debator.py:19` |
| **Role** | Raw f-string prompt, passed directly to `llm.invoke()` |
| **Summary** | Advocates for asset protection, minimum volatility, steady growth. Must counter Aggressive and Neutral analysts, questioning their optimism. Injects same data as Aggressive. |

### 16. Neutral Risk Analyst
| | |
|---|---|
| **File** | `tradingagents/agents/risk_mgmt/neutral_debator.py:19` |
| **Role** | Raw f-string prompt, passed directly to `llm.invoke()` |
| **Summary** | Provides balanced perspective, challenges both Aggressive and Conservative viewpoints. Advocates for moderate, sustainable strategy. Injects same data as the other two risk analysts. |

---

### 17. Portfolio Manager
| | |
|---|---|
| **File** | `tradingagents/agents/managers/portfolio_manager.py:42` |
| **Role** | f-string prompt → `with_structured_output` (with free-text fallback) |
| **Summary** | Synthesises the risk analysts' debate into the final `PortfolioDecision`. Uses the 5-tier rating scale. Context includes Research Manager's plan, Trader's proposal, optionally past lessons (`past_context` from memory log), and the full risk debate history. Appends `get_language_instruction()` suffix. |

---

## Phase B (Post-Trade) Prompts

### 18. Reflector (Deferred Reflection)
| | |
|---|---|
| **File** | `tradingagents/graph/reflection.py:20` |
| **Role** | System message → `quick_thinking_llm.invoke()` |
| **Summary** | Instructs the LLM to produce exactly 2–4 sentences of plain prose reviewing a past decision: (1) was the directional call correct? (cite alpha), (2) which part of thesis held/failed, (3) one concrete lesson. Output is stored verbatim in the memory log. Human message provides raw return and alpha vs SPY. |

---

## Utility Prompts (Injected into Other Prompts)

### 19. Language Instruction
| | |
|---|---|
| **File** | `tradingagents/agents/utils/agent_utils.py:23` |
| **Variable** | Return value of `get_language_instruction()` |
| **Summary** | Returns `f" Write your entire response in {lang}."` if `output_language` is not "English". Returns empty string for English (default). Injected into analysts (#1, #3, #5, #7) and Portfolio Manager (#17). Not injected into internal debate agents (#9, #10, #14, #15, #16). |

### 20. Instrument Context
| | |
|---|---|
| **File** | `tradingagents/agents/utils/agent_utils.py:37` |
| **Variable** | Return value of `build_instrument_context(ticker)` |
| **Summary** | Returns a short snippet: `"The instrument to analyze is \`{ticker}\`. Use this exact ticker in every tool call, report, and recommendation, preserving any exchange suffix (e.g. \`.TO\`, \`.L\`, \`.HK\`, \`.T\`)."` Injected into all agent prompts that touch instruments. |

---

## Execution Order (Graph Flow)

```
Analysts (#1–#8, sequentially: market → social → news → fundamentals)
  → Bull Researcher (#9) ↔ Bear Researcher (#10)  [debate, up to max_debate_rounds]
  → Research Manager (#11, structured output)
  → Trader (#12–#13, structured output)
  → Aggressive (#14) ↔ Conservative (#15) ↔ Neutral (#16)  [risk debate]
  → Portfolio Manager (#17, structured output)

Post-trade (deferred):
  → Reflector (#18)  [log reflection to memory]
```

---

## Prompt Architecture Notes

- **Shared collab preamble** (#2, #4, #6, #8): All four analysts share identical system prompt structure — just injected with different specialized instructions and tool lists.
- **No ChatPromptTemplate for debate agents**: Bull/Bear researchers (#9, #10) and risk analysts (#14, #15, #16) use raw f-strings directly — no message templating, no conversation history management beyond manual `\n` concatenation.
- **Structured output agents**: Research Manager (#11), Trader (#12–#13), and Portfolio Manager (#17) use `with_structured_output` via LangChain's `bind_structured()`, with graceful free-text fallback when a provider doesn't support it.
- **Language isolation**: Internal debate agents always operate in English (`get_language_instruction()` not applied). Only user-facing outputs (analysts, PM) respect `output_language`.
