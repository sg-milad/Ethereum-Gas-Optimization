# **Introduction**  
These notes are from the Udemy course “[Advanced Solidity: Understanding and Optimizing Gas Costs](https://www.udemy.com/course/advanced-solidity-understanding-and-optimizing-gas-costs/)”.

## Table of Contents

1. [Tx Fee Calculation](#tx-fee-calculation)  
2. [Block Gas Limit](#block-gas-limit)  
3. [EIP-1559 Fee Mechanism](#eip-1559-fee-mechanism)  
4. [Function Selector](#function-selector)  
5. [Summary: 5 Places to Save Gas](#summary-5-places-to-save-gas)  
   6. [On Deployment](#on-deployment)  
   7. [During Computation](#during-computation)  
   8. [Transaction Data](#transaction-data)  
   9. [Memory](#memory)  
   10. [Storage](#storage)  
11. [Payable vs Non-Payable Functions](#payable-vs-non-payable-functions)  
12. [Intrinsic Gas & 21 000 Base Cost](#intrinsic-gas--21-000-base-cost)  
13. [`--optimize-runs`](#optimize-runs)  
14. [Gas Cost Rules](#gas-cost-rules)  
15. [Gas Refund in Ethereum](#gas-refund-in-ethereum)  
16. [Variable Packing](#variable-packing)  
17. [Array Length](#array-length)  
18. [Memory vs Calldata](#memory-vs-calldata)  
19. [Memory Is Never Cleared](#memory-is-never-cleared)  
20. [Function Names](#function-names)  
21. [Less Than vs Less Than or Equal To](#less-than-vs-less-than-or-equal-to)  
22. [Bit Shifting](#bit-shifting)  
23. [Useful Links](#useful-links)  

---

## Tx Fee Calculation

- **Tx fee (in gwei)** = Gas price × Gas limit (or actual gas used by the transaction)  
- Reflects what you pay to include your transaction on-chain.

---

## Block Gas Limit

The block gas limit is the maximum total gas all transactions in a block can consume.

- Unlike Bitcoin’s byte-limited blocks, each Ethereum block is sized by the sum of gas consumed by its transactions.  
- Caps how many transactions—or how much computation—a miner/validator can pack into one block, keeping block validation and propagation within acceptable time.

**Typical mainnet values:**

- Pre-EIP-1559 (before Aug 2021): ~12–15 million gas  
- Post-EIP-1559 (London hard fork):  
  - Target gas used: 15 million  
  - Hard cap: 30 million (blocks above target burn more base fee, steering back toward 15 M)  
- Early 2025: Effective limits ~25–30 million → ~1 000–1 200 tx/sec.

**Why it matters:**

- **Throughput vs decentralization:** larger limits increase capacity but slow propagation, which can centralize mining.  
- **Fee dynamics:** sets upper and target bounds for EIP-1559’s base-fee adjustment.

---

## EIP-1559 Fee Mechanism

EIP-1559 splits transaction costs into:

- Base fee: burned each block; adjusts based on prior block’s gas usage vs 15 M target.  
- Priority fee (tip): paid to miners/validators.  
- Users set `maxPriorityFeePerGas` and `maxFeePerGas` to cap spending.

---

## Function Selector

The first 4 bytes (8 hex chars) of the Keccak-256 hash of `functionName(argTypes…)`. Placed in call data to identify which function to invoke.

---

## Summary: 5 Places to Save Gas

### On Deployment

Gas scales with deployed bytecode size (constructor + runtime).  
- Use external libraries or interfaces to share common code.  
- Mark static values as `immutable` or `constant` so they’re in bytecode, not storage.  
- Remove dead code.  
- Enable and tune the Solidity optimizer.

### During Computation

Covers every opcode executed at runtime.  
- Compile with higher `--optimize-runs` for hot paths.  
- Wrap arithmetic in `unchecked { … }` where safe.  
- Use custom errors (`error MyError();`) instead of `require("…")`.

### Transaction Data

Calldata costs 4 gas per zero byte, 16 gas per non-zero byte.  
- Pack parameters tightly (e.g. smaller types, `abi.encodePacked`).  
- Prefer `calldata` for `external` params.

### Memory

Costs 3 gas per 32-byte word plus a quadratic surcharge after 724 bytes (`⌊words²/512⌋`).  
- Reuse memory buffers and minimize large temporary arrays.

### Storage

Persistent state writes/reads:  
- SSTORE zero→non-zero: 20 000 gas  
- SSTORE non-zero→non-zero: 5 000 gas  
- Clearing a slot: refund up to 50% of tx gas.  
- Pack multiple small variables into one slot; use `constant`/`immutable`; clear slots when safe.

---

## Payable vs Non-Payable Functions

Non-payable functions include an implicit `require(msg.value == 0)` check. Mark as `payable` if you don’t need to reject ETH transfers to skip that check and save gas.

---

## Intrinsic Gas & 21 000 Base Cost

Minimum gas before execution:  
1. ECDSA signature recovery (msg.sender)  
2. RLP encoding & P2P propagation  
3. Consensus checks (nonce, balance, chain ID)  

Transactions with `gasLimit < 21 000` are rejected.

---

## `--optimize-runs`

Compiler flag tuning size vs runtime:  
- 1–10: favors smaller bytecode (rarely-called contracts)  
- 200: balanced default  
- 1000+: favors execution efficiency (hot-path contracts)

---

## Gas Cost Rules

- Storage zero→non-zero: 20 000 gas  
- Storage non-zero→non-zero: 5 000 gas  
- Storage non-zero→zero: refund (≤50% of gas used)  
- First slot access per tx: +2 100 gas; subsequent: +100 gas  
- Smaller integer types still occupy a full 32-byte slot.

---

## Gas Refund in Ethereum

Deleting storage (`delete mapping[key]`) or `selfdestruct()` refunds 15 000 gas per slot, capped at 50% of total gas used.

---

## Variable Packing

Storage slots are 256 bits. Back-to-back small vars ≤256 bits share one slot. Masking adds ~20 gas per access—only pack when multiple vars fit.

---

## Array Length

- Dynamic storage arrays: `.length` is an SLOAD (~100 gas).  
- Fixed-size arrays: length is constant—no SLOAD.  
- Memory arrays: `.length` is an MLOAD (~3 gas).  
- Cache `.length` before loops; `push()` zero-inits; `pop()` clears and refunds.

---

## Memory vs Calldata

- Calldata: immutable, zero-copy; 4 gas/zero-byte, 16 gas/non-zero-byte.  
- Memory: mutable; ~3 gas per 32-byte word read/write.

---

## Memory Is Never Cleared

Within a call, Solidity’s free-memory pointer only moves forward. It resets only at the start of external calls; no mid-call reclamation. Large expansions incur quadratic surcharges.

---

## Function Names

Function selectors are sorted ascending in dispatch. Jumping between non-adjacent selectors costs ~22 gas. Name heavier functions so their selectors sort earlier.

---

## Less Than vs Less Than or Equal To

`<`/`>` → `LT`/`GT` (3 gas); `<=`/`>=` add `ISZERO` (~6 gas). Use strict comparisons to save ~3 gas each.

---

## Bit Shifting

- **Shift vs. Multiply/Divide:** `x << n`/`x >> n` uses EVM’s `SHL`/`SHR` at **3 gas** instead of `MUL`/`DIV` at **5 gas**.
- **Bit shifts:** Excess bits are simply dropped (no revert), so `2 << 255` yields 0 rather than throwing.
---
## Useful Links

- [Solidity Official Docs](https://docs.soliditylang.org/)  
- [Ethereum Yellow Paper](https://ethereum.github.io/yellowpaper/)  
- [EVM Opcode List](https://www.evm.codes/)  
- [Udemy Course](https://www.udemy.com/course/advanced-solidity-understanding-and-optimizing-gas-costs/)
- [Rare skill gas-optimization Book](https://www.rareskills.io/post/gas-optimization)
- [solidity-gas-optimizations-cheat-sheet](https://0xmacro.com/blog/solidity-gas-optimizations-cheat-sheet)
- [Deconstructing a Solidity Smart Contract](https://blog.openzeppelin.com/deconstructing-a-solidity-smart-contract-part-i-introduction-832efd2d7737)
- [Rare gas puzzles](https://github.com/RareSkills/gas-puzzles)
- [Best resources for Solidity gas optimizations](https://github.com/0xisk/awesome-solidity-gas-optimization)
