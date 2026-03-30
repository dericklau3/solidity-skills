# foundry-test Skill Design

## Summary

Add a new `foundry-test` skill under `skills/` for Solidity / Foundry projects. The skill is intended for post-development test completion: it should inspect contracts under `src/`, review the current `test/` layout, identify meaningful testing gaps, directly add clear high-value tests, and verify the result with `forge test`.

The skill is test-focused, not a general refactor or audit skill. Its job is to improve confidence in contract behavior by adding and strengthening tests with a unit-test-first approach, while allowing targeted integration, fuzz, and invariant coverage where those test types are justified.

## Goals

- Provide a repeatable post-development workflow for strengthening Foundry test coverage.
- Default to scanning the repository's `src/` contracts rather than only the files touched in the current task.
- Require explicit review of existing `test/` coverage before adding new tests.
- Prioritize high-value unit tests first, then add integration, fuzz, and invariant tests when they materially improve coverage.
- Reuse the project's existing Foundry testing patterns whenever possible.
- End with `forge test` verification and report results clearly.

## Non-Goals

- Maximize line coverage or branch coverage mechanically.
- Rewrite production contracts just to make testing easier unless a minimal test-enabling change is clearly necessary.
- Replace a separate security review with generic adversarial tests.
- Introduce broad new test infrastructure by default when the repository does not already use it.
- Guess intended behavior when the expected assertion cannot be inferred reliably from code and context.

## Trigger and Scope

The skill is intended to be used after Solidity development work is complete and the user wants missing or weak tests to be reviewed and strengthened.

Default scope rules:

- Inspect Solidity contracts under `src/`.
- Inspect existing tests under `test/`.
- Read `README.md` and `foundry.toml` when present.
- Read supporting mocks, fixtures, scripts, and helper contracts when they are needed to understand behavior.
- Allow the user to narrow scope to specific contracts or test files if requested.

The skill should not assume that every contract in `src/` deserves every available test type. Repository-wide scanning is the default discovery mode, not a promise of exhaustive generated coverage.

## Required Coverage Standard

The skill should identify and prioritize gaps across these categories:

- successful paths
- revert and failure paths
- permission and role enforcement
- state transitions and storage effects
- emitted events
- boundary conditions
- cross-contract interaction behavior
- fuzzable input constraints
- invariant-worthy system properties

Coverage should be value-driven. The skill should prefer a smaller number of high-signal tests over a larger number of brittle or repetitive tests.

## Execution Flow

### 1. Build Context

Before writing or changing tests, the skill should read enough project context to understand the intended behavior:

- `README.md`
- `foundry.toml`, when present
- target contracts under `src/`
- existing tests under `test/`
- supporting mocks, fixtures, scripts, and helper contracts when necessary

If required project files cannot be read, the skill must say so explicitly and avoid claiming the test pass is complete.

### 2. Audit Existing Test Coverage

The skill should inspect the current test suite and identify missing or weak coverage for the in-scope contracts. This audit should be behavior-oriented, not file-count-oriented.

The audit should look for:

- important public behavior with no direct assertion
- missing revert-path coverage
- missing access-control checks
- unverified event emission
- missing state transition assertions
- edge conditions and boundary values that are not exercised
- complex multi-contract flows that only have unit coverage
- functions or modules that are good candidates for fuzzing
- modules with meaningful system invariants that are currently untested

### 3. Patch Clear Gaps by Test Layer

The skill should add tests in this default order:

1. high-value unit tests
2. targeted integration tests
3. selective fuzz tests
4. selective invariant tests

This ordering is intentional. Unit tests are the default baseline. Integration, fuzz, and invariant tests are additive layers used when they materially improve assurance.

### 4. Verify with `forge test`

After edits, the skill should run `forge test`.

If verification fails, the skill should report whether the failure appears caused by:

- the newly added or modified tests
- a pre-existing failure in the repository
- project configuration or dependency issues unrelated to the new tests

Do not claim completion without reporting the `forge test` result.

## Test Layer Rules

### Unit Tests

Unit tests are the default and highest-priority output of the skill.

The skill should prefer unit tests that clearly express:

- expected return values
- storage updates
- emitted events
- revert selectors or revert reasons, when stable and meaningful
- behavior under boundary inputs
- role and permission restrictions

Unit tests should stay close to the contract behavior they validate and should follow the project's existing setup and naming style.

### Integration Tests

Integration tests should be added when behavior spans multiple contracts, deployment wiring, callbacks, token movement, or state transitions that are hard to validate meaningfully in isolated unit tests.

The skill should avoid turning every unit behavior into an integration scenario. Integration tests are for validating realistic flows and contract interaction boundaries.

### Fuzz Tests

Fuzz tests should be added selectively, not mechanically.

Good fuzz targets include:

- arithmetic or accounting logic with broad input ranges
- validation logic with clear input constraints
- state transitions that should hold across many inputs
- idempotence or monotonicity properties that can be expressed simply

The skill should avoid low-value fuzz tests that only restate trivial assertions or create noisy failures without improving confidence.

### Invariant Tests

Invariant tests should only be added when the module has a clear, meaningful system property worth preserving across sequences of calls.

Good invariant candidates include:

- balance conservation
- asset and share consistency
- bounded state-machine transitions
- permissions that should never be bypassed
- accounting relationships that must always hold

The skill should not force invariant tests onto simple modules that have no real long-lived system property beyond what ordinary unit tests already prove.

## Patch Policy

If intended behavior can be inferred reliably from implementation, naming, existing tests, mocks, scripts, and project documentation, the skill should directly add or improve tests.

If a testing gap is local and unambiguous, patch it directly.

If a gap would require broad fixture redesign, speculative business assumptions, or major testing architecture changes, the skill should stop short of guessing and report the uncertainty.

If the expected outcome of a behavior is unclear, for example whether a boundary should revert or clamp, the skill must not invent the assertion. It should identify the exact ambiguity and leave that behavior unpatched until clarified.

## Style and Maintenance Rules

- Follow the repository's existing Foundry test style, helpers, fixtures, and naming conventions.
- Reuse mocks and setup utilities before creating new ones.
- Prefer tests with clear failure diagnostics over clever but opaque helper abstractions.
- Keep new tests scoped to the behavior being validated.
- Do not introduce large new harnesses or frameworks unless the repository already relies on them.
- Do not expand a testing pass into unrelated production-code cleanup.

## Output Contract

The skill's final response should include:

- which contracts and test files were reviewed
- which major testing gaps were identified
- which tests were added or updated
- which test layers were used:
  - unit
  - integration
  - fuzz
  - invariant
- which gaps were intentionally left unpatched because semantics were ambiguous or the required scaffolding was too broad
- whether `forge test` succeeded
- if `forge test` failed, whether the failure appears caused by the new tests or by pre-existing repository issues

## Risks and Guardrails

- Do not treat coverage percentage as the primary success metric.
- Do not generate broad low-signal tests just to touch every function.
- Do not guess assertions that the codebase does not support clearly.
- Do not let invariant or fuzz testing dominate the workflow when simpler direct tests would communicate behavior better.
- Do not overlap the skill's mission with a full security audit; keep the focus on testing behavior, not producing audit findings.

## Recommended Skill Structure

The skill should mirror the style of the existing repository skills:

- concise frontmatter with name, description, license, and metadata
- a direct workflow-oriented body
- explicit must / must-not guidance
- verification requirements with `forge test`
- output expectations for the final response

## Open Decisions Resolved

The following default behavior is part of the approved design:

- default operating mode is review first, then directly patch clear testing gaps
- default discovery scope scans the repository's `src/`
- existing `test/` files are part of the required context before writing new tests
- unit tests are the default priority, with integration tests added when behavior crosses contract boundaries
- fuzz and invariant tests are supported by default, but only when they add meaningful value
- success is judged by behavior coverage quality and `forge test` verification, not by raw coverage expansion
