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

## Required repository settings

The workflows depend on settings that live in GitHub, not in the repo files:

- **Branch protection on `main`**: require a pull request and require the `Lint, Test & Build`, `E2E` and `Secret Scan` status checks to pass. This is the real gate; the pre-push hook is only a local backstop. (On some plans private repos cannot use branch protection; the pre-push hook and the in-workflow checks are then your only guard.)
- **Actions can create and approve pull requests**: Settings > Actions > General > Workflow permissions, enable "Allow GitHub Actions to create and approve pull requests" and choose read and write permissions, so the `auto-merge` job's `gh pr merge` works.
- **Allow auto-merge / delete branch on merge**: optional, but keeps the repo tidy after `--delete-branch`.
- **A `production` environment**: Settings > Environments > New environment, named exactly `production` (`deploy.yml` uses `environment: production`). Add required reviewers there if deploys should need a manual approval, and restrict it to the `main` branch. Store the deploy secrets on the environment rather than the repo where possible.
- **Secrets**: `DEPLOY_HOST`, `DEPLOY_USER`, `DEPLOY_SSH_KEY` (a dedicated deploy key, not a personal one), plus your app's own secrets referenced in `deploy.yml`.
- **Variables** (optional): `NOTIFY_WEBHOOK_URL` for failure and sensitive-merge notifications.

## Other stacks

`ci.yml` assumes npm scripts and a Postgres service container, but only the build/test steps are stack-specific. The [`examples/`](examples/) folder has minimal `lint-test-build` jobs, with a coverage gate, for:

- [`examples/python/ci.yml`](examples/python/ci.yml): Python with ruff, mypy and pytest
- [`examples/dotnet/ci.yml`](examples/dotnet/ci.yml): .NET with `dotnet format`, `dotnet test` and coverlet
- [`examples/go/ci.yml`](examples/go/ci.yml): Go with gofmt, `go vet` and a coverage threshold

Each example shows only the language-specific job; keep the secret-scan, sensitive-paths and auto-merge jobs from the root `ci.yml`. See [CONTRIBUTING.md](CONTRIBUTING.md) if you want to improve the template.

## What this deliberately doesn't do

- No monorepo-specific paths (the real pipeline this is based on runs everything from a `frontend/` subdirectory — adjust `working-directory` if you need that).
- No specific test framework assumed beyond "has an `npm run test:coverage` and `npm run test:e2e` script" — bring your own.
- No branch protection rules — those are configured in repo settings, not in a workflow file.

## License

MIT
