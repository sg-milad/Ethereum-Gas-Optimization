**Introduction**  
These notes are from the Udemy course [Advanced Solidity: Understanding and Optimizing Gas Costs](https://www.udemy.com/course/advanced-solidity-understanding-and-optimizing-gas-costs/).

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

---

## Tx Fee Calculation

- **Tx fee (in gwei)** = Gas price × Gas limit (or gas used by tx)

---

## Block Gas Limit

The block gas limit is the maximum amount of computational work (measured in “gas”) that can be included in a single block.

- Unlike Bitcoin’s fixed-size blocks (in bytes), Ethereum blocks are “sized” by total gas.  
- Caps the amount of computation per block, bounding work and keeping propagation times reasonable.

**Typical mainnet values:**

- Prior to EIP-1559 (Aug 2021): ~12–15 million gas  
- After EIP-1559 (London): target 15 million; hard cap 30 million  
- Early 2025: effective limits ~25–30 million (≈1 000–1 200 tps)

**Why it matters:**

- **Throughput vs decentralization**: larger blocks = more tx but slower propagation.  
- **Fee dynamics**: sets the stage for EIP-1559’s base-fee adjustment.

---

## EIP-1559 Fee Mechanism

Ethereum’s EIP-1559 separates transaction costs into:

- **Base fee**: burned each block, adjusts up/down toward target.  
- **Priority fee (tip)**: paid to miners/validators.  
- Users specify **max priority fee** and **max fee per gas**.

---

## Function Selector

A **function selector** is the first 4 bytes (8 hex chars) of the Keccak-256 hash of a function signature. It determines which function is called.

---

## Summary: 5 Places to Save Gas

### On Deployment

- Gas ∝ bytecode size (constructor + runtime).  
- Shrink bytecode with libraries, `immutable`/`constant`, remove dead code.  
- Enable and tune the Solidity optimizer.

### During Computation

- Covers every EVM opcode executed.  
- Use higher `--optimize-runs`, `unchecked {}` for loops, and custom errors.

### Transaction Data

- Calldata cost: 4 gas/zero-byte, 16 gas/non-zero-byte.  
- Pack parameters tightly, prefer `calldata`.

### Memory

- 3 gas per 32-byte word + quadratic surcharge >724 bytes.  
- Reuse buffers, minimize ephemeral arrays.

### Storage

- SSTORE zero→non-zero: 20 000 gas; non-zero→non-zero: 5 000 gas.  
- Clearing slots yields refunds (up to 50%).  
- Pack multiple small vars into one slot; use `constant`/`immutable`.

---

## Payable vs Non-Payable Functions

- Non-payable functions include a `msg.value == 0` check.  
- Mark as `payable` to skip that check and save gas if you don’t need it.

---

## Intrinsic Gas & 21 000 Base Cost

Minimum gas per tx before any opcodes:

1. ECDSA signature recovery for `msg.sender`.  
2. RLP encoding & network propagation.  
3. Protocol integrity checks (nonce, balance, chain ID).

Gas <21 000 is rejected.

---

## `--optimize-runs`

Solidity compiler flag to balance size vs runtime:

- **Low (1–10)**: optimize for deployment size.  
- **Medium (200)**: default balance.  
- **High (1000+)**: optimize for repeated calls.

---

## Gas Cost Rules

- Storage zero→non-zero: 20 000 gas  
- Storage non-zero→non-zero: 5 000 gas  
- Storage non-zero→zero: refund (≤50% of gas used)  
- First access to slot: +2 100 gas; subsequent: +100 gas  
- Smaller integer types still occupy 32 bytes.

---

## Gas Refund in Ethereum

- **Deleting storage** (`delete mapping[key]`) or `selfdestruct()` refunds 15 000 gas per slot.  
- Refund capped at 50% of total gas used.

---

## Variable Packing

- Storage slots are 256 bits (32 bytes).  
- Pack back-to-back small vars ≤256 bits into one slot.  
- Masking costs ~20 gas per access; only pack when multiple vars fit.

---

## Array Length

- **Dynamic storage**: `.length` is one SLOAD (~100 gas).  
- **Fixed-size**: compile-time constant, no SLOAD.  
- **Memory**: `.length` is MLOAD (~3 gas).  
- Cache lengths before loops; `push()` zero-inits, `pop()` refunds.

---

## Memory vs Calldata

- **Calldata**: immutable, zero-copy; 4 gas/zero-byte, 16 gas/non-zero-byte.  
- **Memory**: mutable; ~3 gas per 32-byte word read/write.

---

## Memory Is Never Cleared

Solidity’s free-memory pointer only moves forward. Memory resets at each external call, but no mid-call reclamation. Large expansions incur quadratic surcharge.

---

## Function Names

Selectors are sorted ascending; jumping between distant selectors costs ~22 gas. Name high-gas functions so their selectors sort earlier.

---

## Less Than vs Less Than or Equal To

- `<`/`>` compile to `LT`/`GT` (3 gas).  
- `<=`/`>=` add `ISZERO` (~6 gas).  
- Use strict comparisons to save ~3 gas each.

---

## Bit Shifting

- `x << n`/`x >> n` use `SHL`/`SHR` (3 gas) vs `MUL`/`DIV` (5 gas).  
- Excess bits are dropped (no revert), e.g. `2 << 255` → 0.

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
