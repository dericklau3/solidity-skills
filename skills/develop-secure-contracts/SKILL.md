---
name: develop-secure-contracts
description: "Use when developing Solidity contracts with OpenZeppelin access control, reentrancy protection, emergency pause, ERC20, ERC721, or UUPS upgrades, with strict input validation and asset-flow ordering."
license: AGPL-3.0-only
metadata:
  author: derick
---

# Develop Contracts with OpenZeppelin

## Execution Contract

Implement the requested contracts using the applicable OpenZeppelin components. Follow this skill's validation, asset-flow, state-transition, and verification rules; complete authorized implementation instead of stopping at advice. Explicit user instructions and higher-priority instructions take precedence. Report blocked steps truthfully and continue independent work.

Read project configuration, target code, tests, deployment scripts, and exact installed dependencies before editing. Preserve unrelated changes. Verify compiler support for `require(condition, CustomError(...))` with the actual compiler/backend; do not impose a fixed compiler version, silently upgrade it, or silently replace the required syntax with revert strings or `if/revert`. Resolve an incompatibility before dependent implementation.

## Use OpenZeppelin Components

Select components required by the requested behavior; do not add every component to every contract. Import installed source instead of copying it or recreating equivalent primitives. Verify paths, APIs, hooks, constructors, and initializers from that version.

| Requirement | Integration rule |
| --- | --- |
| Permissions | Use `Ownable`/`Ownable2Step` for one administrative authority or `AccessControl` for distinct roles. Explicitly gate minting, configuration, pause/unpause, asset recovery, and upgrades as applicable. Do not create overlapping authority systems without a business need. |
| Reentrancy | Use the installed `ReentrancyGuard` or its supported upgradeable equivalent on asset-moving entrypoints and other mutating entrypoints sharing exposed accounting. Respect the guard's initialization and EVM requirements. Protected external functions should use internal helpers rather than call one another through nested guard entry. |
| Emergency pause | Use `Pausable` and authorized pause/unpause entrypoints. Define precisely which operations stop and whether emergency exits remain available. Inheritance alone does not enforce pause: apply guards or the appropriate token pause extension/hooks. |
| ERC20 | Extend `ERC20` and only the required extensions. Use `SafeERC20` for external token transfers. Preserve standard transfer, allowance, mint/burn, and event semantics. |
| ERC721 | Extend `ERC721` and required extensions. Treat safe transfers and safe minting as callback boundaries; verify receiver handling and token ownership rules. |
| UUPS | Use the installed UUPS implementation pattern and `ERC1967Proxy`, with upgrade-compatible stateful components where required. Restrict `_authorizeUpgrade` to the intended authority. |

For UUPS, initialize the proxy atomically when deployed, initialize each required parent exactly once in the correct order, and disable initializers on the implementation. Use reinitializers only for deliberate migrations. Preserve existing proxy storage, inheritance, namespaced storage, and upgrade authorization. A new implementation serving an existing proxy is an upgrade, even before that implementation is deployed. Compare against the deployed baseline or use the project's upgrade validator before claiming layout compatibility.

## Validate at Function Entry

At the start of each implemented function, validate its applicable input preconditions with `require(validCondition, CustomError(...))`, before business-state changes or asset-moving calls. An entry validation helper/modifier may contain these checks. For internal helpers, establish preconditions at entry; do not repeat a check when all reaching callers demonstrably enforce the same condition and it cannot change before use.

Validate address roles, supported assets, amounts, array lengths/bounds, IDs, deadlines, limits, and cross-parameter relationships according to the business rules. Do not invent invalid-input rules: for example, valid ERC20 zero transfers and valid token ID zero must remain valid. Retain installed OpenZeppelin permission/guard checks instead of rewriting library internals for syntax uniformity.

```solidity
require(receiver != address(0), InvalidReceiver(receiver));
require(amount > 0, InvalidAmount(amount)); // Only for operations requiring a positive amount.
require(amount <= available, InsufficientBalance(available, amount));
```

Declare or reuse specific custom errors using the project's error organization. Error arguments must be cheap and side-effect-free: `require` evaluates them even on success. Validate bounds before indexing and conditions before arithmetic that could fail first. State-dependent checks must use authoritative state. Conditions only knowable after a transfer require post-transfer validation as well.

## Asset-Flow Ordering

Separate business accounting from the reentrancy lock and narrowly scoped callback-authentication context. The lock must be active before external interactions; writing that protection is not premature crediting of a deposit.

### Inflow: validate → receive assets → verify receipt → update business state

- Validate parameters, permissions, pause state, and the permitted transition first.
- Acquire reentrancy protection before transferring assets. For ERC20 use `safeTransferFrom`; for ERC721 use the intended safe custody transfer. Do not credit balances, shares, deposits, or rewards before receipt succeeds.
- Define exact-amount versus fee-on-transfer support. For exact-amount custody, verify the received balance delta equals the requested amount; reject discrepancies. If fee-on-transfer support is specified, account from actual receipt and enforce relevant minimums. Never credit an unverified requested amount.
- For ERC721 custody, authenticate the expected token contract, operator/from/token ID and callback context, and verify ownership after transfer. Required receiver callbacks may acknowledge the expected transfer but must not credit the deposit early or open another business entrypoint. Reject unsolicited/duplicate callbacks. Do not mark the required callback `nonReentrant` if it would prevent the protected deposit from completing; authenticate it and prohibit business mutations instead.
- For payable native-asset entrypoints, the EVM delivers `msg.value` before the body. Validate `msg.value` and then update accounting; no additional incoming transfer is necessary. Direct donations or forced balances must not silently create user credit.
- After receipt verification, update all related accounting and emit the successful-operation event. Do not allow callbacks or dependent view consumers to use an inconsistent asset/accounting snapshot; guard sensitive reads or avoid exposing that transient state to integrations.

### Outflow: validate → update business state → transfer assets

- Validate the recipient, entitlement, amount, liquidity, permissions, pause policy, and transition.
- Under reentrancy protection, debit balances/claims, consume one-time status, and update related totals before transferring assets. For redemption, burn/debit shares before payout.
- Use `safeTransfer` for ERC20, the required safe transfer for ERC721, and a checked native-value call or suitable installed OpenZeppelin utility. Any failed transfer must revert the entire operation, rolling back accounting.
- Treat ERC721 safe minting and transfers as interactions: establish the corresponding business state before invoking receiver callbacks. Do not swallow transfer errors or record success after a failure.

For a combined operation, compose these phases explicitly: receive incoming assets before crediting them; finalize outgoing accounting before sending assets. Do not introduce an advance-credit/flash-loan path under an ordinary deposit/withdrawal request.

## Enforce the Allowed Call Flow

Derive legal states, actors, prerequisites, transitions, and postconditions from the requested behavior and existing code. Implement those rules as checks, not comments alone. Unexpected callers, unsupported transitions, duplicate claims, wrong-order actions, stale/expired inputs, mismatched callbacks, and failed external operations must revert atomically with an appropriate error; preserve useful dependency errors rather than swallowing them.

Do not silently skip failed batch items, return a success-shaped default, retry a different business path, or catch and continue unless partial success/recovery is an explicitly defined feature. Reject unsupported fallback calls and ordinary unexpected native transfers where applicable; forced native balances cannot be prevented by a reverting `receive` alone.

Use `require` with custom errors for expected preconditions; a branch with no valid continuation may use `revert CustomError(...)`. Reserve `assert` for genuinely internal invariants, not untrusted-input validation.

## Verify and Finish

Compile with the project's actual settings. Add and run focused tests for input errors, unauthorized access, pause/unpause and emergency policy, inflow receipt-before-credit, outflow debit-before-callback, receipt mismatches, transfer failures with rollback, reentrancy including cross-entrypoint callbacks, and forbidden/duplicate transitions. Include token-standard and UUPS initialization/authorization/state-preservation tests where relevant. Use callback-aware mocks to observe ordering, not just final balances.

Update affected callers, deployment scripts, error assertions, interfaces, and documentation. Run touched-file formatting and inspect the diff. Cover all affected suites for shared accounting, inheritance, or initialization changes; run the full suite when scope cannot be bounded or the user/project requires it. After required checks pass, repeat only for changes, failures, or unresolved concerns.

Briefly report implemented behavior, actual verification, and remaining blockers. Do not claim successful execution or upgrade compatibility without evidence.
