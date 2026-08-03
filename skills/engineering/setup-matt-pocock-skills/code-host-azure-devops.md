## Code host: Azure DevOps

This repo's code lives in **Azure DevOps Repos**, which is not where its issues live — the issue tracker is configured above. This section covers the one surface the tracker can't: **pull requests**.

Use the **Azure DevOps MCP server** ([`@azure-devops/mcp`](https://github.com/microsoft/azure-devops-mcp)) for all operations; fall back to the [`azure-devops` Azure CLI extension](https://learn.microsoft.com/en-us/cli/azure/repos/pr) (`az repos pr ...`, auto-installs on first use) when no MCP server is wired up. Tool names differ by server and by version — the shapes below are v2.9.0, which consolidated per-action tools into verb-parameterised ones — so match tools by **capability** and `action`, not by exact name.

### Repo configuration

Fill these in once; skills read them instead of rediscovering them every run.

- **Organization**: `<org>` — the MCP server is started per-org, and `az` reads it from `az devops configure -d organization=...` or git config
- **Project**: `<project>`
- **Repository**: `<repo>`
- **Repository GUID**: `<guid>` — from `repo_repository` (`list`). Pin it; see the first trap below
- **Enabled MCP domains**: `<domains>` — the server is started with `-d`; `repositories` is required here, `work-items` is absent unless the tracker is Azure Boards

### Conventions

- **Read a PR**: `repo_pull_request` (`get`) with `includeLabels` / `includeChangedFiles` / `includeWorkItemRefs` as needed — all three default to **false**. `az repos pr show --id <id>`.
- **List open PRs**: `repo_pull_request` (`list`) — `status` already defaults to `Active`. Filters: `created_by_me`, `created_by_user` (email), `i_am_reviewer`, `user_is_reviewer`, `sourceRefName`, `targetRefName`. `az repos pr list --status active`.
- **Read the diff**: `az repos pr checkout --id <id>` puts the source branch in the working tree, then review with ordinary `git diff` — the best option, and what `/code-review` wants. Without the CLI, `repo_pull_request` (`get`) with `includeChangedFiles` lists the paths and `repo_file` (`get_content`) fetches each one.
- **Comment on a PR**: `repo_pull_request_thread_write` (`create`) with `content`; add `filePath` + `rightFileStartLine` for an inline comment. Reply with (`reply`) and a `threadId`. Resolve with (`update_status`).
- **Read comments**: `repo_pull_request_thread` (`list`), then (`list_comments`) per thread.
- **Labels**: `repo_pull_request_write` (`update`) with `labels` — read the current set first (see below). `az repos pr create --labels` sets them at creation, but `az repos pr update` has **no** `--labels`, so the CLI cannot change them afterwards.
- **Vote**: `repo_pull_request_write` (`vote`) — `Approved` / `ApprovedWithSuggestions` / `NoVote` / `WaitingForAuthor` / `Rejected`. `az repos pr set-vote --id <id> --vote approve`.
- **Reviewers**: `repo_pull_request_write` (`update_reviewers`) with `reviewerAction: add | remove`. `az repos pr reviewer add/remove`.
- **Abandon a PR**: comment the explanation, then `repo_pull_request_write` (`update`) with `status: "Abandoned"`. `az repos pr update --id <id> --status abandoned`.
- **Build status** (the nearest thing to `gh pr checks`): `pipelines_build` (`get_status`), available when the server is started with the `pipelines` domain.

### Four things that bite

- **Naming a repo requires the project too.** `repositoryId` accepts a name only when `project` is also passed, and the `Project/RepoName` slash form fails outright — the server's own error text tells you to use the GUID from `repo_repository` (`list`) instead. Pin that GUID in the config block above and pass it. Where one ADO project holds one repo, the project and repo names are the same string, which makes a missing `project` easy to miss.
- **PR labels are a full-array write.** `labels` on `repo_pull_request_write` (`update`) replaces the whole set; there is no add or remove. Read, modify the array, write it back. The trap is sharper than it looks: reading the current labels needs `includeLabels: true` on (`get`), which **defaults to false** — so the obvious get-then-update sequence returns no labels and then writes that emptiness back, silently wiping them.
- **The MCP server cannot complete a PR.** Its `status` enum is `Active | Abandoned` only — there is no `Completed`. Abandoning works; merging does not. To merge, use `az repos pr update --id <id> --status completed`, or set `autoComplete: true` via (`update`) and let policies land it, or the web UI. **Never report a PR as merged on the strength of an MCP write** — verify with a fresh (`get`).
- **Comments are threads, not a flat list.** There is no `gh pr view --comments` equivalent: reading a discussion is (`list`) followed by (`list_comments`) per thread. Writing is worse — a bare (`create`) opens a *new* thread, so continuing an existing discussion means finding its `threadId` first and using (`reply`). The CLI has no PR comment command at all; the fallback is `az devops invoke --area git --resource pullRequestThreads` (run `az devops invoke` bare to list valid area/resource pairs).

### Pull requests as a triage surface

**PRs as a request surface: no.** _(Set to `yes` if this repo treats external PRs as feature requests; `/triage` reads this flag.)_

Think twice before setting it to `yes` here. GitHub's `authorAssociation` — the field that separates a stranger's contribution from a maintainer's in-flight branch — **has no Azure DevOps equivalent**. `repo_pull_request` (`list`) filters by `created_by_me` and `created_by_user` (email) and exposes no membership concept at all, because ADO Repos is an internal-team model with no fork-from-a-stranger flow. So "external" cannot be derived; if this repo turns the flag on, define it explicitly here — an author email allow/deny list, or a label applied by hand — and have `/triage` read that definition. Guessing is worse than leaving the flag off.

### Work-item links

Azure DevOps links a PR to work items via `workItems` (space-separated ids) on `repo_pull_request_write` (`create`), `az repos pr work-item add/list/remove`, and `#<id>` references in PR descriptions and commit messages. Whether those ids mean anything here depends on the tracker configured above: they point at Azure Boards work items, and if the tracker is a different system its own ids are a **separate namespace**. Where a repo's commits carry a bare `<id>:` prefix or branches follow `<type>/<id>-<slug>`, record which system that id belongs to rather than assuming — resolving it against the wrong tracker silently returns the wrong ticket.
