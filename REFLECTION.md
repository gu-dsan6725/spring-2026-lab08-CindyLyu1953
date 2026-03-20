# Lab 08 Reflection Write-up

## Streaming Stock Agent (EXERCISE.md)

### 1. How did the LLM know to call your new tool?

The LLM uses the **tool schema** provided in `STOCK_TOOLS` to decide when to call each tool. Each tool has:
- **name**: `"compare_stocks"` — identifies the tool
- **description**: A natural-language explanation such as "Compare two stocks side-by-side with current price, company name, and market cap. Use this when the user asks to compare two stocks, e.g. 'Compare AAPL and MSFT', 'Which is better Tesla or Ford?'" — helps the model match user intent to the right tool
- **parameters**: JSON Schema specifying `symbol1` and `symbol2`

The model reads the user query, maps it to the description of `compare_stocks`, and chooses that tool when the query fits a comparison request.

---

### 2. What happens if the tool schema description is unclear?

If the description is vague or ambiguous, the LLM may:
- **Use the wrong tool** (e.g., call `get_stock_price` twice instead of `compare_stocks`)
- **Misinterpret parameters** (e.g., wrong order for symbol1 vs symbol2)
- **Avoid the tool** and respond from general knowledge without real data
- **Call the tool inconsistently** for similar queries

Clear, example-driven descriptions help the model reliably pick the right tool when appropriate.

---

### 3. How does the agent decide between `compare_stocks` and `get_stock_price`?

The agent compares:
- **Query intent**: Comparison of two stocks → `compare_stocks`; single-stock price check → `get_stock_price`
- **Tool descriptions**: Descriptions explicitly say when each tool should be used. For `compare_stocks`, phrases like "compare two stocks" and examples such as "Compare AAPL and MSFT" guide the choice.
- **Parameters**: Single ticker → `get_stock_price`; two symbols → `compare_stocks`

The system prompt and prior context also influence tool choice. Without a clear comparison intent, single-stock queries default to `get_stock_price`.

---

### 4. How would you add validation for parameter values?

Possible validation:

1. **In the tool function**: Validate inside `_compare_stocks()` before calling yfinance:
   ```python
   def _validate_symbol(symbol: str) -> str:
       if not symbol or len(symbol) > 10:
           raise ValueError("Invalid symbol")
       return symbol.upper()
   ```

2. **JSON Schema**: Use `pattern`, `minLength`, and `maxLength` to constrain symbols at the schema level.

3. **Pre-processing**: Add a validation layer that checks symbols against a known list or format before the agent calls the tool.

4. **Error handling**: Return clear errors from the tool for invalid symbols so the model can correct itself or ask the user.

---

### 5. What if the user asks to compare 3 stocks instead of 2?

The current `compare_stocks` tool only accepts two symbols. Options:

1. **Tool extension**: Add optional parameters like `symbol3`, `symbol4`, etc., or a `symbols: list[str]` array so the tool supports more than two symbols.
2. **Chained calls**: Have the LLM call `compare_stocks` multiple times (e.g., A vs B, A vs C, B vs C) and combine results.
3. **User clarification**: Instruct the model to ask the user to choose two of the three stocks, or to compare them pairwise.
4. **New tool**: Add a `compare_multiple_stocks` tool with a list parameter for 3+ symbols.

---

## Personal Financial Analyst (EXERCISE.md)

The Personal Financial Analyst exercise does not list explicit "Questions to Consider." Below are brief reflections aligned with the Key Learning Points.

### Orchestrator–Workers Pattern

A single orchestrator coordinates several specialized workers. This keeps concerns separated and makes it easier to change or add agents without redesigning the whole flow.

### Model Selection Strategy

Using Sonnet for orchestration and Haiku for sub-agents reduces cost while keeping coordination and reasoning quality high. Sonnet handles planning, coordination, and synthesis; Haiku handles well-defined, prompt-guided tasks.

### MCP Protocol Integration

MCP provides a standard way to connect to external data (bank and credit card servers). Servers run in separate processes, tools are discovered via the protocol, and the orchestrator can call them without knowing low-level details.

### Agent vs. Tool Decision

Tools are used for data fetching and simple operations (e.g., `get_bank_transactions`, `get_credit_card_transactions`, file writes). Sub-agents are used for harder tasks that need research, negotiation planning, or tax analysis. This split keeps simple operations fast and cheap and reserves more complex reasoning for agents.

### Parallel Execution

The orchestrator prompt tells the model to invoke research, negotiation, and tax agents in parallel. Because these agents work on the same data and produce independent outputs, parallel execution improves speed and the SDK manages coordination for the orchestrator.
