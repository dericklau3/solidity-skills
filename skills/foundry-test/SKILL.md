---
name: foundry-test
description: "Review and strengthen Solidity / Foundry test coverage after development. Scans `src/` and existing `test/`, prioritizes high-value unit tests, adds targeted integration, fuzz, and invariant coverage when justified, and verifies with `forge test`."
license: AGPL-3.0-only
metadata:
  author: derick
---

# Review and Strengthen Foundry Tests

## Core Workflow

### Understand the Request Before Editing

Use this skill after Solidity development is complete and the user wants missing or weak tests to be reviewed, added, or strengthened. By default, inspect the repository's Solidity contracts under `src/` and the current tests under `test/`, unless the user narrows scope to specific contracts or files.

### Read the Project First

Before writing or changing tests:

1. Read `README.md`
2. Read `foundry.toml` when it exists
3. Search `src/` for the relevant Solidity contracts
4. Read existing tests under `test/`
5. Read supporting mocks, helper contracts, fixtures, and scripts when they are needed to resolve intended behavior

If a required file cannot be read, say so explicitly and do not pretend the test pass is complete.

### Default Scope

By default, inspect:

- Solidity contracts under `src/`
- existing tests under `test/`
- user-specified contracts or test files

Repository-wide discovery does not mean every contract needs every test type. Test depth should follow value and clarity, not mechanical completeness.

### Coverage Priorities

Review and prioritize gaps across:

- successful paths
- revert and failure paths
- permission and role enforcement
- state transitions and storage effects
- emitted events
- boundary conditions
- cross-contract interaction behavior
- fuzzable input constraints
- invariant-worthy system properties

Favor a smaller number of high-signal tests over a larger number of repetitive low-value tests.

## Test Layer Rules

### 1. Unit Tests First

Unit tests are the default and highest-priority output of this skill.

Prefer unit tests that clearly validate:

- expected return values
- storage updates
- emitted events
- revert selectors or revert reasons when they are stable and meaningful
- boundary inputs
- role and permission restrictions

### 2. Targeted Integration Tests

Add integration tests when behavior depends on multiple contracts, deployment wiring, callbacks, token movement, or realistic flow sequencing that isolated unit tests do not capture well.

Do not convert every unit behavior into an integration test.

### 3. Selective Fuzz Tests

Add fuzz tests when they materially improve confidence.

Good fuzz targets include:

- arithmetic or accounting logic with broad input ranges
- validation logic with clear constraints
- monotonicity or idempotence properties
- state transitions that should hold across many inputs

Do not generate fuzz tests mechanically for every function.

### 4. Selective Invariant Tests

Add invariant tests only when the module has a clear long-lived system property worth preserving across call sequences.

Good invariant candidates include:

- balance conservation
- asset and share consistency
- bounded state-machine transitions
- permissions that should never be bypassed
- accounting relationships that must always hold

Do not force invariant tests onto simple modules that ordinary unit tests already cover adequately.

## Editing Rules

### Audit Before Patching

Review the existing tests before writing new ones. Identify what is missing, weak, or redundant before adding coverage.

### Patch Clear Gaps Directly

If intended behavior can be inferred reliably from the implementation, naming, existing tests, mocks, scripts, and project documentation, directly add or improve tests.

### Stop on Ambiguity

If the expected outcome is not clear from code and context, do not invent assertions. Name the exact behavior that is ambiguous, explain why it is unclear, and leave it unpatched until clarified.

### Keep Scope Tight

Do not turn a testing pass into a production refactor, fixture redesign, audit report, or coverage-maximization exercise.

## Verification

After edits, run `forge test`.

If it fails, explain whether the failure appears caused by:

- the new or modified tests
- a pre-existing repository issue
- project configuration or dependency problems unrelated to the new tests

Do not claim completion without reporting the `forge test` result.

## Output Contract

Default final output should include:

- reviewed contracts and test files
- major testing gaps identified
- tests added or updated
- which test layers were used: unit, integration, fuzz, invariant
- skipped gaps and why they were skipped
- `forge test` result
- any remaining ambiguity or test-risk notes

## Response Style

- Be direct
- Stay file-grounded
- Prefer behavior coverage over mechanical coverage expansion
- Do not guess assertions
- Do not blur the boundary between testing work and security review
