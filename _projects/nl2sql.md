---
title: "NL2SQL: LLM-Powered Natural Language to Database Query Engine"
collection: projects
type: "Industry Project"
permalink: /projects/nl2sql
# venue: "Seagate Technology LLC"
date: 2023-08-01
excerpt: "Multi-stage LLM pipeline achieving 91% accuracy that converts plain-English questions into optimized SQL queries, cutting query fulfillment from days to under 3 seconds. Built during AI/ML internship at Seagate Technology."
---

What if anyone in your organization could query complex enterprise databases — without writing a single line of SQL? During my internship at Seagate Technology, I designed and built exactly that: a multi-stage LLM pipeline that converts plain-English questions into precise, optimized database queries in real time — cutting query fulfillment from days to seconds.

The Problem
======

Enterprise databases are powerful but inaccessible. At any large manufacturing or tech company, only a small fraction of employees have the SQL skills to extract what they need. Everyone else is either writing brittle ad-hoc queries that break on schema changes, or submitting tickets and waiting days for the data team to fulfill their requests.

The gap between *"I need this information to make a decision"* and *"I can actually get it"* is one of the most expensive inefficiencies in large organizations.

The Solution: A Prompt-Engineered LLM Pipeline
======

The system uses a layered architecture — not a single LLM call. Each stage is purpose-built to handle a specific failure mode that a naive approach would hit.

### Pipeline Architecture

```
  User Input (Natural Language)
          │
          ▼
  ┌───────────────────┐
  │   Intent Filter   │  ← Classifies: queryable / ambiguous / out-of-scope
  └────────┬──────────┘
           ▼
  ┌───────────────────┐
  │   Schema Loader   │  ← Pulls only relevant table metadata (not full DB)
  └────────┬──────────┘
           ▼
  ┌───────────────────┐
  │   Prompt Engine   │  ← Few-shot prompts, dynamically selected by query type
  └────────┬──────────┘
           ▼
  ┌───────────────────┐
  │   LLM Inference   │  ← Generates candidate SQL
  └────────┬──────────┘
           ▼
  ┌───────────────────┐
  │  Validation Layer │  ← AST parse + schema consistency check
  └────────┬──────────┘
           │  ✗ fail → feeds error context back to LLM for retry
           ▼  ✓ pass
  ┌───────────────────┐
  │  Query Executor   │  ← Runs against DB, returns structured results
  └───────────────────┘
```

Showcase: Input → Output
======

Below are representative examples of the conversions the system handles — from simple lookups to multi-table analytical queries with CTEs. Complexity increases with each example.

### Level 1 — Simple Lookup

**Natural Language Input:**
> *"Show me all products manufactured in Q3 2023"*

**Generated SQL:**
```sql
SELECT 
    p.product_id, 
    p.product_name, 
    p.category, 
    p.manufacture_date
FROM products p
WHERE p.manufacture_date BETWEEN '2023-07-01' AND '2023-09-30'
ORDER BY p.manufacture_date DESC;
```

---

### Level 2 — Filtered Aggregation with JOIN

**Natural Language Input:**
> *"What's the average yield rate per production line for drives that passed final inspection last month?"*

**Generated SQL:**
```sql
SELECT 
    pl.line_id,
    pl.line_name,
    ROUND(AVG(pr.yield_rate), 2)  AS avg_yield_rate,
    COUNT(*)                       AS total_units_passed
FROM production_runs pr
JOIN production_lines    pl ON pr.line_id       = pl.line_id
JOIN inspection_results  ir ON pr.run_id        = ir.run_id
WHERE ir.status = 'PASSED'
  AND ir.inspection_date >= DATE_TRUNC('month', CURRENT_DATE - INTERVAL '1 month')
  AND ir.inspection_date <  DATE_TRUNC('month', CURRENT_DATE)
GROUP BY pl.line_id, pl.line_name
ORDER BY avg_yield_rate DESC;
```

---

### Level 3 — Multi-Table Analytical Query with CTEs

**Natural Language Input:**
> *"Compare defect rates between the two highest-volume facilities this quarter, broken down by defect category"*

**Generated SQL:**
```sql
WITH facility_volumes AS (
    SELECT 
        f.facility_id,
        f.facility_name,
        COUNT(*) AS total_units
    FROM production_runs pr
    JOIN facilities f ON pr.facility_id = f.facility_id
    WHERE pr.run_date >= DATE_TRUNC('quarter', CURRENT_DATE)
    GROUP BY f.facility_id, f.facility_name
    ORDER BY total_units DESC
    LIMIT 2
),
defect_summary AS (
    SELECT 
        pr.facility_id,
        dc.category_name,
        COUNT(*)                                                        AS defect_count,
        COUNT(*) * 100.0 / COUNT(*) OVER(PARTITION BY pr.facility_id)  AS defect_rate_pct
    FROM defects d
    JOIN defect_categories dc  ON d.category_id  = dc.category_id
    JOIN production_runs   pr  ON d.run_id       = pr.run_id
    WHERE pr.run_date >= DATE_TRUNC('quarter', CURRENT_DATE)
      AND pr.facility_id IN (SELECT facility_id FROM facility_volumes)
    GROUP BY pr.facility_id, dc.category_name
)
SELECT 
    fv.facility_name,
    ds.category_name,
    ds.defect_count,
    ROUND(ds.defect_rate_pct, 2) AS defect_rate_pct
FROM defect_summary ds
JOIN facility_volumes fv ON ds.facility_id = fv.facility_id
ORDER BY fv.facility_name, ds.defect_rate_pct DESC;
```

---

### Edge Case — Ambiguous Intent (Caught Before SQL Generation)

**Natural Language Input:**
> *"How are we doing?"*

**System Response:**
> *Your question is too broad to generate a query. Could you be more specific? For example: production yield, defect rates, inventory levels, or facility performance.*

The intent filter classifies this as non-queryable *before* it ever reaches the LLM inference stage — no wasted API calls, no hallucinated SQL.

Prompt Engineering: The Core Innovation
======

Getting an LLM to write syntactically valid SQL is easy. Getting it to write *semantically correct* SQL against an unfamiliar enterprise schema — consistently, at scale — is the actual engineering problem. Here's how the prompt strategy was structured:

```python
SYSTEM_PROMPT = """
You are an enterprise SQL query assistant. Convert the user's natural 
language question into a precise SQL query.

RULES:
1. Only reference tables and columns present in the provided schema.
2. Use explicit JOINs — never implicit comma joins.
3. If the question is ambiguous, respond with a clarification request 
   instead of guessing.
4. For date-based queries, default to the most recent relevant period 
   unless the user specifies otherwise.
5. Prefer CTEs over nested subqueries for readability.
6. NEVER generate DROP, DELETE, UPDATE, or DDL statements.

SCHEMA CONTEXT:
{schema_context}

EXAMPLES (selected by query classifier):
{few_shot_examples}

USER QUESTION:
{user_input}
"""
```

Four techniques drove the jump from prototype to production-grade accuracy:

**Schema injection** — Only the tables and columns likely relevant to the query are loaded into the prompt. A full enterprise schema would blow past the context window and dilute attention; selective injection keeps the LLM focused and cuts hallucinated column references by over 70%.

**Dynamic few-shot selection** — The system classifies the incoming query (lookup / aggregation / join / time-series / comparison) and loads 3–5 curated examples *of that type*. Static examples across all query types performed significantly worse.

**Strict guardrails** — Explicit rules block destructive statements and force the model toward safer, more readable patterns (CTEs over nested subqueries). These aren't suggestions in the prompt — they're enforced constraints.

**Validation feedback loop** — If the generated SQL fails the AST parse or schema consistency check, the error message and the original query are sent back to the LLM for a second pass. This self-correction loop recovered ~60% of first-pass failures.

Results
======

| Metric | Before (Manual / Ticket Queue) | After (NL2SQL) |
|---|---|---|
| Time to query result | 2–5 business days | < 3 seconds |
| SQL skill required | Intermediate to Advanced | None |
| Query accuracy (test suite) | N/A | 91% |
| Error rate | ~15% (typos, wrong schema refs) | < 4% (validator catches remainder) |
| Non-technical user access | Blocked | Full self-service |

The system didn't just speed up queries — it removed the bottleneck entirely for non-technical stakeholders.

Tech Stack
======

* **LLM Layer:** OpenAI API (GPT-3.5 / GPT-4), multi-stage prompt chaining
* **Validation:** AST-based SQL parsing, schema-consistency verification
* **Backend:** Python, FastAPI
* **Orchestration:** Apache Airflow (schema refresh & pipeline scheduling)
* **Database:** PostgreSQL, with abstraction layer for multi-DB compatibility

Key Takeaways
======

**Prompt engineering is systematic engineering.** The gap between a 60% and 91% accuracy system wasn't a single clever prompt — it was structured few-shot selection, guardrail rules, selective schema injection, and a validation feedback loop working together.

**Democratizing access is the real ROI.** The technical system is impressive on paper, but the actual value was in how many more people could get the data they needed — without bottlenecking the engineering team.

**Production reliability requires layers.** A single LLM call is never enough for enterprise use. The validation layer, intent filter, and graceful fallback for ambiguous inputs turned a prototype into something the organization could trust.

---

*Built during Summer 2023 AI/ML Internship at Seagate Technology LLC. Examples above are representative of the query types and complexity levels handled by the system.*