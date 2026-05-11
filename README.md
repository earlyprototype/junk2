# junk2

Throwaway target for smoke-testing **Kanbanger v2.1.0** (the original MCP server
at `github.com/earlyprototype/kanbanger`).

Not a real project. Two simulation files in `notes/` exist purely so the repo
isn't empty of content. The interesting bit is `_kanban.md` and the MCP wiring
in `.mcp.json`.

## Where to start

Open `TEST_RUNBOOK.md` — it explains what to do in a Claude Code session
opened in this directory.

## What this is testing

Whether v2.1.0 install + the 6 MCP tools (`list_tasks`, `add_task`,
`move_task`, `delete_task`, `get_sync_status`, `sync_to_github`) work cleanly
when pointed at a brand-new repo, independent of the planning workspace
(`kanbanger-platform`) and the v3.0 successor (`kanbanger-partymix`).

GitHub Projects sync is out of scope for this run.
