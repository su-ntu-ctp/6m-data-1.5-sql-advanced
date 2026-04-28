# 🎓 Instructor Guide — Lesson 1.5: SQL Advanced — Joins, Window Functions & CTEs

> **Branch:** `feature/instructor-guide`
> **Audience:** Instructors and teaching assistants
> **Companion to:** `lesson.md`, `pre-class.md`, `assignment.md`

---

## 1. Lesson Overview & Instructor Objectives

| | |
|---|---|
| **Duration** | 3 hours |
| **Format** | Flipped Classroom + Hands-On SQL in DbGate |
| **Dataset** | Insurance claims database (car, client, claim, address tables) |
| **Learner entry point** | Completed 1.4 (DML); can write SELECT/WHERE/GROUP BY/HAVING |

By the end of this lesson learners should be able to:

1. Combine data from multiple tables using INNER, LEFT, RIGHT, and FULL OUTER joins.
2. Understand when to use UNION vs. JOIN.
3. Add calculated columns (running totals, rankings) without losing row detail using window functions.
4. Simplify complex multi-step queries using Common Table Expressions (CTEs).

**Instructor's primary job:** These three concepts (JOINs, window functions, CTEs) represent the biggest jump in SQL complexity. The analogies and visual scaffolding are essential — learners who don't have a mental model before seeing the syntax tend to memorise rather than understand, and break apart when scenarios change.

---

## 2. Concept Analogies

### JOINs — "The Party Guest List"

> "Imagine you're hosting a party. You have a Guest List (clients) and a Gifts Log (claims). Now you want to know who brought what gift."

| Join Type | Scenario | Who Shows Up |
|-----------|----------|-------------|
| INNER JOIN | "Only guests who brought a gift" | Rows that match in *both* tables |
| LEFT JOIN | "All guests — with or without a gift (NULLs for the empty-handed)" | All left table rows; NULLs where no match |
| RIGHT JOIN | "All gifts — even orphaned ones without a named guest" | All right table rows; NULLs where no match |
| FULL OUTER JOIN | "Everyone and every gift, matched or not" | All rows from both, NULLs where unmatched |

**The "NULL as an empty box" clarification:** When a LEFT JOIN finds no match in the right table, it doesn't delete the row — it puts `NULL` in all the right-table columns. Ask: "If we LEFT JOIN clients to claims, and a client has never made a claim, what appears in their `claim_amt` column?" → `NULL`. This is valuable: `NULL` tells you something meaningful (no claim exists), which is different from zero claims.

**Which table should be on the LEFT?** Rule of thumb: put the table representing your *primary subject* on the left. If you're asking "what do I know about my clients?", clients is the left table. If you're asking "what claims were filed, and by whom?", claims is the left table.

---

### UNION — "Stacking Two Lists"

> "JOIN is like adding columns (horizontal expansion). UNION is like stacking rows (vertical expansion). JOINing employees and departments gives you wider rows. UNIONing employees and contractors gives you a longer list of people."

The visual in lesson.md (join_vs_union.png) is essential — refer to it explicitly. Without the visual, learners confuse UNION with JOIN every time.

---

### Window Functions — "The Annotated Class List"

> "GROUP BY is like calculating the class average and writing down ONE number on the board. Everyone loses their individual identity — the class becomes a statistic.
>
> A window function is like keeping every student's row in your spreadsheet, but adding a new column that says: 'class average is 1.73m' next to every row. Each student is still there; they just have extra context."

**The key phrase to repeat:** "Window functions ADD a column; they don't COLLAPSE rows."

**RANK() ties and gaps:**
Show the sample table in the lesson. Ask: "If two students tie for 2nd place, what rank does the next student get?" → 4th (not 3rd). This surprises learners. Explain: "RANK() preserves the natural position — if two people are tied at position 2, the next person is genuinely in position 4." Compare to `DENSE_RANK()` which doesn't skip: 1, 2, 2, 3.

---

### CTEs — "Writing Down Your Recipe Steps"

> "A subquery inside a subquery inside another query is like writing a recipe in a single sentence: 'Bake the cake that uses flour from the farm that hired workers who were trained by chefs who...' Unreadable. A CTE is like writing the steps on separate lines: Step 1 — make the dough. Step 2 — bake it. Step 3 — frost it. Same result, readable by a human."

**CTEs vs. subqueries — practical guidance for learners:**
- If you reference the same query result more than once → use a CTE (write it once, use twice)
- If the nested query is more than 10 lines → use a CTE (readability)
- If it's a simple one-off filter → a subquery is fine

---

## 3. Real-World Use Cases

### JOINs — The Analytics Foundation

Every business intelligence report uses JOINs. A sales dashboard might join:
- `orders` ↔ `products` (to get product names and categories)
- `orders` ↔ `customers` (to get customer demographics)
- `customers` ↔ `regions` (to get geographic breakdowns)

Without JOINs, analysts would need to export data to Excel and do VLOOKUPs — which is slower, error-prone, and doesn't scale.

**The LEFT JOIN for data quality audits:** A LEFT JOIN where the right table has NULLs identifies *missing* relationships. "Show me all clients who have NEVER filed a claim" is: `SELECT * FROM client LEFT JOIN claim ON client.id = claim.client_id WHERE claim.id IS NULL`. This pattern is used in customer churn analysis, KYC (Know Your Customer) compliance checks, and inventory audits.

---

### Window Functions — Finance & Operations

**Running totals** are used everywhere in finance: cumulative revenue by month, running loan balance, cumulative visitor count. The pattern `SUM(amount) OVER (ORDER BY date)` appears in almost every financial dashboard.

**RANK()** is used in sales leaderboards, student grade reports, and product performance rankings. The `QUALIFY rank <= 3` pattern for finding top-N per group is a classic interview question.

**Real-world example:** An e-commerce company wants to find the top 3 best-selling products *per category* by month. A window function with `PARTITION BY category ORDER BY sales DESC` solves this in one query; without window functions it would require multiple CTEs or subqueries.

---

### CTEs — Code Maintenance

Production SQL at companies like Airbnb uses CTEs extensively. A well-written CTE query reads like documentation:

```sql
WITH churned_customers AS (...),
     their_last_orders AS (...),
     revenue_at_risk AS (...)
SELECT ...
```

A new analyst joining the team can read this query and understand the business logic without needing a comment. Subquery-heavy code without CTEs is often called "spaghetti SQL" — technically correct, impossible to maintain.

---

## 4. Activity Facilitation Notes

### Part 1: Joins (55 min)

**Start with DESCRIBE and SUMMARIZE (10 min):**
Before any JOINs, learners must understand the data. Ask them to describe all 4 tables and answer:
- "How many claims are in the dataset? What's the maximum claim amount?"
- "How does the `claim` table connect to `client`? What column?"
- "If a car_id in claim doesn't exist in car, what does that tell you about data quality?"

**Exercise 3 (4-table join) is the centrepiece of Part 1:**
This is the first time learners join more than 2 tables. Walk through the query mentally before writing any SQL:
1. "What is our primary subject?" (Each claim)
2. "What do we want to know about it?" (Client name, car type, city)
3. "Which tables do we need to add?" (car, client, address)
4. "What columns connect them?" (claim.car_id → car.id, claim.client_id → client.id, client.address_id → address.id)

*Map this on the whiteboard first* — draw boxes for each table and arrows for the joins. The visual prevents learners from writing joins blindly.

**Common mistakes:**
- Forgetting to alias tables (writing `claim.id` vs. `cl.id`) → results in ambiguous column errors
- Joining in the wrong order → produces Cartesian products
- Using the wrong join type → missing rows without understanding why

---

### Part 2: Window Functions (55 min)

**The "no collapse" moment:** Before showing any code, write on the board:

```
GROUP BY: Input 10 rows → Output 1 row (per group)
Window:   Input 10 rows → Output 10 rows (plus a new column)
```

Ask: "When would you WANT to keep all 10 rows but still know the group average?" → When you need both individual detail AND group context in the same result. Examples: "Show me each transaction alongside the monthly total", "rank each employee within their department while keeping their individual salary."

**Running total table:** Show the 4-row running total table from the lesson before showing any code. Let learners calculate the running totals manually. Then show the SQL that produces exactly that result. Code is much more memorable when learners have already computed the answer by hand.

**PARTITION BY:** Ask: "What if you want a separate running total for each car?" → `PARTITION BY car_id`. The partition resets the window for each group — like starting a new class average for each class.

---

### Part 3: CTEs (55 min)

**The subquery → CTE refactor:** Show the subquery version of Exercise 5 first, then refactor it into a CTE. Ask learners to read both versions and say which is easier to understand. They will almost always prefer the CTE — use this as motivation for why CTEs exist.

**Exercise 6 (the combined challenge):** This is the hardest exercise in the lesson. It combines a 4-table JOIN, a RANK() window function, and a QUALIFY filter. Recommend:
1. First, write the 4-table JOIN alone and verify it works.
2. Add the RANK() window function to that query.
3. Wrap it in a CTE and add the QUALIFY filter.

Building incrementally prevents "blank page" paralysis and makes debugging easier.

**Multiple CTEs:** Show the pattern of chaining CTEs. Each CTE can reference the previous ones — they build on each other like steps. The analogy: "avg_claims is Chapter 1. overall_avg is Chapter 2. The final SELECT is the conclusion that uses both chapters."

---

## 5. Timing & Pacing Notes

| Part | Planned | Common Overrun | Mitigation |
|------|---------|---------------|-----------|
| Part 1: Joins | 55 min | DESCRIBE/SUMMARIZE exploration runs long | Cap exploration at 10 min; cut Right and Full Outer if time is short (they're less common in practice) |
| Part 2: Window Functions | 55 min | The RANK/QUALIFY combination confuses learners | Do the running total first; if the group runs out of time, RANK/QUALIFY becomes optional self-study |
| Part 3: CTEs | 55 min | Exercise 6 is a stretch goal — many learners won't finish | Set expectation: "If you finish Exercise 5, you've mastered CTEs. Exercise 6 is the stretch challenge." |

---

## 6. Common Learner Questions

**Q: "Which is better — a subquery or a CTE?"**
A: They produce the same result. Use CTEs when: you need to reference the result more than once, or when readability matters (production code, code review, debugging). Use subqueries for simple, one-off filters in ad-hoc analysis.

**Q: "When would I use a FULL OUTER JOIN in real life?"**
A: Data reconciliation — comparing two systems to find discrepancies. "Show me all records from System A and System B, highlighting what's in one but not the other." Common in finance and compliance workflows.

**Q: "Can window functions be used with CTEs?"**
A: Yes — and this is common in production SQL. The pattern in Exercise 6 is exactly this. A CTE computes the ranked results; the outer query filters them. This is often cleaner than using QUALIFY directly.

**Q: "What's the difference between RANK and DENSE_RANK?"**
A: `RANK` skips numbers after ties (1, 2, 2, 4). `DENSE_RANK` does not skip (1, 2, 2, 3). Use `DENSE_RANK` when learners/users will be confused by gaps in the ranking; use `RANK` for genuine position counting (e.g., "you came 4th in the race because two people tied for 2nd").

**Q: "How does QUALIFY work? I've never seen it in MySQL."**
A: `QUALIFY` is a DuckDB/Snowflake extension — not standard SQL. The equivalent in MySQL/PostgreSQL is to wrap the window query in a CTE or subquery and filter in the outer WHERE. Knowing both approaches is valuable; QUALIFY is just the more readable version.
