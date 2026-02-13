---
name: ethereum-building/deploy
description: Phase 5 — Production deployment of contracts and frontend.
---

# Phase 5: Deploy

Everything works on localhost. Now go live.

This covers both contract deployment and frontend deployment. Skip what doesn't apply — if there's no frontend, skip those sections.

## Key Management

**Before deploying anything**, set up a production wallet:

**Recommended — Foundry keystore:**
```bash
cast wallet import deployer --interactive
# Enter private key, set a password
# Now use --account deployer instead of --private-key
```

**Alternative — env var:**
Store `DEPLOYER_PRIVATE_KEY` in `contracts/.env` (NOT the frontend `.env.local`). Use `--private-key $DEPLOYER_PRIVATE_KEY`.

**NEVER commit private keys.** Add `.env` and keystore paths to `.gitignore`.

## Contract Deployment

### Before You Deploy
Estimate cost — dry run without `--broadcast`:
```bash
forge script script/Deploy.s.sol --rpc-url $RPC
```
Tell the user the approximate USD cost before spending real ETH.

### Option A: Straight to Mainnet
For simple contracts where you're confident after fork testing:
```bash
forge script script/Deploy.s.sol \
  --rpc-url $MAINNET_RPC \
  --broadcast \
  --verify \
  --etherscan-api-key $ETHERSCAN_KEY \
  --account deployer
```
The `--verify` flag auto-verifies on Etherscan in the same transaction. Test with real wallet, small amounts ($1-10).

### Option B: Testnet First
For complex contracts, multi-party testing, or first-time deployers:
```bash
forge script script/Deploy.s.sol \
  --rpc-url $SEPOLIA_RPC \
  --broadcast \
  --verify \
  --account deployer
```
Test with team. Then repeat for mainnet. Sepolia ETH is free via faucets.

### Option C: Deploy to an L2
Base, Arbitrum, Optimism — same Solidity, lower fees:
```bash
forge script script/Deploy.s.sol \
  --rpc-url $BASE_RPC \
  --broadcast \
  --account deployer
```

Verify separately:
```bash
forge verify-contract <address> <ContractName> \
  --chain base \
  --etherscan-api-key $BASESCAN_KEY
```
If `--chain base` doesn't resolve, use `--verifier-url https://api.basescan.org/api` explicitly.

**Key differences from mainnet:** Gas is 100-1000x cheaper. Block times differ. Some precompiles may differ — test on fork first with `anvil --fork-url $BASE_RPC`.

**When to choose L2:** High-frequency interactions, gas-sensitive apps, gaming, social. Choose mainnet for DeFi composability or deep liquidity.

## Update Frontend for Production

1. **Update `frontend/contracts/deployedContracts.ts`** — add the production chainId:
   ```typescript
   export const deployedContracts = {
     1: { // mainnet (or 8453 for Base, 42161 for Arbitrum)
       MyContract: {
         address: "0x..." as const,
         abi: [...] as const,
       },
     },
   } as const;
   ```
2. **Update `frontend/lib/wagmi.ts`** — swap `foundry` for production chain:
   ```typescript
   import { mainnet } from "viem/chains"; // or base, arbitrum, optimism
   chains: [mainnet],
   ```
3. **Set production RPC** — Alchemy or Infura. Never ship with public RPCs.
4. **Remove dev-only code** — dev wallet connectors, console.logs, test addresses.

## Frontend to Vercel (Recommended)

```bash
cd frontend && vercel
```

Set environment variables in Vercel dashboard:
- `NEXT_PUBLIC_WALLETCONNECT_PROJECT_ID`
- `NEXT_PUBLIC_ALCHEMY_API_KEY`
- `NEXT_PUBLIC_CHAIN=mainnet`

No monorepo complexity — it's a standalone Next.js app.

## Frontend to IPFS (Decentralized Alternative)

Add to `next.config.ts` (or `.js`/`.mjs`):
```javascript
output: "export",
trailingSlash: true,
```

```bash
cd frontend
rm -rf .next out  # always clean first — stale builds produce same CID
next build
# Upload out/ to IPFS: Pinata, BuidlGuidl IPFS, or ipfs add -r out
```

CID changed = real deploy. Same CID = stale build.

## Pre-Publish Checklist

- [ ] OG image (1200x630) with absolute production URL, not localhost
- [ ] Page title and description updated
- [ ] Favicon set
- [ ] Production RPC configured (Alchemy/Infura)
- [ ] No test wallets or hardcoded dev addresses in code
- [ ] All env vars set in production

## ENS Subdomain (Optional)

1. Create subdomain on app.ens.domains
2. Set IPFS content hash: `ipfs://<CID>`
3. Verify via `.limo` gateway (may cache 5-15 min)

## When Things Go Wrong

| Problem | Fix |
|---------|-----|
| "No signer available" | Missing `--account deployer` or `--private-key` flag |
| Deploy tx fails | Check gas estimate, ensure wallet has enough ETH + buffer |
| Verification fails | Check Etherscan API key, compiler version. For L2s: `--verifier-url` explicitly |
| Frontend can't reach contracts | Chain ID or address mismatch between wagmi config and deployed contracts |
| IPFS routes 404 | Missing `trailingSlash: true` in next.config |

## Exit Criteria

- [ ] Contracts verified on block explorer (green checkmark)
- [ ] Frontend loads on production URL
- [ ] OG image unfurls correctly when shared
- [ ] Tested with real wallet on production

You're live. Ship it.
