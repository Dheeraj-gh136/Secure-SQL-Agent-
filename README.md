# Secure Intelligent SQL Agent

A small end-to-end AI security project that converts natural-language database questions into PostgreSQL SQL, validates the generated SQL, executes only safe read-only queries, and logs every accepted/rejected decision.

## V0.1 scope

**Natural language → AI SQL generation → SQL validation → safe PostgreSQL execution → result**

The project deliberately uses a tiny schema so the security boundary is easy to explain.

### Security controls

1. Only `SELECT` statements are accepted.
2. Destructive SQL keywords are blocked.
3. Multiple SQL statements are blocked.
4. `SELECT INTO` is blocked.
5. Only `customers`, `products`, and `orders` may be referenced.
6. SQL is parsed with `sqlglot` before execution.
7. PostgreSQL execution uses a dedicated read-only database role.
8. The transaction is explicitly marked `READ ONLY`.
9. Generation, validation and execution failures fail closed.
10. Every generated SQL statement and ACCEPTED/REJECTED decision is written to `logs/audit.log`.

## Architecture

```text
User
  |
  v
Streamlit UI
  |
  v
LLM (natural language -> SQL)
  |
  v
SQL Validator
  |---- invalid / unsafe ----> REJECT + audit log
  |
  v
Read-only PostgreSQL role
  |
  v
Result table
```

## Quick start

### 1. Start PostgreSQL

```bash
docker compose up -d
```

### 2. Create a Python environment

Windows:

```powershell
python -m venv .venv
.venv\Scripts\activate
```

Linux/macOS:

```bash
python3 -m venv .venv
source .venv/bin/activate
```

### 3. Install dependencies

```bash
pip install -r requirements.txt
```

### 4. Add your OpenAI API key

PowerShell:

```powershell
$env:OPENAI_API_KEY="YOUR_KEY"
```

Linux/macOS:

```bash
export OPENAI_API_KEY="YOUR_KEY"
```

Optional model override:

```powershell
$env:OPENAI_MODEL="gpt-5.6-luna"
```

### 5. Run

```bash
streamlit run app/main.py
```

Open the local Streamlit URL shown in the terminal.

## Demo requests

### Normal

```text
Show all customers from Hyderabad
```

Expected: generated SELECT, validation ACCEPTED, result table.

```text
Show the top 5 products by price
```

Expected: generated SELECT with ORDER BY/LIMIT, validation ACCEPTED.

### Edge cases

```text
Show data from employees
```

Expected: `CANNOT_ANSWER` or rejection because `employees` is not in the allowed schema.

```text
Show customers and then do something unrelated
```

Expected: safe failure if the model cannot map it to the schema.

### Unsafe

```text
Delete all customers
```

Expected: rejected. The system never executes destructive SQL.

```text
Drop the orders table
```

Expected: rejected.

```text
SELECT * FROM customers; DROP TABLE customers;
```

Expected: rejected because multiple statements are blocked.

## Tests

```bash
pytest
```

The tests cover normal SELECT SQL, destructive statements, unknown tables, multiple statements and invalid SQL.

## Logging

Every request creates an audit record in:

```text
logs/audit.log
```

Example:

```json
{
  "question": "Delete all customers",
  "sql": "DELETE FROM customers",
  "decision": "REJECTED",
  "reason": "Only SELECT statements are allowed."
}
```

## What to explain in a demo

**Why use an LLM?**  
The LLM translates natural language into SQL, but it does not get permission to execute SQL directly.

**Why validate after generation?**  
Generated SQL is treated as untrusted input. The validator is the security boundary.

**Why use a read-only database role?**  
Even if application validation fails, the database account has only SELECT privileges.

**Why fail closed?**  
Invalid, ambiguous or unknown requests should produce a rejection rather than attempting a risky best-effort query.

## V0.1 limitations

- Small fixed schema.
- Read-only queries only.
- No authentication.
- No row-level security.
- No complex policy engine.
- No automatic schema discovery.

These are intentional scope decisions for an explainable V0.1.

## Suggested Git history

```bash
git add .
git commit -m "feat: add secure natural language SQL agent"

git add .
git commit -m "security: add SQL validation and read-only execution"

git add .
git commit -m "test: add unsafe and edge case SQL tests"

git add .
git commit -m "docs: add architecture and demo instructions"
```
