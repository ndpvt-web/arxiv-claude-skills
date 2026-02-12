---
name: "solagent-specialized-multi-agent-framework"
description: "Generate secure, functionally correct Solidity smart contracts using a dual-loop refinement process: an inner loop that compiles and tests with Forge until all tests pass, and an outer loop that runs Slither static analysis to eliminate security vulnerabilities. Triggers: 'write a Solidity contract', 'generate smart contract', 'secure Solidity code', 'fix smart contract vulnerabilities', 'audit and fix Solidity', 'create ERC-20/ERC-721 contract with tests'"
---

# SolAgent: Dual-Loop Refinement for Secure Solidity Code Generation

This skill enables Claude to generate production-grade Solidity smart contracts through a structured dual-loop refinement workflow inspired by the SolAgent paper. Instead of producing a single-pass draft and hoping it works, Claude iteratively refines contracts by (1) compiling and running Foundry/Forge tests to ensure functional correctness, then (2) running Slither static analysis to detect and eliminate security vulnerabilities — mirroring how expert Solidity developers actually work.

## When to Use

- When the user asks to write a new Solidity smart contract (ERC-20, ERC-721, DeFi protocols, governance, etc.)
- When the user provides a failing Solidity contract and wants it fixed to pass its test suite
- When the user wants to audit existing Solidity code for security vulnerabilities and auto-remediate them
- When the user needs to generate a contract that interacts with existing on-chain dependencies (interfaces, libraries, inherited contracts)
- When the user asks to "make this contract secure" or "harden this smart contract"
- When the user wants to scaffold a Foundry project with contract + tests and iterate until everything passes

## Key Technique: Dual-Loop Refinement with Tool Feedback

The core insight from SolAgent is that smart contract generation is an **impossible triangle** for single-pass LLMs: you cannot simultaneously satisfy functional correctness, security compliance, and complex dependency resolution in one shot. The solution is to decompose the problem into two nested feedback loops with concrete tool outputs driving each iteration.

**Inner Loop (Correctness via Forge):** After generating an initial contract, compile it with `forge build` and run its test suite with `forge test`. Parse the output for compilation errors, assertion failures, and stack traces. Feed these specific failure messages back into the next refinement pass. Continue until all tests pass or the pass rate stagnates for 2 consecutive rounds.

**Outer Loop (Security via Slither):** Once the contract compiles and passes tests, run `slither . --json -` to perform static analysis. Parse the JSON output for vulnerabilities categorized by severity (high/medium/low). Apply targeted fixes — especially the checks-effects-interactions pattern for reentrancy, proper access control, and safe math — then re-run the inner loop to confirm the security fixes don't break functionality. Terminate when no high/medium findings remain, or when fixes begin oscillating (detected via output similarity > 0.9 between rounds).

**Dependency Resolution:** For contracts that import interfaces or libraries, use file system inspection to read existing project files, understand the dependency graph, and ensure imports resolve correctly before compilation.

## Step-by-Step Workflow

1. **Analyze requirements and project structure.** Read the user's specification. If a Foundry project exists, run `ls` on the project root, `src/`, `test/`, and `lib/` directories to understand existing contracts, interfaces, libraries, and test files. Read `foundry.toml` for compiler settings and remappings.

2. **Read all dependency files.** If the contract inherits from or imports other contracts, read every imported file to understand the interface signatures, storage layouts, events, and modifiers the new contract must conform to. Do not guess at interfaces — read them.

3. **Generate the initial Solidity implementation.** Write the contract following these principles:
   - Use Solidity ^0.8.x with explicit SPDX license identifiers
   - Apply checks-effects-interactions pattern by default (update state before external calls)
   - Include proper access control (Ownable, role-based, or custom modifiers)
   - Emit events for all state changes
   - Add input validation at function entry with descriptive `require`/`revert` messages
   - Handle edge cases (zero address, overflow on older compilers, empty arrays)

4. **Run the inner loop — compile and test with Forge.**
   ```bash
   cd <project_root> && forge build 2>&1
   ```
   If compilation fails, parse errors (missing imports, type mismatches, visibility issues) and fix them. Then:
   ```bash
   forge test -vvv 2>&1
   ```
   Parse test results for failures. For each failing test, identify the assertion, the expected vs. actual values, and the relevant code path. Fix the contract logic and re-run. Repeat until all tests pass or pass rate stagnates for 2 rounds.

5. **Run the outer loop — static analysis with Slither.**
   ```bash
   slither . --json /tmp/slither-output.json 2>&1
   ```
   Parse the JSON for detectors that fired. Prioritize by severity:
   - **High:** reentrancy, unprotected selfdestruct, arbitrary send, delegatecall to untrusted
   - **Medium:** unchecked return values, missing zero-address checks, tx.origin usage
   - **Low:** naming conventions, unused state variables, missing events
   Fix high and medium findings first. For each fix, explain the vulnerability and the remediation.

6. **Re-validate after security fixes.** After applying Slither-driven fixes, re-run `forge test -vvv` to confirm the security changes did not break any tests. If tests regress, iterate the inner loop again before proceeding.

7. **Detect stopping conditions.** Stop iterating when:
   - All tests pass AND no high/medium Slither findings remain (success)
   - Test pass rate has not improved for 2 consecutive rounds (stagnation)
   - Slither output is nearly identical between rounds — similarity > 0.9 (oscillation)
   When stopping due to stagnation or oscillation, return the best version seen so far (highest test pass rate with fewest vulnerabilities).

8. **Track the best version.** Maintain a mental scoreboard: `score = (tests_passed / total_tests) * 100 - (high_vulns * 10 + medium_vulns * 3)`. Always return the highest-scoring version, not necessarily the latest iteration.

9. **Report results to the user.** Summarize: which tests pass, which Slither findings remain (if any), what security patterns were applied, and any known limitations of the generated contract.

## Concrete Examples

**Example 1: ERC-20 Token with Minting and Burning**

User: "Create an ERC-20 token called GoldCoin (GLD) with owner-only minting and public burning. I already have OpenZeppelin in `lib/`. Write the contract and make sure it passes the tests in `test/GoldCoin.t.sol`."

Approach:
1. Read `test/GoldCoin.t.sol` to understand expected function signatures and test assertions
2. Read `lib/openzeppelin-contracts/contracts/token/ERC20/ERC20.sol` and `Ownable.sol` for interfaces
3. Write `src/GoldCoin.sol` inheriting ERC20 and Ownable, implementing `mint(address, uint256)` with `onlyOwner` and `burn(uint256)` as public
4. Run `forge build` — fix any compilation errors
5. Run `forge test -vvv` — parse failures, fix logic (e.g., missing `_mint` call in constructor for initial supply)
6. Run `slither .` — address any findings (e.g., add zero-address check in `mint`)
7. Re-run `forge test -vvv` to confirm security fix didn't break tests

Output:
```solidity
// SPDX-License-Identifier: MIT
pragma solidity ^0.8.20;

import {ERC20} from "@openzeppelin/contracts/token/ERC20/ERC20.sol";
import {Ownable} from "@openzeppelin/contracts/access/Ownable.sol";

contract GoldCoin is ERC20, Ownable {
    constructor(uint256 initialSupply) ERC20("GoldCoin", "GLD") Ownable(msg.sender) {
        _mint(msg.sender, initialSupply);
    }

    function mint(address to, uint256 amount) external onlyOwner {
        require(to != address(0), "GoldCoin: mint to zero address");
        _mint(to, amount);
    }

    function burn(uint256 amount) external {
        _burn(msg.sender, amount);
    }
}
```
Result: 12/12 tests passing. Slither: 0 high, 0 medium findings.

**Example 2: Fixing a Reentrancy Vulnerability in a Withdrawal Contract**

User: "My `Vault.sol` contract has a reentrancy bug. Slither flags `withdraw()` as vulnerable. Fix it."

Approach:
1. Read `src/Vault.sol` — identify that `withdraw()` sends ETH before updating `balances[msg.sender]`
2. Apply checks-effects-interactions: move `balances[msg.sender] = 0` before the `.call{value: ...}` line
3. Add a reentrancy guard (either OpenZeppelin's `ReentrancyGuard` or a manual `locked` modifier)
4. Run `forge test -vvv` to confirm existing tests still pass
5. Run `slither .` to confirm the reentrancy detector no longer fires

Before (vulnerable):
```solidity
function withdraw() external {
    uint256 amount = balances[msg.sender];
    (bool ok, ) = msg.sender.call{value: amount}("");
    require(ok);
    balances[msg.sender] = 0; // STATE UPDATE AFTER EXTERNAL CALL
}
```

After (secure):
```solidity
function withdraw() external nonReentrant {
    uint256 amount = balances[msg.sender];
    require(amount > 0, "Vault: nothing to withdraw");
    balances[msg.sender] = 0; // STATE UPDATE BEFORE EXTERNAL CALL
    (bool ok, ) = msg.sender.call{value: amount}("");
    require(ok, "Vault: transfer failed");
}
```
Result: All tests passing. Slither reentrancy detector no longer fires.

**Example 3: Generating a Contract with Complex Dependencies**

User: "I need a Staking contract that uses our custom `IRewardDistributor` interface and `MathLib` library. The files are somewhere in the project."

Approach:
1. Run `find`/`ls` to locate `IRewardDistributor.sol` and `MathLib.sol` in the project tree
2. Read both files to understand the exact function signatures (`distribute(address,uint256)`, `mulDiv(uint256,uint256,uint256)`)
3. Write `Staking.sol` with correct import paths and proper usage of the interface and library
4. Run `forge build` — if import paths are wrong, read `foundry.toml` remappings and fix
5. Run `forge test -vvv` — iterate on logic until staking/unstaking/reward-claim tests pass
6. Run `slither .` — fix any unchecked return value warnings on `IRewardDistributor.distribute()` calls

## Best Practices

- **Do:** Always read the test file before writing the contract. Tests define the ground truth for expected behavior, function signatures, and edge cases.
- **Do:** Apply checks-effects-interactions by default in every function that makes external calls. Update all state variables before any `.call`, `.transfer`, or `.send`.
- **Do:** Parse Forge and Slither output literally — copy exact error messages and assertion values into your reasoning rather than guessing what went wrong.
- **Do:** Use Foundry remappings (`foundry.toml`) to resolve import paths rather than hardcoding relative paths like `../../lib/...`.
- **Avoid:** Ignoring low-severity Slither findings silently. Acknowledge them to the user even if you choose not to fix them, explaining why (e.g., false positive, acceptable risk).
- **Avoid:** Infinite iteration. Cap at 5-7 rounds total. If the contract isn't converging, report the best version and explain what remains unresolved.
- **Avoid:** Adding `unchecked` blocks to suppress Slither warnings about arithmetic — this trades a false positive for a real overflow risk.

## Error Handling

| Problem | Detection | Resolution |
|---|---|---|
| Forge not installed | `command not found: forge` | Tell user to install Foundry: `curl -L https://foundry.paradigm.xyz \| bash && foundryup` |
| Slither not installed | `command not found: slither` | Tell user to install: `pip install slither-analyzer` |
| Import resolution failure | `Source not found` in forge build | Read `foundry.toml` for remappings; run `forge install` if deps missing |
| Slither crashes on valid code | Non-zero exit with no JSON | Fall back to `slither . --exclude-dependencies` or skip outer loop, warn user |
| Tests pass but Slither flags keep reappearing | Same detector fires after fix | Check if it's a false positive; add `// slither-disable-next-line` with justification comment |
| Oscillating fixes | Fix for security breaks tests, test fix reintroduces vulnerability | Step back, redesign the function's control flow rather than patching symptoms |

## Limitations

- **Requires Foundry and Slither installed locally.** This workflow depends on real tool execution. Without them, Claude can still generate Solidity code but cannot run the feedback loops.
- **Test quality bounds output quality.** The inner loop can only verify what the tests cover. If tests are incomplete or incorrect, the contract may pass all tests while still being buggy.
- **Slither is not exhaustive.** It catches known vulnerability patterns (reentrancy, unchecked calls, etc.) but misses business logic flaws, economic exploits, and novel attack vectors. A formal audit is still necessary for production contracts.
- **Complex cross-contract interactions** (flash loans, callback patterns, proxy upgrades) may require manual reasoning beyond what the dual-loop can catch automatically.
- **Gas optimization is secondary.** The workflow prioritizes correctness and security. Gas-optimal patterns (assembly blocks, storage packing) should be applied after the dual loop converges, not during it.

## Reference

**Paper:** [SolAgent: A Specialized Multi-Agent Framework for Solidity Code Generation](https://arxiv.org/abs/2601.23009v1) (Chen et al., 2026)

Key takeaway: Single-pass LLM generation achieves ~25% Pass@1 on rigorous Solidity benchmarks. Adding a dual-loop refinement with Forge (correctness) and Slither (security) as concrete tool feedback raises this to 64.39% while reducing security vulnerabilities by 39.77% versus human-written code. The critical insight is that tool feedback — not more prompting — is what closes the gap.