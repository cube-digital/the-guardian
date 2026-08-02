# Project Onboarding

Use this checklist when connecting a project repository to the engineering platform.

## 1. Add Wrapper Workflows

Copy the relevant files from `templates/project/.github/workflows/` into the project repository.

For a Django backend, start with:

```text
.github/workflows/ci.yml
.github/workflows/security.yml
.github/workflows/scorecard.yml
.github/workflows/codex-review.yml
```

Add release workflows only after CI/security are passing.

## 2. Add Project Instructions

Create or update:

```text
AGENTS.md
```

This file should explain:

- what the project does
- high-risk paths
- security-sensitive domains
- coding conventions
- test expectations
- commands used locally and in CI

The custom Codex PR reviewer uses this context. Human reviewers should also be able to read it.

For custom Codex review, also add:

```text
.github/codex/review-policy.yml
```

## 3. Add Dependency Policy

Copy:

```text
templates/project/.github/policies/dependency-review.yml
```

Tune allowed licenses and fail thresholds per project.

## 4. Add Renovate

Copy:

```text
templates/project/renovate.json
```

Start with grouped weekly updates and immediate security updates.

## 5. Configure Repository Secrets And Environments

Prefer GitHub OIDC roles over static cloud keys.

For deployable repositories, create GitHub environments:

```text
dev
staging
production
```

Production should require manual approval.

## 6. Configure GitHub Rulesets

Protect `main` with required checks:

```text
Django Quality Gate
Security Gate
```

If Codex PR review is enabled in blocking mode, add the Codex review status check once the team trusts its signal.

## 7. Roll Out In Advisory Mode First

Recommended sequence:

1. Add workflows but do not block merge.
2. Fix obvious failures.
3. Turn CI into a required check.
4. Turn security into a required check.
5. Add Codex PR review as advisory.
6. Promote specific AI/policy checks to blocking only after measuring noise.
