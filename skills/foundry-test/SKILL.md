---
name: foundry-test
description: "Review and strengthen Solidity / Foundry tests after development. Use when users need high-value unit, fuzz, invariant, integration, or realistic-flow tests added or improved, `test/docs/*.md` maintained, and `forge test` verification reported."
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

### Dependency Rule: Test the Project's Actual Integration

When contracts inherit from or compose OpenZeppelin, Solady, Chainlink, Uniswap, proxy libraries, or other dependencies:

1. Locate the installed source through `foundry.toml`, `remappings.txt`, `lib/`, `node_modules/`, or package config.
2. Read the exact component, mock, example, or test in the dependency when behavior depends on hooks, modifiers, role checks, initializers, token callbacks, oracle semantics, or upgradeability.
3. Use dependency examples and generated baselines as references for integration shape, but keep project tests focused on the user's behavior and invariants.
4. Do not copy dependency tests wholesale unless the project intentionally vendors them.
5. Add regression tests around required overrides, initializer order, storage compatibility, and interacting extensions when those are part of the risk.

### Methodology

The primary workflow is **behavior discovery from project source and existing tests**:

1. Inspect contracts, tests, scripts, docs, mocks, and helpers.
2. Inspect dependency source when imported behavior shapes the expected result.
3. Identify entrypoints, assets, roles, accounting relationships, permissions, state transitions, and states that should never occur.
4. Choose the lightest test layer that proves each behavior.
5. Patch tests and matching `test/docs` together.
6. Run `forge test` and report the result.

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
6. Read `remappings.txt`, `lib/`, `node_modules/`, and package config when dependency behavior matters.
7. Read mocks, fixtures, and helper contracts only as needed to understand current setup.

### Step 2: Map Behaviors to Test Layers

Use the lightest layer that proves the behavior:

1. **Unit test** - for one specific operation under specified conditions: return values, state updates, asset movement, emitted events, boundary checks, revert paths, and access checks.
2. **Integration test** - for behavior that depends on multiple contracts, deployment setup, external interactions, asset movement, or realistic sequencing.
3. **Realistic flow test** - for behavior whose risk comes from how a user or operator actually reaches it through public entrypoints.
4. **Fuzz test** - for arithmetic, validation, monotonicity, idempotence, or state transitions with broad input ranges.
5. **Invariant test** - only for long-lived system properties that should hold across call sequences.

Remember the layer split: unit tests prove a concrete operation, fuzz tests explore parameter space, and invariant tests explore protocol state space.

Do not promote every unit behavior into an integration test. Do not generate fuzz or invariant tests mechanically.

For dependency-backed behavior, add the narrowest test that proves the project's integration point: an override is called, a role modifier gates the path, an initializer cannot be skipped or repeated, a callback is handled safely, or a library-imposed invariant remains true after the user's flow.

Testing priority is value- and state-risk first:

1. Fund safety and asset movement.
2. Accounting and solvency.
3. Permissions and privileged operations.
4. Core state transitions.
5. Math, rounding, shares, prices, ratios, and BPS.
6. Boundary conditions.
7. Ordinary business logic.

Prioritize cases that could cause fund loss, unlimited minting, locked funds, unauthorized withdrawal, accounting drift, insolvency, duplicate claims, fee bypass, or wrong pricing.

### Step 3: Design Unit Tests

For each important `public` or `external` state-changing function, decide whether a unit test is needed. Typical candidates include `deposit`, `withdraw`, `swap`, `stake`, `unstake`, `claim`, `borrow`, `repay`, `liquidate`, `mint`, and `burn`.

High-signal unit tests normally cover:

- Happy path: the intended operation succeeds through the relevant caller.
- State changes: core storage, supply, shares, positions, rewards, and global totals change correctly.
- Asset movement: user, protocol, treasury, fee receiver, and other recipient balances change correctly.
- Revert conditions: zero amounts, insufficient balance or allowance, missing permission, exceeded max, invalid state, repeated operation, invalid address, expired deadline, or domain-specific invalid input.
- Access control: authorized caller succeeds and unauthorized caller reverts with the expected custom error or selector when available.
- Boundaries: `0`, `1`, minimum allowed, maximum allowed, near-maximum, just-enough balance or allowance, and one-less-than-enough.
- Events: important state-changing operations emit the expected event with meaningful indexed and non-indexed values.

Do not write tests that only prove "does not revert" when state, balance, event, or accounting assertions are available.

### Step 4: Design Fuzz Tests

Use fuzz tests when the risk is hidden in a broad input range rather than a single hand-picked example. Strong candidates include:

- Fee, reward, share, exchange-rate, price, interest, percentage, and BPS calculations.
- Deposit, withdraw, swap, stake, borrow, repay, and claim amounts.
- Lock duration, vesting time, reward duration, deadlines, epochs, and time-dependent state.
- Ratios such as slippage, collateralization, discount, and reward weights.

Prefer `bound()` to keep inputs in meaningful protocol ranges. Use `vm.assume()` sparingly; if most generated inputs are filtered out, redesign the input model with bounds, actor selection, or state-aware setup.

Fuzz tests should assert properties, not merely replay a unit test with random numbers. Useful examples:

- Fees stay within `0 <= fee <= amount`.
- Deposit increases protocol assets and user shares according to the protocol's rounding rules.
- Deposit followed by withdraw returns the expected assets when no fees, yield, or rounding beyond the protocol's design applies.
- Increasing an input produces a monotonic or otherwise expected change.
- Invalid parameter ranges revert with the expected error.

### Step 5: Design Invariant Tests

Only add invariant tests after understanding the protocol and covering the important single-operation behavior. Do not start with invariants, and do not copy another protocol's invariants without re-deriving them from the current business rules.

Derive invariants from the protocol's actual promises:

- **Accounting:** internal accounting matches real assets or the sum of user balances.
- **Solvency:** protocol-controlled assets are enough to satisfy user claims.
- **Conservation:** assets do not appear or disappear unless mint, burn, fee, reward, or loss rules explicitly allow it.
- **Supply:** minted minus burned supply, or summed user shares, matches reported total supply.
- **Balance:** user and protocol balances cannot enter impossible or debt-creating states.
- **Permission:** unprivileged actors cannot gain admin powers or drain privileged funds after any call sequence.
- **State machine:** paused, closed, liquidated, claimed, locked, or expired states cannot be bypassed.
- **Economic:** repeated small operations, donations, first-depositor paths, rounding, share inflation, or fee bypass cannot extract value beyond protocol rules.

For stateful invariant testing, create a handler that exposes only realistic user or operator actions worth exploring. Use `targetContract(address(handler))`, and use `targetSelector()` when the random call set needs narrowing.

Handler design rules:

- Generate valid but diverse operations; avoid handlers where nearly every call reverts.
- Use `bound()` and state-aware action guards to keep calls meaningful without hiding bugs.
- Model multiple actors when the protocol has multiple users; choose actors from input seeds or local helper logic.
- Track ghost variables such as total deposited, withdrawn, fees, rewards, minted, or burned when protocol state alone is not enough to check accounting.
- Keep invariant functions focused on core protocol properties; a few meaningful invariants are better than many trivial assertions.

When an invariant fails, use Foundry's call sequence to reduce the smallest realistic reproducer. Preserve the failing test or reproducer until the protocol behavior or test assumption is resolved.

### Step 6: Prefer Project-Realistic Setup

When the project has reusable setup or deployment helpers, prefer using them so tests exercise the same inputs, order, environment assumptions, and post-deploy configuration as the project expects.

Use mocks or direct helper shortcuts only when the shortcut is not the behavior under test. If the test claims to prove a real user path, the action should go through the relevant public entrypoint.

For fork tests, pin the block when possible and document assumptions that affect assertions.

### Step 7: Patch Tests

Add or modify tests only after the missing behavior is clear.

Keep changes focused:

- Reuse local naming, fixtures, helper patterns, and assertion style.
- Add new helpers only when they remove meaningful repetition.
- Avoid production refactors unless they are necessary to make the test possible and are within the user's request.
- Use approximate assertions only when external math, rounding, timing, or unit conversion makes exact equality inappropriate.
- Use Foundry tools such as `vm.prank`, `vm.startPrank`, `deal`, `vm.deal`, `vm.warp`, `vm.roll`, `vm.expectRevert`, `vm.expectEmit`, `bound`, `vm.assume`, `targetContract`, and `targetSelector` as needed, but prefer existing project helpers when they exist.

If a new test fails, do not immediately weaken the assertion or rewrite the test to go green. Decide whether the test assumption is wrong or the protocol behavior is wrong. If the protocol may be wrong, keep the reproducing test, record trigger conditions, expected behavior, actual behavior, affected functions/assets, and the risk.

Do not hide real issues by widening tolerance, adding `assume()` to exclude the failing case, or changing production behavior unless the user has confirmed the protocol bug and the fix is in scope.

### Step 8: Maintain Test Documentation

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

When focused verification is useful, run the narrow command first, then the full suite when feasible:

- `forge test --match-path <path> -vvvv`
- `forge test --match-test <testName> -vvvv`
- `forge test`

If an invariant fails, inspect Foundry's failing call sequence and reduce it to a minimal reproducer when feasible.

If RPC access, dependencies, or environment variables are missing, report the exact blocker and the command that could not be completed.

Do not claim completion without reporting the verification result.

## Output Contract

Default final output should include:

- Files reviewed
- Tests added or updated
- Test docs added or updated
- Unit tests added or improved, including the operations and guarantees covered
- Fuzz tests added or improved, including parameter ranges and properties checked
- Invariants added or improved, including why each invariant must hold, how it is checked, and which contracts it covers
- Gaps intentionally skipped and why
- `forge test` result
- Potential protocol issues found during testing, including location, trigger, expected result, actual result, risk, and reproducing test when applicable
- Remaining ambiguity or risk
