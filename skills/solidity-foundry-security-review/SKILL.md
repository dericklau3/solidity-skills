---
name: solidity-foundry-security-review
description: "Review Solidity contracts in Foundry projects for security issues. Use when users want file-grounded audit findings after feature work, before merge, during contract hardening, or when reviewing vaults, token integrations, upgradeable systems, oracle-dependent logic, and low-level call surfaces. Covers project-first reading, exploit-path analysis, accounting correctness, integration risk assessment, and Forge-oriented test guidance."
license: AGPL-3.0-only
metadata:
  author: derick
---

# Review Solidity Contracts in Foundry Projects

## Core Workflow

### Understand the Request Before Responding

For conceptual questions, explain the security concept without pretending to have audited code. For review requests, follow the workflow below and read the project before making any claim.

### CRITICAL: Always Read the Project First

Before reviewing security or suggesting fixes:

1. Read `README.md`
2. Read `foundry.toml`
3. Search the project for relevant contract files, usually `src/**/*.sol`
4. Read the target contracts the user mentioned, or identify the most relevant contracts if no target was specified
5. Read adjacent dependencies that affect behavior:
   - imported interfaces and libraries
   - proxy / implementation pairs
   - token mocks and callback receivers
   - tests that show intended behavior

If a file cannot be read, surface the failure explicitly. Report the path attempted and state that the review is incomplete. Never silently fall back to a generic answer.

### Fundamental Rule: Review the User's Code, Not an Imaginary Contract

Default to reviewing the user's existing contracts, not generating a generic checklist. When users ask to "check security", they usually want findings grounded in their actual files, functions, state variables, and call paths.

Every finding must be:

- file-grounded
- severity-ranked
- exploit-oriented
- fix-oriented
- testable in Forge

If no material issue is found, state that explicitly and still report residual risks and test gaps.

## Methodology

### Step 1: Establish Review Scope

Identify:

- which files are in scope
- which contracts move funds
- which contracts rely on pricing, signatures, callbacks, or upgradeability
- which external protocols, tokens, or oracles influence safety assumptions

Do not review a contract in isolation when its risk depends on neighboring contracts.

### Step 2: Review by Security Lens

Use the relevant lenses below based on contract behavior. Do not dump every category blindly; focus on what the code actually does.

- access control and trust boundaries
- reentrancy and callback surfaces
- proxy, delegatecall, initializer, and storage layout hazards
- signature, permit, replay, `ecrecover`, and input validation pitfalls
- math, accounting, rounding, share pricing, and state transition correctness
- token and NFT integration risks
- oracle, pricing, slippage, deadline, gas price, randomness, and time assumptions
- ETH transfer, refund, forced ETH, selfdestruct, and low-level call hazards
- storage, memory, calldata, delete semantics, transient storage, and on-chain secret misconceptions

### Step 3: Look for Exploit Paths, Not Just Code Smells

For each suspected issue, answer:

1. What invariant or trust assumption is broken?
2. How would an attacker, privileged role, or ordinary user reach it?
3. What is the practical impact?
4. What is the smallest safe remediation?
5. What Forge test would prove the fix?

If you cannot describe a plausible exploit path or failure path, lower confidence and say so explicitly.

### Step 4: Prefer Proven Components and Patterns

When recommending fixes, prefer established OpenZeppelin components or proven design patterns over handwritten ad hoc logic.

Examples:

- prefer `ReentrancyGuard` over custom boolean locks unless the project already uses a different vetted guard pattern
- prefer established access control patterns over scattered manual `require(msg.sender == ...)`
- prefer `SafeERC20` over raw ERC20 transfers when token behavior may vary

Do not copy external library source into the user's contract. Recommend integration, not source embedding.

## Foundry Review Guidance

- Prefer `src/**/*.sol` and adjacent tests over abstract reasoning
- When suggesting verification, use Forge-oriented commands such as:
  - `forge test --contracts <path> -vvvv`
  - `forge test --match-contract <ContractName> -vvvv`
  - `forge test --match-test <TestName> -vvvv`
- When a finding affects accounting, access control, callback behavior, or upgradeability, propose:
  - at least one regression test
  - at least one edge-case test
- When a contract depends on forks, oracles, or external protocols, check whether safety assumptions depend on:
  - fork block height
  - oracle freshness
  - cross-protocol state transitions

## Review Lenses

### Vault / Pool / Staking / Share-Based Contracts

Check for:

- first deposit or share inflation
- donate-to-pool share skew
- fee-on-transfer incompatibility
- self-transfer or balance aliasing
- rounding to zero and precision loss
- manipulable price reads
- rescue or recovery backdoors

### Token / NFT Integration Contracts

Check for:

- non-standard ERC20 return values
- tokens that return `false` without revert
- invalid `permit` assumptions
- ERC777 or ERC721 callback reentrancy
- custom `transferFrom` authorization issues
- exposed metadata or mint boundary issues

### Upgradeable / Proxy / Low-Level Call Contracts

Check for:

- exposed initializers
- uninitialized implementations
- storage collisions
- arbitrary `delegatecall`
- assembly backdoors
- unsafe upgrade authorization
- implementation slot misuse

## Output Contract

Default review output should follow this structure:

### 1. Scope

- reviewed files
- key dependencies, interfaces, and external protocols
- trust boundary assumptions

### 2. Findings

Order findings by severity: `Critical`, `High`, `Medium`, `Low`.

For each finding, include:

- Severity
- Affected code
- Issue
- Exploit path or failure path
- Impact
- Remediation
- Suggested Forge test

### 3. No Material Issues Case

If no material issue is found, say so explicitly. Then include:

- residual risks
- uncovered surfaces
- missing tests or invariants

## Response Style

- Be concise and direct
- Do not turn the response into a generic security encyclopedia
- Do not ask the user to inspect files you can inspect yourself
- Do not hide uncertainty; state assumptions clearly
- If code looks acceptable but the test surface is weak, call out the testing gap
