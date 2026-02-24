# Prompt-Driven Planning-in-Code with TinyDB

## 1. Overview

This system implements a **planning-in-code architecture** using a language model and TinyDB.

Instead of asking the model to describe a plan in JSON or call tools step-by-step, the model is instructed to:

1. Interpret a natural language request
2. Write executable Python code that encodes the full plan
3. Execute that code in a controlled sandbox

The central principle:

> The model writes the plan as Python.
> The system executes it safely.

---

# 2. Database Schema

The system operates on two TinyDB tables.

## 2.1 Inventory Table (`inventory_tbl`)

| Field               | Type   | Description                                |
| ------------------- | ------ | ------------------------------------------ |
| `item_id`           | string | Unique product identifier (e.g., SG001)    |
| `name`              | string | Style of sunglasses (e.g., Aviator, Round) |
| `description`       | string | Text description                           |
| `quantity_in_stock` | int    | Current stock available                    |
| `price`             | float  | Price in USD                               |

---

## 2.2 Transactions Table (`transactions_tbl`)

| Field                       | Type   | Description                           |
| --------------------------- | ------ | ------------------------------------- |
| `transaction_id`            | string | Unique identifier (e.g., TXN001)      |
| `customer_name`             | string | Name of customer or `OPENING_BALANCE` |
| `transaction_summary`       | string | Short transaction description         |
| `transaction_amount`        | float  | Monetary value                        |
| `balance_after_transaction` | float  | Running balance                       |
| `timestamp`                 | string | ISO-8601 datetime                     |

---

# 3. Prompt Design

The prompt enforces:

* Dynamic query construction via `Query()`
* No hardcoded filters
* Automatic read vs mutate decision
* Strict transaction policy
* Mandatory `answer_text`
* Controlled output format

---

## 3.1 Prompt Template

```python
PROMPT = """You are a senior data assistant. PLAN BY WRITING PYTHON CODE USING TINYDB.

Database Schema & Samples (read-only):
{schema_block}

Execution Environment (already imported/provided):
- Variables: db, inventory_tbl, transactions_tbl
- Helpers: get_current_balance(tbl) -> float, next_transaction_id(tbl, prefix="TXN") -> str
- Natural language: user_request: str

PLANNING RULES:
- Derive ALL filters from user_request.
- Build TinyDB queries dynamically using Query().
- If ambiguous, default to read-only (DRY RUN).

TRANSACTION POLICY:
- No aggregated multi-item transactions.
- One transaction per item.
- Validate stock before mutation.
- If insufficient stock → abort mutation.

HUMAN RESPONSE REQUIREMENT:
- Set `answer_text` (1–2 sentence response).
- Set `STATUS` to:
  "success", "no_match", "insufficient_stock",
  "invalid_request", "unsupported_intent".

OUTPUT CONTRACT:
Return ONLY executable Python inside:

<execute_python>
# code here
</execute_python>

User request:
{question}
"""
```

---

# 4. Code Generation Stage

The system first generates the plan-as-code.
No execution occurs at this stage.

## 4.1 Code Generation Function

```python
def generate_llm_code(
    prompt: str,
    *,
    inventory_tbl,
    transactions_tbl,
    model: str = "gpt-4.1-mini",
    temperature: float = 0.2,
) -> str:
    """
    Generates a plan encoded as executable Python.
    Returns the FULL LLM response including <execute_python> tags.
    """
    schema_block = inv_utils.build_schema_block(inventory_tbl, transactions_tbl)
    formatted_prompt = PROMPT.format(schema_block=schema_block, question=prompt)

    resp = client.chat.completions.create(
        model=model,
        temperature=temperature,
        messages=[
            {
                "role": "system",
                "content": "You write safe, well-commented TinyDB code."
            },
            {"role": "user", "content": formatted_prompt},
        ],
    )

    return resp.choices[0].message.content or ""
```

---

# 5. Execution Sandbox

The generated code must be extracted and executed in a controlled namespace.

The sandbox:

* Exposes only TinyDB tables and helper functions
* Blocks file I/O
* Blocks network access
* Captures stdout
* Captures errors
* Extracts `answer_text`

---

## 5.1 Code Extraction Helper

```python
import re

def _extract_execute_block(text: str) -> str:
    """
    Extracts Python between <execute_python>...</execute_python>.
    If no tags exist, assumes raw Python.
    """
    if not text:
        raise RuntimeError("Empty content passed to executor.")

    match = re.search(
        r"<execute_python>(.*?)</execute_python>",
        text,
        re.DOTALL | re.IGNORECASE,
    )

    return match.group(1).strip() if match else text.strip()
```

---

## 5.2 Code Execution Function

```python
import io
import sys
import traceback
from typing import Optional, Dict, Any
from tinydb import Query

def execute_generated_code(
    code_or_content: str,
    *,
    db,
    inventory_tbl,
    transactions_tbl,
    user_request: Optional[str] = None,
) -> Dict[str, Any]:
    """
    Executes model-generated Python safely.
    """

    code = _extract_execute_block(code_or_content)

    SAFE_GLOBALS = {
        "Query": Query,
        "get_current_balance": inv_utils.get_current_balance,
        "next_transaction_id": inv_utils.next_transaction_id,
        "user_request": user_request or "",
    }

    SAFE_LOCALS = {
        "db": db,
        "inventory_tbl": inventory_tbl,
        "transactions_tbl": transactions_tbl,
    }

    stdout_buffer = io.StringIO()
    old_stdout = sys.stdout
    sys.stdout = stdout_buffer

    error_text = None

    try:
        exec(code, SAFE_GLOBALS, SAFE_LOCALS)
    except Exception:
        error_text = traceback.format_exc()
    finally:
        sys.stdout = old_stdout

    printed_logs = stdout_buffer.getvalue().strip()

    answer = (
        SAFE_LOCALS.get("answer_text")
        or SAFE_LOCALS.get("answer_rows")
        or SAFE_LOCALS.get("answer_json")
    )

    return {
        "code": code,
        "stdout": printed_logs,
        "error": error_text,
        "answer": answer,
        "inventory_after": inventory_tbl.all(),
        "transactions_after": transactions_tbl.all(),
    }
```

---

# 6. End-to-End Agent

The complete pipeline is wrapped in a single function.

## 6.1 Customer Service Agent

```python
def customer_service_agent(
    question: str,
    *,
    db,
    inventory_tbl,
    transactions_tbl,
    model: str = "o4-mini",
    temperature: float = 1.0,
) -> dict:
    """
    Full planning + execution pipeline.
    """

    # 1. Generate plan-as-code
    full_content = generate_llm_code(
        question,
        inventory_tbl=inventory_tbl,
        transactions_tbl=transactions_tbl,
        model=model,
        temperature=temperature,
    )

    # 2. Execute safely
    exec_result = execute_generated_code(
        full_content,
        db=db,
        inventory_tbl=inventory_tbl,
        transactions_tbl=transactions_tbl,
        user_request=question,
    )

    return {
        "full_content": full_content,
        "execution": exec_result,
    }
```

---

# 7. Architectural Summary

The system follows this pipeline:

Natural Language
→ Inject Live Schema
→ Model Writes Python Plan
→ Extract Execution Block
→ Execute in Sandbox
→ Return `answer_text`

This pattern:

* Reduces tool orchestration complexity
* Makes reasoning inspectable
* Enforces transaction safety
* Separates planning from execution
* Keeps mutations deterministic

---

# 8. Core Principle

This architecture treats Python as a reasoning language.

The model does not explain the plan.
The model writes the plan.
The system executes it under constraints.
