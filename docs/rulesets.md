# Recommended GitHub Rulesets

Configure these rules at the organization level where possible.

## Main Branch

Target:

```text
main
```

Recommended rules:

- Require a pull request before merging.
- Require at least one approving human review.
- Dismiss stale approvals when new commits are pushed.
- Require all conversations to be resolved.
- Require status checks:
  - `Django Quality Gate` for Django projects.
  - `Security Gate`.
- Require branches to be up to date before merging if your team can tolerate the extra queue time.
- Block force pushes.
- Block branch deletion.

## Production Tags

Target:

```text
v*
```

Recommended rules:

- Restrict who can create matching tags.
- Require signed tags if your team already has signing habits.
- Trigger production deploys only from protected tags.

## GitHub Environments

Recommended environments:

```text
dev
staging
production
```

Production should require:

- manual reviewer approval
- deployment timeout
- restricted secrets
- no static long-lived cloud credentials

## Bypass Policy

Keep bypass small and explicit:

- no broad admin bypass
- small break-glass group only
- audit every bypass

