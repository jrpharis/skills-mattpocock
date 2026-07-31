---
"mattpocock-skills": patch
---

Add **Shortcut** as a first-class issue tracker in `/setup-matt-pocock-skills`, alongside GitHub, GitLab, and local markdown.

- **MCP-first, REST fallback.** Shortcut has no official CLI, so the new `issue-tracker-shortcut.md` template drives the Shortcut MCP server, with a `curl` equivalent against the v3 API for repos with no MCP wired up. Operations are named by capability (`stories-create`, `stories-search`, …) because tool-name prefixes differ between the official server and the claude.ai connector.
- **Detection doesn't come from `git remote`.** For a Shortcut user the tracker and the code host are separate systems, so setup looks for a Shortcut MCP server, `sc-<id>` branch/commit prefixes, and `SHORTCUT_API_TOKEN` instead.
- **Two Shortcut-specific traps are called out in the template**: `labels` is a full-set write with no add/remove variant (so `/triage` must read-modify-write or it wipes labels), and there is no "close" — a workspace typically has several `done`-type states that aren't interchangeable, so the closed state is pinned by id at setup time.
- **Wayfinding** maps onto an epic labelled `wayfinder:map` whose stories are the decision tickets, with native `blocked by` story relations and a single-call frontier query (`epic` + `isDone` + `isBlocked` + `hasOwner`).
- **No PRs-as-a-request-surface flag** for Shortcut — it isn't a code host, so `/triage` shouldn't look for PRs there.

Docs page for `setup-matt-pocock-skills` re-synced.
