---
name: foundry-test
description: "Use when adding or strengthening Solidity / Foundry tests, including unit, fuzz, invariant, integration, and regression tests."
license: AGPL-3.0-only
metadata:
  author: derick
---

# Write and Verify Foundry Tests

## Follow This Skill

Follow this skill's applicable scope, execution rules, verification requirements, and output contract. Do not silently skip required steps or substitute advice for authorized implementation. Explicit user instructions and higher-priority instructions take precedence. If a required step cannot be completed, state the specific limitation, continue independent work, and do not claim that step was completed.

## Execution Scope

Invoking this skill, or requesting a test review or improvement, means identify gaps, write the relevant tests, and run `forge test`. Complete that work directly; do not stop at findings, recommendations, or a proposed test plan. An explicit instruction to explain only, run existing tests only, or remain read-only overrides this default.

Use the user's named files or behavior as the scope. Otherwise use the project's configured source and test directories. Preserve unrelated changes and existing test organization. Changes to production behavior require a fix to be within the user's authorized scope; a request for tests alone does not authorize protocol changes.

If one expected behavior cannot be established, leave that assertion unresolved and continue independent tests. Ask only for information that materially determines correctness and cannot be obtained from the project. Reuse authorization already given.

## Discover and Implement

1. Read `foundry.toml`, relevant contracts, existing tests, fixtures, and mocks. Inspect README, setup/deployment scripts, and exact installed dependencies when they determine behavior. Resolve custom paths and remappings instead of assuming `src/`, `test/`, or a remembered library API.
2. Derive expected behavior from entrypoints, assets, roles, accounting units, permissions, and state transitions. Prioritize fund safety, accounting/solvency, permissions, core state transitions, arithmetic/rounding, and boundaries. Tests are evidence of intent, not the specification by themselves.
3. Choose the lightest layer that proves each missing guarantee using the table below. Reuse local naming, fixtures, helpers, and assertions. Do not replace the suite or copy dependency tests wholesale.
4. Exercise realistic setup and public entrypoints for claimed user flows. Mocks and shortcuts are appropriate only when they do not bypass the behavior under test. Cover required hooks, initializer ordering, callbacks, and upgrade integration when relevant. Pin fork blocks when possible and keep environmental assumptions beside assertions.
5. Write tests, run the relevant commands, diagnose failures, and finish the authorized work. Do not generate separate `test/docs` files.

| Layer | Use it to prove |
| --- | --- |
| Unit | A specific operation's result, state, asset movement, events, boundary, revert, or permission check. |
| Integration / realistic flow | Interacting contracts, deployment setup, or a sequence reached through actual public entrypoints. |
| Fuzz | A property over a meaningful parameter domain, such as conservation, monotonicity, rounding bounds, or invalid-input rejection. |
| Stateful invariant | A protocol promise across diverse sequences of user/operator actions. |

Avoid mechanical coverage expansion. Derive invariants from the current protocol and establish important single-operation behavior before adding stateful exploration.

## Test Quality

Place one or two purpose-comment lines immediately after the opening brace of every new or materially changed `test...()` or invariant function. State the behavior or failure prevented; for fuzz tests include the property and input domain, and for invariants the promise across call sequences.

- Assert meaningful outputs: balances, shares/debt/rewards, fees, events, error selectors, permissions, and unrelated users' state as applicable. Use approximate assertions only with a justified rounding, conversion, timing, or external-math tolerance.
- For regressions, construct the minimal state that activates the failed condition without accidentally enabling another successful path. Check critical boundaries and repeat/reordered operations where relevant.
- When the defective version is available, run the regression against it and confirm failure due to the original bug, not compilation, setup, or environment errors; then run the same behavioral assertion against the fixed version and confirm it passes. Keep the comparison isolated from unrelated work. If the defective version or comparison environment is unavailable, state that pre-fix failure was not verified. A passing test on the fixed version alone does not establish that it detects the original bug. If implementing the fix is outside the authorized scope, preserve and report the valid failing regression instead of changing production code.
- For fuzz inputs, prefer `bound()` for continuous ranges and narrow `vm.assume()` for constraints not conveniently represented by bounds. Include edge values; never filter out a counterexample merely to pass.
- For stateful invariants, derive accounting, solvency, conservation, supply, permission, state-machine, and economic properties only where the business rules promise them. Model fees, losses, mint/burn, and permitted donations explicitly.
- Use realistic handlers, multiple actors where relevant, `targetContract(address(handler))`, and selective `targetSelector()` when needed. Keep actions diverse and meaningful instead of mostly reverting. Use ghost variables for flows not recoverable from protocol state. Guards must not hide invalid protocol states.
- Reduce a failing invariant sequence to a realistic reproducer. Determine whether a failure comes from the test assumption, production behavior, or environment. Preserve valid reproducers; never weaken assertions, widen tolerances, or alter production behavior merely to get green tests.

## Verification Scope

Always execute tests for work performed under this skill; select the scope deliberately:

| Change / request | Required execution |
| --- | --- |
| One isolated test or local behavior | Relevant `forge test --match-path <path>` or `forge test --match-test <name>`; add `-vvvv` when diagnosing a failure. |
| Shared fixture/helper, core accounting, public behavior, or interacting contracts | Focused tests first, then every affected suite; use full `forge test` when dependencies are broad or cannot be bounded reliably. |
| Repository-wide test strengthening or an explicit full-suite request | Full `forge test`, including applicable project-required profiles. |
| Test comments only | Relevant test file; no extra fuzz/invariant tests or unrelated full-suite run. |

Run touched-file `forge fmt --check` and inspect the diff after edits. Honor project-required checks. Once required checks pass, broaden or repeat only for new changes, failures, or unresolved concerns. Reuse valid results for unchanged inputs/configuration.

For missing RPC access, dependencies, or environment variables, continue available independent checks and state the exact blocked command. Distinguish new failures from baseline failures; a focused pass is not a green full suite.

## Final Response

Briefly state which tests were written and the actual verification result. Include a remaining blocker or valid failing reproducer only when it affects completion. Keep scenario explanations in test comments; do not output a separate findings report, recommendations list, or test plan.
