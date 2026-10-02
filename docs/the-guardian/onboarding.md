# Project Onboarding

<!--
platform-guide-version: v1
Source: cube-digital/the-guardian, docs/the-guardian/onboarding.md
These three files are copied, not referenced live. When re-running onboarding,
diff against the source repo at the tag your wrapper workflows consume
(the `@vX` in your .github/workflows/*.yml `uses:` lines) and bump this stamp.
-->

Use this file only to choose the order of the onboarding guides.

## Order

1. Read `docs/the-guardian/QA_TEST_GENERATION_GUIDE.md`.
2. Use it in an AI model session with full context of this repository to generate committed tests and, when useful, `.github/testing/critical-test-plan.md`.
3. Review the generated tests as normal project code.
4. Read `docs/the-guardian/WRAPPER_GENERATION_GUIDE.md`.
5. Use it to generate the project wrapper files, `AGENTS.md`, AI review policy, and GitHub Actions wiring.
6. Open the onboarding PR.

The wrapper guide should consume committed tests. It should not generate tests itself.
