# SQL Query Reviewer

An AI skill that reviews SQL and the code that builds it, with a PostgreSQL focus. It checks injection risk and parameterisation, correctness (row multiplication, NULL logic, pagination), N+1 patterns, index usage, locking during migrations, and timezone handling. Findings come back as a severity-ranked table with evidence and a concrete fix, followed by a corrected query.

## Install

Claude Code:

```
git clone https://github.com/AIX-store/skill-sql-query-reviewer ~/.claude/skills/sql-query-reviewer
```

Other agents (Codex, Cursor, Claude Desktop projects): copy the contents of `SKILL.md` into the agent's instructions or rules file.

## Example prompts

- "Review the SQL in this PR for injection and performance issues."
- "This report query is slow on 20M rows. Here is the EXPLAIN output - what is wrong?"
- "Check this migration before I run it on a live table."

## Licence

MIT, © 2026 CodeMeld LLC.

Listed on AiX: https://aixstore.ai/store
