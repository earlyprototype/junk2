# Kanbanger v2.1.0 Smoke Test — Run 1 Report

**Date:** 2026-05-11
**Target:** `earlyprototype/kanbanger` v2.1.0
**Workspace:** `C:/Users/Fab2/Desktop/AI/_tools/junk2`
**Verdict:** PARTIAL PASS — 7 steps pass, Step 8 reveals a whitelist-asymmetry bug.

---

## 1. Scope

Walk `TEST_RUNBOOK.md` against the Kanbanger v2.1.0 MCP server, pointed at the seeded board in `_kanban.md`. Steps 1–7 exercise core CRUD and read paths; Step 8 probes documented v2.1.0 column-whitelist behaviour. `sync_to_github` and GitHub Projects V2 were explicitly out of scope per `CLAUDE.md`.

The session was opened in `junk2/` so the project-level `.mcp.json` would scope the kanbanger MCP server to this repo's `_kanban.md` (not to `kanbanger-platform/` or `kanbanger-partymix/`).

---

## 2. Pre-flight: MCP server did not connect at session start

The first thing I did after reading the runbook was confirm the kanbanger MCP was wired up. It wasn't.

| Probe | Result |
|---|---|
| `ListMcpResourcesTool` (no filter) | Only `plugin:everything-claude-code:exa` returned — kanbanger absent. |
| `ListMcpResourcesTool(server="kanbanger")` | "No resources found." |
| `ReadMcpResourceTool(server="kanbanger", uri="kanban://current-board")` | **"Server 'kanbanger' is not connected"** |
| `ToolSearch` for `list_tasks` / `add_task` / `kanban` / `kanbanger` | Nothing returned. |

The runbook's own Step 2 check (*"If `kanbanger` is missing, the `.mcp.json` didn't load — troubleshoot before continuing."*) had failed.

### Root cause

Python module import was succeeding — but resolving to the wrong file:

```
$ python -c "import kanbanger_mcp; print(kanbanger_mcp.__file__); print(kanbanger_mcp.__version__)"
C:\Users\Fab2\Desktop\AI\_tools\kanbanger-partymix\kanbanger_mcp\__init__.py
2.1.0
```

`pip list | grep kanban` showed why:

```
kanbanger-partymix    0.0.1    C:\Users\Fab2\Desktop\AI\_tools\kanbanger-partymix   (editable)
```

A different package (`kanbanger-partymix`, installed in editable / `pip install -e` mode) owned the `kanbanger_mcp` import name on this machine. The `.mcp.json` in `junk2/` had no `PYTHONPATH` pin, so `python -m kanbanger_mcp` ran the other package's code, not `earlyprototype/kanbanger`'s. Whether the actual MCP transport failure was caused by a start-up incompatibility in the shadowing package or by something else is unverified — the auto-mode classifier blocked the diagnostic-run probe.

### Fix

Per user instruction ("go to the earlyprototype/kanbanger repo on github and install from there"):

1. `pip uninstall -y kanbanger-partymix` — removes the shadowing editable install.
2. `pip install "kanban-project-sync[mcp] @ git+https://github.com/earlyprototype/kanbanger.git"` — installs the real package. **Package metadata name is `kanban-project-sync`, not `kanbanger`** — my first attempt with `kanbanger[mcp]` failed with `inconsistent name: expected 'kanbanger', but metadata has 'kanban-project-sync'` and I retried.
3. Verify: `python -c "import kanbanger_mcp; print(kanbanger_mcp.__file__)"` now resolves to `C:/Users/Fab2/AppData/Local/Programs/Python/Python312/Lib/site-packages/kanbanger_mcp/__init__.py`, version `2.1.0`, installed from commit `857a047b90006d3680ff6834286460be5fa6e7ce`.
4. User hot-reloaded the kanbanger MCP (`/mcp` → "Reconnected to kanbanger."). The 6 tools surfaced as deferred tools; I loaded schemas via `ToolSearch select:…` and proceeded.

---

## 3. Run 1 — step-by-step

### Step 1 — `list_tasks()` (no args) — PASS

```json
{
  "BACKLOG": ["Add a feature flag for dark mode", "Wire up email notifications"],
  "TODO": ["Draft onboarding copy", "Pick a logo color"],
  "DOING": ["Sketch initial wireframes"],
  "DONE": ["Create junk2 repo", "Wire up Kanbanger MCP server"]
}
```

7 tasks across 4 columns. Matches seed exactly.

### Step 2 — `list_tasks(column="TODO")` — PASS

```json
{ "TODO": ["Draft onboarding copy", "Pick a logo color"] }
```

Two tasks as expected.

### Step 3 — `add_task("Smoke test task", "TODO", "Added by runbook step 3")` — PASS (with quirks)

Tool returned `"Successfully added task 'Smoke test task' to TODO"`. Re-listing TODO showed the task present with description preserved.

**Quirk:** the task was **prepended** to the top of TODO (1st position), not appended as the runbook expected ("3rd entry"). And it was inserted on the line immediately after the `## TODO` header with **no blank line** between header and the new task — other columns retain a blank line:

```
## TODO
*   [ ] Smoke test task - Added by runbook step 3   ← new, on line directly under header

*   [ ] Draft onboarding copy
*   [ ] Pick a logo color
```

### Step 4 — `move_task` TODO → DOING → DONE — PASS

Both moves succeeded. After the second move, DONE showed `[x] Smoke test task - Added by runbook step 3` — auto-check on entry to DONE works. Description preserved through both moves. Same prepend + missing-blank-line cosmetic quirk reproduced in DOING then DONE on each insert.

After both moves, TODO returned to a clean shape (header → blank line → original two tasks). The cosmetic quirk only manifests at the column the task currently lives in.

### Step 5 — `delete_task("Smoke test task", "DONE")` — PASS

Tool returned success. `list_tasks()` confirmed task removed. `_kanban.md` byte-for-byte identical to seed shape — spacing restored.

### Step 6 — `get_sync_status()` — PASS

```json
{
  "synced_tasks": 0,
  "state_file": "not found",
  "message": "No sync state found. Run sync_to_github() first."
}
```

Returned immediately. **No hang** — the Phase-0 risk from `kanbanger-platform` is not present in this code path.

### Step 7 — three resources — PASS

| Resource | MIME | Content |
|---|---|---|
| `kanban://current-board` | `text/markdown` | Full `_kanban.md` content. |
| `kanban://stats` | `application/json` | `{BACKLOG:2, TODO:2, DOING:1, DONE:2, total:7, in_progress:1, completed:2, pending:4}` — arithmetic correct. |
| `kanban://sync-status` | `application/json` | `{synced:false, synced_tasks:0, message:"No sync state found. Board has not been synced to GitHub yet."}` |

### Step 8 — REVIEW edge probes — PARTIAL

Two probes, two different behaviours:

**8a. `add_task("Edge probe", "REVIEW")`** → clean error:
```
"Error: Invalid column 'REVIEW'. Must be one of: BACKLOG, TODO, DOING, DONE"
```
This is the documented v2.1.0 whitelist behaviour. **PASS.**

**8b. `list_tasks(column="REVIEW")`** → **no error**, returned:
```json
{ "REVIEW": [] }
```
The runbook predicted *"Error on filter validation. (The underlying parser tolerates a `## REVIEW` header in markdown, but the MCP tool layer rejects the filter.)"* — and explicitly listed this outcome under **What FAIL looks like**: *"Step 8 succeeds on REVIEW (would be unexpected — file as a finding)."*

So the column whitelist is enforced **asymmetrically** between `add_task` (validated) and `list_tasks` (unvalidated). This is a real product finding.

---

## 4. Findings

### Finding 1 — `add_task` prepends and omits the column-header blank line

**Severity:** LOW (cosmetic, transient).
**Where:** v2.1.0 `add_task` tool, markdown writer.
**Behaviour:** Inserted entry sits at line N+1 where N is the `## <COLUMN>` header line, with no blank line between them. Other columns retain header → blank → tasks.
**Recover:** Cosmetic only — `delete_task` or `move_task` out of the column restores the shape.
**Question raised:** Is insertion-at-top intentional? The runbook author expected append-to-bottom. Doc one or the other.

### Finding 2 — `list_tasks` does not validate the `column` argument

**Severity:** MEDIUM (silent failure mode for typos).
**Where:** v2.1.0 `list_tasks` tool, parameter validation.
**Behaviour:** Accepts any string in `column=`, returns `{"<that-string>": []}`. `add_task` correctly rejects.
**Impact:** A user filtering by a typo (`"TDOO"`) or a column they think exists (`"REVIEW"`, `"BLOCKED"`) gets a silent empty list back, not an error.
**Recommendation:** Make the validation symmetric across both tools. Reject unknown column names everywhere so typos and schema drift are visible.

### Finding 3 — `.mcp.json` ↔ runbook "Doctor output" disagree about `PYTHONPATH`

**Severity:** MEDIUM (test-validity hazard — caught only because I was paying attention).
**Where:** This repo (`junk2/`) — `.mcp.json` and `TEST_RUNBOOK.md` "Doctor output" section.
**Behaviour:** The "Doctor output" section asserts `.mcp.json` pins `PYTHONPATH=C:/Users/Fab2/Desktop/AI/_tools/_kanbanger` to force v2.1.0 resolution. The actual `.mcp.json` has no `PYTHONPATH` entry — only `KANBANGER_WORKSPACE`, `GITHUB_TOKEN`, `GITHUB_REPO`, `GITHUB_PROJECT_NUMBER`, `MCP_USE_ANONYMIZED_TELEMETRY`.
**Impact:** Without that pin, `python -m kanbanger_mcp` resolved to whichever `kanbanger_mcp` happened to be the installed package on this machine. The runbook's "test v2.1.0 independent of other workspaces" promise was not actually enforced by the wiring; it was assumed.
**Resolution options:**
- (a) Add the `PYTHONPATH` env var to `.mcp.json` so the wiring matches the doc claim.
- (b) Remove the claim from "Doctor output" and replace with an explicit install step: clone+install in a per-project venv, or pip-install-from-GitHub as a setup probe. The latter is what I had to do manually.

### Note: `sync_to_github` not exercised

Out of scope per `CLAUDE.md` ("Phase 0 of the kanbanger-platform planning workspace observed it hangs indefinitely on misconfiguration. Do not invoke it unless explicitly asked."). Skipped.

---

## 5. Files touched during this session

| Path | Change |
|---|---|
| `TEST_RUNBOOK.md` | Filled in the `### Run 1 — <date>` results section (table, verdict, unexpected-behaviour list, planning-feedback list). |
| `RUN_1_REPORT.md` | New — this file. |
| `_kanban.md` | Round-tripped through CRUD steps; ended byte-for-byte identical to seed. |

Pip environment was also modified — uninstalled the editable install that was shadowing `kanbanger_mcp`, installed `kanban-project-sync 2.1.0` from `git+https://github.com/earlyprototype/kanbanger.git@857a047`.

---

## 6. Overall verdict

**Kanbanger v2.1.0 ships its 6 MCP tools and 3 resources operably against a fresh repo.** Core CRUD round-trips cleanly, the v2.1.0 column whitelist is correctly enforced in `add_task`, and `get_sync_status` does not hang.

Three findings worth fixing — one cosmetic (insertion shape), one product (whitelist asymmetry), one repo-level (wiring docs vs reality). None block the v2.1.0 smoke milestone.

`sync_to_github` and concurrency / lock-file / large-board / cross-platform are still untouched and remain in the runbook's "Out of scope (widen by request)" section.
