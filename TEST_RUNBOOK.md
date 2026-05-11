# TEST_RUNBOOK — Kanbanger v2.1.0 smoke test

## What this is

A handover from the Claude Code session that **set up** this repo (the
session in `kanbanger-platform\`) to the next Claude Code session that will
**run the tests** (a fresh session opened in this directory).

The setup session couldn't run the tests itself because the Kanbanger MCP
server is per-directory — it loads from the `.mcp.json` in whichever
directory Claude Code was opened in. The setup session was opened in
`kanbanger-platform\`, so its MCP server points at `kanbanger-platform\_kanban.md`,
not at `junk2\_kanban.md`. A fresh session opened here gets the right config.

## How to start the test session

1. Open a new Claude Code session with this directory as cwd:
   ```
   cd C:\Users\Fab2\Desktop\AI\_tools\junk2
   claude .
   ```
   (Or open this folder in your usual Claude Code launcher.)

2. Confirm the MCP server is wired up — paste this into chat:
   > Show me what MCP servers are loaded.

   You should see one called `kanbanger` (and possibly others from global
   config). If `kanbanger` is missing, the `.mcp.json` didn't load —
   troubleshoot before continuing.

## Baseline state at handover

- **GitHub repo:** `github.com/earlyprototype/junk2` (currently PUBLIC — heads
  up if you intended private). Empty before this commit; seed commit pushed
  during setup.
- **Seed board:** 4 columns (BACKLOG/TODO/DOING/DONE), 7 seed tasks.
- **`kanban-doctor` result:** see "Doctor output" section below (filled in
  during setup).
- **Out of scope:** `sync_to_github` and GitHub Projects V2 (no
  `GITHUB_PROJECT_NUMBER` configured).

## The smoke test — paste these prompts in order

For each step, paste the prompt and record the result. **If a tool errors
unexpectedly or hangs >10 seconds, stop and note it** before moving on.

### Step 1 — Read the board (no-op probe)

> Call the kanbanger MCP tool `list_tasks` with no arguments and show me the JSON output.

**Expect:** JSON listing 7 tasks across BACKLOG/TODO/DOING/DONE. Tasks in
DONE have `[x]` checkboxes; others have `[ ]`.

### Step 2 — Filter by column

> Call `list_tasks` with `column="TODO"` and show me the JSON.

**Expect:** 2 tasks ("Draft onboarding copy", "Pick a logo color"). Only TODO.

### Step 3 — Add a task

> Call `add_task` with title "Smoke test task", column "TODO", description "Added by runbook step 3".

**Expect:** Success. Re-run `list_tasks(column="TODO")` and see the new task as the 3rd entry.

### Step 4 — Move it through the workflow

> Call `move_task` for "Smoke test task" from "TODO" to "DOING".

**Expect:** Success. Confirm by `list_tasks(column="DOING")` — should now have 2 entries.

> Then call `move_task` for "Smoke test task" from "DOING" to "DONE".

**Expect:** Success. Confirm by `list_tasks(column="DONE")` — task is there with `[x]` checkbox auto-marked.

### Step 5 — Delete it

> Call `delete_task` for "Smoke test task" in column "DONE".

**Expect:** Success. `list_tasks` no longer shows it.

### Step 6 — Sync status (read-only, safe)

> Call `get_sync_status` and show me the JSON.

**Expect:** JSON returns — probably indicates "never synced" or similar.
**Should NOT hang.** If it hangs >5s, cancel — that's the Phase 0 finding.

### Step 7 — Read the 3 resources

> Read the resource `kanban://current-board` and show me the content.

**Expect:** Current `_kanban.md` content as markdown.

> Read the resource `kanban://stats` and show me the content.

**Expect:** JSON with task counts per column.

> Read the resource `kanban://sync-status` and show me the content.

**Expect:** Similar to step 6 output (sync metadata JSON).

### Step 8 — Known-edge probes (optional, surfaces v2.1.0 limitations)

> Try calling `add_task` with title "Edge probe" and column "REVIEW". I expect it to error.

**Expect:** Error — `REVIEW` is not in the v2.1.0 column whitelist. This is
the documented limitation, not a regression.

> Try `list_tasks` with `column="REVIEW"`.

**Expect:** Error on filter validation. (The underlying parser tolerates a
`## REVIEW` header in markdown, but the MCP tool layer rejects the filter.)

## What "PASS" looks like

- Steps 1–7 all succeed with expected output.
- Step 8 errors with a clear "column not allowed" or similar message
  (this is correct v2.1.0 behaviour).
- No hangs (>10s) on any step.
- After step 5, `_kanban.md` content is back to the seed state.

## What "FAIL" looks like

- Any step 1–7 errors unexpectedly.
- Any step hangs.
- `add_task` succeeds but the task doesn't appear in `_kanban.md`.
- `move_task` succeeds but the task didn't actually move (still in old column).
- Step 8 *succeeds* on REVIEW (would be unexpected — file as a finding).

## Record results here

Add a section below for each run.

### Run 1 — <date> — <pass/fail/partial>

| Step | Result | Notes |
|---|---|---|
| 1. list_tasks (all) | | |
| 2. list_tasks (TODO) | | |
| 3. add_task | | |
| 4. move_task ×2 | | |
| 5. delete_task | | |
| 6. get_sync_status | | |
| 7. resources ×3 | | |
| 8. REVIEW probes | | |

**Overall verdict:**

**Unexpected behaviour:**

**Items to feed back to kanbanger-platform Phase 1:**

---

## Doctor output (filled in during setup)

Setup session ran `kanban_doctor.py` from `_kanbanger\` (v2.1.0 source)
with the env vars below, against `junk2\_kanban.md`. Result: **EXIT 0**,
10 PASS / 2 WARN / 0 FAIL / 1 SKIP.

```
kanban-doctor -- preflight checks
workspace: C:\Users\Fab2\Desktop\AI\_tools\junk2

Environment
-----------
  [PASS] Python version: 3.12.3
  [WARN] .env file: no .env in C:\Users\Fab2\Desktop\AI\_tools\junk2
         -> If env vars come from your shell instead, this is fine. Otherwise: create .env with GITHUB_TOKEN and GITHUB_REPO

GitHub credentials
------------------
  [PASS] GITHUB_TOKEN: present (ghp_...aFbc)
  [PASS] GITHUB_TOKEN format (classic-PAT check): looks like a classic PAT (ghp_ + 36 chars)
  [PASS] GITHUB_TOKEN authenticates: authenticated as earlyprototype

Repo and Projects
-----------------
  [PASS] GITHUB_REPO: earlyprototype/junk2
  [PASS] Repo visible to token: earlyprototype/junk2
  [WARN] Token can access Projects V2: query succeeded but no projects linked to repo
         -> Link a project: https://github.com/earlyprototype/junk2/projects
  [SKIP] Project Status field has required options: skipped (no projects to check)

Workspace
---------
  [PASS] _kanban.md in workspace: C:\Users\Fab2\Desktop\AI\_tools\junk2\_kanban.md (4 column header(s) found)

MCP server (optional)
---------------------
  [PASS] MCP_USE_ANONYMIZED_TELEMETRY: set to 'false' (telemetry banner suppressed)
  [PASS] mcp_use library: installed (MCP server can run)
  [PASS] kanbanger_mcp package: importable (version 2.1.0)

Summary
  [PASS]: 10
  [WARN]: 2
  [FAIL]: 0
  [SKIP]: 1
```

**Two warnings, both expected:**
- No `.env` — auth comes from `.claude\settings.local.json` injected into MCP
  server's spawn env via Claude Code. Not a problem.
- No GitHub Project linked — sync is out of scope for this run.

**Important install detail (worth flagging to the planning workspace):**
The pip-installed `kanbanger_mcp` on this machine comes from `kanbanger-partymix\`
(the diverged v3.0 successor with 5-column schema and review-gate tools), NOT
from `_kanbanger\` (v2.1.0). To run v2.1.0 cleanly, `.mcp.json` in this repo
pins `PYTHONPATH=C:/Users/Fab2/Desktop/AI/_tools/_kanbanger`. Without that
pin, the next Claude Code session would unwittingly test the wrong code.
This is a real finding for the kanbanger-platform Phase 1 install story.

---

## Out of scope (widen by request)

- `sync_to_github` — not exercised. To extend: provide a
  `GITHUB_PROJECT_NUMBER` for `earlyprototype/junk2` and rerun.
- Multi-session concurrency / `.kanban.lock` behaviour.
- Performance under large boards.
- Cross-platform (Mac/Linux) install.
