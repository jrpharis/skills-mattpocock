# Issue tracker: Shortcut

Issues and specs (you may know a spec as a PRD) for this repo live as **Shortcut stories**.

Shortcut has no official CLI. Use the **Shortcut MCP server** for all operations; fall back to the REST API (`https://api.app.shortcut.com/api/v3`, header `Shortcut-Token: $SHORTCUT_API_TOKEN`) when no MCP server is wired up. MCP tool names differ by server — `stories-create` on the official [`@shortcut/mcp`](https://github.com/useshortcut/mcp-server-shortcut), `mcp__claude_ai_Shortcut__stories-create` via the claude.ai connector — so match tools by **capability**, not by exact name.

## Workspace configuration

Fill these in once; skills read them instead of rediscovering them every run.

- **Workspace slug**: `<slug>` — story URLs are `https://app.shortcut.com/<slug>/story/<id>`
- **Default team**: `<name>` (`<uuid>`)
- **Default workflow**: `<name>` (`<id>`) — from `workflows-list` / `workflows-get-default`
- **State to open in**: `<name>` (`<id>`)
- **State that means "closed"**: `<name>` (`<id>`)

## Conventions

- **Create a story**: `stories-create` — `name`, `description` (markdown), `type` (`feature`/`bug`/`chore`), plus `team` / `epic` / `workflow_state_id` where known. REST: `POST /stories`.
- **Read a story**: `stories-get-by-id` with `full: true` — returns comments, labels, relations, and tasks in one call. REST: `GET /stories/{id}`.
- **List / query stories**: `stories-search` with structured filters (`label`, `state`, `owner`, `team`, `epic`, `type`, `isDone`, `isBlocked`, `hasOwner`, `created`). Archived stories are excluded by default. REST: `GET /search/stories?query=<operators>`, using the same operator syntax as the web UI (`label:needs-triage !is:done`).
- **Comment on a story**: `stories-create-comment`. REST: `POST /stories/{id}/comments`.
- **Apply / remove labels**: `stories-update` with `labels: [{ name }]` — read the current labels first (see below). REST: `PUT /stories/{id}`.
- **Close a story**: comment the explanation, then `stories-update` with the `workflow_state_id` recorded above. REST: `PUT /stories/{id}`.
- **Blocking / relations**: `stories-add-relation` with `relationshipType: "blocked by"`. REST: `POST /story-links` with `{ subject_id: <blocker>, verb: "blocks", object_id: <blocked> }`.
- **Branch name for a story**: `stories-get-branch-name`.

### Three things that bite

- **Labels replace, they do not merge.** There is no add-label or remove-label operation — `stories-update` writes the whole `labels` array. Always read the story's current labels, modify the array, and write the full set back; passing a single label wipes the rest. This is the easiest way for `/triage` to do real damage.
- **There is no "close".** A story is closed by moving it into a workflow state whose type is `done` — and a workspace usually has **several** such states (`Dev`, `Beta`, `Done`, `UAT Verification`, `Staging`, `Canceled` …), which are not interchangeable: landing a finished story in `Canceled` misreports the outcome. That's why the closed state is pinned by id above rather than resolved by type. Multiple workflows also mean the right state depends on which team owns the story.
- **Story references are `sc-<id>`.** Branches and commit messages carry `sc-1234` (Shortcut's own branch-name convention) or `[sc-1234]`; a bare `#1234` is not a Shortcut convention. Resolve any of these with `stories-get-by-id`.

## Pull requests as a triage surface

Not applicable — Shortcut is not a code host, so pull requests never appear in it and `/triage` should not go looking for them here. If this repo wants external PRs in the triage queue, that surface belongs to the code host: keep the `gh` / `glab` PR recipes from the GitHub or GitLab template alongside this file and flip that flag there.

## When a skill says "publish to the issue tracker"

Create a Shortcut story, setting the team and epic where they're known and applying the triage label the skill asks for (see `triage-labels.md`).

## When a skill says "fetch the relevant ticket"

`stories-get-by-id` with `full: true`. The user will normally pass a story id or a `sc-<id>` reference.

## Wayfinding operations

Used by `/wayfinder`. The **map** is an epic; its stories are the tickets.

- **Map**: an epic named `Wayfinder: <effort>` and labelled `wayfinder:map`, its description holding the Notes / Decisions-so-far / Fog body. `epics-create`, then `epics-update` to attach the label (create takes no labels). Epic search filters on `hasLabel` but not on a label name, so the `Wayfinder:` name prefix is what makes the map findable via `epics-search`.
- **Child ticket**: a story in that epic, labelled `wayfinder:<type>` (`research`/`prototype`/`grilling`/`task`). Once claimed, the story is owned by the driving dev. Map order is ascending story id (creation order).
- **Blocking**: Shortcut's **native story relations** — the canonical, UI-visible representation, and what `is:blocked` reads. `stories-add-relation` with `relationshipType: "blocked by"`. Available on every tier, so no body-convention fallback is needed. A ticket is unblocked when every blocker reaches a `done` state.
- **Frontier query**: `stories-search` with `epic: <id>`, `isDone: false`, `isBlocked: false`, `hasOwner: false` — Shortcut evaluates all three natively, so the frontier is one call. First in map order wins.
- **Claim**: `stories-assign-current-user` — the session's first write.
- **Resolve**: `stories-create-comment` with the answer, move the story to the closed state, then append a context pointer (gist + link) to the epic's Decisions-so-far via `epics-update`.
