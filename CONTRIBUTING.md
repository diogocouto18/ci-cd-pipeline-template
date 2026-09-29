# Contributing

Thanks for helping improve this template. Keep changes small and focused: one concern per pull request.

## Workflow

1. Fork the repo (or create a branch if you have write access) and branch from `master`.
2. Make your change. Never push directly to `master`; open a pull request instead.
3. Explain what changed and why in the PR description, and reference the issue it closes (`Closes #N`).

## Commit messages

Use the format `[Type] Brief Description In Title Case`, with no body unless strictly necessary.

Types: `[Doc]` `[Config]` `[CI]` `[Fix]` `[Chore]` `[Test]`

Example: `[CI] Pin Actions To Commit SHAs`

Write code, comments, docs and commit messages in English.

## Workflow file rules

- Pin every third-party action to a full commit SHA with a version comment, for example `uses: actions/checkout@<40-char-sha>  # v4.4.0`. Dependabot keeps them current.
- Grant the least privilege possible: `permissions: contents: read` at the top level, elevate per job only where needed.
- Keep the template generic: no project-specific paths, hostnames or secrets. Placeholders must be clearly commented.
- If you change the pipeline's behavior, update the README (and the matching file under `examples/`) in the same PR.

## Validating your changes

The `Self-test` workflow (`actionlint`, `yamllint`, `shellcheck`) runs on every PR. To run the same checks locally:

```bash
docker run --rm -v "$PWD:/repo" -w /repo rhysd/actionlint:1.7.7
yamllint --strict -c .yamllint.yml .github/
shellcheck --shell=sh .husky/pre-push
```

Examples under `examples/` are not covered by the self-test; run `actionlint` on them by hand when you touch them.

## Reporting problems

Open an issue describing what you expected and what happened. Never include secrets, tokens or private hostnames in issues or PRs.
