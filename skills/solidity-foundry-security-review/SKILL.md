---
name: solidity-foundry-security-review
description: "Review Solidity contracts in Foundry projects for security and business-logic issues. Use when users want file-grounded audit findings after feature work, before merge, during hardening, or when reviewing accounting, value flows, state machines, permissions, integrations, upgradeability, pricing, callbacks, or low-level calls. Requires project-first reading, business model reconstruction, invariant and exploit-path analysis, and Forge-oriented test guidance."
license: AGPL-3.0-only
metadata:
  author: derick
---

# Review Solidity Contracts in Foundry Projects

## Core Workflow

### Understand the Request Before Responding

For conceptual questions, explain the security concept without pretending to have audited code. For review requests, proceed with the workflow below.

### CRITICAL: Always Read the Project First

Before reporting findings or suggesting fixes:

1. **Search the user's project** for contracts, tests, scripts, interfaces, libraries, mocks, and docs.
2. **Read the relevant files** before making claims. Include neighboring contracts when behavior crosses file boundaries.
3. **Default to reviewing the user's existing system, not an imagined generic contract**. When users say "review security", they usually want findings grounded in their actual business logic and call paths.

If a file cannot be read, surface the failure explicitly. Report the path attempted and why the review is incomplete. Never silently fall back to a generic checklist.

### Fundamental Rule: Reconstruct the Business Logic Before Looking for Bugs

Most serious Solidity bugs are violations of the protocol's intended rules, not isolated syntax hazards.

Before classifying a suspected issue, establish:

1. **Actors** - who can call each meaningful path, and under what assumptions.
2. **Assets and value flows** - what moves, what is minted, burned, locked, paid, credited, or accounted.
3. **State machine** - which states exist, how they transition, and which transitions should be impossible.
4. **Accounting model** - what totals, balances, shares, debts, rewards, limits, or rates must stay consistent.
5. **Trust assumptions** - which dependencies, operators, inputs, time assumptions, or off-chain facts the system relies on.

Do not rely only on a vulnerability taxonomy. A finding should explain which intended business rule, invariant, or trust assumption breaks.

### Methodology

The primary workflow is **business-model-driven exploit analysis**:

1. Inspect project files and understand intended behavior.
2. Build a compact model of actors, assets, states, and invariants.
3. Trace realistic success and failure paths through public entrypoints.
4. Check generic Solidity risks only after mapping them to the project's actual behavior.
5. Report only file-grounded findings with exploit or failure paths and Forge test ideas.

See [Business Model and Exploit Analysis](#business-model-and-exploit-analysis) for the full procedure.

## Business Model and Exploit Analysis

Procedural guide for reviewing Foundry projects without reducing the task to a generic checklist.

**Prerequisite:** Always reconstruct the business model first.

### Step 1: Establish Scope and Intent

1. Read `README.md`, `foundry.toml`, and project docs when they exist.
2. Search `src/`, `test/`, `script/`, and relevant interfaces or libraries.
3. Identify the contracts in scope and the neighboring contracts required to understand them.
4. Read tests to infer expected behavior, but do not assume tests are complete or correct.
5. Note any missing context that prevents a complete review.

### Step 2: Build the Protocol Model

Before hunting for findings, write down the model you are reviewing:

- External actors and privileged actors
- User-facing actions and operator actions
- Assets, balances, credits, debts, limits, and other accounting variables
- State transitions and terminal states
- External integrations and their assumptions
- Time, price, randomness, signature, or configuration assumptions

Use this model to decide what "correct" means. Business-logic bugs are usually mismatches between this model and the implementation.

### Step 3: Derive Invariants and Failure Conditions

Convert the model into properties the code should preserve.

Examples of useful property types:

- A user cannot receive more value than they are entitled to.
- Total accounting cannot drift from actual assets or recorded obligations.
- State transitions cannot skip required prerequisites.
- Privileged actions cannot violate user guarantees unless explicitly designed.
- External inputs cannot make stale, circular, or inconsistent values look valid.

These are examples, not a checklist. Derive properties from the project's own rules.

### Step 4: Trace Exploit Paths Through Public Entrypoints

For each suspected issue, answer:

1. Which invariant, business rule, or trust assumption breaks?
2. Which actor can reach the path using normal entrypoints?
3. What sequence of calls or state changes triggers the issue?
4. What is the impact in assets, permissions, accounting drift, denial of service, or broken user guarantees?
5. What is the smallest safe remediation?
6. What Forge test would prove the bug and the fix?

If you cannot describe a plausible exploit or failure path, do not present the item as a strong finding. Lower confidence or move it to residual risk.

### Step 5: Apply Solidity Security Lenses

After the business model is clear, apply only the lenses relevant to the code:

- Access control and trust boundaries
- Reentrancy and external-call ordering
- Upgradeability, delegatecall, initialization, and storage layout
- Signature validation, replay protection, and authorization
- Arithmetic, rounding, accounting, and state transition correctness
- Token and NFT integration behavior
- Pricing, freshness, slippage, randomness, and time assumptions
- Native asset transfers, refunds, forced value, and low-level calls
- Storage, memory, calldata, delete semantics, and on-chain secret misconceptions

Do not dump all categories into the final answer. Use them to find issues, then report only code-grounded results.

### Step 6: Recommend Minimal Fixes and Forge Tests

For each finding:

- Prefer proven libraries or established patterns over custom logic when applicable.
- Do not copy external library source into the user's contract.
- Suggest the smallest fix that restores the broken invariant.
- Suggest at least one Forge regression test that fails before the fix and passes after it.
- Add edge-case or invariant test guidance when the bug is accounting, sequencing, or state-machine related.

## Foundry Review Guidance

Prefer project files over abstract reasoning. Useful verification commands include:

- `forge test -vvvv`
- `forge test --match-contract <ContractName> -vvvv`
- `forge test --match-test <TestName> -vvvv`

When the project depends on forked state or external systems, call out any missing RPC, fork block, environment variable, or dependency that limits review confidence.

## Output Contract

Default review output should follow this structure:

### 1. Scope and Model

- Reviewed files
- Relevant dependencies and neighboring contracts
- Actors, assets, major flows, and key assumptions
- Invariants or business rules used during review

### 2. Findings

Order findings by severity: `Critical`, `High`, `Medium`, `Low`.

For each finding, include:

- Severity
- Affected code
- Broken invariant or business rule
- Exploit path or failure path
- Impact
- Remediation
- Suggested Forge test
- Confidence when useful

### 3. No Material Issues Case

If no material issue is found, say so explicitly. Then include:

- Residual risks
- Uncovered surfaces
- Business rules or invariants that still need tests

## Response Style

- Be concise and direct
- Lead with findings, not background
- Do not turn the response into a generic security encyclopedia
- Do not ask the user to inspect files you can inspect yourself
- State uncertainty and assumptions clearly
- If code looks acceptable but the business model or test surface is weak, call that out
