# ci-cd-pipeline-template

A GitHub Actions CI/CD pipeline template: feature branch → PR → CI → auto-merge on green → explicit deploy trigger → a pre-push hook as a technical backstop. Proven in production on a real Next.js/Prisma app; genericized here so it's not tied to any specific stack.

## The pattern

```
feature branch
      │
      ▼
  open a PR ──────► CI runs: lint, typecheck, test (with a coverage gate),
      │              E2E against a real service container, secret scanning
      │
      ▼
 all green? ──────► auto-merge (squash + delete branch)
      │
      ▼
 merge to main ───► deploy workflow explicitly triggered
```

**Why explicitly trigger deploy instead of just `on: push: branches: [main]`?** Because the auto-merge step runs with the default `GITHUB_TOKEN`, and GitHub deliberately does not let a push made with that token trigger other workflows (an anti-recursion safeguard). Without the explicit `gh workflow run deploy.yml` call in `ci.yml`'s `auto-merge` job, your deploy would silently never fire after an auto-merged PR.

**Why a pre-push hook too, if CI already gates main?** Belt and suspenders. Branch protection with required status checks is the real gate — but it's a paid feature on some plans for private repos. The `.husky/pre-push` hook blocks accidental direct pushes to `main` locally, as a technical backstop that costs nothing and needs no server-side config.

## What's included

- **`.github/workflows/ci.yml`** — lint/typecheck/test/build, E2E against a real Postgres container (swap for whatever your app depends on), gitleaks secret scanning, a "sensitive paths" check that flags (without blocking) PRs touching CI config or agent instruction files, and auto-merge + deploy-trigger.
- **`.github/workflows/deploy.yml`** — SSH + Docker Compose deploy to a VPS, with a first-deploy-only `.env` write (so manual server-side edits survive future deploys) and a migration step before bringing the new containers up.
- **`.husky/pre-push`** — blocks direct pushes to `main`.

## Adopting this

1. Copy `.github/workflows/` and `.husky/` into your repo.
2. Fill in the placeholders in `ci.yml` and `deploy.yml` — they're commented where you need your own values (build/test commands, env vars, the DB migration command).
3. Set repo secrets: `DEPLOY_HOST`, `DEPLOY_USER`, `DEPLOY_SSH_KEY`, plus whatever your app needs.
4. Optionally set the `NOTIFY_WEBHOOK_URL` repo variable if you want failure/sensitive-merge notifications (works with any webhook that accepts `{"text": "..."}` — Slack, Discord, Mattermost, etc. all do).
5. Wire up Husky if you don't already have it:
   ```bash
   npm install --save-dev husky
   npx husky init
   # then replace the generated .husky/pre-push with this repo's version
   ```
6. In GitHub repo settings, make sure Actions has permission to create and approve pull requests if you want the `auto-merge` job's `gh pr merge` to work.

## Auto-merge safety notes

**Fork PRs are never auto-merged.** The `auto-merge` job only runs when `github.event.pull_request.head.repo.full_name == github.repository`. Fork PRs still get CI, but a maintainer has to merge them by hand.

**Trade-off: sensitive-path PRs are still auto-merged.** The `sensitive_paths` job only flags PRs touching `.github/workflows/`, `CLAUDE.md` or `AGENTS.md` (via `notify-sensitive-merge`); it does not block them. That keeps the flow fully automatic on plans without required reviewers, but it means a change that weakens CI can merge itself once the (possibly weakened) checks pass. If that is not acceptable for your repo, do one of these:

- add `needs.sensitive_paths.outputs.hit != 'true'` to the `auto-merge` job's `if:` so those PRs wait for a human;
- or enable branch protection with required reviewers / CODEOWNERS on those paths (the stronger option where your plan supports it).

**Deploys do not run twice for one merge.** With the default setup only one deploy fires per merge:

- Auto-merge pushes with `GITHUB_TOKEN`, which does not trigger `deploy.yml`'s `push: main` trigger, so the explicit `gh workflow run` step is the only deploy.
- If someone merges manually, only the `push` trigger fires, and `auto-merge` fails at `gh pr merge` before reaching the deploy step (the PR is already merged).
- If you replace `GITHUB_TOKEN` with a PAT or GitHub App token, pushes start triggering `deploy.yml` themselves: remove the explicit `Trigger deploy` step, or the merge will deploy twice.
- Keep a `concurrency` group (`cancel-in-progress: false`) on `deploy.yml` so any overlapping runs are serialized instead of running in parallel.

## What this deliberately doesn't do

- No monorepo-specific paths (the real pipeline this is based on runs everything from a `frontend/` subdirectory — adjust `working-directory` if you need that).
- No specific test framework assumed beyond "has an `npm run test:coverage` and `npm run test:e2e` script" — bring your own.
- No branch protection rules — those are configured in repo settings, not in a workflow file.

## License

MIT
