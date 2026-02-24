---

# Prompt 3 — LLM-Powered SQL Query Generation with Reflection

## 1. Overview

In this section, the system uses a Large Language Model (LLM) to:

1. Convert natural-language questions into SQL queries
2. Execute those queries
3. Reflect on correctness
4. Improve the query if necessary

The goal is to let the user focus on asking questions, while the model handles SQL generation and refinement.

This workflow consists of:

* Query generation
* Logical reflection
* Execution feedback reflection
* Final validated result

---

# 2. Step 1 — Generate SQL from Natural Language

The first function converts a natural-language question into a valid SQLite query.

The model is given:

* The database schema
* The user’s question

It must return only SQL.

---

## 2.1 SQL Generation Function

```python
def generate_sql(question: str, schema: str, model: str) -> str:
    prompt = f"""
    You are a SQL assistant. Given the schema and the user's question, write a SQL query for SQLite.

    Schema:
    {schema}

    User question:
    {question}

    Respond with the SQL only.
    """
    response = client.chat.completions.create(
        model=model,
        messages=[{"role": "user", "content": prompt}],
        temperature=0,
    )
    return response.choices[0].message.content.strip()
```

### Design Notes

* `temperature=0` ensures deterministic SQL generation.
* The model is restricted to SQLite syntax.
* The output must contain only SQL (no explanation).

---

## 2.2 Example Usage

```python
schema = """
Table name: transactions
id (INTEGER)
product_id (INTEGER)
product_name (TEXT)
brand (TEXT)
category (TEXT)
color (TEXT)
action (TEXT)
qty_delta (INTEGER)
unit_price (REAL)
notes (TEXT)
ts (DATETIME)
"""

question = "Which color of product has the highest total sales?"

sql_v1 = generate_sql(question, schema, model="openai:gpt-4.1")
```

At this stage:

* The model produces SQL (Version 1)
* The query has not yet been validated against real data

---

# 3. Improving SQL Queries with Reflection

Initial SQL may be syntactically correct but logically incomplete.

Common issues:

* Missing filters
* Incorrect grouping
* Incorrect aggregation
* Ignoring negative values
* Wrong interpretation of "sales"

Reflection improves reliability.

Two levels of reflection are used:

1. Logical reflection (review SQL text only)
2. Execution-based reflection (review actual results)

---

# 4. Step 2 — Logical Reflection on SQL

This function evaluates whether the SQL logically answers the question.

It does not execute SQL.
It inspects only:

* The original question
* The SQL query
* The schema

---

## 4.1 SQL Refinement Function

```python
def refine_sql(
    question: str,
    sql_query: str,
    schema: str,
    model: str,
) -> tuple[str, str]:
    """
    Reflect on whether a query's *shown output* answers the question,
    and propose an improved SQL if needed.
    Returns (feedback, refined_sql).
    """
    prompt = f"""
You are a SQL reviewer and refiner.

User asked:
{question}

Original SQL:
{sql_query}

Table Schema:
{schema}

Step 1: Briefly evaluate if the SQL OUTPUT fully answers the user's question.
Step 2: If improvement is needed, provide a refined SQL query for SQLite.
If the original SQL is already correct, return it unchanged.

Return STRICT JSON with two fields:
{{
  "feedback": "<1-3 sentences explaining the gap or confirming correctness>",
  "refined_sql": "<final SQL to run>"
}}
"""
    response = client.chat.completions.create(
        model=model,
        messages=[{"role": "user", "content": prompt}],
        temperature=0,
    )

    content = response.choices[0].message.content
    try:
        obj = json.loads(content)
        feedback = str(obj.get("feedback", "")).strip()
        refined_sql = str(obj.get("refined_sql", sql_query)).strip()
        if not refined_sql:
            refined_sql = sql_query
    except Exception:
        feedback = content.strip()
        refined_sql = sql_query

    return feedback, refined_sql
```

### Why JSON Output?

Structured JSON ensures:

* Stable parsing
* Predictable workflow integration
* Safe fallback handling

---

# 5. Step 3 — Execution-Based Reflection

Logical reflection checks reasoning “on paper.”

Execution-based reflection checks the real output.

This catches subtle issues such as:

* Negative totals from returns
* Missing `WHERE action='sale'`
* Incorrect grouping
* Null values

This produces a refined Version 2 (V2) query.

---

# 6. End-to-End SQL Workflow

The full pipeline integrates:

1. Schema extraction
2. SQL generation (V1)
3. Execution of V1
4. Reflection with execution feedback
5. SQL refinement (V2)
6. Execution of V2

---

## 6.1 Workflow Function

```python
def run_sql_workflow(
    db_path: str,
    question: str,
    model_generation: str = "openai:gpt-4.1",
    model_evaluation: str = "openai:gpt-4.1",
):
    """
    End-to-end workflow to generate, execute, evaluate, and refine SQL queries.
    """

    # 1) Extract Schema
    schema = utils.get_schema(db_path)
    utils.print_html(schema, title="Step 1 — Extract Database Schema")

    # 2) Generate SQL (V1)
    sql_v1 = generate_sql(question, schema, model_generation)
    utils.print_html(sql_v1, title="Step 2 — Generate SQL (V1)")

    # 3) Execute V1
    df_v1 = utils.execute_sql(sql_v1, db_path)
    utils.print_html(df_v1, title="Step 3 — Execute V1")

    # 4) Reflect and refine to V2
    feedback, sql_v2 = refine_sql_external_feedback(
        question=question,
        sql_query=sql_v1,
        df_feedback=df_v1,
        schema=schema,
        model=model_evaluation,
    )
    utils.print_html(feedback, title="Step 4 — Reflection Feedback")
    utils.print_html(sql_v2, title="Step 4 — Refined SQL (V2)")

    # 5) Execute V2
    df_v2 = utils.execute_sql(sql_v2, db_path)
    utils.print_html(df_v2, title="Step 5 — Final Answer (V2)")
```

---

# 7. Running the Workflow

```python
run_sql_workflow(
    "products.db",
    "Which color of product has the highest total sales?",
    model_generation="openai:gpt-4.1",
    model_evaluation="openai:gpt-4.1"
)
```

---

# 8. Model Selection Considerations

Available models:

* `openai:gpt-4o`
* `openai:gpt-4.1`
* `openai:gpt-4.1-mini`
* `openai:gpt-3.5-turbo`

Observations:

* `gpt-4.1` performs best for reflection-heavy tasks.
* Lower-tier models may require multiple iterations.
* Results are stochastic; outputs may vary slightly per run.

---

# 9. Architectural Pattern

This workflow demonstrates a broader principle:

Initial generation is fast.
Reflection improves reliability.
Execution feedback improves correctness.

The full pipeline becomes:

Natural Language
→ SQL (V1)
→ Execute
→ Reflect
→ Refine (V2)
→ Execute
→ Final Answer

This pattern generalizes beyond SQL to:

* Data pipelines
* Code generation
* API orchestration
* Automated analytics

---

# 10. Key Insight

LLMs are good at generating.
They become significantly better when allowed to critique themselves.

Generation + Reflection + Execution Feedback
creates a far more reliable system than single-pass generation alone.
