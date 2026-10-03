# Security Policy

## Reporting a Vulnerability

Please do not report security vulnerabilities through public issues, pull requests, or discussions.

Use GitHub's private vulnerability reporting instead:

1. Go to the [Security tab](https://github.com/diogocouto18/ci-cd-pipeline-template/security) of this repository.
2. Click **Report a vulnerability**.
3. Describe the issue, the affected file(s), and steps to reproduce.

You can expect an initial response within a few days. This is a personal, best-effort project, so there is no formal SLA.

## Scope

This repository is a template for CI/CD pipelines. Issues of interest include, for example:

- Workflow patterns that could leak secrets or allow privilege escalation (for instance, unsafe use of `pull_request_target` or untrusted input in `run:` steps).
- Insecure defaults in the SSH/Docker Compose deploy workflow.
- Mistakes in the example configuration that could lead users into an insecure setup.

Vulnerabilities in third-party actions or tools used by the template should be reported to their respective maintainers.

## Supported Versions

Only the latest version on the `master` branch is supported.
