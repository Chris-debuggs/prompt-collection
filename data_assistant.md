## Table

As you can see above, the schemas of each table are as follows:

<div style="border:1px solid #BFDBFE; border-left:6px solid #3B82F6; background:#EFF6FF; border-radius:6px; padding:16px; font-family:system-ui,-apple-system,Segoe UI,Roboto,Ubuntu,Cantarell,Noto Sans,sans-serif; line-height:1.6; color:#1E3A8A;">

  <h4 style="margin-top:0; color:#1E40AF;">Inventory Table (<code>inventory_tbl</code>)</h4>
  <ul>
    <li><strong>item_id</strong> (string): Unique product identifier (e.g., SG001).</li>
    <li><strong>name</strong> (string): Style of sunglasses (e.g., Aviator, Round).</li>
    <li><strong>description</strong> (string): Text description of the product.</li>
    <li><strong>quantity_in_stock</strong> (int): Current stock available.</li>
    <li><strong>price</strong> (float): Price in USD.</li>
  </ul>
  <h4 style="margin-top:1em; color:#1E40AF;">Transactions Table (<code>transactions_tbl</code>)</h4>
  <ul>
    <li><strong>transaction_id</strong> (string): Unique identifier (e.g., TXN001).</li>
    <li><strong>customer_name</strong> (string): Name of the customer, or <code>OPENING_BALANCE</code> for initial entry.</li>
    <li><strong>transaction_summary</strong> (string): Short description of the transaction.</li>
    <li><strong>transaction_amount</strong> (float): Amount of money for this transaction.</li>
    <li><strong>balance_after_transaction</strong> (float): Running balance after applying the transaction.</li>
    <li><strong>timestamp</strong> (string): ISO-8601 formatted date/time of the transaction.</li>
  </ul>
</div>

## The plan

Once the schema is clear, you’ll build the **prompt** that instructs the model to *plan by writing code* and then execute that code. As Andrew emphasized, the code is the plan: the model explains each step in comments, then carries it out. Your prompt below also makes the model self-decide whether the request is read-only or a state change, and it enforces safe execution (no I/O, no network, TinyDB Query only, consistent mutations).


## Prompt 1

````python
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
````
### From Prompt to Code (Planning in Code)

Let’s generate code that **is the plan**.

Instead of asking the model to output a plan in JSON and running it step-by-step with many tiny tools, let’s have it **write Python that encodes the whole plan** (e.g., “filter this, then compute that, then update this row”). The function `generate_llm_code`:

1. **Builds a live schema** from `inventory_tbl` and `transactions_tbl` so the model sees real fields, types, and examples.
2. **Formats the prompt** with that schema plus the user’s question.
3. **Calls the model** to produce a **plan-with-code** response — typically an `<execute_python>...</execute_python>` block whose body contains the step-by-step logic.
4. **Returns the full response** (including the plan and the code).  
   *We don’t execute anything in this step.*

Why this pattern? Let’s leverage Python/TinyDB as a rich toolbox the model already “knows,” so it can compose multi-step solutions directly in code instead of relying on a growing set of bespoke tools. We’ll extract and run the code in a later step.

## Usage 

````python
# ---------- 1) Code generation ----------
def generate_llm_code(
    prompt: str,
    *,
    inventory_tbl,
    transactions_tbl,
    model: str = "gpt-4.1-mini",
    temperature: float = 0.2,
) -> str:
    """
    Ask the LLM to produce a plan-with-code response.
    Returns the FULL assistant content (including surrounding text and tags).
    The actual code extraction happens later in execute_generated_code.
    """
    schema_block = inv_utils.build_schema_block(inventory_tbl, transactions_tbl)
    prompt = PROMPT.format(schema_block=schema_block, question=prompt)

    resp = client.chat.completions.create(
        model=model,
        temperature=temperature,
        messages=[
            {
                "role": "system",
                "content": "You write safe, well-commented TinyDB code to handle data questions and updates."
            },
            {"role": "user", "content": prompt},
        ],
    )
    content = resp.choices[0].message.content or ""
    
    return content
````

### Try a Sample Prompt (Planning-in-Code)

We’ll use the same prompt Andrew used in the lecture:

> **Prompt:** “Do you have any round sunglasses in stock that are under $100?”

Before generating any code, let’s manually inspect the TinyDB tables to see if there are truly *round* frames (word-only match) and what their prices look like. Run the next cell to preview the inventory and highlight items that match the word-only “round” filter.

