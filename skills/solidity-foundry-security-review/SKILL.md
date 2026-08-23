---
name: solidity-foundry-security-review
description: "Use when reviewing Solidity contracts or Foundry projects for security and protocol-level risks, including business logic, asset flows, accounting, economic attacks, trust boundaries, integrations, upgradeability, pricing, callbacks, signatures, or low-level EVM behavior."
license: AGPL-3.0-only
metadata:
  author: derick
---

# Review Solidity Contracts and Protocols in Foundry Projects

## Core Principle

Do not begin with a vulnerability taxonomy. First reconstruct what the protocol promises, how value moves, who can change state, and which properties must always hold. A material finding is a demonstrated violation of a business rule, invariant, or trust assumption—not merely unusual code.

Priority: **business logic > fund safety > accounting > economic attacks > permissions and trust > implementation hazards**.

## Review Modes

Choose the mode from the user's request and state the effective scope.

### Targeted Security Review

Use when the user names a contract, feature, patch, or risk surface. Review that scope plus every neighboring contract, dependency, test, script, and configuration needed to trace its behavior. Apply all relevant security lenses, but do not silently claim coverage of the rest of the protocol. List unreviewed surfaces and residual risks.

Read [Protocol Audit Methodology](references/protocol-audit-methodology.md) when the target includes asset custody, accounting, economic mechanisms, or any of its specialized surfaces. Use only the sections relevant to the target.

### Full Protocol Audit

Use when the user requests a complete, repository-wide, protocol-wide, exhaustive, or audit-style assessment. Read [Protocol Audit Methodology](references/protocol-audit-methodology.md) completely before analyzing findings. Follow its ordered phases and completion gate. Finding one severe issue does not end the audit.

### Conceptual Question

Explain the concept without implying that code was reviewed. Ask for or inspect the project only when the user requests a code-grounded conclusion.

## Non-Negotiable Review Rules

### Read the Project Before Making Claims

1. Search for contracts, interfaces, libraries, tests, scripts, mocks, deployment configuration, docs, and upgrade tooling.
2. Read the in-scope files and all neighboring components required to understand cross-contract behavior.
3. Treat tests as evidence of intent, not proof of correctness or completeness.
4. If relevant files cannot be read, report the attempted paths and explain the resulting coverage limit. Never substitute a generic checklist silently.

### Verify Installed Dependencies

Before reasoning about OpenZeppelin, Solady, Uniswap, Chainlink, proxy libraries, token standards, or other dependencies:

1. Resolve the installed version through `foundry.toml`, `remappings.txt`, `lib/`, `node_modules/`, or package configuration.
2. Read the exact imported source, including inherited hooks, modifiers, constructors, initializers, storage, and documented extension points.
3. Do not rely on a remembered API or assume a well-known library makes the integration safe.
4. Prefer importing, inheriting, composing, or configuring proven components when recommending a fix. Never copy external library source into the user's contract.
5. Treat proxies, initializers, namespaced storage, storage gaps, `delegatecall`, and upgrade scripts as storage-layout and authorization scope.

### Use a Capable Attacker Model

Unless the protocol explicitly prevents it, assume an attacker can:

- create many accounts and deploy arbitrary contracts;
- control callbacks, return values, and intentional reverts from attacker-owned code;
- use flash liquidity, MEV, transaction ordering, and cross-protocol composition;
- transfer tokens or force native value directly to protocol addresses;
- choose extreme inputs, wait for favorable chain state, and combine valid operations in one transaction;
- introduce unusual but permitted token behavior.

Do not reduce the attacker to an honest EOA following the happy path.

## Core Workflow

### 1. Establish Scope and Intent

- Read `README.md`, `foundry.toml`, docs, target contracts, tests, scripts, and configuration.
- Identify the contracts in scope and the adjacent components required for complete call paths.
- Record exclusions, unavailable dependencies, missing RPC/fork state, and unverified design assumptions.
- For a full audit, inventory all production contracts and externally reachable entrypoints.

### 2. Build the Protocol Model

Write a compact working model before hunting for bugs:

- protocol goal and architecture;
- production entrypoint inventory and the primary lifecycle each entrypoint belongs to;
- unprivileged and privileged actors;
- asset custody and end-to-end value flows;
- state machines and valid/invalid transitions;
- accounting authority for assets, shares, debts, rewards, fees, and reserves;
- price, time, signature, randomness, configuration, and off-chain assumptions;
- external integrations and failure behavior;
- role-to-capability-to-maximum-impact trust matrix.

You must be able to explain the primary entrypoints, where assets are held, which component controls accounting and pricing, and which components can move value or change critical state. If you cannot, continue modeling before producing findings.

Keep the model operational rather than narrative-only. For any non-trivial review, maintain concise working artifacts as needed:

- asset-flow notes covering source, custodian, accounting source of truth, conversion formula/unit, normal exit, emergency exit, privileged destination, and unsupported direct-transfer behavior;
- a role-to-capability-to-maximum-impact matrix, including indirect authorities and dangerous role combinations;
- an invariant map from property → state-changing functions/callbacks/privileged actions/external state that can affect it;
- an external dependency map covering the value or data relied on, who can influence it, failure behavior, pause/upgrade assumptions, and whether users can still exit;
- a candidate ledger that distinguishes confirmed findings, rejected theories, unresolved risks, missing evidence, and unreviewed surfaces.

### 3. Derive Invariants and Failure Conditions

Derive properties from the protocol's own rules rather than copying generic examples. Cover as applicable:

- conservation of assets and obligations;
- user ownership and withdrawal entitlement;
- solvency and collateralization;
- share/asset, debt, reward, reserve, and fee consistency;
- state-transition prerequisites and terminal states;
- privilege boundaries and user guarantees;
- price freshness, unit consistency, and manipulation resistance.

For each invariant, identify every operation and external state change that can affect it.

### 4. Trace End-to-End and Composed Paths

Trace public entrypoints through internal functions, neighboring contracts, callbacks, and external protocols until the final state and value transfer. Examine:

- normal lifecycles such as deposit → mint → earn → withdraw;
- reordered, repeated, interrupted, and cross-function sequences;
- multi-user interactions and transferable positions/shares;
- external-call intermediate states and read-only observations;
- direct asset transfers, donations, forced value, and stale external state;
- failure paths, pause paths, recovery paths, and partial updates.

Always ask whether a sequence of individually valid operations creates an unintended result.

### 5. Apply the Relevant Security Lenses

After the model is clear, inspect the applicable surfaces in [Protocol Audit Methodology](references/protocol-audit-methodology.md):

- asset flow, business logic, invariants, accounting, precision, and rounding;
- economic model, oracle manipulation, flash liquidity, and MEV;
- roles, access control, signatures, upgrades, and operational controls;
- reentrancy, callbacks, external protocols, and token compatibility;
- low-level EVM behavior, assembly, denial of service, and griefing.

For a full audit, cover every methodology section and mark non-applicable sections with a reason. For a targeted review, cover every lens reachable from the selected scope and disclose the rest as unreviewed.

When a request is ambiguous, do not silently expand a narrow target into a full protocol audit. State the effective scope you inferred, review the reachable protocol context deeply, and list the protocol-level sections that would still require a full audit.

### 6. Validate Candidates

A formal finding requires all of:

```text
root cause + reachable attack/failure path + concrete impact
```

For each candidate:

1. Identify the broken invariant, business rule, or trust assumption.
2. Identify the actor, prerequisites, required privilege, capital, and external conditions.
3. Trace the exact calls and state changes.
4. Quantify affected assets, users, accounting drift, liveness loss, or privilege impact where possible.
5. Challenge the candidate against guards, actual dependency behavior, transaction atomicity, and protocol assumptions.
6. Build the smallest practical Forge PoC, numerical example, or state-transition proof for material findings.

If the path or impact cannot be established, lower confidence, classify it as unresolved risk, or discard it. Do not turn style issues or hypothetical discomfort into vulnerabilities.

### 7. Recommend Minimal Fixes and Tests

- Restore the broken invariant with the smallest safe change.
- Specify what to check, where to check it, and which security property the change restores.
- Identify required imports, inheritance/composition, overrides, initializers, and storage-layout effects when a library component is appropriate.
- Suggest or implement a Forge regression test that fails before the fix and passes after it.
- Use fuzz tests for mathematical boundaries and input combinations.
- Use stateful invariant tests for multi-user, multi-function, long-sequence accounting and state-machine properties.
- Do not alter intended protocol behavior merely to make a fuzz or invariant test pass.

Never claim that a test, command, PoC, fork, or tool was executed unless it actually ran successfully. Report missing RPC, fork block, environment, compiler, or dependency constraints.

## Finding Standard

Order findings by `Critical`, `High`, `Medium`, `Low`, then `Informational`. Calibrate severity using impact, likelihood, attack cost, required privilege, affected assets/users, and exploit complexity.

Each finding must include:

1. **Title** — root cause and consequence in one sentence.
2. **Severity and confidence** — with the decisive factors when not obvious.
3. **Affected code** — precise files, lines, functions, and relevant dependencies.
4. **Broken invariant or trust assumption**.
5. **Root cause** — the code, design, accounting, formula, permission, state update, or integration error.
6. **Attack path or failure scenario** — ordered prerequisites, calls, state changes, and result.
7. **Impact** — attacker gain, user/protocol loss, accounting corruption, privilege escalation, or liveness failure.
8. **Proof** — minimal Forge PoC, concrete values, state trace, or mathematical reasoning when warranted.
9. **Recommendation** — an executable minimal remediation and the property it restores.
10. **Suggested regression test**.

High and Critical findings require a verifiable path or PoC. Do not lower severity merely because an attack uses flash liquidity or is operationally complex if realistic impact remains severe.

## Output Contract

Lead with material findings. Then provide enough model and coverage information to make the conclusions auditable.

### Targeted Review Output

1. Findings ordered by severity.
2. Scope and reviewed files.
3. Relevant protocol model, asset flows, trust assumptions, and invariants.
4. Tests/PoCs actually performed and their results.
5. Residual risks, unavailable evidence, and unreviewed surfaces.

### Full Protocol Audit Output

1. Executive summary and overall risk.
2. Protocol overview and architecture.
3. Asset flow and accounting model.
4. Roles, trust assumptions, and maximum impact.
5. Key invariants.
6. Attack surface and methodology coverage.
7. Findings ordered by severity.
8. Unit, fuzz, invariant, fork, and PoC testing actually performed.
9. Unresolved risks, exclusions, and unverifiable assumptions.

If no material issue is found, say so explicitly; do not imply the absence of all risk. Include residual risks, uncovered surfaces, and the invariants that still need stronger tests.

## Completion Gate

Before calling a targeted review complete, confirm that the selected scope, all reachable value/call paths, applicable security lenses, candidate validation, and coverage limits are addressed.

Before calling a full protocol audit complete, use the completion checklist in [Protocol Audit Methodology](references/protocol-audit-methodology.md). Every phase must be completed or explicitly marked blocked/non-applicable with evidence. A severe early finding is not a stopping condition.

## Response Style

- Be concise, skeptical, and file-grounded.
- Lead with findings rather than generic background.
- Do not ask the user to inspect files you can inspect yourself.
- Separate confirmed findings, unresolved risks, and design/trust assumptions.
- State uncertainty, scope limitations, and unexecuted tests clearly.
- Do not dump a generic vulnerability encyclopedia into the final answer.
