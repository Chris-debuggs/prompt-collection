Perfect. You want the same content, but cleaned up into proper Markdown — no inline HTML styling, structured like a technical report, readable in GitHub.

Here is your reorganized section rewritten cleanly.

---

## Tables

The system operates over two TinyDB tables: `inventory_tbl` and `transactions_tbl`.

### Inventory Table (`inventory_tbl`)

| Field               | Type   | Description                                |
| ------------------- | ------ | ------------------------------------------ |
| `item_id`           | string | Unique product identifier (e.g., SG001)    |
| `name`              | string | Style of sunglasses (e.g., Aviator, Round) |
| `description`       | string | Text description of the product            |
| `quantity_in_stock` | int    | Current stock available                    |
| `price`             | float  | Price in USD                               |

---

### Transactions Table (`transactions_tbl`)

| Field                       | Type   | Description                                                  |
| --------------------------- | ------ | ------------------------------------------------------------ |
| `transaction_id`            | string | Unique identifier (e.g., TXN001)                             |
| `customer_name`             | string | Name of the customer, or `OPENING_BALANCE` for initial entry |
| `transaction_summary`       | string | Short description of the transaction                         |
| `transaction_amount`        | float  | Amount of money for this transaction                         |
| `balance_after_transaction` | float  | Running balance after applying the transaction               |
| `timestamp`                 | string | ISO-8601 formatted date/time of the transaction              |

---

## The Plan

Once the schema is clear, the next step is to build a prompt that instructs the model to **plan by writing Python code**, and then execute that code.

The central idea is:

> The code is the plan.

The model does not describe the reasoning in prose. Instead, it writes executable Python that:

* Parses the user’s request
* Builds dynamic TinyDB queries
* Decides whether the action is read-only or mutating
* Applies strict transaction rules
* Produces a short customer-facing response

The prompt enforces:

* No file I/O
* No network access
* TinyDB `Query()` for filtering
* Sequential and consistent mutations
* Clear human response contract

---

## Prompt 1

```python
PROMPT = """You are a senior data assistant. PLAN BY WRITING PYTHON CODE USING TINYDB.

Database Schema & Samples (read-only):
{schema_block}

Execution Environment (already imported/provided):
- Variables: db, inventory_tbl, transactions_tbl  # TinyDB Table objects
- Helpers: get_current_balance(tbl) -> float, next_transaction_id(tbl, prefix="TXN") -> str
- Natural language: user_request: str  # the original user message

PLANNING RULES (critical):
- Derive ALL filters/parameters from user_request (shape/keywords, price ranges "under/over/between", stock mentions,
  quantities, buy/return intent). Do NOT hard-code values.
- Build TinyDB queries dynamically with Query(). If a constraint isn't in user_request, don't apply it.
- Be conservative: if intent is ambiguous, do read-only (DRY RUN).

TRANSACTION POLICY (hard):
- Do NOT create aggregated multi-item transactions.
- If the request contains multiple items, create a separate transaction row PER ITEM.
- For each item:
  - compute its own line total (unit_price * qty),
  - insert ONE transaction with that amount,
  - update balance sequentially (balance += line_total),
  - update the item’s stock.
- If any requested item lacks sufficient stock, do NOT mutate anything; reply with STATUS="insufficient_stock".

HUMAN RESPONSE REQUIREMENT (hard):
- You MUST set a variable named `answer_text` (type str) with a short, customer-friendly sentence (1–2 lines).
- This sentence is the only user-facing message. No dataframes/JSON, no boilerplate disclaimers.
- If nothing matches, politely say so and offer a nearby alternative (closest style/price) or a next step.

ACTION POLICY:
- If the request clearly asks to change state (buy/purchase/return/restock/adjust):
    ACTION="mutate"; SHOULD_MUTATE=True; perform the change and write a matching transaction row.
  Otherwise:
    ACTION="read"; SHOULD_MUTATE=False; simulate and explain briefly as a dry run (in logs only).

FAILURE & EDGE-CASE HANDLING (must implement):
- Do not capture outer variables in Query.test. Pass them as explicit args.
- Always set a short `answer_text`. Also set a string `STATUS` to one of:
  "success", "no_match", "insufficient_stock", "invalid_request", "unsupported_intent".
- no_match: No items satisfy the filters → suggest the closest in style/price, or invite a different range.
- insufficient_stock: Item found but stock < requested qty → state available qty and offer the max you can fulfill.
- invalid_request: Unable to parse essential info (e.g., quantity for a purchase/return) → ask for the missing piece succinctly.
- unsupported_intent: The action is outside the store’s capabilities → provide the nearest supported alternative.
- In all cases, keep the tone helpful and concise (1–2 sentences). Put technical details (e.g., ACTION/DRY RUN) only in stdout logs.

OUTPUT CONTRACT:
- Return ONLY executable Python between these tags (no extra text):
  <execute_python>
  # your python
  </execute_python>

CODE CHECKLIST (follow in code):
1) Parse intent & constraints from user_request (regex ok).
2) Build TinyDB condition incrementally; query inventory_tbl.
3) If mutate: validate stock, update inventory, insert a transaction (new id, amount, balance, timestamp).
4) ALWAYS set:
   - `answer_text` (human sentence, required),
   - `STATUS` (see list above).
   Also print a brief log to stdout, e.g., "LOG: ACTION=read DRY_RUN=True STATUS=no_match".
5) Optional: set `answer_rows` or `answer_json` if useful, but `answer_text` is mandatory.

TONE EXAMPLES (for `answer_text`):
- success: "Yes, we have our Classic sunglasses, a round frame, for $60."
- no_match: "We don’t have round frames under $100 in stock right now, but our Moon round frame is available at $120."
- insufficient_stock: "We only have 1 pair of Classic left; I can reserve that for you."
- invalid_request: "I can help with that—how many pairs would you like to purchase?"
- unsupported_intent: "We can’t refurbish frames, but I can suggest similar new models."

Constraints:
- Use TinyDB Query for filtering. Standard library imports only if needed.
- Keep code clear and commented with numbered steps.

User request:
{question}
"""
```

---

## From Prompt to Code (Planning in Code)

Instead of asking the model to:

* Output a JSON plan
* Call multiple small tools
* Execute step-by-step logic externally

We instruct it to:

Write the entire reasoning process directly in Python.

The generated code becomes the executable plan.

This approach leverages:

* The model’s fluency in Python
* TinyDB as a structured reasoning substrate
* A controlled execution sandbox

The flow is:

Natural Language
→ Prompt Injection (with live schema)
→ Model Writes Python Plan
→ Extract `<execute_python>` Block
→ Execute Safely
→ Return `answer_text`

---

If you'd like, I can now restructure the rest (Usage, Examples, Executor, Agent wiring) into the same clean report-style format so the entire document reads consistently.
