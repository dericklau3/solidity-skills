---
name: foundry-test
description: "Review and strengthen Solidity / Foundry tests after development. Use when users need test gaps identified, high-value tests added or improved, realistic user flows covered, deployment scripts reused where relevant, `test/docs/*.md` maintained, and `forge test` verification reported."
license: AGPL-3.0-only
metadata:
  author: derick
---

# Review and Strengthen Foundry Tests

## Core Workflow

### Understand the Request Before Editing

For conceptual questions ("How should I test this contract?"), explain without editing code. For requests to add, review, improve, or document tests, proceed with the workflow below.

### CRITICAL: Always Read the Project First

Before writing or changing tests:

1. **Search the user's project** for Solidity contracts, Foundry tests, deployment scripts, helpers, mocks, fixtures, and `test/docs`.
2. **Read the relevant files** to understand the existing behavior, test style, setup model, and documentation conventions.
3. **Default to integration with the existing test suite, not replacement**. Add focused tests and helpers in the local style. Only reorganize or replace broad test structure when explicitly requested.

If a file cannot be read, surface the failure explicitly. Report the path attempted and the reason. Never silently fall back to generic testing advice as if the project context does not exist.

### Fundamental Rule: Test Behavior, Not Coverage Counts

Before adding any test, identify the behavior or guarantee it proves:

1. **Clear behavior gap exists?** Add or improve the smallest test that proves it.
2. **Behavior is already covered?** Do not add a duplicate just to increase test count.
3. **Expected behavior is ambiguous?** Stop and name the ambiguity instead of inventing assertions.

Favor a smaller number of high-signal tests over broad mechanical coverage expansion.

### Methodology

The primary workflow is **behavior discovery from project source and existing tests**:

1. Inspect contracts, tests, scripts, docs, mocks, and helpers.
2. Identify externally meaningful behaviors, failure paths, state transitions, and multi-step flows.
3. Choose the lightest test layer that proves each behavior.
4. Patch tests and matching `test/docs` together.
5. Run `forge test` and report the result.

See [Behavior Discovery and Test Strengthening](#behavior-discovery-and-test-strengthening) for the full procedure.

## Behavior Discovery and Test Strengthening

Procedural guide for strengthening Foundry tests without turning the task into a rewrite, audit, or coverage-maximization pass.

**Prerequisite:** Always follow the behavior-first rule above.

### Step 1: Identify the Test Surface

1. Read `README.md` and `foundry.toml` when they exist.
2. Search `src/` for the contracts in scope.
3. Read related tests under `test/`.
4. Read deployment or setup scripts under `script/` when behavior depends on deployed configuration.
5. Read existing `test/docs/*.md` when present.
6. Read mocks, fixtures, and helper contracts only as needed to understand current setup.

### Step 2: Map Behaviors to Test Layers

Use the lightest layer that proves the behavior:

1. **Unit test** - for isolated return values, state updates, emitted events, boundary checks, revert paths, and access checks.
2. **Integration test** - for behavior that depends on multiple contracts, deployment setup, external interactions, asset movement, or realistic sequencing.
3. **Realistic flow test** - for behavior whose risk comes from how a user or operator actually reaches it through public entrypoints.
4. **Fuzz test** - for arithmetic, validation, monotonicity, idempotence, or state transitions with broad input ranges.
5. **Invariant test** - only for long-lived system properties that should hold across call sequences.

Do not promote every unit behavior into an integration test. Do not generate fuzz or invariant tests mechanically.

### Step 3: Prefer Project-Realistic Setup

When the project has reusable setup or deployment helpers, prefer using them so tests exercise the same inputs, order, environment assumptions, and post-deploy configuration as the project expects.

Use mocks or direct helper shortcuts only when the shortcut is not the behavior under test. If the test claims to prove a real user path, the action should go through the relevant public entrypoint.

For fork tests, pin the block when possible and document assumptions that affect assertions.

### Step 4: Patch Tests

Add or modify tests only after the missing behavior is clear.

Keep changes focused:

- Reuse local naming, fixtures, helper patterns, and assertion style.
- Add new helpers only when they remove meaningful repetition.
- Avoid production refactors unless they are necessary to make the test possible and are within the user's request.
- Use approximate assertions only when external math, rounding, timing, or unit conversion makes exact equality inappropriate.

### Step 5: Maintain Test Documentation

When adding or materially changing a test file, create or update its matching documentation under `test/docs`.

Default convention:

- `test/Foo.t.sol` -> `test/docs/FooTest.md`

If the repository already uses another convention, follow the local convention.

Each test doc should capture:

- The test file's purpose and setup model
- The major scenarios or test groups
- Important helpers and what they prepare
- Fork, mock, timing, rounding, or environment assumptions that matter

Documentation should describe behavior, not repeat every assertion line-by-line.

## Verification

After edits, run `forge test`.

When focused verification is useful, run the narrow command first, then the full suite when feasible. If RPC access, dependencies, or environment variables are missing, report the exact blocker and the command that could not be completed.

Do not claim completion without reporting the verification result.

## Output Contract

Default final output should include:

- Files reviewed
- Tests added or updated
- Test docs added or updated
- Test layers used and why
- Gaps intentionally skipped and why
- `forge test` result
- Remaining ambiguity or risk
