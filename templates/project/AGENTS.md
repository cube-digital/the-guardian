# Project Agent Instructions

Describe the project for AI reviewers and coding agents.

## Project Context

- Runtime:
- Framework:
- Database:
- Deployment:
- Critical domains:

## High-Risk Paths

```text
path/to/auth/**
path/to/payments/**
path/to/security-sensitive-code/**
```

## Review Priorities

Agents should prioritize:

- security bugs
- authorization bugs
- data isolation issues
- idempotency failures
- broken migrations
- missing tests for risky behavior
- unsafe dependency or deployment changes

## Local Verification

```bash
replace with project commands
```

