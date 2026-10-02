# Custom Codex PR Agents

The Guardian supports a custom AI code review layer using `openai/codex-action`.

The goal is to keep the review agent customizable:

- central workflow owns the GitHub/Codex mechanics
- project repository owns domain rules and high-risk paths
- GitHub rulesets decide whether the result blocks merge

## Architecture

```text
Project PR
  -> project wrapper workflow
  -> central reusable codex-pr-review.yml
  -> openai/codex-action
  -> Codex reads AGENTS.md, review policy, PR diff
  -> Codex posts one sticky PR review comment
  -> optional blocking check for P0/P1 findings
```

## Recommended Rollout

Start advisory:

```yaml
blocking: false
```

After two weeks, inspect:

- useful findings
- false positives
- missed issues found by humans
- whether developers acted on comments
- which risk profiles need tuning

Only then consider blocking on P0/P1 findings.

## Security Notes

Use the minimum privileges:

```yaml
permissions:
  contents: read
  pull-requests: write
  issues: write
```

The central workflow uses:

```yaml
sandbox: read-only
safety-strategy: drop-sudo
```

Do not expose deployment secrets to this workflow. Keep review separate from release.

## Project Configuration

Each project should keep:

```text
AGENTS.md
.github/codex/review-policy.yml
.github/workflows/codex-review.yml
```

The policy file should describe domain-specific risks, for example:

```yaml
risk_profiles:
  payments:
    paths:
      - src/payments/**
    required_review:
      - webhook signature validation
      - idempotency and replay protection
      - transaction boundaries
      - audit trail
      - duplicate provider event tests
```

