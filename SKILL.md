---
name: sql-query-reviewer
description: Reviews SQL - with a PostgreSQL focus - and the code that builds it, for correctness, injection risk and missing parameterisation, N+1 query patterns, missing or unusable indexes, locking and transaction hazards, NULL semantics, and timezone handling. Returns findings grouped by severity, each with the evidence and a concrete fix. Use when reviewing a migration, a slow query, an ORM-generated query, a PR that adds raw SQL, or before running a data-fix script in production.
---

# SQL Query Reviewer

Read each query as the database will execute it and as an attacker or a busy production table would stress it.

## When to use
- A PR adds or changes SQL, an ORM query, or a migration.
- A query is slow, deadlocks, or returns surprising rows.
- A one-off UPDATE/DELETE is about to run against production.

## Inputs
The SQL text (or the code that builds it), table definitions and existing indexes if available, rough row counts, and how the query is called (once per request, in a loop, in a background job). If you only have the query, state your assumptions about sizes.

## Review checklist (work through in this order)
1. **Injection / parameterisation**
   - Any user input concatenated or interpolated into SQL text is a finding. Values must be bound parameters (`$1`, `?`, named binds).
   - Identifiers (table, column, sort direction) cannot be parameters; they must come from an allow-list, then be quoted.
   - `LIKE` patterns: escape `%` and `_` in user input. `IN (...)` lists: use `= ANY($1)` with an array parameter rather than building a string.
2. **Correctness**
   - JOINs that multiply rows (missing join condition, one-to-many feeding a SUM).
   - LEFT JOIN turned into INNER by a WHERE filter on the right table.
   - GROUP BY / DISTINCT used to hide duplicates.
   - `LIMIT` without `ORDER BY`; pagination by OFFSET on a changing set.
   - UPDATE/DELETE without a WHERE, or with one that can match more than intended. Ask for a `SELECT count(*)` with the same predicate first.
3. **NULL semantics**
   - `col = NULL` never matches; use `IS NULL`. `col <> 'x'` excludes NULL rows.
   - `NOT IN (subquery)` returns nothing if the subquery yields a NULL; prefer `NOT EXISTS`.
   - `COUNT(col)` skips NULLs; `SUM` of no rows is NULL - wrap in `COALESCE` when a number is expected.
4. **Time and timezones**
   - Prefer `timestamptz`. Comparing `timestamp` to `now()` mixes zones silently.
   - Day boundaries: `date_trunc('day', ts AT TIME ZONE 'Asia/Riyadh')` style conversion must happen before truncating, using the business's zone.
   - Ranges should be half-open: `>= start AND < end`, never `BETWEEN` on timestamps.
5. **Performance**
   - N+1: a query inside a loop over another result. Suggest a single query with `JOIN` or `= ANY($1)`.
   - Predicates that defeat indexes: functions on the column (`lower(email)` without an expression index), leading-wildcard `LIKE`, implicit casts, `OR` across different columns.
   - Missing index on foreign keys used in joins or cascaded deletes. Composite index column order should match equality columns first, then range.
   - `SELECT *` in hot paths; unbounded result sets.
   - Ask for `EXPLAIN (ANALYZE, BUFFERS)` on a realistic dataset before claiming a fix; never run ANALYZE on a destructive statement outside a rolled-back transaction.
6. **Locking and transactions**
   - `CREATE INDEX` without `CONCURRENTLY` on a large live table blocks writes.
   - `ALTER TABLE ... ADD COLUMN ... DEFAULT <volatile>` or type changes rewrite the table.
   - Long transactions holding row locks; read-modify-write without `SELECT ... FOR UPDATE` or an atomic `UPDATE ... SET x = x + 1`.
   - Large batch updates: chunk them and commit per batch.
   - Migrations should set `lock_timeout` so they fail fast instead of queueing behind traffic.
7. **Data safety**: irreversible changes (dropping columns, deleting rows) need a backup or a soft-delete plan.

## Severity scale
- **Critical** - injection, data loss, wrong results in money/permissions, table-wide lock in production.
- **High** - wrong results in edge cases, N+1 on a hot path, full scans on large tables.
- **Medium** - missing index on moderate tables, fragile pagination, timezone ambiguity.
- **Low** - style, readability, `SELECT *`, naming.

## Output format
```
| # | Severity | Location | Finding | Evidence | Fix |
```
Follow with a corrected version of the query (or the relevant code) and a short list of assumptions you made. If there are no findings at a level, say so explicitly.

## Worked example
Input (Node): `db.query("SELECT * FROM orders WHERE customer_id = " + id + " AND created_at BETWEEN '" + from + "' AND '" + to + "'")`
- Critical: string concatenation of `id`, `from`, `to` - injection. Fix: `WHERE customer_id = $1 AND created_at >= $2 AND created_at < $3`.
- Medium: `BETWEEN` on timestamps double-counts the boundary. Fix: half-open range.
- Medium: check for an index on `(customer_id, created_at)`.
- Low: select only needed columns.

## Pitfalls
- Reporting "add an index" without saying which columns and in what order.
- Ignoring how often the query runs; a slow monthly report differs from a per-request query.
- Trusting ORM output without looking at the generated SQL.
