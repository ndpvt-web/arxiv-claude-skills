---
name: "an-cost-efficient-agentic-framework"
description: "Audit Ethereum smart contracts for business logic vulnerabilities using Heimdallr's four-phase agentic pipeline: function-level code reorganization via dependency graph clustering, heuristic Plan-Remind-Solve reasoning with adversarial state injection, automatic multi-step exploit chaining, and cascaded false-positive filtration. Trigger phrases: 'audit this smart contract', 'find vulnerabilities in this Solidity code', 'check this DeFi protocol for exploits', 'smart contract security review', 'detect business logic bugs in this contract', 'chain exploit paths in this protocol'."
---

# Heimdallr-Style Smart Contract Auditing

This skill enables Claude to perform structured, multi-phase security audits of Ethereum smart contracts following the Heimdallr framework (Hu et al., 2026). Rather than scanning line-by-line or relying on generic static analysis heuristics, the approach reorganizes contract code into semantically cohesive batches using dependency graph clustering, then applies a Plan-Remind-Solve reasoning loop with adversarial state assumptions, chains individual findings into composite multi-step exploits, and finally filters results through a three-layer cascaded verification to eliminate false positives. This method specifically targets **business logic vulnerabilities** — flaws in protocol-specific economic mechanisms, state transitions, access control workflows, and cross-contract interactions — that traditional tools consistently miss.

## When to Use

- When the user asks to audit a Solidity smart contract or set of contracts for security vulnerabilities
- When reviewing a DeFi protocol (lending, DEX, staking, vault) for economic exploit paths
- When the user wants to find business logic bugs that static analyzers like Slither or Mythril would miss
- When asked to construct a proof-of-concept exploit chain across multiple contract functions
- When triaging audit contest submissions and need to prioritize findings by exploitability
- When verifying whether a reported vulnerability is a true positive or false positive
- When the user pastes Solidity code and asks "is this safe?" or "can this be exploited?"

## Key Technique

**Function-Level Code Reorganization (Contextual Profiling).** Instead of feeding entire contracts into an LLM (which wastes context and loses coherence), Heimdallr builds a static directed dependency graph where nodes are functions and edges represent control flow (direct calls) and data flow (shared state variable access). It then applies the Louvain community detection algorithm to partition this graph into densely connected subgraphs — each representing a logically cohesive unit of business logic. Functions are scored by importance using a weighted combination of betweenness centrality and PageRank: `Score(f) = alpha * Betweenness(f) + beta * PageRank(f)`. High-scoring functions are audited first. This reorganization means the LLM sees related functions together regardless of which `.sol` file they live in, dramatically reducing context noise.

**Plan-Remind-Solve with Adversarial State Injection.** For each batch, the auditor runs a three-step loop: (1) **Plan** — identify which functions look potentially vulnerable and why; (2) **Remind** — retrieve concrete exploit patterns from a knowledge base organized by vulnerability category (reentrancy, oracle manipulation, precision loss, etc.); (3) **Solve** — analyze each candidate through three complementary lenses: basic semantic analysis, adversarial state assumptions (block variable manipulation, malicious calldata, flash-loan unlimited capital), and symbolic constraint solving (modeling arithmetic in Z3 to prove whether invariant violations are reachable at boundary values). Individual findings are then chained: if the postcondition of vulnerability A satisfies the precondition of vulnerability B, they merge into a composite multi-step exploit.

**Cascaded Verification.** Raw findings pass through three sequential filters: (1) Contextual Aggregation — re-evaluate each finding under global contract scope to check if safeguards elsewhere neutralize the risk; (2) Semantic Deduplication — cluster similar findings by embedding similarity, keep only the highest-confidence representative; (3) Threat Model Assessment — discard findings that require unrealistic assumptions like dishonest contract owners or compromised external oracles (unless the user explicitly includes those in scope).

## Step-by-Step Workflow

1. **Collect all contract source code.** Read every `.sol` file the user provides. If the user gives a single file, also ask whether there are imported dependencies or interfaces that affect the logic (e.g., OpenZeppelin, protocol-specific libraries).

2. **Build the dependency graph.** For each contract, enumerate all functions (public, external, internal, private). Map edges for: direct function calls (`functionA` calls `functionB`), shared state variable reads/writes, modifier applications, and inheritance overrides. Represent this as an adjacency list or mental model.

3. **Cluster into semantic batches.** Group functions into cohesive units based on their connectivity. Functions that share state variables or call each other belong together. Prioritize batches containing functions with high centrality (called by many others or sitting on critical paths between entry points and state changes). A typical DeFi protocol yields 3-8 batches: e.g., "deposit/withdraw/accounting", "liquidation/health-check", "governance/parameter-setting", "oracle-integration".

4. **Score and prioritize.** Within each batch, rank functions by importance. External/public functions callable by anyone rank highest. Functions that modify balances, ownership, or protocol parameters rank above pure view functions. Start the audit with the highest-priority batch.

5. **Plan — Identify suspicious patterns.** For the current batch, read each function and flag candidates that exhibit: unchecked external calls, arithmetic on user-supplied values without overflow/underflow guards, state reads before writes (potential reentrancy), missing access control, reliance on `block.timestamp` or `block.number`, token transfer patterns without return-value checks, or price calculations using spot reserves.

6. **Remind — Match against known vulnerability patterns.** For each flagged candidate, explicitly recall the relevant exploit pattern:
   - **Reentrancy**: state update after external call; cross-function reentrancy via shared state
   - **Oracle manipulation**: spot price used as oracle; TWAP with insufficient window
   - **Precision loss cascade**: repeated `mulDiv` or `mulDown` operations accumulating rounding errors
   - **Flash loan attacks**: no same-block restrictions on deposit+borrow+withdraw
   - **Access control**: missing `onlyOwner`/role checks on sensitive setters
   - **Front-running**: predictable state transitions exploitable via mempool observation

7. **Solve — Analyze with adversarial assumptions.** For each candidate, test under three adversarial profiles:
   - *Environment tampering*: What if `block.timestamp` is at the edge of a range? What if this is called in the same block as another transaction?
   - *Interaction hijacking*: What if the external call target is a malicious contract that re-enters? What if `transferFrom` returns false silently?
   - *Resource infinity*: What if the attacker has unlimited capital via flash loan? Can they manipulate pool ratios, oracle prices, or collateral values?
   For arithmetic vulnerabilities, mentally model the constraints: identify the invariant the code assumes, find boundary inputs where rounding or overflow breaks it, and verify whether the deviation is exploitable (i.e., profitable for the attacker).

8. **Chain exploits.** Review all individual findings across batches. For each pair (A, B), check: does successfully exploiting A create a state that makes B exploitable? Common chains: oracle manipulation -> undercollateralized borrow -> bad debt; precision loss -> repeated micro-transactions -> invariant drift -> profitable withdrawal; access control bypass -> parameter manipulation -> economic drain.

9. **Verify through cascaded filtration.** For each finding:
   - *Contextual check*: Does the full contract have a safeguard (e.g., a `nonReentrant` modifier, a pause mechanism, an admin-only rescue function) that mitigates this?
   - *Deduplication*: Are multiple findings describing the same root cause? Merge them and keep the most severe framing.
   - *Threat model check*: Does this exploit require the contract owner to be malicious (centralization risk, not a bug)? Does it require an oracle to be compromised externally? If so, flag it as an informational/centralization finding, not a critical vulnerability.

10. **Report findings.** For each verified vulnerability, output: severity (Critical/High/Medium/Low/Informational), title, affected function(s) with line references, root cause explanation, step-by-step exploit scenario, estimated impact (funds at risk), and a recommended fix with code.

## Concrete Examples

**Example 1: Precision Loss in a DEX Pool**

User: "Audit this Balancer-style weighted pool contract for vulnerabilities."

Approach:
1. Build dependency graph: `swap()` calls `_upscaleArray()`, `_downscaleArray()`, `calcOutGivenIn()` which use `mulDown()` and `divDown()` from FixedPoint library.
2. Cluster: swap/join/exit functions form one batch; fee/governance another.
3. Plan: `calcOutGivenIn` performs multiple sequential `mulDown` operations — flag for precision loss.
4. Remind: Precision loss cascade pattern — repeated rounding-down in fixed-point math can accumulate, allowing an attacker to extract value through many small swaps.
5. Solve with adversarial assumptions (resource infinity): With flash-loaned capital, an attacker executes 65+ micro-swaps. Each swap rounds down by up to 8-9 wei. Over many iterations, the pool's internal accounting diverges from actual token balances. The attacker profits from the accumulated rounding errors.
6. Verify: No per-block swap limit exists. No minimum swap size enforced. Finding is valid.

Output:
```
## [CRITICAL] Precision Loss Cascade in Weighted Pool Swap

**Affected:** `WeightedPool.swap()` -> `WeightedMath.calcOutGivenIn()` (lines 142-189)

**Root Cause:** Sequential `mulDown()` and `divDown()` operations in the swap
calculation accumulate rounding errors of up to 8-9 wei per operation. No minimum
swap amount is enforced.

**Exploit Scenario:**
1. Attacker flash-loans large token amount
2. Executes 65+ micro-swaps in a single transaction
3. Each swap extracts ~8 wei more than it should due to rounding
4. Accumulated extraction drains pool value over repeated transactions

**Impact:** Pool invariant diverges from actual balances; estimated extractable
value scales with pool TVL and number of micro-swaps per block.

**Recommendation:** Enforce a minimum swap amount that exceeds the maximum
per-operation rounding error, or use `mulUp`/`divUp` for output calculations
to ensure rounding always favors the pool.
```

**Example 2: Cross-Contract Reentrancy in a Lending Protocol**

User: "Check this lending protocol for reentrancy — it uses `nonReentrant` on all external functions."

Approach:
1. Build dependency graph: `deposit()`, `borrow()`, `withdraw()`, `liquidate()` are external; each calls internal helpers `_updateAccountState()`, `_checkHealthFactor()`.
2. Cluster: deposit/withdraw share `balances` state; borrow/liquidate share `debts` state; both groups share `healthFactor` computation.
3. Plan: All external functions have `nonReentrant` — but `deposit()` writes to `balances` mapping, then calls an ERC-777 token's `transfer` which triggers a `tokensReceived` hook. The hook can call `borrow()` — a *different* function, so the single-function reentrancy guard does not block it.
4. Remind: Cross-function reentrancy pattern — `nonReentrant` per-function does not prevent reentrancy across different functions sharing state.
5. Solve: Attacker deposits ERC-777 token -> `tokensReceived` callback fires -> attacker calls `borrow()` before `deposit()` finishes updating internal accounting -> health factor computed on stale state -> attacker borrows more than collateral allows.
6. Verify: Contextual check — is there a global reentrancy lock? If the modifier uses a contract-level boolean (not per-function), the attack is blocked. If per-function, the attack succeeds.

Output:
```
## [HIGH] Cross-Function Reentrancy via ERC-777 Token Hook

**Affected:** `LendingPool.deposit()` (line 87) -> `LendingPool.borrow()` (line 134)

**Root Cause:** `nonReentrant` modifier uses per-function locking. When `deposit()`
transfers an ERC-777 token, the `tokensReceived` hook executes before deposit
accounting completes. The hook can invoke `borrow()`, which reads stale balances.

**Exploit Scenario:**
1. Attacker calls `deposit()` with an ERC-777 collateral token
2. During token transfer, `tokensReceived` hook triggers
3. Hook calls `borrow()` — permitted because it has a separate reentrancy lock
4. `borrow()` reads `balances[attacker]` which is already credited but internal
   accounting (health factor) has not yet been updated
5. Attacker borrows against inflated health factor, then withdraws

**Impact:** Attacker can borrow up to 100% of deposited collateral value instead
of the protocol's intended LTV ratio.

**Recommendation:** Replace per-function `nonReentrant` with a contract-level
reentrancy guard (single shared boolean). Alternatively, follow checks-effects-
interactions: update all internal state before any external call.
```

**Example 3: Quick Triage of a Single Function**

User: "Is this `withdrawAll` function safe?"
```solidity
function withdrawAll() external {
    uint256 balance = balances[msg.sender];
    (bool success, ) = msg.sender.call{value: balance}("");
    require(success, "Transfer failed");
    balances[msg.sender] = 0;
}
```

Approach:
1. Single function — skip graph construction. Directly apply Plan-Remind-Solve.
2. Plan: State update (`balances[msg.sender] = 0`) happens *after* external call (`msg.sender.call`). Classic pattern.
3. Remind: Reentrancy — the exact pattern from the DAO hack.
4. Solve: Attacker deploys contract with `receive()` fallback that calls `withdrawAll()` again. On re-entry, `balances[msg.sender]` still holds the original value. Drains contract.
5. Verify: No `nonReentrant` modifier. No checks-effects-interactions. Confirmed exploitable.

Output:
```
## [CRITICAL] Reentrancy in withdrawAll()

State update on line 5 (`balances[msg.sender] = 0`) occurs after the external
call on line 3. An attacker contract's `receive()` function can re-enter
`withdrawAll()` before the balance is zeroed, draining the entire contract.

Fix — move the state update before the external call:
  uint256 balance = balances[msg.sender];
  balances[msg.sender] = 0;  // <-- move before call
  (bool success, ) = msg.sender.call{value: balance}("");
  require(success, "Transfer failed");
```

## Best Practices

- **Do:** Always build a dependency graph before auditing multi-contract systems. Analyzing functions in isolation misses cross-function and cross-contract interactions — the most severe vulnerability class.
- **Do:** Apply all three adversarial profiles (environment tampering, interaction hijacking, resource infinity) to every flagged candidate. Many business logic bugs only manifest under flash-loan conditions or same-block manipulation.
- **Do:** Chain findings. A medium-severity precision loss combined with a medium-severity oracle read can compose into a critical exploit. Report the chain, not just the parts.
- **Do:** Run cascaded verification before reporting. Re-check each finding against the full contract context. A reentrancy finding is invalid if a global lock exists; a precision loss is low-severity if a minimum amount is enforced.
- **Avoid:** Reporting centralization risks (admin can rug) as vulnerabilities unless the user explicitly includes trusted-role assumptions in scope. These are design decisions, not bugs.
- **Avoid:** Hallucinating vulnerability patterns that don't match the actual code flow. If you cannot trace a concrete step-by-step exploit path, downgrade the finding to informational or discard it.
- **Avoid:** Auditing only the "obvious" functions. Business logic bugs hide in helper functions, modifiers, and library code. The dependency graph ensures nothing is skipped.

## Error Handling

- **Incomplete source code:** If the user provides partial contracts with unresolved imports, explicitly list what's missing and explain which audit steps are limited. Offer to audit what's available with caveats.
- **Proxy/upgradeable contracts:** If the contract uses a proxy pattern, warn that the implementation can change. Audit the current implementation but flag that findings may not persist across upgrades.
- **No vulnerabilities found:** Do not fabricate findings. Report that the audited code appears sound under the tested adversarial profiles, list exactly which profiles were tested, and note any areas where formal verification would add confidence (e.g., complex math invariants).
- **Ambiguous severity:** When a finding's exploitability depends on external conditions (e.g., oracle liveness, governance actions), report it with conditional severity: "Critical if oracle can be manipulated in a single block; Low if TWAP with 30-minute window is used."

## Limitations

- This approach is LLM-based and cannot replace formal verification for mathematical invariants. For complex DeFi math (AMM curves, interest rate models), recommend Z3 or Certora Prover for definitive proofs.
- Claude cannot execute Solidity code or run a forked mainnet simulation. Exploit chains are reasoned about, not empirically tested. Always recommend the user validate critical findings with a Foundry/Hardhat PoC.
- Very large protocols (50+ contracts, 10K+ LOC) may exceed practical context limits even with batching. Prioritize the highest-centrality batches and flag uncovered areas.
- The adversarial profiles assume standard EVM behavior. L2-specific quirks (sequencer uptime feeds, custom precompiles, different gas semantics) require additional domain-specific assumptions.
- This method targets business logic vulnerabilities. For low-level EVM issues (storage collisions in proxies, inline assembly bugs, gas griefing), supplement with Slither or Mythril.

## Reference

Hu, X., Chan, W.Y., Shi, Y., Sun, Q., & Wang, W.-C. (2026). *An Effective and Cost-Efficient Agentic Framework for Ethereum Smart Contract Auditing.* arXiv:2601.17833v1. [https://arxiv.org/abs/2601.17833v1](https://arxiv.org/abs/2601.17833v1)

Key sections to study: Section 3 (Contextual Profiling via Louvain clustering), Section 4 (Plan-Remind-Solve auditor with adversarial state injection profiles), Section 5 (Cascaded verification filters), and the Balancer V2 case study demonstrating precision loss cascade detection.