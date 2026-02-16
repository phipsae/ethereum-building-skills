---
name: ethereum-building/contracts
description: Phase 1 — Write and deploy Solidity contracts with Foundry on a mainnet fork.
---

# Phase 1: Contracts

Write Solidity. Deploy to your local Anvil fork. Export ABIs for the frontend.

## Versions

- **Solidity >=0.8.20** — built-in overflow protection, latest features
- **OpenZeppelin v5** — NOT v4. They have different import paths:
  - v4 (WRONG): `import "@openzeppelin/contracts/security/ReentrancyGuard.sol";`
  - v5 (RIGHT): `import "@openzeppelin/contracts/utils/ReentrancyGuard.sol";`

## Gotchas — Hard Rules

These are the things you WILL get wrong if you don't read this:

1. **Token decimals vary.** USDC = 6, WBTC = 8, most tokens = 18. ALWAYS call `decimals()`. Never hardcode 18.
2. **ERC-20 approve is two-step.** User must `approve()` your contract first, then your contract calls `transferFrom()`. Prefer exact approval amounts. If using `type(uint256).max` (standard for trusted protocols like Uniswap/Aave), document the trust assumption.
3. **Reentrancy kills.** CEI pattern (Checks-Effects-Interactions) + `ReentrancyGuard` on every function with an external call. No exceptions.
4. **No floating point.** Use basis points: 500/10000 = 5%. Never raw percentages.
5. **SafeERC20 for everything.** USDT doesn't return a bool on `transfer()`. Without SafeERC20, your contract silently fails.
6. **Vault inflation attack.** If you're building a vault/pool: use a virtual offset for the first depositor. Without it, an attacker can inflate the share price and steal from the next depositor.
7. **Never use DEX spot prices as oracles.** They can be manipulated in a single transaction. Use Chainlink with staleness checks.
8. **Every maintenance function needs an incentive.** Who calls `harvest()`? Who calls `rebalance()`? If the answer is "someone will," it won't happen. Design a caller reward:
   ```solidity
   uint256 public constant HARVEST_REWARD_BPS = 100; // 1% to caller
   function harvest() external {
       uint256 yield = externalProtocol.claim();
       uint256 callerReward = (yield * HARVEST_REWARD_BPS) / 10000;
       rewardToken.transfer(msg.sender, callerReward);
       rewardToken.transfer(treasury, yield - callerReward);
       emit Harvested(msg.sender, yield, callerReward);
   }
   ```

## Events

Emit events for ALL state changes. Every function that modifies state must emit at least one event:
```solidity
event Staked(address indexed user, uint256 amount);
event Withdrawn(address indexed user, uint256 amount, uint256 reward);
```
Index addresses with `indexed` for efficient filtering. Events enable frontend reactivity (no polling), off-chain indexing (The Graph), and bot monitoring. They cost minimal gas and are critical for production dApps.

## Foundry Setup

**All forge commands must run from inside `contracts/`.** Always `cd contracts` first.

```bash
cd contracts
forge install OpenZeppelin/openzeppelin-contracts --no-git
```

Create `contracts/remappings.txt`:
```
@openzeppelin/contracts/=lib/openzeppelin-contracts/contracts/
```

Verify `contracts/lib/forge-std/` exists. If not: `forge install foundry-rs/forge-std --no-git`

Add fuzz testing config to `contracts/foundry.toml`:
```toml
[fuzz]
runs = 1000
```

Write contracts in `contracts/src/`. Write deploy scripts in `contracts/script/`.

## Deploy Script

Agents frequently get deploy scripts wrong. Use this pattern:

```solidity
// SPDX-License-Identifier: MIT
pragma solidity ^0.8.20;

import {Script, console} from "forge-std/Script.sol";
import {MyContract} from "../src/MyContract.sol";

contract Deploy is Script {
    function run() external returns (MyContract) {
        vm.startBroadcast();
        MyContract c = new MyContract(/* constructor args */);
        vm.stopBroadcast();

        console.log("MyContract deployed at:", address(c));
        return c;
    }
}
```

**Multi-contract deploys:** Deploy dependencies first. Pass addresses, never hardcode:
```solidity
function run() external {
    vm.startBroadcast();
    Token token = new Token();
    Staking staking = new Staking(address(token));
    Rewards rewards = new Rewards(address(token), address(staking));
    vm.stopBroadcast();
}
```

**Deploy to local fork (for testing — production deployment is in Phase 5):**
```bash
forge script script/Deploy.s.sol \
  --rpc-url http://localhost:8545 \
  --broadcast \
  --private-key 0xac0974bec39a17e36ba4a6b4d238ff944bacb478cbed5efcae784d7bf4f2ff80
```
That private key is Anvil's default account 0. For production, use `--account deployer` (Foundry keystore) or `--private-key $DEPLOYER_PRIVATE_KEY`.

## Integrating External Protocols

Building on Uniswap, Aave, Chainlink, etc.? They're already deployed on your fork.

1. Create `contracts/src/interfaces/` — define minimal interfaces with ONLY the functions you call:
   ```solidity
   interface ISwapRouter {
       struct ExactInputSingleParams { ... }
       function exactInputSingle(ExactInputSingleParams calldata) external payable returns (uint256);
   }
   ```
2. Fetch verified addresses from `https://ethskills.com/addresses/SKILL.md` — never guess them.
3. Interact directly in your contracts — they exist on the fork at real addresses.

## ABI Export for Frontend

After deploying, the frontend needs ABIs and addresses:

1. **ABIs** are in `contracts/out/<ContractName>.sol/<ContractName>.json` — extract only the `abi` field.
2. **Deployed addresses** are in `contracts/broadcast/Deploy.s.sol/<chainId>/run-latest.json` — look for `contractAddress` in the `transactions` array.
3. Create `frontend/contracts/deployedContracts.ts` (create the directory first: `mkdir -p frontend/contracts`):

```typescript
export const deployedContracts = {
  31337: {
    MyContract: {
      address: "0x5FbDB2315678afecb367f032d93F642f64180aa3" as const,
      abi: [/* paste ABI array here */] as const,
    },
    // add more contracts...
  },
} as const;
```

The `as const` is mandatory — without it, viem can't infer types and wagmi hooks lose type safety.

## When Things Go Wrong

| Problem | Fix |
|---------|-----|
| `forge build` import errors | Check `remappings.txt`: `@openzeppelin/contracts/=lib/openzeppelin-contracts/contracts/` |
| Deploy script reverts | Check constructor args, verify `anvil` is running, check `vm.startBroadcast()` is present |
| "No signer available" | Missing `--private-key` flag on `forge script` command |
| Can't find deployed address | Check `contracts/broadcast/Deploy.s.sol/31337/run-latest.json` |

## Exit Criteria

- [ ] All contracts compile (`forge build` succeeds)
- [ ] Deploy script tested on local fork (contracts deployed and ABIs exported)
- [ ] `frontend/contracts/deployedContracts.ts` created with ABIs + addresses

Next → load `testing/SKILL.md`
