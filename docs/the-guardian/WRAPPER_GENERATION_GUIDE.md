# Wrapper Generation Guide

<!-- platform-guide-version: v1 | source: cube-digital/the-guardian, docs/the-guardian/WRAPPER_GENERATION_GUIDE.md -->

Use this prompt in an AI model session that has full access to the target repository.

The purpose of this guide is to onboard any new or existing project into The Guardian, the centralized engineering platform. It should generate the project-side wrapper files, project-specific agent instructions, a review policy, and CI wiring for the tests that exist in the repository.

The CI pipeline should run committed tests, not hidden runtime-generated tests. Runtime agents may suggest additional tests, but merge gates should use tests that humans can review.

## Central Platform

The reusable platform repository is:

```text
cube-digital/the-guardian
```

Use that exact name in every `uses:` reference you generate. Do not substitute a different organization, and do not guess at a fork or mirror.

## Mental Model

The project repository is the execution environment.

The central `the-guardian` repository is the reusable instruction library.

When a project workflow contains:

```yaml
uses: cube-digital/the-guardian/.github/workflows/<workflow>.yml@v1
```

GitHub loads the workflow definition from the central repository, but the workflow runs in the project repository context:

```text
actions/checkout checks out the project repository
secrets and variables come from the project repository or its GitHub organization
status checks appear on the project pull request
the AI review step reads the project's AGENTS.md and .github/ai-review/review-policy.yml
test commands run against the project code
Docker builds use the project's Dockerfiles and manifests
deployments use the project's environment variables and cloud permissions
```

## Output Goal

Create a PR-ready onboarding change for the target repository.

The change should include:

```text
project-side wrapper workflows
project-specific AI agent instructions
project-specific AI review policy
CI commands that run the repository's committed tests
dependency/security configuration
optional release wrapper if the repository deploys artifacts
clear final notes for humans
```

Do not produce generic filler. Every generated file should reflect what was learned from the repository.

## Expected Directory Tree

Prefer this structure:

```text
AGENTS.md
.github/
  ai-review/
    review-policy.yml
  policies/
    dependency-review.yml
  workflows/
    ai-review.yml
    ci.yml
    security.yml
    scorecard.yml
    release-dev.yml
  images.json
renovate.json
```

Create deployment files only if the repository actually builds or deploys artifacts:

```text
.github/workflows/release-dev.yml
.github/images.json
```

If files with these names already exist, inspect them first. Do not overwrite existing project workflows blindly. Either update them carefully or create platform-prefixed alternatives:

```text
.github/workflows/platform-ci.yml
.github/workflows/platform-ai-review.yml
.github/workflows/platform-security.yml
.github/workflows/platform-scorecard.yml
.github/workflows/platform-release-dev.yml
```

## Step 1: Inspect The Repository

Before writing files, inspect the repository thoroughly.

Run or adapt these commands:

```bash
pwd
git remote -v
find . -maxdepth 2 -type f | sort | sed -n '1,220p'
find .github -maxdepth 4 -type f -print 2>/dev/null | sort
find .docker -maxdepth 3 -type f -print 2>/dev/null | sort
find .deploy -maxdepth 4 -type f -print 2>/dev/null | sort
find . -maxdepth 3 -type f \( -name 'README*' -o -name 'AGENTS.md' -o -name 'package.json' -o -name 'pnpm-lock.yaml' -o -name 'yarn.lock' -o -name 'package-lock.json' -o -name 'pyproject.toml' -o -name 'uv.lock' -o -name 'requirements.txt' -o -name 'poetry.lock' -o -name 'go.mod' -o -name 'Cargo.toml' -o -name 'Dockerfile' -o -name 'docker-compose.yml' -o -name 'compose.yml' \) -print | sort
```

Then read the relevant files:

```bash
sed -n '1,260p' README.md 2>/dev/null
sed -n '1,260p' AGENTS.md 2>/dev/null
find .github/workflows -maxdepth 1 -type f -print -exec sed -n '1,260p' {} \; 2>/dev/null
```

Depending on the stack, inspect:

```bash
sed -n '1,260p' pyproject.toml 2>/dev/null
sed -n '1,260p' package.json 2>/dev/null
sed -n '1,220p' go.mod 2>/dev/null
sed -n '1,220p' Cargo.toml 2>/dev/null
find src app backend frontend services packages apps libs -maxdepth 3 -type d 2>/dev/null | sort | sed -n '1,240p'
find . -path '*/tests/*' -o -path '*/__tests__/*' -o -path '*/e2e/*' -o -path '*/spec/*' | sed -n '1,240p'
```

Do not run expensive test suites unless the user asks. For onboarding generation, reading and targeted analysis is enough.

## Step 2: Identify Stack And Test Tooling

Identify:

```text
primary language/framework
package manager
database
cache/queue/background jobs
API framework
frontend framework
browser/e2e framework
container/deployment shape
existing CI workflows
existing tests and test commands
```

Examples:

```text
Python/Django -> pytest, Django TestCase, APIClient, manage.py check, migration checks
Python/FastAPI -> pytest, TestClient, service/API tests
Node/React/Next -> pnpm/npm/yarn, Vitest/Jest, Testing Library, Playwright/Cypress
Go -> go test ./..., go vet ./..., gofmt
Rust -> cargo test, cargo clippy, cargo fmt
Monorepo -> one QA plan with stack-specific wrappers or TODOs per service
```

If the central platform does not yet provide a reusable CI workflow for the detected stack, do not fake one. Create the AI review/security/scorecard wrappers and add a clear TODO describing the missing central workflow. If a QA test plan already exists, reference it instead of recreating it.

## Step 3: Find Critical Paths

Map the risky areas before generating review policy and workflow wrappers.

Look for:

```text
authentication/login/session/JWT/OAuth
authorization/permissions/RBAC/multi-tenant scoping
payments/billing/webhooks/subscriptions
database models, migrations, transactions, data integrity
file upload/download/export/import
PII/user/customer data
admin actions
background jobs/queues/workers/schedulers
third-party integrations
LLM/AI calls and prompt/data exposure
browser automation
infrastructure/deployment config
Dockerfiles and GitHub workflows
cryptography/signing/certificates
email/SMS/notifications
feature flags and environment-specific behavior
public API contracts and webhooks
```

For each critical path, identify:

```text
files/directories involved
why it is risky
what could break
what security issue could appear
which committed unit, integration, or e2e tests cover it, if any
what CI command should enforce it
```

## Step 4: Generate `AGENTS.md`

Create or update:

```text
AGENTS.md
```

Use this structure:

```markdown
# <Project Name> - Agent Instructions

## Project Context
Describe what this repository does, based on README and code.

## Tech Stack
List languages, frameworks, package managers, databases, queues, infra, and deployment.

## Project Structure
Describe the important directories and ownership boundaries.

## High-Risk Areas
List security-sensitive and correctness-sensitive areas with concrete paths.

## Coding Rules
Describe project-specific architecture, style, dependency, logging, error handling, API, database, and security conventions.

## Testing Expectations
Point to the repository's committed unit, integration, and e2e tests. Do not generate new tests from this wrapper guide.

## Local Verification
List commands developers should run before pushing.

## PR Review Expectations
Explain what the AI model and humans should prioritize during review.
```

Make it specific to the project. Avoid generic advice that could apply to every repository.

## Step 5: Consume Existing Test Outputs

Before generating CI wrappers, check whether the repository already contains committed test files and a critical test plan.

Look for:

```text
.github/testing/critical-test-plan.md
new or modified unit tests
new or modified integration tests
new or modified end-to-end tests
test framework configuration
```

If `.github/testing/critical-test-plan.md` exists, read it and use it as the source of truth for:

```text
critical paths
generated tests
recommended future tests
CI commands
known gaps or human-confirmed TODOs
```

If no critical test plan exists, do not generate tests from this wrapper guide. Instead, identify the existing test commands and add a final note that no committed critical test plan was found.

## Step 6: Generate AI Review Policy

Create:

```text
.github/ai-review/review-policy.yml
```

Use a structure like:

```yaml
review_goals:
  - find correctness regressions in critical paths
  - identify security issues before merge
  - verify tests cover risky behavior
  - flag missing migrations, broken contracts, and unsafe deploy changes

risk_profiles:
  authentication:
    paths:
      - "path/to/auth/**"
    required_review:
      - session handling
      - token handling
      - permission bypasses
      - missing negative tests

  authorization:
    paths:
      - "path/to/permissions/**"
    required_review:
      - object-level access
      - tenant isolation
      - admin-only actions
      - forbidden-path tests

  data_integrity:
    paths:
      - "path/to/models/**"
      - "path/to/migrations/**"
    required_review:
      - migrations
      - transactions
      - backward compatibility
      - rollback impact

  ci_cd:
    paths:
      - ".github/workflows/**"
      - "Dockerfile"
      - ".docker/**"
    required_review:
      - secret exposure
      - least privilege permissions
      - image build safety
      - deploy environment targeting
```

Only include profiles that match real repository paths.

## Step 7: Generate Workflow Wrappers

### AI PR Review

Create:

```text
.github/workflows/ai-review.yml
```

Use:

```yaml
name: AI PR Review

on:
  pull_request:
    types: [opened, synchronize, reopened, ready_for_review]

permissions:
  contents: read
  pull-requests: write
  issues: write

jobs:
  ai-review:
    if: github.event.pull_request.draft == false
    uses: cube-digital/the-guardian/.github/workflows/ai-pr-review.yml@v1
    with:
      instructions-file: AGENTS.md
      review-policy-file: .github/ai-review/review-policy.yml
      review-profile: strict
      high-risk-paths: |
        .github/workflows/**
      local-verification-command: |
        echo "Set project-specific verification commands in this wrapper."
      blocking: false
      model: ""
      effort: ""
    secrets:
      AI_PROVIDER_API_KEY: ${{ secrets.AI_PROVIDER_API_KEY }}
```

Replace `high-risk-paths` and `local-verification-command` with values from the repository analysis.

Keep `blocking: false` by default during rollout. Make it blocking only after the team trusts the signal.

### CI

Create:

```text
.github/workflows/ci.yml
```

If the project is Python/Django and the central platform supports it, use:

```yaml
name: CI

on:
  pull_request:
  push:
    branches: [main]

permissions:
  contents: read
  checks: write
  pull-requests: write

jobs:
  ci:
    uses: cube-digital/the-guardian/.github/workflows/python-django-ci.yml@v1
    with:
      python-version: "<detected-python-version>"
      install-command: "<detected install command>"
      lint-command: "<detected lint command>"
      format-check-command: "<detected format check command or empty string>"
      type-check-command: "<detected type check command or empty string>"
      django-check-command: "<detected Django check command or empty string>"
      migration-check-command: "<detected migration check command or empty string>"
      test-command: "<detected command that runs committed unit/integration/e2e tests>"
      postgres-version: "<detected-or-default>"
      redis-version: "<detected-or-default>"
```

For other stacks, use the matching central workflow if it exists. If it does not exist yet, add a TODO in the final response and, if present, `.github/testing/critical-test-plan.md`.

### Security

Create:

```text
.github/workflows/security.yml
```

Use:

```yaml
name: Security

on:
  pull_request:
  push:
    branches: [main]
  schedule:
    - cron: "17 4 * * 1"

permissions:
  contents: read
  pull-requests: read
  security-events: write
  actions: read

jobs:
  security:
    uses: cube-digital/the-guardian/.github/workflows/security.yml@v1
    with:
      codeql-languages: '["javascript-typescript"]'
      dependency-review-config: .github/policies/dependency-review.yml
      trivy-scan-ref: .
      semgrep-config: p/default
```

Choose `codeql-languages` based on detected languages:

```text
Python -> ["python"]
JavaScript/TypeScript -> ["javascript-typescript"]
Go -> ["go"]
Java/Kotlin -> ["java-kotlin"]
C/C++ -> ["c-cpp"]
C# -> ["csharp"]
Ruby -> ["ruby"]
Swift -> ["swift"]
```

### Scorecard

Create:

```text
.github/workflows/scorecard.yml
```

Use:

```yaml
name: Scorecard

on:
  workflow_dispatch:
  schedule:
    - cron: "21 3 * * 1"

permissions:
  contents: read
  security-events: write
  id-token: write

jobs:
  scorecard:
    uses: cube-digital/the-guardian/.github/workflows/scorecard.yml@v1
```

### Deployment

Create release wrappers only if the repository already has deployable services, Dockerfiles, or deployment definitions.

Create:

```text
.github/workflows/release-dev.yml
.github/images.json
```

Use repository variables and secrets. Do not hardcode AWS account IDs, ECR registries, credentials, service names, or cluster names unless they are already public/non-sensitive project conventions.

The release workflow should reference variables like:

```text
DEV_AWS_REGION
DEV_AWS_ROLE_TO_ASSUME
DEV_ECR_REGISTRY
DEV_ECS_CLUSTER_NAME
DEV_SMOKE_COMMAND
```

## Step 8: Dependency And Renovate Config

Create:

```text
.github/policies/dependency-review.yml
```

Use:

```yaml
fail-on-severity: high
fail-on-scopes:
  - runtime
vulnerability-check: true
license-check: true
allow-licenses:
  - Apache-2.0
  - BSD-2-Clause
  - BSD-3-Clause
  - ISC
  - MIT
  - MPL-2.0
  - Python-2.0
```

Create:

```text
renovate.json
```

Use:

```json
{
  "$schema": "https://docs.renovatebot.com/renovate-schema.json",
  "extends": [
    "github>cube-digital/the-guardian//policies/renovate-base.json"
  ]
}
```

If the central platform repo is private and Renovate cannot access it, mention that a copied local Renovate config may be needed.

## Step 9: Validate Generated Files

After creating files:

1. Parse every YAML file you created or changed.
2. Parse every JSON file you created or changed.
3. Search for stale or wrong platform names:

```bash
rg -n "Qodo|qodo|CodeRabbit|coderabbit|engineering-platform|reusable-ai-review|reusable-python|reusable-security|reusable-docker|trivy-scan-path|package-manager|coverage-command" .
```

4. Search for placeholders:

```bash
rg -n "<project|<detected|<real|your-org" .github AGENTS.md renovate.json
```

Every `<project…>`, `<detected…>`, and `your-org` hit is an unfilled placeholder and must be replaced before the PR is opened. References to `cube-digital/the-guardian` are correct and should stay.

## Final Response

Return a concise summary:

```text
Files created/updated
Detected stack and test frameworks
Generated workflows
Critical paths identified
Test commands wired into CI
Critical test plan status
Required GitHub secrets
Required GitHub variables
Placeholders still needing replacement
Existing workflows left untouched
Assumptions/risks
Validation performed
```

Do not claim the workflows have run in GitHub. Only say local syntax validation was performed.
