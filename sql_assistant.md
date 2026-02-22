## Prompt 3

### 3.1. Use an LLM to Query a Database

In this step, you will use a function that turns your natural-language questions into SQL queries.

You provide your question and the database schema as input. The LLM then generates the SQL query that answers your question.

This way, **you** can focus on asking questions while the model takes care of writing the query.


````python
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
````

----
````python
# Example usage of generate_sql

# We provide the schema as a string
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

# We ask a question about the data in natural language
question = "Which color of product has the highest total sales?"

utils.print_html(question, title="User Question")

# Generate the SQL query using the specified model
sql_V1 = generate_sql(question, schema, model="openai:gpt-4.1")

# Display the generated SQL query
utils.print_html(sql_V1, title="SQL Query V1")
````
----
### Improving SQL Queries with Reflection

In this section, you will learn how to **improve SQL queries using reflection**.

First, the LLM can review only the **SQL text** against the question and schema, and propose improvements if needed.
Then, the LLM can also consider the **actual query execution results** to catch subtle issues such as negative totals, missing filters, or grouping errors.

Together, these approaches show how reflection makes your SQL workflow more **reliable and accurate**—by first checking the logic on paper and then validating it against real data.

#### First Attempt: Refine a SQL query

In this function, you ask an LLM to **review** a SQL query against the original question and the schema (e.g., as the one defined in section 3.1.). The model reflects on whether the query fully answers the question and, if not, proposes an improved version.

* **Inputs**:

  * the user’s **question**
  * the **original SQL query**
  * the **table schema**

* **Outputs**:

  * **feedback** → a short evaluation (e.g., “valid but missing a date filter”)
  * **refined_sql** → the final SQL (unchanged if correct, or updated if improvements are needed)

This function does **not execute SQL**. It only inspects the query and suggests refinements when the logic doesn’t perfectly match the intent.


````python
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
        # Fallback if model doesn't return valid JSON
        feedback = content.strip()
        refined_sql = sql_query

    return feedback, refined_sql
````

----
### Putting it all together — Building the Database Query Workflow

In this step, **you** will use a function that automates the entire workflow of creating, running, and improving SQL queries with an LLM.

The workflow runs through these key steps:

1. **Extract the database schema**
2. **Generate an initial (V1) SQL query** from your natural language question
3. **Reflect on V1 using execution feedback** — review the actual query results and refine the SQL if needed
4. **Execute the improved (V2) SQL query** to ensure it fully answers your question

At the end, **you** will see:

* Both the initial and refined queries
* Their respective outputs
* Feedback from the LLM explaining the refinement

This makes it easier for **you** to work with SQL queries in a smoother, more accurate, and fully automated way.

````python
def run_sql_workflow(
    db_path: str,
    question: str,
    model_generation: str = "openai:gpt-4.1",
    model_evaluation: str = "openai:gpt-4.1",
):
    """
    End-to-end workflow to generate, execute, evaluate, and refine SQL queries.

    Steps:
      1) Extract database schema
      2) Generate SQL (V1)
      3) Execute V1 → show output
      4) Reflect on V1 with execution feedback → propose refined SQL (V2)
      5) Execute V2 → show final answer
    """

    # 1) Schema
    schema = utils.get_schema(db_path)
    utils.print_html(
        schema,
        title="📘 Step 1 — Extract Database Schema"
    )

    # 2) Generate SQL (V1)
    sql_v1 = generate_sql(question, schema, model_generation)
    utils.print_html(
        sql_v1,
        title="🧠 Step 2 — Generate SQL (V1)"
    )

    # 3) Execute V1
    df_v1 = utils.execute_sql(sql_v1, db_path)
    utils.print_html(
        df_v1,
        title="🧪 Step 3 — Execute V1 (SQL Output)"
    )

    # 4) Reflect on V1 with execution feedback → refine to V2
    feedback, sql_v2 = refine_sql_external_feedback(
        question=question,
        sql_query=sql_v1,
        df_feedback=df_v1,          # external feedback: real output of V1
        schema=schema,
        model=model_evaluation,
    )
    utils.print_html(
        feedback,
        title="🧭 Step 4 — Reflect on V1 (Feedback)"
    )
    utils.print_html(
        sql_v2,
        title="🔁 Step 4 — Refined SQL (V2)"
    )

    # 5) Execute V2
    df_v2 = utils.execute_sql(sql_v2, db_path)
    utils.print_html(
        df_v2,
        title="✅ Step 5 — Execute V2 (Final Answer)"
    )
````

----
### Run the SQL Workflow

Now it’s time for **you** to execute the complete SQL processing pipeline. You can try different combinations of the following OpenAI models, each with different capabilities and performance:

* `openai:gpt-4o`
* `openai:gpt-4.1`
* `openai:gpt-4.1-mini`
* `openai:gpt-3.5-turbo`

💡 In this workflow, `openai:gpt-4.1` often gives the best results for self-reflection tasks.

**Important:** Because Large Language Models (LLMs) are stochastic, every run may return slightly different results.
You are encouraged to experiment with different models and combinations to find the setup that works best for **you**.

````python
run_sql_workflow(
    "products.db", 
    "Which color of product has the highest total sales?",
    model_generation="openai:gpt-4.1",
    model_evaluation="openai:gpt-4.1"
)
````
