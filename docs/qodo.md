# Qodo Integration

Qodo is an optional hosted candidate for the AI code review layer.

This platform repository now includes a custom Codex PR review workflow. Use Qodo only if you decide you want a hosted governance product instead of owning the review agent yourselves.

```text
Qodo or custom Codex = review intelligence
GitHub Actions = deterministic checks
GitHub rulesets = enforcement
```

## Suggested Rollout

1. Install Qodo on one pilot repository.
2. Configure it in advisory mode.
3. Point it at the project's `AGENTS.md` or equivalent guidance.
4. Track useful findings versus noisy findings for two weeks.
5. Tune review rules and high-risk paths.
6. Only make Qodo blocking after it consistently produces useful, low-noise findings.

## What Qodo Should Review Strictly

High-risk backend areas:

- authentication and authorization
- workspace or tenant isolation
- payment webhooks and billing state
- signing/OAuth callback flows
- file upload/download/export
- secret handling
- LLM calls and prompt/data exposure
- migrations that mutate critical data
- deployment workflows and cloud config

## What Qodo Should Not Block On Initially

Do not make these blocking at first:

- stylistic suggestions
- optional refactors
- naming preferences
- comments about code that deterministic tools already cover

Let Ruff, tests, security scanners, and rulesets handle deterministic enforcement.
