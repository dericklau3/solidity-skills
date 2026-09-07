---
name: foundry-post-dev-optimization
description: "Use for post-development Solidity / Foundry gas optimization and proven semantic no-op pruning, including deployment-aware error and storage optimization."
license: AGPL-3.0-only
metadata:
  author: derick
---

# Optimize Solidity Contracts After Development

## Follow This Skill

Follow this skill's applicable scope, execution rules, verification requirements, and output contract. Do not silently skip required steps or substitute advice for authorized implementation. Explicit user instructions and higher-priority instructions take precedence. If a required step cannot be completed, state the specific limitation, continue independent work, and do not claim that step was completed.

## Scope and Execution

Optimize gas and remove proven semantic no-ops in the requested contracts; otherwise use the configured source directory. For an optimization request, implement clear, supported improvements directly. An explicit review-only request stays read-only. This skill does not authorize new features, a security audit, or a broad refactor.

Read compiler/optimizer settings, target contracts, affected callers, tests, scripts, and exact installed dependencies. Trace overrides, data/control flow, external calls, and observable outputs before choosing changes. Inspect existing gas measurement workflows. Preserve unrelated changes; exclude vendored code, generated files, and unrelated formatting from edits. Inspect interfaces and deployment artifacts when needed to determine compatibility.

## Establish Deployment Status

Use explicit user statements and project deployment records for the affected contracts and supported chains. No deployment file or address in the checkout is not proof that a contract has never been deployed. If status remains unknown, continue deployment-independent optimizations and ask only when a deployment-dependent change is otherwise ready.

| Status | Error and storage policy |
| --- | --- |
| Confirmed not deployed on-chain and not an upgrade to existing on-chain state | Prioritize gas: directly replace revert strings with suitable custom errors and pack/reorder storage when justified. These changes are within an optimization request and need no additional compatibility approval. |
| Deployed, including a new implementation intended for an existing proxy | Preserve error selectors/revert contracts and storage compatibility. An unbroadcast implementation is not an undeployed system when it must read existing proxy state. |
| Unknown | Preserve existing error and storage contracts until status is established; complete other proven optimizations. |

Before-deployment permission covers error representation and storage arrangement, not arbitrary business, permission, asset-flow, function-signature, or event changes. Keep input domains and failure conditions intact. Changing variable widths requires proof that every supported value fits, including intermediate calculations and future configurations; do not introduce truncation or new overflow behavior.

For pre-deployment changes, update affected error assertions, callers, interfaces, deployment/initialization scripts, and documentation. Regenerate affected ABI/layout artifacts through the project's normal process. Inspect inheritance, namespaced storage, assembly slot references, and any layout-sensitive tooling. Describe these as intentional pre-deployment gas changes, not semantic no-op deletions.

## Select and Prove Changes

### Local, behavior-preserving changes

Implement when caller/data/control-flow and side-effect evidence is complete:

- Safe `memory` to `calldata`, redundant conversions/calculations, or repeated storage reads/lengths. Cached values must remain valid across mutations, hooks, callbacks, and external calls.
- Dead assignments, ignored private returns, unreachable private branches, and duplicate guards whose preconditions hold on every reachable path.
- Pass-through temporaries or wrappers only when they add no useful domain meaning, authorization, invariant, side effect, diagnostic, or gas benefit.
- Redundant helper storage-reference parameters when the helper already receives the canonical mapping key. Prove every caller passes `mapping[key]` for that same key and preserve aliasing, delete/reload, and reassignment behavior. Keep explicit references when callers need a pre-fetched slot or another mapping entry.
- Reuse of an installed library component only after reading its actual source and establishing equivalent behavior. Import it; do not paste library source.

Tests alone do not prove redundancy. State the code invariant supporting each non-trivial deletion.

### Changes requiring stronger evidence

- Apply custom errors and storage packing/reordering under the deployment table above.
- Use `unchecked` only after proving all reachable arithmetic ranges.
- Optimize loops, merge branches, or alter dependency hooks only after preserving evaluation order, supported inputs, external-call ordering, side effects, and relevant inheritance/initializer requirements.
- Retain the two-pass count-and-fill array pattern when replacing it would require manual memory-array length truncation. Do not make such truncation a default optimization; classify assembly techniques under the next section.

### Recommendations unless separately authorized

Assembly, function/event ABI redesign, deployed-system compatibility changes, and broad multi-contract rewrites require an explicit broader request and strong evidence. Do not remove untrusted-input validation, authorization, fund-transfer checks, oracle assumptions, callback/reentrancy protection, or supported-version safeguards without proving the same guarantee remains enforced on every reachable path.

For an unclear candidate, preserve that code, record the missing evidence, and continue independent candidates. Do not stop the entire pass or repeat an already-resolved approval question.

## Verification

1. Establish a comparable baseline before claiming measured gas savings: same compiler, optimizer, EVM target, fixtures, inputs, and initial state. Prefer existing benchmark, snapshot, or gas-report workflows. Separate deployment gas from runtime gas and identify affected operations.
2. After edits, run focused tests for changed behavior and touched-file `forge fmt --check`. Run `forge build` when compilation/artifacts are not already covered or the project requires it.
3. For shared helpers, core accounting, public behavior, storage changes, or inheritance changes, test all affected suites; use full `forge test` when dependencies are broad or cannot be bounded reliably. A repository-wide pass or explicit full-suite request requires full verification. Local independent changes do not automatically require unrelated suites.
4. For any authorized deployed-proxy upgrade, compare against the deployed version's layout artifacts or use the project's upgrade validator. Missing baseline layout means compatibility is unverified; passing unit tests is insufficient.
5. Compare gas after the change. Without an actual before/after measurement, report expected gas benefit or structural simplification. Do not claim measured savings from intuition or from a successful test alone.

Honor required project checks and inspect the final diff. Once required checks pass, rerun or expand only for new changes, failures, or unresolved concerns. Distinguish regressions, baseline failures, and environment blockers; continue unaffected checks and report any incomplete validation.

## Final Response

Report the implemented improvements, concise proof for non-trivial pruning, deployment status relevant to compatibility changes, and verification/gas results. Mention deferred candidates only when material, with the missing evidence or reason. Omit empty categories and repeated inventories.
