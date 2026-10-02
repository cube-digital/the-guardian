# Test Generation Standards — QA Engineering Guide

<!-- platform-guide-version: v1 | source: cube-digital/the-guardian, docs/the-guardian/QA_TEST_GENERATION_GUIDE.md -->

This document tells an AI model **how to analyze a repository and generate unit, integration, and e2e tests** the way a senior QA engineer would. It is meant to be referenced (directly or via `@`-include) from the wrapper prompt in a project repo, so the central pipeline can produce tests that are targeted, non-redundant, and actually catch regressions — not just tests that inflate coverage numbers.

Follow the phases in order. Do not skip Phase 0–2 and jump straight to writing tests — the single biggest failure mode of AI-generated test suites is testing what's easy to test instead of what's risky to break.

---

## Phase 0 — Repository Reconnaissance

Before writing a single test, build a mental model of the system:

1. **Identify the stack**: language, framework, runtime, package manager, test runner already in use (don't introduce a second test framework unless none exists).
2. **Map the entry points**: HTTP routes/controllers, GraphQL resolvers, RPC handlers, queue consumers, CLI commands, cron/scheduled jobs, webhooks, event listeners.
3. **Map the data layer**: ORMs/models, migrations, raw SQL, caches, external datastores. Note anything with cascading deletes, soft-deletes, or unique constraints — these are common bug sources.
4. **Map external dependencies**: third-party APIs, payment processors, auth providers (OAuth/SAML/OIDC), email/SMS senders, feature-flag services, file storage. These are where mocking strategy matters most.
5. **Find existing tests**: read what's there before adding more. Note the assertion style, fixture/factory patterns, naming conventions, and mocking conventions already established — new tests should look like they belong, not like a foreign import.
6. **Read config/env handling**: `.env.example`, `config/*`, secret managers, feature flags. You need this to know what must never appear literally in a test fixture (see Secrets Handling below).

**Output of this phase**: a short internal map (stack, entry points, data layer, external deps, existing conventions). Use it to drive every later phase — don't re-derive it per file.

---

## Phase 1 — Critical Surface Identification

Not all code deserves equal test investment. Rank surfaces by **blast radius if broken**, not by how easy they are to test.

### Tier 1 — Always test thoroughly (unit + integration, often e2e)
- Authentication and session handling (login, token refresh, logout, password reset)
- Authorization / access control (role checks, ownership checks, tenant isolation, IDOR-prone endpoints)
- Payment, billing, and anything touching money or credits
- Data mutation endpoints (create/update/delete), especially ones with side effects (cascades, webhooks fired, emails sent)
- Anything handling PII (user data export, GDPR delete, profile updates)
- Idempotency-sensitive operations (payment webhooks, retried jobs, "at-least-once" consumers)
- Input validation and sanitization boundaries (anywhere untrusted input crosses into a query, shell command, template render, or file path)
- Rate limiting / abuse prevention logic
- Migrations and schema changes with data-loss potential

### Tier 2 — Test for correctness, moderate depth
- Business logic / calculations (pricing, scoring, aggregation, date/timezone math)
- Read endpoints with filtering, pagination, sorting
- State machines / workflow transitions
- Background jobs without external side effects

### Tier 3 — Light coverage, smoke-level only
- Pure UI presentation components with no logic
- Simple CRUD read paths with no business rules
- Generated/boilerplate code, config loaders

**Rule of thumb**: if the failure mode is "wrong number shown" → Tier 2/3. If the failure mode is "user A sees user B's data," "money moves incorrectly," "credentials leak," or "system becomes unavailable" → Tier 1, no exceptions.

---

## Phase 2 — Test Type Selection

For each surface identified in Phase 1, decide test type using this decision tree rather than defaulting to unit tests everywhere:

| Question | If yes → |
|---|---|
| Is the logic pure (no I/O, deterministic given inputs)? | **Unit test** |
| Does correctness depend on how 2+ components interact (DB query + ORM mapping, service + real cache, API + serializer)? | **Integration test** |
| Does the behavior only manifest across a full request lifecycle (middleware chain, auth guard, real routing) or across a full user journey (multi-step UI flow)? | **E2E test** |
| Is it a Tier 1 surface with external side effects (webhook fired, email sent, payment captured)? | **Integration test with real or containerized dependency**, not a mocked unit test — mocking the exact thing that tends to break defeats the purpose |

**Anti-pattern to actively avoid**: writing a unit test that mocks every collaborator so thoroughly it only proves "the mock returns what the mock was told to return." If a test's assertions would still pass after deleting the implementation and hardcoding the mocked return value, it is not a real test — reclassify it as integration or rewrite it to exercise real logic.

Don't chase a fixed pyramid ratio for its own sake. The shape falls out of Phase 1: heavy unit coverage on pure logic, integration coverage concentrated on Tier 1 surfaces, e2e coverage limited to the handful of journeys that actually represent revenue or trust (signup → first action, checkout, permission boundaries).

---

## Phase 3 — Writing Unit Tests

- **Structure**: Arrange–Act–Assert (or Given–When–Then). One logical behavior per test.
- **Naming**: describe behavior and condition, not implementation — `returns_403_when_user_lacks_role` not `test_permission_check_2`.
- **Isolation**: no shared mutable state between tests; no test-order dependence. If a test fails when run alone but passes in the suite (or vice versa), that's a bug in the test, fix it before moving on.
- **Mock only true boundaries**: network calls, clock/time, randomness, filesystem, external services. Do not mock the module under test's own internal collaborators just to "simplify" — that hides real bugs in the interaction.
- **Cover the edge shape, not just the happy path**, for every function under test:
  - Empty / null / undefined input
  - Boundary values (0, -1, max int, empty string, empty array, exactly-at-limit)
  - Wrong type / malformed input (if the language doesn't statically prevent it)
  - Concurrent/duplicate calls if the function isn't naturally idempotent
  - Error paths: does it throw the right error type, or silently swallow?
- **Time and randomness**: inject/mock the clock and RNG rather than sleeping in tests or relying on `Date.now()` — flakiness here is one of the top causes of untrustworthy suites.
- **One assertion concept per test** — multiple `expect` calls are fine if they verify one behavior; don't bundle unrelated behaviors into one test to save file space.

---

## Phase 4 — Writing Integration Tests

- **Use real dependencies where feasible**: an actual test database (via containers/test schema, not a shared dev DB), a real cache instance, a real message broker in test mode. Prefer this over mocking the ORM or query layer — that's exactly where integration bugs live (n+1 queries, wrong join, migration mismatch, serialization drift).
- **Reset state between tests**: transactional rollback or truncate-and-reseed per test. Never let one test's leftover data affect another's assertions.
- **Test the actual contract**: for HTTP APIs, assert on status code, response shape, and headers — not just "the handler function returned an object." Use the same client library production code would use, or the framework's official test client.
- **External third-party services** (payment processors, email providers, OAuth providers): use their official sandbox/test mode if available; otherwise a local fake/stub server that implements the real wire protocol — not a hand-mocked object standing in for the whole client. Never hit live third-party endpoints from CI.
- **Cover cross-component failure modes**: what happens when the DB is unreachable mid-transaction, when a downstream API times out, when a webhook is delivered twice.
- **Assert side effects, not just return values**: did the row actually persist with correct fields, did the email-send job get enqueued, did the cache get invalidated.

---

## Phase 5 — Writing E2E Tests

- **Cover journeys, not screens**: pick the handful of flows where failure = lost trust or lost revenue (signup, login, checkout/payment, primary create-action, permission-denied paths). Resist the urge to e2e-test every page — that's what integration/unit tests are for.
- **Test data must be seeded and torn down deterministically**: never depend on data left over from a previous run or manual QA session.
- **Prefer stable selectors** (data-testid / accessible role+name) over CSS classes or DOM structure, so tests don't break on unrelated styling changes.
- **Eliminate flakiness sources deliberately**: no arbitrary `sleep`; wait on explicit conditions (element visible, network idle, specific state). Retries should be a last resort for genuinely non-deterministic infra (not a substitute for fixing a race condition in the test).
- **Run against an environment that mirrors production configuration** (same auth flow, same feature flags default state) — an e2e suite that only passes with test-only shortcuts enabled isn't testing the real system.
- **Include at least one negative/authorization e2e path**: e.g., a logged-out user hitting a protected page redirects to login; a user without permission sees a 403/blocked UI, not just "happy path only" coverage.

---

## Secrets, Credentials, and Sensitive Data — Non-Negotiable Rules

1. **Never write a real secret, API key, token, or credential into a test file, fixture, or commit** — including ones copy-pasted from `.env` "to make the test pass locally." Use obviously-fake values (`test_sk_dummy_...`, `not-a-real-key`) or pull from a test-only secrets provider/CI secret store.
2. **If a repo scan turns up a hardcoded real secret while generating tests, do not reproduce it anywhere (including test output, PR description, or logs) — flag it as a finding and stop**, don't silently work around it.
3. **Use synthetic PII only** (faker-generated names/emails/addresses) — never copy real user data from a database dump, support ticket, or log sample into a fixture.
4. **Auth tests must cover both directions**: valid credentials succeed, invalid/expired/malformed credentials fail, and — critically — a valid token for user A must not grant access to user B's resources (authorization/IDOR check, not just authentication).
5. **Don't disable security middleware to make a test pass** (e.g., turning off CSRF/auth checks in test config) unless that middleware is explicitly out of scope for the test and the bypass is scoped narrowly, documented, and reverted for e2e/integration runs that should exercise it.

---

## Quality Bar — Definition of Done for Any Generated Test

A generated test is not complete unless it satisfies all of the following:

- [ ] Fails when the behavior it targets is broken (mentally — or actually — revert the fix and confirm the test catches it)
- [ ] Passes reliably in isolation and in the full suite, in any run order
- [ ] Has no `sleep`/timing hacks; waits on explicit conditions
- [ ] Contains no real secrets, tokens, or copied production/user data
- [ ] Has a name that tells you what broke, without opening the test body
- [ ] Cleans up any state it creates (DB rows, files, queued jobs) regardless of pass/fail
- [ ] Runs in reasonable time for its tier (unit: milliseconds; integration: low seconds; e2e: reserved for the journeys that justify the cost)
- [ ] Doesn't duplicate an existing test's coverage under a new name

---

## Anti-Patterns to Avoid

- **Coverage theater**: generating tests that execute a line without asserting anything meaningful about its behavior.
- **Snapshot-everything**: blanket snapshot tests on complex objects that nobody reviews on diff — they catch that something changed, not whether it's correct.
- **Testing the mock**: asserting that a mocked function was called, when the real question is what the system does as a result.
- **Over-mocking integration boundaries**: mocking the DB/ORM in what's labeled an "integration test," which quietly turns it into a unit test with extra ceremony.
- **Flaky-by-design e2e**: broad, brittle selectors and fixed sleeps that make the suite untrustworthy within a few weeks.
- **Ignoring existing conventions**: introducing a new assertion library, mocking framework, or file layout when the repo already has one.
- **Testing framework code**: writing tests that just re-verify that the ORM/framework itself works, rather than the application's use of it.

---

## Pre-Flight Checklist (run this before generating any test file)

1. Have I completed Phase 0 recon for this repo (stack, entry points, data layer, external deps, existing test conventions)?
2. Have I classified the surface under test into Tier 1/2/3 (Phase 1)?
3. Have I chosen the test type via the Phase 2 decision tree, not by default?
4. Do I know what real secrets/PII exist in this repo's config so I can guarantee none leak into fixtures?
5. Am I extending existing test file/naming/fixture conventions rather than inventing new ones?
6. For Tier 1 surfaces: am I testing the authorization boundary explicitly, not just the happy path?
7. Will every test I generate satisfy the Quality Bar checklist above?

If the answer to any of these is "no," go back and resolve it before writing test code — this is what separates a QA-grade test suite from an AI-generated coverage number.
