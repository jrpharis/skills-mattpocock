---
"mattpocock-skills": patch
---

Teach `/setup-matt-pocock-skills` that the issue tracker and the code host can be different systems, and add **Azure DevOps** as the first code host recorded on its own.

On GitHub and GitLab the tracker *is* the code host, so those templates fold their PR recipes inline. Shortcut isn't, and its template used to hand-wave the gap — "keep the `gh` / `glab` PR recipes from the GitHub or GitLab template alongside this file" — which leaves an ADO shop with no recipes at all. The new `code-host-azure-devops.md` is a **section**, not a fifth tracker choice: setup appends it to `docs/agents/issue-tracker.md` when the chosen tracker isn't the code host and the remote points at ADO. No new config file, and no downstream skill changes — `/triage` already reads the PR surface from the tracker config.

- **MCP-first, `az` fallback**, matching the Shortcut template's shape: the `@azure-devops/mcp` server for all operations, `az repos pr` when none is wired up. Tools are matched by capability and `action` rather than exact name, because v2.9.0 consolidated per-action tools into verb-parameterised ones.
- **Four traps are documented** rather than discovered the hard way: naming a repo fails without `project` (pin the GUID from `repo_repository`); PR labels are a full-array write whose `includeLabels` read flag defaults to *false*, so the obvious get-then-update sequence wipes them; the MCP server's status enum is `Active | Abandoned` with no `Completed`, so **no agent can honestly report a PR as merged** through it (`az repos pr update --status completed` can); and comments are threads, so a bare create opens a new one instead of replying.
- **The PR-as-request-surface flag stays off**, with the reason spelled out: GitHub's `authorAssociation` has no ADO equivalent, so "external contributor" cannot be derived and has to be defined by hand if a team wants it.
- **Detection is host-aware.** The explore step now recognises ADO remotes (`ssh.dev.azure.com`, `dev.azure.com`, `visualstudio.com`) and states that a remote identifies the code host, not necessarily the tracker.

Also fixed nearby: `setup-matt-pocock-skills`'s summary line still claimed GitHub was the default and local markdown the only alternative; `to-tickets` listed Linear as an example tracker, which was never a template; and `triage`'s agent-brief doc hard-coded "a GitHub issue or PR". Docs page re-synced.

Azure Boards is deliberately **not** added as a tracker — that's a separate, additive change.
