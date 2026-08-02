# Engineering Platform Starter

Central reusable automation for GitHub-hosted projects.

This repository is intended to be versioned and consumed by application repositories through GitHub reusable workflows.

## Goals

- Standardize CI, security, and release mechanics across projects.
- Keep project repositories small and project-specific.
- Make quality gates enforceable through GitHub rulesets.
- Avoid hiding project context inside the platform repository.

## Repository Layout

```text
.github/workflows/
  python-django-ci.yml       Reusable CI for Django/DRF backends
  codex-pr-review.yml        Reusable custom Codex PR review agent
  security.yml               Reusable dependency/SAST/filesystem security gate
  docker-ecs-release.yml     Reusable Docker build, push, attestation, ECS release
  scorecard.yml              Reusable scheduled OpenSSF Scorecard workflow

docs/
  onboarding.md              How to onboard a project repo
  rulesets.md                Recommended GitHub branch/tag rules
  qodo.md                    How Qodo fits into this stack
  codex-pr-agents.md         How custom Codex PR agents work

policies/
  dependency-review.yml      Baseline dependency policy
  renovate-base.json         Baseline Renovate config preset

templates/project/
  Example files copied into project repositories
```

## Consumption Model

Application repositories should keep thin wrapper workflows:

```yaml
jobs:
  ci:
    uses: your-org/engineering-platform/.github/workflows/python-django-ci.yml@v1
    with:
      python-version: "3.13"
      install-command: uv sync --frozen
      lint-command: uv run ruff check .
      test-command: uv run pytest
    secrets: inherit
```

The platform owns the process. The project owns the commands, domain instructions, deploy targets, and exceptions.

## Custom Codex PR Review

Projects can opt into a custom AI review agent:

```yaml
jobs:
  codex-review:
    uses: your-org/engineering-platform/.github/workflows/codex-pr-review.yml@v1
    with:
      instructions-file: AGENTS.md
      review-policy-file: .github/codex/review-policy.yml
      review-profile: strict-backend
      blocking: false
    secrets:
      OPENAI_API_KEY: ${{ secrets.OPENAI_API_KEY }}
```

Start with `blocking: false`. Promote to blocking only after the team has measured useful findings versus noise.

## Versioning

Tag stable releases:

```bash
git tag v1.0.0
git push origin v1.0.0
```

Projects should consume pinned tags:

```yaml
uses: your-org/engineering-platform/.github/workflows/python-django-ci.yml@v1.0.0
```

For fast internal iteration, projects may temporarily consume `@main`, but production repositories should pin a release tag.
