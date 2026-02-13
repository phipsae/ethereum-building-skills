# Ethereum Building Skill

An opinionated playbook for AI agents building production Ethereum dApps. Give your agent one URL — it learns the correct workflow, tools, and gotchas for building on-chain in 2026.

## How It Works

Give any URL to your AI agent. It reads the skill and builds correctly.

```
https://raw.githubusercontent.com/phipsae/ethereum-building-skills/main/SKILL.md
```

The meta skill routes the agent to 5 sub-skills, loaded on demand to keep context lean:

| Phase | Skill | What It Corrects |
|-------|-------|-----------------|
| 1 | [Contracts](contracts/SKILL.md) | Deploy script pattern, OZ v5 imports, token decimals, reentrancy, approve flow |
| 2 | [Testing](testing/SKILL.md) | What agents forget to test — decimals, access control, fuzz testing, edge cases |
| 3 | [Security](security/SKILL.md) | Audit checklist — flash loans, cross-function reentrancy, oracle manipulation |
| 4 | [Frontend](frontend/SKILL.md) | wagmi v2 provider setup, tx confirmation UX, approve flow, chain config |
| 5 | [Deploy](deploy/SKILL.md) | Key management, mainnet/testnet/L2 paths, Vercel + IPFS, verification |

## Stack

| Concern | Tool |
|---------|------|
| Smart contracts | Foundry (forge, cast, anvil) |
| Contract library | OpenZeppelin v5 |
| Local dev | Anvil fork of mainnet |
| Frontend | Next.js + wagmi v2 + viem + RainbowKit |
| Deploy | Foundry for contracts, Vercel or IPFS for frontend |

## Why This Exists

AI agents building Ethereum dApps make the same mistakes:

- **Use blank local chains** instead of forking mainnet (no tokens, no protocols, no liquidity)
- **Hardcode 18 decimals** (USDC is 6, WBTC is 8)
- **Show "Success" after signing** instead of waiting for tx confirmation
- **Skip fuzz testing** entirely
- **Hallucinate contract addresses** instead of looking them up
- **Forget `vm.startBroadcast()`** in deploy scripts
- **Miss the wagmi v2 provider wrapping** (every hook throws `WagmiProviderNotFoundError`)

This skill fixes all of them. It only teaches what LLMs get wrong — if the model already knows something, it's not in here.

## Usage Examples

**Full dApp build:**
```
Read https://raw.githubusercontent.com/phipsae/ethereum-building-skills/main/SKILL.md
and build me a staking dApp where users stake ETH and earn rewards.
```

**Just security review:**
```
Read https://raw.githubusercontent.com/phipsae/ethereum-building-skills/main/security/SKILL.md
and review my contracts in ./contracts/src/
```

**Just frontend:**
```
Read https://raw.githubusercontent.com/phipsae/ethereum-building-skills/main/frontend/SKILL.md
and build a frontend for my deployed contracts.
```

## How It's Structured

```
├── SKILL.md              ← Meta router (always loaded first, ~4KB)
├── contracts/SKILL.md    ← Phase 1: Solidity + Foundry
├── testing/SKILL.md      ← Phase 2: What agents forget to test
├── security/SKILL.md     ← Phase 3: Audit checklist
├── frontend/SKILL.md     ← Phase 4: wagmi + RainbowKit
└── deploy/SKILL.md       ← Phase 5: Production deployment
```

The meta file is ~4KB. Sub-skills are 50-210 lines each. Total context when building: meta (~4KB) + one sub-skill at a time. Never all 5 at once.

## Related

- [EthSkills](https://github.com/austintgriffith/ethskills) — Corrects LLM misconceptions about Ethereum (gas prices, addresses, standards)
- [Ethereum Wingman](https://github.com/phipsae/ethereum-wingman) — SE2-based Ethereum development tutor

## License

MIT
