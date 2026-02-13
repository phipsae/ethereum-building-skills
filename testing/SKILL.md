---
name: ethereum-building/testing
description: Phase 2 — Foundry tests focused on what AI agents consistently miss.
---

# Phase 2: Testing

You know how to write Foundry tests. This skill tells you what you're forgetting to test.

## Quick Reference

- Tests go in `contracts/test/`
- Run: `forge test`
- Debug: `forge test -vvvv`
- Single test: `forge test --match-test testFunctionName`

## What You're Forgetting to Test

These are the gaps agents leave in every test suite:

### 1. Token Decimal Handling
Agents almost always hardcode 18 decimals. Test with real tokens on the fork:
- USDC (6 decimals)
- WBTC (8 decimals)
- DAI (18 decimals)

If your contract handles multiple tokens, test all three. The bugs are always in the math when decimals don't match.

### 2. Access Control — Both Directions
Don't just test that the owner can call admin functions. Test that non-owners get reverted:
```solidity
function testNonOwnerCannotWithdraw() public {
    vm.prank(attacker);
    vm.expectRevert();
    vault.withdraw(1 ether);
}
```

### 3. The Full Approve → Action Flow
Test the complete two-step process:
- Without approval → reverts
- With approval → succeeds
- Approval for wrong amount → reverts or partial success

### 4. Edge Cases That Lose Money
- Zero amounts (deposit 0, withdraw 0, transfer 0)
- `type(uint256).max` inputs
- Empty arrays
- Re-entrancy from a malicious receiver contract (write one that calls back)

### 5. State After Failed Transactions
The #1 source of exploits: partial state changes. If a function modifies state then reverts mid-execution, verify nothing leaked:
```solidity
function testStateConsistentAfterRevert() public {
    uint256 balanceBefore = token.balanceOf(address(vault));
    vm.expectRevert();
    vault.riskyAction();
    assertEq(token.balanceOf(address(vault)), balanceBefore);
}
```

### 6. Multi-Contract Interactions
If contract A calls contract B, test the full chain:
- Happy path (A → B succeeds)
- B reverts — does A handle it?
- B returns unexpected data — does A validate?

## Fork Test Setup — Funding Accounts

Your tests run against a mainnet fork. Accounts start empty (except Anvil's pre-funded ones).

```solidity
function setUp() public {
    // Give ETH to any address
    vm.deal(user, 100 ether);

    // Get ERC-20 tokens from a whale
    address whale = 0x...; // look up current top holder on Etherscan
    vm.prank(whale);
    usdc.transfer(user, 1_000_000e6); // 1M USDC (6 decimals!)

    // For multiple calls from same address
    vm.startPrank(whale);
    usdc.transfer(user, 500_000e6);
    dai.transfer(user, 500_000e18);
    vm.stopPrank();
}
```

Always reset state in `setUp()` — fork tests share the same snapshot.

## Fuzz Testing

Agents skip fuzz tests by default. Don't.

- Every function that takes a `uint256` should be fuzzed
- Bound inputs to realistic ranges: `amount = bound(amount, 1, 1_000_000e18)`
- Set minimum runs in `foundry.toml`:
  ```toml
  [fuzz]
  runs = 1000
  ```

**Invariant tests** for protocol-level properties:
- "Total shares never exceeds total assets"
- "Sum of all user balances equals contract balance"
- "Price per share is monotonically non-decreasing"

```solidity
function invariant_totalSharesNeverExceedAssets() public view {
    assertLe(vault.totalShares(), vault.totalAssets());
}
```

## When Things Go Wrong

| Problem | Fix |
|---------|-----|
| Cryptic revert, no message | `forge test -vvvv` for full stack trace |
| Fork tests timeout | RPC URL invalid or rate-limited — try `https://eth.llamarpc.com` |
| Tests pass alone, fail together | Shared state — ensure `setUp()` resets everything |
| Fuzz finds failure | Failing input is printed — write a regression unit test with that exact input |

## Exit Criteria

- [ ] All tests pass
- [ ] Fuzz tests run ≥1000 iterations
- [ ] Edge cases from checklist above are covered (decimals, access control, approve flow, zero amounts)

Next → load `security/SKILL.md`
