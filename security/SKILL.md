---
name: ethereum-building/security
description: Phase 3 — Security audit checklist for Ethereum contracts. Verify nothing slipped through.
---

# Phase 3: Security Review

Stop. Before you move to the frontend, review every contract against this checklist. This is not "write secure code" — that was Phase 1. This is "verify nothing slipped through."

Go through each item. Check it against the actual code. If it doesn't apply, document why.

## Audit Checklist

- [ ] **Access control** on ALL admin/privileged functions — not just the obvious ones. Check modifiers on every `external` and `public` function.
- [ ] **Reentrancy** — CEI pattern + `nonReentrant` on all functions with external calls.
- [ ] **Cross-function reentrancy** — can calling function A during function B's external call cause issues? Check functions that share state.
- [ ] **Token decimals** — verified not hardcoding 18. Check every arithmetic operation involving token amounts.
- [ ] **Oracle usage** — Chainlink with staleness checks (`updatedAt` is recent), not DEX spot prices.
- [ ] **Return values** — using `SafeERC20` for all token operations. Raw `transfer()`/`transferFrom()` silently fails with USDT.
- [ ] **Integer math** — no division before multiplication. Basis points (10000), not raw percentages.
- [ ] **Input validation** on all external function parameters. Zero addresses, zero amounts, array lengths.
- [ ] **Events** emitted for all state changes. Required for frontend reactivity and off-chain indexing.
- [ ] **Approval hygiene** — exact approvals preferred. `type(uint256).max` only for trusted protocols, with documented justification.
- [ ] **Incentive design** — who calls each maintenance function (harvest, rebalance, liquidate) and why would they?
- [ ] **First depositor protection** for vault/pool contracts. Virtual offset or minimum deposit.
- [ ] **Flash loan resistance** — can any state be manipulated by borrowing + acting + repaying in a single transaction?
- [ ] **Frontrunning** — can a pending transaction be profitably frontrun? Consider commit-reveal or slippage parameters.

## Why This Matters

One-line reminders of what happens when you skip these:

- **The DAO (2016):** reentrancy → $60M drained
- **Euler (2023):** unprotected `donateToReserves` allowed debt manipulation → $197M
- **Cream Finance (2021):** oracle manipulation via flash loan → $130M
- **Nomad Bridge (2022):** missing input validation → $190M
- **Harvest Finance (2020):** flash loan price manipulation → $34M

## When in Doubt

Can't determine if a checklist item applies? **Implement the protection anyway.** The cost of an unnecessary guard is near zero gas. The cost of a missing one is catastrophic.

## Exit Criteria

- [ ] Every checklist item explicitly PASSES or has documented justification for why it doesn't apply
- [ ] No items left unchecked

**If any item fails** → go back to Phase 1 (contracts), fix the issue, re-run Phase 2 (tests), then re-audit.

Next → load `frontend/SKILL.md`
