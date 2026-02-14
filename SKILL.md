---
name: ethereum-building
description: Opinionated playbook for building production Ethereum dApps. Use when asked to build, create, or develop smart contracts, dApps, tokens (ERC-20, ERC-721), DeFi protocols, staking apps, NFT projects, DAOs, or any Solidity/blockchain project. Also use for Ethereum-related tasks like writing Foundry tests, auditing contract security, building a wagmi frontend, or deploying contracts to mainnet/L2s. Covers the full lifecycle — contracts, testing, security, frontend, and deployment.
---

# Ethereum Building Skill

> Last updated: February 2026

You are building an Ethereum dApp. This skill tells you how to do it correctly. Read this file first — it gives you the workflow and routes you to detailed sub-skills for each phase.

## Stack

- **Contracts:** Foundry (forge, cast, anvil) + OpenZeppelin v5 + Solidity >=0.8.20
- **Frontend:** Next.js (TypeScript) + wagmi v2 + viem + RainbowKit
- **Local dev:** Anvil fork of mainnet (real protocol state, real tokens)
- **Deploy:** Foundry for contracts, Vercel or IPFS for frontend

## Hard Rules (apply to ALL phases)

1. **Fork mode for protocol integration.** If your dApp interacts with ANY external protocol (tokens, Uniswap, Aave, Chainlink, etc.): `anvil --fork-url <rpc>`. For dApps using only native ETH with no external contracts, plain `anvil` is acceptable.
   - **RPC options (in order of preference):** your own Alchemy/Infura URL, `https://eth.llamarpc.com`, `https://rpc.ankr.com/eth`, `https://eth.drpc.org`. If one rate-limits you, try the next.
2. **NEVER hallucinate contract addresses.** If you need a protocol address (Uniswap, USDC, Aave, etc.), fetch it from `https://ethskills.com/addresses/SKILL.md`. If that URL returns HTML, use the raw GitHub source. Never guess.
3. **Nothing is automatic on Ethereum.** Every public function that changes state needs a caller with an incentive. Design for this.
4. **Secrets in `.env.local` only.** RPC URLs, API keys, private keys — never committed, always in `.gitignore`.
5. **All forge/cast commands run from inside `contracts/`.** Always `cd contracts` before running `forge build`, `forge test`, `forge script`, etc. Running from the project root will fail or compile wrong files.

## Project Setup

**Existing project?** If you see a `foundry.toml` or `contracts/` directory, skip setup and route to the relevant phase below.

**Starting from scratch:**

```bash
# 1. Init project root
mkdir <project-name> && cd <project-name>
git init

# 2. Contracts (Foundry)
forge init contracts --no-git
# Verify forge-std: ls contracts/lib/forge-std/ (if missing: cd contracts && forge install foundry-rs/forge-std --no-git)

# 3. Frontend (Next.js + wagmi)
npx create-next-app@latest frontend --typescript --tailwind --eslint --app --no-src-dir --import-alias "@/*" --yes
cd frontend && npm install wagmi viem @rainbow-me/rainbowkit @tanstack/react-query && cd ..

# 4. Environment
echo 'NEXT_PUBLIC_WALLETCONNECT_PROJECT_ID=<get from cloud.walletconnect.com>' > frontend/.env.local
echo 'NEXT_PUBLIC_ALCHEMY_API_KEY=<your key>' >> frontend/.env.local

# 5. Fork mainnet (in a separate terminal) — see RPC options in Hard Rules above
anvil --fork-url https://eth.llamarpc.com
# Note: Anvil defaults to chainId 31337 even when forking mainnet
# For ETH-only dApps with no external protocols, plain `anvil` is also fine

# 6. Enable block mining (so txs don't hang)
cast rpc anvil_setIntervalMining 1
```

**Funding test wallets:** Anvil pre-funds 10 accounts with 10,000 ETH each. For ERC-20 tokens, impersonate a whale:
```bash
cast rpc anvil_impersonateAccount <whale_address>
cast send <token> "transfer(address,uint256)" <your_address> <amount> --from <whale_address>
cast rpc anvil_stopImpersonatingAccount <whale_address>
# Find current top holders at etherscan.io/token/<token>#balances
```

## Workflow

### Full Build (user asks to "build X")

Present an architecture plan first: which contracts, their interfaces, deploy order, cross-contract calls, frontend pages, external protocol integrations. **Include reasonable default values for constructor parameters** (thresholds, deadlines, fees) — explain your choices to the user. For dev/demo, use values that allow easy testing (e.g., a 2-minute deadline, not 30 seconds). Get user approval, then execute phases sequentially:

1. **Contracts** → read `contracts/SKILL.md`
2. **Testing** → read `testing/SKILL.md`
3. **Security** → read `security/SKILL.md`
4. **Frontend** → read `frontend/SKILL.md`
5. **Deploy** → read `deploy/SKILL.md`

### Partial Workflows (user asks for a specific task)

| User says | Load |
|-----------|------|
| "Review my contract for security" | `security/SKILL.md` only |
| "Build a frontend for my contracts" | `frontend/SKILL.md` only |
| "Write tests for my contracts" | `testing/SKILL.md` only |
| "Build contracts only, no UI" | Phases 1 → 2 → 3 |
| "Deploy to production" | `deploy/SKILL.md` only |
| "Fix a bug in my contract" | Phases 1 → 2 → 3 (fix, test, re-check) |
| "Add a feature to my dApp" | Phases 1 → 2 → 3 → 4 → 5 (skip setup) |

### How to Load Sub-Skills

Sub-skill paths are relative to this file's location. If you fetched this file from a URL, replace `SKILL.md` in the URL with the sub-skill path.

Example: if this file is at `https://raw.githubusercontent.com/phipsae/ethereum-building-skills/main/SKILL.md`, then:
- `contracts/SKILL.md` → `https://raw.githubusercontent.com/phipsae/ethereum-building-skills/main/contracts/SKILL.md`
- `frontend/SKILL.md` → `https://raw.githubusercontent.com/phipsae/ethereum-building-skills/main/frontend/SKILL.md`

**Important:** When fetching sub-skills via URL, use raw file reads (curl, Read tool) — NOT summarizing tools like WebFetch. The sub-skills contain exact code templates that must be followed verbatim.

Only load the sub-skill you need — never load all 5 at once.
