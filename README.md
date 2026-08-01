# CI/CD Setup Guide

This matches your architecture exactly:

```
Company Account (main / testing / development)
      ▲  (03) manual DEPLOY-gated merge: development -> testing
      |
      ▲  (02) auto PR + auto-merge (fork PR): team development -> company development
      |
Team Account (development / dev-*)
      ▲  (01) auto PR + auto-merge: dev-* -> team development
```

## File placement

| File | Goes in | Branch trigger |
|---|---|---|
| `team-repo/.github/workflows/01-dev-branch-to-development.yml` | Every **team** repo | push to `dev-*` |
| `team-repo/.github/workflows/02-sync-development-to-company.yml` | Every **team** repo | push to `development` |
| `company-repo/.github/workflows/03-deploy-development-to-testing.yml` | The **company** repo | manual dispatch only |

Copy the two `team-repo/...` files into each `cubeage-team-*` fork, and the one `company-repo/...` file into the company repo. Nothing in the files hardcodes a team name — configuration is via variables/secrets below.

## One-time setup

### 1. Company repository
- Create an **environment** named `production-deploy` (Settings → Environments).
  - Add your Team Leads as **Required reviewers** on that environment. This means even though only Team Leads have access to the company account in the first place, triggering workflow 03 still pauses for an explicit approval — a second layer of control over the deploy decision.
- Branch protection on `main` and `testing`:
  - Restrict who can push/merge to Team Leads only.
  - This is what structurally guarantees interns (who never touch the company account) can never merge into `testing` or `main` — workflow 02 only ever targets `development`.

### 2. A "sync" bot/service account
Create (or designate) one GitHub account/PAT that:
- Has **write access to `development` only** on the company repo (enforced via branch protection excluding it from `testing`/`main`).
- Generate a fine-grained PAT for it with `contents: write` and `pull requests: write` scoped to the company repo.

### 3. Each team repository
- Repo secret: `COMPANY_SYNC_TOKEN` = the PAT from step 2.
- Repo variable: `COMPANY_REPO` = `"company-org/project-name"` (the company repo's `owner/name`).
- Settings → Actions → General → Workflow permissions: set to **Read and write permissions**, and check **Allow GitHub Actions to create and approve pull requests** (needed for workflow 01's `gh pr create`/`merge` using the built-in `GITHUB_TOKEN`).

### 4. Team repo branch protection (optional but recommended)
Even though mistakes here are sandboxed, you may still want:
- `development` protected from force-push, so workflow 02 always has clean history to sync.

## Why PRs instead of direct pushes

Both workflow 01 and 02 open a PR and then auto-merge it (`gh pr merge --admin`), rather than pushing directly. This keeps a visible audit trail (who pushed what, when it synced) in both repos' PR history, while still requiring zero manual approval, matching your "automatic, no Team Lead review" requirement for those two stages. Workflow 03 is the only manual, gated step, matching "deployment decision must always remain under the Team Lead's control."

## Extending to a new project

Per project, you only need to:
1. Copy the three files into the right repos.
2. Set `COMPANY_REPO` (team repos) and the `production-deploy` environment reviewers (company repo).
3. Ensure the sync bot's PAT/branch protection is configured once per company repo (can be reused across many team forks of the same project).

No workflow file needs editing — everything project-specific is a variable or secret.