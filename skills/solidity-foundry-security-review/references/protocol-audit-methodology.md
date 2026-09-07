# Protocol Audit Methodology

Read this file completely for a full protocol audit. For a targeted review, read the sections reachable from the selected scope and record which surfaces were not reviewed.

This methodology is limited to read-only static analysis. Do not write PoCs, change project files, or perform local validation. Read existing tests as context only; deliver findings and written remediation recommendations.

## Ordered Audit Phases

Use this order so implementation-level patterns do not distract from protocol-level failures:

1. Protocol understanding and architecture
2. Asset flow and custody
3. Roles and trust model
4. Business logic and state machines
5. Invariants
6. Accounting
7. Economic model
8. Oracles
9. External integrations
10. Access control
11. Upgradeability
12. Signature security
13. Reentrancy and callbacks
14. Precision and rounding
15. Flash liquidity
16. MEV and transaction ordering
17. Token compatibility
18. Low-level EVM and assembly
19. Denial of service and griefing
20. Operational and administrative security
21. Candidate validation and findings

Do not stop after finding a severe issue. Complete every applicable phase, or mark it blocked/non-applicable with a concrete reason.

## Working Artifacts

Use these artifacts as lightweight audit notes when the protocol is non-trivial. They are not required final-report headings, but the final answer should expose enough of them to make conclusions reviewable.

- **Entrypoint inventory:** externally reachable function, caller assumptions, lifecycle step, value moved, critical state changed, external calls, and reachable privileged paths.
- **Asset-flow table:** asset, source, custodian, accounting source of truth, conversion formula/unit, normal destination, emergency/privileged destination, and direct-transfer/donation behavior.
- **Role matrix:** role, direct capability, indirect capability through shared helpers or role-admin powers, maximum impact, and intended trust assumption.
- **Invariant map:** invariant, operations that can affect it, attacker-controlled variables, external dependencies, boundary values, and the source evidence and reasoning supporting it.
- **External dependency map:** dependency, service/value relied on, who can influence outputs, revert/pause/upgrade/staleness behavior, callback surface, and user-exit behavior during failure.
- **Candidate ledger:** suspected issue, broken rule, actor, prerequisites, path, impact, existing guards, evidence status, and final disposition as finding, unresolved risk, or rejected.

Prefer these concrete traces over broad vulnerability checklists. A checklist item matters only when it can affect a protocol promise, invariant, asset flow, trust boundary, or liveness guarantee.

## 1. Protocol Understanding and Architecture

Read documentation, production contracts, interfaces, libraries, tests, configuration, deployments, upgrades, and relevant installed dependencies. Produce a compact map containing:

- protocol purpose and intended guarantees;
- core contracts and their relationships;
- externally reachable production entrypoints and their lifecycle phase;
- user and privileged roles;
- external protocols and off-chain actors;
- critical assets and state;
- primary user entrypoints;
- the component that holds assets, controls accounting, decides prices, and can move value.

Do not begin findings analysis until the main lifecycle and control boundaries can be explained coherently.

## 2. Asset Flow and Custody

Trace every relevant native asset, ERC-20/721/1155, LP token, vault share, debt token, reward token, and synthetic asset.

For each asset, determine:

- its source and entrypoint;
- every custodian and transfer boundary;
- who can transfer, mint, burn, lock, approve, or rescue it;
- the formula and unit used for each conversion;
- every normal, emergency, privileged, and external destination;
- whether internal accounting records the actual movement.

Trace complete chains such as user → vault → strategy → external protocol → token transfer. Include deposit, mint, stake, swap, borrow, repay, liquidate, claim, redeem, withdraw, rescue, and emergency paths as applicable.

Look for trapped assets, excess withdrawals, cross-user allocation, unaccounted balances, double valuation, and failure paths that leave value or accounting in an intermediate state.

## 3. Roles and Trust Model

Inventory owner, admin, operator, manager, guardian, keeper, oracle, upgrader, pauser, minter, signer, multisig, governance, role admins, and indirect authorities.

Build a matrix:

| Role | Direct and indirect capability | Maximum impact | Intended trust assumption |
|---|---|---|---|
| Example: upgrader | Replace implementation | Arbitrary protocol behavior | Timelocked multisig |

Check excessive privilege, missing checks, escalation paths, shared internal functions reached by weaker roles, role-admin configuration, initialization, permission transfer, front-runnable ownership handoff, and dangerous combinations of individually limited roles.

Distinguish an explicitly trusted role's documented power from a vulnerability. It becomes a finding when implementation exceeds the trust model or violates a guarantee the role is supposed to preserve. Record concentrated trust even when it is accepted design risk.

## 4. Business Logic and State Machines

Model full lifecycles rather than isolated functions. Analyze normal, reordered, repeated, skipped, interrupted, and cross-user sequences. Examples include:

- deposit → mint → transfer share → redeem;
- deposit → borrow → price change → liquidate;
- stake → claim → unstake → restake → claim;
- pause → emergency action → unpause → normal exit.

Search for duplicate claims/withdrawals/accounting, state skips, prerequisite bypass, limit/cooldown/fee/pause/blacklist bypass, reused positions, repeated valuation, partial failure, and combinations of valid operations that produce an invalid result.

For each transition, record caller, prerequisites, state read, state written, value moved, external calls, and terminal condition.

## 5. Invariants

Derive invariants independently from documentation and code. Useful families include:

- users cannot withdraw more value than their entitlement;
- assets and obligations are conserved except for explicit fees, yield, loss, mint, or burn;
- internal accounting reconciles with the protocol's chosen source of truth;
- debt, collateral, and solvency remain within protocol rules;
- cumulative rewards do not exceed funded or scheduled rewards;
- state transitions cannot skip required prerequisites;
- privileged actions preserve documented user guarantees;
- price inputs are fresh, dimensionally correct, and resistant to the assumed attacker.

For each invariant, enumerate functions, callbacks, direct transfers, privileged actions, extreme values, and external state changes capable of affecting it. Connect every material finding to a violated invariant when possible.

## 6. Accounting

Identify the authoritative variables for assets, shares, supply, principal, debt, borrow, interest, rewards, fees, and reserves. Trace every increment, decrement, reset, realization, and transfer.

Compare actual balances with internal accounting only after determining which one is intended as the source of truth. Specifically inspect:

- duplicate, missing, or incorrectly cleared entries;
- state updates before/after external calls and failures;
- transfers that do not synchronize ownership/accounting;
- direct token transfers, forced native value, and donations;
- interest/reward accumulation across time and user actions;
- duplicated fees or debts that fail to increase/decrease;
- share/asset conversion and first/last depositor behavior;
- cross-user contamination and aggregate-vs-user reconciliation.

An accounting difference is not automatically exploitable. Determine whether the attacker can create it, benefit from it, or use it to deny service.

## 7. Precision and Rounding

Review every division, multiplication, percentage, price/share conversion, decimal conversion, interest, fee, and reward formula.

Determine:

- whether multiplication should precede division;
- the units and decimals at every boundary;
- who benefits from rounding down/up;
- whether splitting or repeating operations accumulates profit;
- whether dust can become trapped, stolen, or weaponized;
- behavior at zero, one wei, minimum valid input, boundary ±1, large values, and maximum supported values.

Use concrete dimensional analysis for mixed decimals such as 6-decimal tokens, 8-decimal feeds, and 18-decimal internal precision. Analyze both individual conversions and round trips.

## 8. Economic Model

Analyze incentives and attacker profitability, not only code correctness. Consider limited capital, large capital, flash liquidity, multiple accounts, high transaction counts, favorable timing, MEV, arbitrary contracts, and cross-protocol composition.

Look for risk-free or circular arbitrage, low-cost manipulation, reward loops, reverse incentives, large-user extraction from small users, leverage amplification, tokenomics that leak protocol value, and actions whose private reward exceeds their protocol cost.

For each candidate, estimate attacker capital, attack cost, recoverable principal, protocol/user loss, and net profit. Separate profitable extraction from griefing that causes disproportionate damage without direct profit.

## 9. Oracles

Map every Chainlink feed, TWAP, DEX spot price, custom/signed/off-chain oracle, LP price, share price, fallback, and derived price to the calculations it controls.

Check freshness, heartbeat, round completeness, sign/zero handling, decimals, sequencer downtime, deviation bounds, update delay, pause/fallback behavior, permissions, source switching, circular dependencies, and unit consistency.

Treat a DEX spot price as manipulable unless the protocol proves otherwise. Analyze flash swaps, large trades, low liquidity, donation, LP manipulation, and same-transaction read/write effects. Trace impact into collateral, borrow limits, liquidation, mint/redeem amounts, shares, and rewards.

## 10. External Protocol Integrations

Treat each external protocol and token call as a trust boundary. Read the installed interface and implementation assumptions.

Check return values, revert behavior, callbacks, slippage, deadline, allowance lifecycle, pricing, decimals, pause/upgrade behavior, mutable external state, partial completion, and unexpected return data.

For each dependency answer:

- what service or value does the protocol depend on;
- which outputs affect funds or accounting;
- who can influence those outputs;
- what happens on revert, pause, stale state, changed behavior, or upgrade;
- whether callbacks can observe or reenter intermediate state;
- whether user funds can still exit during dependency failure.

## 11. Access Control

Inventory every externally reachable function that changes critical state, especially setters, updates, upgrades, withdrawals, rescues, mint/burn, pause/unpause, and initialization.

Check `onlyOwner`, `onlyRole`, role admins, custom modifiers, `msg.sender`, `tx.origin`, proxy context, callbacks, meta-transactions, internal helpers shared across privilege levels, and indirect calls that bypass the intended boundary.

Validate who can grant, revoke, renounce, transfer, or recover authority and whether two-step or delayed transitions can be bypassed or permanently blocked.

## 12. Upgradeability

For Transparent, UUPS, Beacon, Diamond, and custom delegatecall proxies, inspect:

- initializer/reinitializer reachability and versioning;
- implementation initialization and constructor assumptions;
- upgrade authorization, proxy admin, and target validation;
- delegatecall context and arbitrary execution paths;
- storage layout, collisions, gaps, namespaced slots, inheritance order, and type changes;
- migration logic and invariants across versions;
- whether upgrades alter accounting meaning or privilege boundaries.

Resolve the actual proxy pattern and installed library source. A constructor deployment cannot be assumed equivalent to an initialized proxy deployment.

## 13. Signature Security

For EIP-712, permits, authorizations, meta-transactions, and off-chain orders, verify that the signed digest binds every security-relevant parameter: signer, action, amount/token/position, recipient, caller when required, nonce, deadline, chain ID, verifying contract, and domain/version.

Check nonce uniqueness and consumption timing, cancellation, expiration, signature malleability, contract signers when supported, replay across chains/contracts/actions, domain separator changes, and partial-fill semantics.

## 14. Reentrancy and Callbacks

Analyze single-function, cross-function, cross-contract, and read-only reentrancy. Trace every external call made while state or accounting is intermediate, including token hooks, ERC-721/1155 receivers, DEX/flash callbacks, fallback/receive, and arbitrary user targets.

`nonReentrant` on one function is not proof of safety. Determine whether a callback can enter a different function, another protocol component, a view used by an external protocol, or a privileged callback surface before invariants are restored.

## 15. Flash Liquidity

Treat flash liquidity as the ability to use very large capital atomically, not as a vulnerability by itself. Re-evaluate oracle inputs, voting, rewards, share prices, liquidity, collateral, liquidation, accounting, and token price under a large temporary balance.

Analyze sequences of borrow → manipulate → trigger protocol action → unwind → repay. A realistic same-transaction exploit should not be dismissed or downgraded merely because flash liquidity is used.

## 16. MEV and Transaction Ordering

Analyze front-running, back-running, sandwiching, and arbitrary ordering around swaps, price updates, liquidation, reward claims, auctions, deposits, withdrawals, and mints.

Check slippage bounds, deadlines, stale quotes, user-controlled minimums, commit/reveal when appropriate, and whether an observer of a pending transaction can profit by executing before or after it. Distinguish unavoidable market MEV from protocol-created value loss or guarantee violations.

## 17. Token Compatibility

Do not assume `amount sent == amount received` or uniform ERC-20 behavior. As applicable, consider fee-on-transfer, rebasing, ERC-777/hooks, no-return/false-return tokens, blacklisting, pausing, upgrades, callbacks, unusual decimals, transfer restrictions, and tokens that revert on zero or special approvals.

Check safe transfer wrappers, allowance races/lifecycle, balance-before/after measurement, share accounting under rebases, denial of exit due to blacklist/pause, and whether permissionless token listing exposes unsupported behavior. Only require compatibility promised by the protocol.

## 18. Low-Level EVM and Assembly

Raise scrutiny for assembly, `call`, `delegatecall`, `staticcall`, CREATE/CREATE2, raw storage, manual ABI encoding, and returndata parsing.

Check calldata offsets, free-memory pointer, memory overwrite, returndata length/copy, dirty bits, selectors, value forwarding, revert bubbling, storage slots, address masking, and compiler assumptions.

For delegatecall, identify whose code executes, whose storage is used, the effective sender/value, whether the target is controllable, and whether storage layouts match. Prove semantic equivalence rather than assuming assembly is a gas-optimized version of high-level Solidity.

## 19. Denial of Service and Griefing

Inspect unbounded loops, attacker-controlled arrays/storage growth, malicious reverts, gas exhaustion, dust positions, forced value, blocked recipients, callbacks, global locks, and external dependency failure.

Assess attack cost from available code and evidence, repeatability, affected scope, duration, and recovery path. A low-cost attack that blocks the whole protocol may be material even without attacker profit. Distinguish single-user self-DoS from protocol-wide liveness loss.

## 20. Operational and Administrative Security

Evaluate pause, emergency withdrawal, oracle fallback, rate/withdraw limits, multisig, timelock, monitoring assumptions, upgrade delay, and recovery.

Model compromised admin keys, stale oracles, paused/upgraded external protocols, missing DEX liquidity, token blacklist/pause, and multisig compromise. Verify which actions remain available while paused, whether users can exit, whether emergency paths preserve accounting, whether timelocks can be bypassed, and whether guardians/rate limits materially constrain loss.

## 21. Candidate Validation and Findings

Do not formalize a candidate without root cause, reachable attack or failure path, and impact. Validate exact dependency behavior and all existing guards. For material issues, cite the source locations and explain why the path is reachable and the impact follows. State evidence gaps without attempting local reproduction.

Severity depends on impact and likelihood, informed by attack cost, privilege, capital, affected assets/users, exploit complexity, and recoverability. Do not report code style as a security issue, equate a failing test with a vulnerability, or assign High/Critical to an unproven theory.

## Completion Checklist

A full protocol audit is complete only when each item is completed or explicitly marked blocked/non-applicable with evidence:

- [ ] Protocol purpose, architecture, components, and main lifecycles are explained.
- [ ] Production contracts, entrypoints, dependencies, deployment/configuration, and upgrade paths are inventoried.
- [ ] Critical assets, custody locations, transfer paths, and exit/emergency paths are traced.
- [ ] Roles, indirect authorities, trust assumptions, and maximum impacts are documented.
- [ ] Business workflows, state machines, cross-function combinations, and failure paths are reviewed.
- [ ] Key invariants are independently derived and mapped to state-changing operations.
- [ ] Asset/share/debt/reward/fee/reserve accounting is reconciled.
- [ ] Precision, decimals, rounding direction, dust, and boundary behavior are checked.
- [ ] Economic incentives, attacker cost/profit, large capital, and cross-protocol composition are considered.
- [ ] Oracle sources, manipulation, freshness, units, fallback, and downstream effects are reviewed.
- [ ] External integrations, callbacks, failure behavior, pause, and upgrade assumptions are reviewed.
- [ ] Access control, role transitions, initialization, and indirect bypasses are reviewed.
- [ ] Upgrade authorization, implementation initialization, migrations, and storage layout are reviewed where applicable.
- [ ] Signature binding, nonce/deadline/domain, and replay protections are reviewed where applicable.
- [ ] Single-, cross-function, cross-contract, and read-only reentrancy are considered.
- [ ] Flash liquidity and transaction-ordering/MEV effects are considered.
- [ ] Supported token behaviors and compatibility assumptions are reviewed from source.
- [ ] Low-level calls, assembly, delegatecall, storage, memory, and returndata handling are reviewed where present.
- [ ] DoS/griefing cost, scope, duration, and recovery are analyzed.
- [ ] Pause, emergency, timelock, multisig, monitoring, and recovery assumptions are reviewed.
- [ ] Every formal finding has affected code, root cause, path, impact, and remediation recommendations.
- [ ] Every High/Critical finding has a code-grounded reachable path and concrete impact.
- [ ] Conclusions are identified as static analysis and do not imply local validation.
- [ ] Exclusions, unavailable evidence, unresolved economic risks, and unverified trust/design assumptions are recorded.

## Prohibited Shortcuts

Do not:

- infer safety from `nonReentrant`, a known library, or passing tests;
- review one function while ignoring reachable cross-function/cross-contract behavior;
- ignore funds, accounting, economics, rounding, admin power, or external dependencies;
- assume standard token behavior, correct oracle output, honest callbacks, or permanently available integrations;
- assume an administrator is honest unless that is an explicit trust assumption;
- report an unproven concern as a vulnerability or inflate severity without path and impact;
- lower severity solely because flash liquidity or a complex transaction sequence is required;
- modify project files, write PoCs, or run local validation;
- claim coverage or execution that did not occur.
