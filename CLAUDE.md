# CLAUDE.md — junk2

You are opening a **throwaway smoke-test target** for Kanbanger v2.1.0
(`github.com/earlyprototype/kanbanger`). This is not a real project.

## What you should do

Open `TEST_RUNBOOK.md` and walk it. That's the entire job.

## Important constraints

- **v2.1.0 supports only 4 columns**: `BACKLOG`, `TODO`, `DOING`, `DONE`.
  A `REVIEW` column is **not** in the whitelist — `add_task(column="REVIEW")`
  and `list_tasks(column="REVIEW")` will error. The parser will tolerate a
  manually-added `## REVIEW` header in `_kanban.md`, but MCP tools won't
  read/write it. This is a known v2.1.0 limitation, not a bug to fix here.

- **`sync_to_github` is out of scope** for this run. Phase 0 of the
  kanbanger-platform planning workspace observed it hangs indefinitely on
  misconfiguration. Do **not** invoke it unless explicitly asked. If you do
  accidentally invoke it, cancel within 5–10 seconds.

- **Do not invest real work here.** This repo is a smoke-test sandbox. Any
  features, bug fixes, or design work belong in `kanbanger-partymix` (the
  v3.0 successor), not here.

- **Token lives in `.claude/settings.local.json`** (gitignored). Do not
  commit it. Do not echo it back to the user.

## Layout

```
junk2/
|-- README.md            One-paragraph orientation
|-- CLAUDE.md            This file
|-- TEST_RUNBOOK.md      The runbook to walk
|-- _kanban.md           Seed board, 4 columns
|-- .mcp.json            Kanbanger MCP wiring
|-- .gitignore
|-- .claude/
|   `-- settings.local.json   GITHUB_TOKEN + GITHUB_REPO (gitignored)
`-- notes/
    |-- idea-a.md        Trivial simulation content
    `-- idea-b.md        Trivial simulation content
```

## If you find issues

Record them as a new section at the bottom of `TEST_RUNBOOK.md` under
`## Run N — <date>` with PASS/FAIL per probe and observed-vs-expected
behaviour. The runbook becomes its own report — no separate report file
needed for a smoke test.
