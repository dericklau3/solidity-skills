---
name: develop-secure-contracts
description: "Develop secure smart contracts using OpenZeppelin Contracts libraries. Use when users need to integrate OpenZeppelin library components — including token standards (ERC20, ERC721, ERC1155), access control (Ownable, AccessControl, AccessManager), security primitives (Pausable, ReentrancyGuard), governance (Governor, timelocks), or accounts (multisig, account abstraction) — into existing or new Solidity contracts. Covers pattern discovery from library source, CLI contract generators, and library-first integration. Supports Solidity."
license: AGPL-3.0-only
metadata:
  author: OpenZeppelin
---

# Develop Secure Smart Contracts with OpenZeppelin

## Core Workflow

### Understand the Request Before Responding

For conceptual questions ("How does Ownable work?"), explain without generating code. For implementation requests, proceed with the workflow below.

### CRITICAL: Always Read the Project First

Before generating code or suggesting changes:

1. **Search the user's project** for existing Solidity contracts (`Glob` for `**/*.sol`)
2. **Read the relevant contract files** to understand what already exists
3. **Default to integration, not replacement** — when users say "add pausability" or "make it upgradeable", they mean modify their existing code, not generate something new. Only replace if explicitly requested ("start fresh", "replace this").

If a file cannot be read, surface the failure explicitly — report the path attempted and the reason. Ask whether the path is correct. Never silently fall back to a generic response as if the file does not exist.

### Fundamental Rule: Prefer Library Components Over Custom Code

Before writing ANY logic, search the OpenZeppelin library for an existing component:

1. **Exact match exists?** Import and use it directly — inherit or compose with it. Done.
2. **Close match exists?** Import and extend it — override only functions the library marks as overridable (virtual, hooks, configurable parameters).
3. **No match exists?** Only then write custom logic. Confirm by browsing the library's directory structure first.

**NEVER copy or embed library source code into the user's contract.** Always import from the dependency so the project receives security updates. Never hand-write what the library already provides:

- Never write a custom `paused` modifier when `Pausable` or `ERC20Pausable` exists
- Never write `require(msg.sender == owner)` when `Ownable` exists
- Never implement ERC165 logic when the library's base contracts already handle it


### CRITICAL: Validation Style — Prefer `require` with Custom Errors

For ordinary precondition checks, validation, access/state guards, and invariant-style input checks, **prefer `require(condition, Errors.Xxx(...))`** over `if (!condition) revert Errors.Xxx(...);`.

Use a centralized `Errors` namespace/library when the project already has one. For new project-specific errors, prefer defining or reusing them in a shared `Errors.sol` rather than scattering error declarations across contracts, unless the existing project has a different explicit convention.

**Preferred:**

```solidity
require(account != address(0), Errors.ZeroAddress());
require(amount > 0, Errors.InvalidAmount());
require(balance >= amount, Errors.InsufficientBalance(balance, amount));
```

**Avoid for simple validation:**

```solidity
if (account == address(0)) revert Errors.ZeroAddress();
if (amount == 0) revert Errors.InvalidAmount();
if (balance < amount) revert Errors.InsufficientBalance(balance, amount);
```

Rules:

1. Write the condition in the **success form**: `require(validCondition, Errors.Xxx())`.
2. Reuse the project's existing `Errors.Xxx(...)` custom errors before creating new ones.
3. Do not replace custom errors with revert strings such as `require(x, "INVALID")` unless the user explicitly requests revert strings.
4. Do not use `if (...) revert ...` merely as a stylistic alternative to a simple `require` guard.
5. `if (...) revert ...` is acceptable only when the revert belongs to genuinely branch-dependent control flow that cannot be expressed cleanly as a simple precondition, or when the user explicitly requests the `if/revert` form.
6. Remember that `require` arguments are evaluated unconditionally. Do not put calls with side effects or unnecessarily expensive computations inside `Errors.Xxx(...)` arguments.
7. When editing an existing function, convert newly touched simple `if (!condition) revert Errors.Xxx();` guards to the preferred `require(condition, Errors.Xxx());` form, while keeping the change focused on the requested area.

### Methodology

The primary workflow is **pattern discovery from library source code**:

1. Inspect what the user's project already imports
2. Read the dependency source and docs in the project's installed packages
3. Identify what functions, modifiers, hooks, and storage the dependency requires
4. Apply those requirements to the user's contract

See [Pattern Discovery and Integration](#pattern-discovery-and-integration) below for the full step-by-step procedure.

### CLI Generators as Reference

Use `npx @openzeppelin/contracts-cli` to generate reference implementations for pattern discovery:
generate a baseline to a file, generate with a feature enabled to another file, diff them, and apply the changes to the user's code. The CLI output is the canonical correct integration — use it as the source of truth for what imports, inheritance, storage, and overrides a feature requires.

See [CLI Generators](#cli-generators) for details on the generate-compare-apply workflow.

If no CLI command exists for what's needed, use the generic pattern discovery methodology from [Pattern Discovery and Integration](#pattern-discovery-and-integration). The absence of a CLI command does not mean the library lacks support — it only means there is no generator.

## Pattern Discovery and Integration

Procedural guide for discovering and applying OpenZeppelin contract integration patterns
by reading dependency source code for Solidity projects.

**Prerequisite:** Always follow the library-first decision tree above
(prefer library components over custom code, never copy/embed source).

### Step 1: Identify Dependencies and Search the Library

1. Search the project for Solidity contract files: `Glob` for `**/*.sol`.
2. Read import statements in existing contracts to identify which OpenZeppelin components
   are already in use.
3. Locate the installed dependency in the project's dependency tree:
   - Hardhat/npm: `node_modules/@openzeppelin/contracts/`
   - Foundry/forge: `lib/openzeppelin-contracts/`
4. Browse the dependency's directory listing to discover available components. Use `Glob`
   patterns against the installed source (e.g., `node_modules/@openzeppelin/contracts/**/*.sol`).
   Do not assume knowledge of the library's contents — always verify by listing directories.
5. If the dependency is not installed locally, clone or browse the canonical OpenZeppelin Contracts repository.

### Step 2: Read the Dependency Source and Documentation

1. Read the source file of the component relevant to the user's request.
2. Look for documentation within the source: Solidity NatSpec comments (`///`, `/** */`) and README files in the component's directory.
3. Determine the integration strategy using the decision tree from the Critical Principle:
   - If the component satisfies the need directly → import and use as-is.
   - If customization is needed → identify extension points the library provides (virtual
     functions, hook functions, configurable constructor parameters). Import and extend.
   - Only if no component covers the need → write custom logic.
4. Identify the **public API**: functions exposed, events emitted, errors defined.
5. Identify **integration requirements** — this is the critical step:
   - Functions the integrator MUST implement (abstract functions, overrides, hooks)
   - Modifiers or guards that must be applied to the integrator's functions
   - Constructor or initializer parameters that must be passed
   - Storage variables or state that must be declared
   - Inheritance required (always via import, never via copy)
6. Search for example contracts or tests in the same repository that demonstrate correct
   usage. Look in `test/`, `tests/`, `examples/`, or `mocks/` directories.

### Step 3: Extract the Minimal Integration Pattern

From Step 2, construct the minimal set of changes needed:

- **Imports** to add
- **Inheritance** to add (always via import from the dependency)
- **Storage** to declare
- **Constructor / initializer** changes (new parameters, initialization calls)
- **New functions** to add (required overrides, hooks, public API)
- **Existing functions to modify** (add modifiers, call hooks, emit events)

If the contract is upgradeable, any of the above may affect storage compatibility. Consult the relevant upgrade skill before applying.

Do not include anything beyond what the dependency requires. This is the minimal diff
between "contract without the feature" and "contract with the feature."

### Step 4: Apply Patterns to the User's Contract

1. Read the user's existing contract file.
2. Apply the changes from Step 3 using the `Edit` tool. Do not replace the entire file —
   integrate into existing code.
3. Check for conflicts: duplicate access control systems, conflicting function overrides,
   incompatible inheritance. Resolve before finishing.
4. Enforce the validation style above in touched code: prefer `require(validCondition, Errors.Xxx(...))`
   for simple guards.
5. Do not ask the user to make changes themselves — apply directly.

### Repository and Documentation Lookup

| Ecosystem | Repository                                                   | Documentation                                                | File Extension | Dependency Location                                          |
| --------- | ------------------------------------------------------------ | ------------------------------------------------------------ | -------------- | ------------------------------------------------------------ |
| Solidity  | [openzeppelin-contracts](https://github.com/OpenZeppelin/openzeppelin-contracts) | [docs.openzeppelin.com/contracts](https://docs.openzeppelin.com/contracts) | `.sol`         | `node_modules/@openzeppelin/contracts/` or `lib/openzeppelin-contracts/` |

### Directory Structure Conventions

Where to find components within the OpenZeppelin Contracts repository:

| Category             | Solidity                                  |
| -------------------- | ----------------------------------------- |
| Tokens               | `contracts/token/{ERC20,ERC721,ERC1155}/` |
| Access control       | `contracts/access/`                       |
| Governance           | `contracts/governance/`                   |
| Proxies / Upgrades   | `contracts/proxy/`                        |
| Utilities / Security | `contracts/utils/`                        |
| Accounts             | `contracts/account/`                      |

Browse these paths first when searching for a component.

### Known Version-Specific Considerations

Do not assume override points from prior knowledge — always verify by reading the installed source. Functions that were `virtual` in an older version may no longer be in the current one, making them non-overridable. The source NatSpec will indicate the correct override point (e.g., `NOTE: This function is not virtual, {X} should be overridden instead`).

A known example: the Solidity ERC-20 transfer hook changed between v4 and v5. Read the installed `ERC20.sol` to confirm which function is `virtual` before recommending an override.

## CLI Generators

The `@openzeppelin/contracts-cli` package generates reference OpenZeppelin contract implementations from the command line. Use it as the reference source in the generate-compare-apply workflow whenever a command exists for the Solidity contract type.

### Discovering Commands and Options

Run `npx @openzeppelin/contracts-cli --help` to list available commands. For a Solidity contract command (e.g., `solidity-erc20`), run `npx @openzeppelin/contracts-cli <command> --help` to see the available options. Do not rely on prior knowledge of what options exist; check `--help` at the start of a conversation since the CLI may have been updated.

### Generate-Compare-Apply Shortcut

When a CLI command exists for the contract type, pipe generated output to temporary files and diff them to keep generated contract code out of the conversation context:

1. **Generate baseline** — run with only required options, all features disabled, pipe to a file:

   ```bash
   npx @openzeppelin/contracts-cli solidity-erc20 --name MyToken --symbol MTK > /tmp/oz-baseline.sol
   ```

2. **Generate with feature** — run again with the feature enabled, pipe to a second file:

   ```bash
   npx @openzeppelin/contracts-cli solidity-erc20 --name MyToken --symbol MTK --pausable > /tmp/oz-variant.sol
   ```

3. **Compare** — diff the two files to identify exactly what changed (imports, inheritance, state, constructor, functions, modifiers):

   ```bash
   diff /tmp/oz-baseline.sol /tmp/oz-variant.sol
   ```

4. **Apply** — edit the user's existing contract to add the discovered changes

For interacting features (e.g., access control + upgradeability), generate a combined variant as well.

### When No CLI Command Exists or a Feature Is Not Covered

The absence of a CLI command does NOT mean the library lacks support. It only means there is no generator for that contract type. Always fall back to the generic pattern discovery methodology in [Pattern Discovery and Integration](#pattern-discovery-and-integration).

Similarly, when a CLI command exists but does not expose an option for a specific feature, do not stop there. Fall back to pattern discovery for that feature: read the installed library source to find the relevant component, extract the integration requirements, and apply them to the user's contract.