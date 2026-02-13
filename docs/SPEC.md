# ethereum-building-skill — Specification

## Vision

An opinionated, lean skill that tells AI agents exactly how to build production Ethereum dApps in February 2026. Five sequential phases, each in its own sub-file, loaded on demand to keep context small. The agent reads a short meta file, understands the workflow, and loads only what it needs for the current phase.

## Target Consumer

AI coding agents (Claude Code, Cursor, Copilot) helping developers of all experience levels build on-chain Ethereum applications. The skill is consumed by the agent, not read directly by developers.

## Core Design Principles

1. **Concise above all.** Each file should be as short as possible while being effective. No filler, no "you could also" hedging.
2. **Only correct what LLMs get wrong.** Don't teach Solidity basics — LLMs know that. Teach the gotchas they miss.
3. **Route, don't bloat.** The meta file is a small router. Sub-skills load on demand. The agent never has all 5 sub-skills in context at once.
4. **Security is embedded, not optional.** Safe code is the default writing style, not a separate concern to bolt on later.
5. **Fork mode, always.** `anvil --fork-url <rpc>`, never a blank local chain. This is a top-level rule in the meta file.
6. **The agent is the scaffolder.** No framework templates. The agent creates exactly the project structure needed — nothing more, nothing less.

## Pain Points This Solves

| Problem | How the skill addresses it |
|---------|--------------------------|
| **Context window bloat** | Routing architecture — meta file ~3-4KB, sub-skills loaded one at a time |
| **Agents write insecure code** | Critical gotchas at the top of the contracts sub-skill, written as hard rules not suggestions |
| **Agents use blank local chains** | Fork mode instruction in the meta file — impossible to miss before any sub-skill loads |
| **Bad frontend UX** | Frontend sub-skill has mandatory UX rules (loading states, approve flow, tx confirmation) |
| **Deploy issues** | Contracts sub-skill ends with deploy-to-fork. Dedicated deploy sub-skill handles production |
| **Agent gets stuck on errors** | Each sub-skill has a "when things go wrong" section with common failures and fixes |

## Architecture

```
ethereum-building-skill/
├── SKILL.md                    # Meta router (~3-4KB, always loaded)
├── contracts/
│   └── SKILL.md                # Phase 1: Write + deploy Solidity with Foundry
├── testing/
│   └── SKILL.md                # Phase 2: Foundry tests (unit, fuzz, fork)
├── security/
│   └── SKILL.md                # Phase 3: Audit/review checklist
├── frontend/
│   └── SKILL.md                # Phase 4: Next.js + wagmi + viem frontend
└── deploy/
    └── SKILL.md                # Phase 5: Production deployment (contracts + frontend)
```

## Meta SKILL.md — What It Contains

The always-loaded file. ~3-4KB. Contains universal rules and routing only — NO code examples, NO detailed workflows, NO phase-specific instructions. Those belong in sub-skills. If the meta file grows past ~4KB, content is leaking in that should be in a sub-skill.

1. **Stack declaration** — Foundry for contracts/testing/security. Next.js (TypeScript) + wagmi v2 + viem + RainbowKit for frontend. Mainnet-first.
2. **Project detection + setup instructions:**
   - If a Foundry project already exists in the working directory, skip setup and route to the relevant phase.
   - If starting from scratch, the agent creates the project structure:
     ```
     project-name/
     ├── contracts/              # forge init --no-git
     │   ├── src/
     │   ├── test/
     │   ├── script/
     │   └── foundry.toml
     └── frontend/              # npx create-next-app (non-interactive)
         ├── app/
         ├── components/
         ├── contracts/          # ABIs + addresses (agent copies from Foundry output)
         └── package.json
     ```
   - Initialize git at the project root, then:
   - Contracts: `forge init contracts --no-git` (avoids nested git repos + submodule issues)
   - Verify forge-std installed: check `contracts/lib/forge-std/` exists. If missing: `cd contracts && forge install foundry-rs/forge-std --no-git`
   - Frontend: `npx create-next-app@latest frontend --typescript --tailwind --eslint --app --no-src-dir --import-alias "@/*" --yes` (all flags required — without them the command prompts interactively and hangs the agent)
   - Install frontend deps: `cd frontend && npm install wagmi viem @rainbow-me/rainbowkit @tanstack/react-query`
   - Create `frontend/.env.local` with: `NEXT_PUBLIC_WALLETCONNECT_PROJECT_ID=<id>` and `NEXT_PUBLIC_ALCHEMY_API_KEY=<key>` (WalletConnect projectId is required — obtain free at cloud.walletconnect.com)
   - Start fork: `anvil --fork-url https://eth.llamarpc.com` (or Alchemy RPC for reliability). Note: Anvil defaults to chainId 31337 even when forking mainnet. Your frontend wagmi config must match this chainId.
   - Enable block mining: `cast rpc anvil_setIntervalMining 1` (so txs don't hang)
   - **Fund test wallets on fork:** Anvil pre-funds 10 accounts with 10,000 ETH each. For ERC-20 tokens, use whale impersonation:
     - `cast rpc anvil_impersonateAccount <whale_address>`
     - `cast send <token> "transfer(address,uint256)" <your_address> <amount> --from <whale_address>`
     - `cast rpc anvil_stopImpersonatingAccount <whale_address>`
     - Find current top holders at Etherscan's token holders page (e.g., `etherscan.io/token/<token_address>#balances`). Whale addresses change over time — always look up the current top holder, don't hardcode.
3. **Universal hard rules** that apply to ALL phases:
   - ALWAYS fork mode (`anvil --fork-url`), NEVER a blank local chain
   - NEVER hallucinate contract addresses — fetch from `https://ethskills.com/addresses/SKILL.md` (raw markdown, agent-readable). If the URL returns HTML instead of markdown, fall back to the raw GitHub source.
   - Nothing is automatic on Ethereum — design incentives for every function
   - Environment variables: RPC URLs and API keys in `.env.local`, NEVER commit secrets. Ensure `.env.local` is in `.gitignore`.
4. **Routing mechanism** — Explicit instructions for loading sub-skills:
   - "To load a sub-skill, read the file at the path relative to this skill's root directory (e.g., `contracts/SKILL.md`)."
   - **Full build pipeline** (default when user asks to "build X"):
     - Phase 1 → 2 → 3 → 4 → 5 in sequence
   - **Partial workflow routing** (when user asks for specific tasks):
     - "Review my contract for security" → Read `security/SKILL.md` only
     - "Build a frontend for my already-deployed contracts" → Read `frontend/SKILL.md` only
     - "Write tests for my existing contracts" → Read `testing/SKILL.md` only
     - "Build contracts only, no UI" → Phases 1 → 2 → 3, skip 4 and 5
     - "Deploy my project to production" → Read `deploy/SKILL.md` only
     - "Fix a bug in my contract" → Phase 1 (fix) → 2 (regression test) → 3 (re-check)
     - "Add a feature to my existing dApp" → Full pipeline (1→2→3→4→5) but skip project setup
   - The agent determines which phases apply based on what the user asks for. Not every task needs all 5 phases.
5. **Agent behavior:**
   - For full builds: Present architecture plan to user first, get approval, then execute applicable phases autonomously.
   - For partial tasks: Load the relevant sub-skill and execute directly.
   - **Architecture plan contents** (for full builds): Which contracts are needed, their public interfaces, how they interact (deploy order, cross-contract calls), which frontend pages/components, and which external protocols are integrated.

## Phase 1: Contracts Sub-Skill

**File:** `contracts/SKILL.md`
**Focus:** Writing Solidity and deploying to a local fork with Foundry.
**Conceptual source:** Extract underlying patterns (gotchas, automation principles, deploy workflows) from ethereum-wingman's knowledge base. Rewrite all commands for the raw Foundry + wagmi stack — do NOT carry over any SE2-specific hooks, commands, or file paths.

### Contents

1. **Tooling versions** (top of file):
   - Solidity `>=0.8.20` (built-in overflow protection, latest language features)
   - OpenZeppelin v5 (`@openzeppelin/contracts`) — NOT v4. v5 has different import paths and renamed functions. If you see `import "@openzeppelin/contracts/security/ReentrancyGuard.sol"` that's v4. v5 is `import "@openzeppelin/contracts/utils/ReentrancyGuard.sol"`.

2. **Critical gotchas as hard rules** (impossible to miss):
   - Token decimals vary (USDC=6, WBTC=8, most=18) — always check `decimals()`
   - ERC-20 approve pattern required — two-step process. Prefer exact approvals for maximum security. If using `type(uint256).max` approvals (industry-standard for trusted protocols like Uniswap, Aave), document the trust assumption.
   - Reentrancy — CEI pattern + ReentrancyGuard on every external call
   - No floating point — use basis points (500/10000 = 5%)
   - SafeERC20 for all token operations (USDT doesn't return bool)
   - Vault inflation attack — virtual offset for first depositor
   - Never use DEX spot prices as oracles — use Chainlink
   - Incentive design — every maintenance function needs a caller incentive

3. **Foundry setup:**
   - Write contracts in `contracts/src/`
   - Install OpenZeppelin: `cd contracts && forge install OpenZeppelin/openzeppelin-contracts --no-git`
   - Create `contracts/remappings.txt` with: `@openzeppelin/contracts/=lib/openzeppelin-contracts/contracts/`
   - Verify forge-std is in `contracts/lib/forge-std/`. If missing: `forge install foundry-rs/forge-std --no-git`

4. **Deploy script pattern** (agents frequently get this wrong — include explicitly):
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
   - Deploy to local fork: `forge script script/Deploy.s.sol --rpc-url http://localhost:8545 --broadcast --private-key 0xac0974bec39a17e36ba4a6b4d238ff944bacb478cbed5efcae784d7bf4f2ff80` (Anvil's default account 0)
   - For production deploys, use `--private-key $DEPLOYER_PRIVATE_KEY` from `.env` or a keystore via `cast wallet import`

5. **Integrating external protocols (Uniswap, Aave, etc.):**
   - Create an `interfaces/` directory. Define minimal interfaces with only the functions you call (e.g., `ISwapRouter`, `IPool`). Do NOT copy entire protocol source trees.
   - In fork mode, external contracts are already deployed at real addresses. Fetch verified addresses from `https://ethskills.com/addresses/SKILL.md` — never hardcode or guess them.
   - In tests: Fork tests call real contracts automatically. Use `vm.deal()` + whale impersonation to fund test accounts with the tokens the protocol needs.

6. **Multi-contract projects:**
   - Deploy ordering matters — deploy dependencies first (e.g., deploy Token before Staking contract that takes Token address)
   - Pass deployed addresses to dependent contracts via deploy script (not hardcoded)
   - Use interfaces for cross-contract calls, import from the actual contract for type safety
   - Deploy script pattern: deploy A → get address → deploy B(addressOfA) → deploy C(addressOfA, addressOfB)

7. **ABI management for frontend:**
   - After `forge build`, ABIs are in `contracts/out/<ContractName>.sol/<ContractName>.json` — extract only the `abi` field, not the entire compiler output.
   - After `forge script --broadcast`, deployed addresses are in `contracts/broadcast/Deploy.s.sol/<chainId>/run-latest.json` — look for `contractAddress` in the `transactions` array.
   - Create `frontend/contracts/deployedContracts.ts` with this shape (the `as const` is mandatory for viem type inference):
     ```typescript
     export const deployedContracts = {
       31337: {  // chainId
         MyContract: {
           address: "0x..." as const,
           abi: [...] as const,
         },
       },
     } as const;
     ```

8. **When things go wrong:**
   - `forge build` fails with import errors → check `remappings.txt` has `@openzeppelin/contracts/=lib/openzeppelin-contracts/contracts/`
   - Deploy script reverts → check constructor args, verify fork is running (`anvil` process alive), check `vm.startBroadcast()` is present
   - "No signer available" → missing `--private-key` flag on `forge script` command
   - Can't find deployed address → check `contracts/broadcast/Deploy.s.sol/31337/run-latest.json`

9. **Phase exit criteria:** Contracts compile. Deploy script runs on local fork. ABI + addresses exported to `frontend/contracts/deployedContracts.ts`.

## Phase 2: Testing Sub-Skill

**File:** `testing/SKILL.md`
**Focus:** What to test in Ethereum contracts — the things agents consistently miss. NOT a Foundry tutorial (LLMs already know `forge test`).
**Conceptual source:** Extract testing strategies and common failure patterns from ethereum-wingman's knowledge base. Rewrite for raw Foundry — no SE2-specific paths or tools.

### Contents

1. **Quick reference** (one line each — agents know Foundry, they just need the path):
   - Tests go in `contracts/test/`. Run: `forge test`. Debug: `forge test -vvvv`.

2. **What agents forget to test (the real value of this sub-skill):**
   - Token decimal handling — test with USDC (6), WBTC (8), and DAI (18). Agents almost always hardcode 18.
   - Access control reversions — test that unauthorized callers get reverted, not just that authorized callers succeed
   - The full approve → action flow — test that the contract reverts without approval, succeeds with it
   - Edge cases that lose money — zero amounts, max uint256, empty arrays, re-entrancy from malicious receiver contracts
   - State consistency after failed transactions — partial state changes are the #1 source of exploits
   - Multi-contract interactions — if contract A calls contract B, test the full chain including failure modes

3. **Fork test setup — funding test accounts:**
   - Use `vm.deal(address, amount)` to give ETH to any address in tests
   - Use `vm.prank(whaleAddress)` + `token.transfer()` to get ERC-20 tokens from real holders
   - For persistent impersonation across multiple calls: `vm.startPrank(whaleAddress)` ... `vm.stopPrank()`
   - Always reset state in `setUp()` — fork tests share the same snapshot

4. **Fuzz testing (agents skip this by default — don't let them):**
   - Every function that takes a `uint256` should be fuzzed
   - Invariant tests for protocol-level properties (e.g., "total shares never exceeds total assets", "sum of all balances equals contract balance")
   - Fuzz runs should be ≥1000 iterations (`[fuzz] runs = 1000` in foundry.toml)

5. **When things go wrong:**
   - Cryptic revert with no message → `forge test -vvvv` for full stack trace
   - Fork tests timeout → RPC URL invalid or rate-limited, try `https://eth.llamarpc.com`
   - Tests pass individually but fail together → shared state, ensure `setUp()` resets everything
   - Fuzz finds failure → failing input is printed, write a regression unit test with that exact input

6. **Phase exit criteria:** All tests pass. Fuzz tests run ≥1000 iterations. Edge cases from checklist above are covered.

## Phase 3: Security Sub-Skill

**File:** `security/SKILL.md`
**Focus:** Audit/review checklist run against finished contracts. This is NOT "write security code" (that's baked into contracts phase). This is "verify nothing slipped through."
**Conceptual source:** Extract security patterns and historical hack references from ethereum-wingman's knowledge base. These are framework-agnostic — no rewriting needed.

**Note on intentional overlap:** The contracts sub-skill has gotchas baked in as writing rules. This checklist deliberately repeats them as verification items PLUS adds items that go beyond the writing phase (flash loans, cross-function reentrancy, access control coverage). The repetition is intentional — when the agent loads this sub-skill, the contracts sub-skill is no longer in context. The agent needs the full checklist here.

This is the thinnest sub-skill (~60-70 lines). Its value is in forcing the agent to explicitly pause and review, not in content volume.

### Contents

1. **Audit checklist** (the agent runs through each item against the actual code):
   - [ ] Access control on ALL admin/privileged functions (not just the obvious ones)
   - [ ] Reentrancy protection — CEI + nonReentrant on all external calls
   - [ ] Cross-function reentrancy — can calling function A during function B's external call cause issues?
   - [ ] Token decimal handling verified (not hardcoding 18)
   - [ ] Oracle usage — Chainlink with staleness checks, not DEX spot prices
   - [ ] Return values checked — using SafeERC20 for all token operations
   - [ ] Integer math — no division before multiplication, basis points not raw percentages
   - [ ] Input validation on all external function parameters
   - [ ] Events emitted for all state changes
   - [ ] Approval hygiene — exact approvals preferred, `type(uint256).max` only for trusted protocols with documented justification
   - [ ] Incentive design — who calls each maintenance function and why?
   - [ ] First depositor protection for vault/pool contracts
   - [ ] Flash loan resistance — can any state be manipulated by borrowing + acting + repaying in one tx?
   - [ ] Frontrunning — can a pending transaction be profitably frontrun?

2. **Historical hacks as cautionary patterns** (one line each — just enough to reinforce why each checklist item matters):
   - The DAO (2016): reentrancy → $60M drained
   - Euler (2023): unprotected `donateToReserves` allowed debt manipulation → $197M
   - Cream Finance (2021): oracle manipulation via flash loan → $130M
   - Nomad Bridge (2022): missing input validation → $190M
   - Harvest Finance (2020): flash loan price manipulation → $34M

3. **When in doubt:** Can't determine if a checklist item applies → err on the side of implementing the protection. The cost of an unnecessary guard is near zero; the cost of a missing one is catastrophic.

4. **Phase exit criteria:** Every checklist item explicitly passes or has documented justification for why it doesn't apply. If any item fails → loop back to contracts phase, fix, re-test, re-audit.

## Phase 4: Frontend Sub-Skill

**File:** `frontend/SKILL.md`
**Focus:** Building a Next.js (TypeScript) + wagmi v2 + viem frontend that correctly handles onchain interactions.
**Conceptual source:** Extract transaction UX patterns (approve flows, loading states, confirmation handling) from ethereum-wingman's knowledge base. Rewrite all hook/component references for raw wagmi v2 — no SE2-specific hooks or components.

### Contents

1. **Provider setup** (agents routinely get this wrong with wagmi v2 — include explicitly):
   - Create `frontend/lib/wagmi.ts`:
     ```typescript
     import { getDefaultConfig } from "@rainbow-me/rainbowkit";
     import { foundry } from "viem/chains"; // for local dev; swap for mainnet/base in production

     export const config = getDefaultConfig({
       appName: "My dApp",
       projectId: process.env.NEXT_PUBLIC_WALLETCONNECT_PROJECT_ID!,
       chains: [foundry],  // chainId 31337 — matches Anvil fork
       ssr: true,
     });
     ```
   - Create `frontend/app/providers.tsx` (must be `"use client"`):
     ```typescript
     "use client";
     import { WagmiProvider } from "wagmi";
     import { RainbowKitProvider } from "@rainbow-me/rainbowkit";
     import { QueryClient, QueryClientProvider } from "@tanstack/react-query";
     import { config } from "@/lib/wagmi";
     import "@rainbow-me/rainbowkit/styles.css";

     const queryClient = new QueryClient();

     export function Providers({ children }: { children: React.ReactNode }) {
       return (
         <WagmiProvider config={config}>
           <QueryClientProvider client={queryClient}>
             <RainbowKitProvider>
               {children}
             </RainbowKitProvider>
           </QueryClientProvider>
         </WagmiProvider>
       );
     }
     ```
   - Wrap children in root `layout.tsx` with `<Providers>{children}</Providers>`
   - Without this wrapping, every wagmi hook throws `WagmiProviderNotFoundError`

2. **ABI + address wiring:**
   - Import `deployedContracts` from `@/contracts/deployedContracts` (created in Phase 1)
   - For external protocols: add their ABI + address to the same directory, same `as const` shape
   - Use wagmi's `useReadContract` / `useWriteContract` with the imported ABI + address
   - Type safety comes from viem's ABI inference — the `as const` on ABIs is mandatory

3. **Transaction UX rules (hard rules, not suggestions):**
   - **Always wait for confirmation.** After `writeContractAsync`, use `useWaitForTransactionReceipt` hook (pass the tx hash) OR import `waitForTransactionReceipt` from `@wagmi/core` and call with `waitForTransactionReceipt(config, { hash })`. Never show success after signing — only after the tx is mined.
   - Every onchain button: disable on click + show loader + stay disabled until confirmation
   - Separate loading state per button — NEVER a shared `isLoading`
   - Approve flow — two-step: first check `allowance()`, then either show "Approve" or "Action" button. Never both.
   - Display addresses with ENS resolution where possible (`useEnsName`)
   - Show USD values next to all token/ETH amounts
   - Reject transaction in wallet → catch the error, re-enable the button, show message

4. **Common patterns:**
   - Read contract state: `useReadContract({ address, abi, functionName, args })`
   - Write to contract: `const { writeContractAsync } = useWriteContract()` → `const hash = await writeContractAsync({ address, abi, functionName, args })` → use `useWaitForTransactionReceipt({ hash })` to track confirmation
   - Watch for events: `useWatchContractEvent({ address, abi, eventName, onLogs })`
   - Format token amounts: always use `formatUnits(amount, decimals)` and `parseUnits(input, decimals)` from `viem` — never hardcode 18

5. **Development chain config:**
   - Anvil fork defaults to chainId 31337, regardless of which chain is forked. The wagmi config must use the `foundry` chain from `viem/chains` (which is chainId 31337) during development.
   - For production, swap `foundry` for `mainnet`, `base`, etc. Use an environment variable to toggle: `const chain = process.env.NEXT_PUBLIC_CHAIN === "mainnet" ? mainnet : foundry`

6. **Browser testing checklist** (final verification):
   - App loads at localhost:3000
   - Wallet connects successfully
   - Wrong network → prompt to switch (RainbowKit handles this by default)
   - Approve flow → button shows "Approving...", disables, then transitions to action button
   - Main action → button disables, shows loader, state updates after confirmation
   - Reject transaction in wallet → UI recovers gracefully, button re-enables

7. **When things go wrong:**
   - `WagmiProviderNotFoundError` → provider wrapping missing or not `"use client"`
   - Hook not returning data → check ABI matches deployed contract, check address is correct, check chainId matches
   - Transaction pending forever → check Anvil is running, block mining is enabled
   - "Chain not configured" → wagmi config missing the target chain (use `foundry` for local dev)
   - Hydration errors → wrap wallet-dependent UI in a client component with `useEffect` mount guard

8. **Phase exit criteria:** Full user journey works on localhost with real wallet. All buttons have loading states. Approve flow correct. Transaction confirmation is awaited before showing success.

## Phase 5: Deploy Sub-Skill

**File:** `deploy/SKILL.md`
**Focus:** Production deployment of contracts and frontend. Going live.
**Conceptual source:** Extract deployment patterns and production config from ethereum-wingman's knowledge base. Rewrite all commands for raw Foundry + Vercel — no SE2 wrappers.

**Note:** This sub-skill covers both contract deployment and frontend deployment. For contracts-only projects (no UI), skip the frontend deployment sections. The file is structured with clear headers so the agent can skip what doesn't apply.

### Contents

1. **Key management for production:**
   - **Recommended:** Use Foundry keystores: `cast wallet import deployer --interactive` (enter private key, set password). Then deploy with `--account deployer` instead of `--private-key`.
   - **Alternative:** Store `DEPLOYER_PRIVATE_KEY` in `.env` (NOT `.env.local` in frontend — this is the contracts `.env`). Load in deploy script with `vm.envUint("PRIVATE_KEY")` or pass via `--private-key $DEPLOYER_PRIVATE_KEY`.
   - NEVER commit private keys. Add `.env` and keystores to `.gitignore`.

2. **Contract deployment — three paths:**

   **Before deploying:** Estimate cost with `forge script script/Deploy.s.sol --rpc-url $RPC` (dry run, no `--broadcast`). Tell the user the approximate USD cost at current gas prices before spending real ETH.

   **Option A: Straight to mainnet** (simple contracts, confident after local fork testing):
   - Fund deployer wallet with real ETH
   - Deploy: `forge script script/Deploy.s.sol --rpc-url $MAINNET_RPC --broadcast --verify --etherscan-api-key $KEY --account deployer`
   - The `--verify` flag auto-verifies on Etherscan in the same step
   - Test with real wallet, small amounts ($1-10)

   **Option B: Testnet first, then mainnet** (complex contracts, multi-party testing, or first-time deployers):
   - Deploy to Sepolia: `forge script script/Deploy.s.sol --rpc-url $SEPOLIA_RPC --broadcast --verify --account deployer`
   - Test with team / collaborators on testnet
   - Then repeat for mainnet once confident
   - Sepolia ETH is free via faucets

   **Option C: Deploy to an L2** (Base, Arbitrum, Optimism — lower fees, same Solidity):
   - L2 deployment uses the exact same Foundry toolchain — contracts don't change
   - Deploy: `forge script script/Deploy.s.sol --rpc-url $BASE_RPC --broadcast --account deployer`
   - Verify: `forge verify-contract <address> <Contract> --chain base --etherscan-api-key $BASESCAN_KEY`. If `--chain <name>` doesn't resolve, use `--verifier-url https://api.basescan.org/api` explicitly.
   - **Key differences from mainnet:** Gas is 100-1000x cheaper. Block times differ. Some precompiles may differ — test on fork first: `anvil --fork-url $BASE_RPC`
   - **When to choose L2 over mainnet:** High-frequency user interactions, gas-sensitive apps, gaming, social. Choose mainnet for DeFi composability with mainnet protocols, or when deep liquidity matters.

3. **Update frontend for production:**
   - Update `frontend/contracts/deployedContracts.ts` — add the production chainId entry (e.g., `1` for mainnet, `8453` for Base) with the real deployed addresses
   - Update `frontend/lib/wagmi.ts`: swap `foundry` chain for the production chain (`mainnet`, `base`, etc.)
   - Set RPC to Alchemy (or Infura) — never ship with public RPCs
   - Remove any dev-only wallet connectors

4. **Frontend deployment to Vercel (recommended):**
   - `cd frontend && vercel` — standard Next.js deployment
   - Set environment variables in Vercel dashboard: `NEXT_PUBLIC_WALLETCONNECT_PROJECT_ID`, `NEXT_PUBLIC_ALCHEMY_API_KEY`, `NEXT_PUBLIC_CHAIN=mainnet`
   - No monorepo complexity — the frontend is a standalone Next.js app

5. **Frontend deployment to IPFS (alternative — fully decentralized):**
   - Add `output: "export"` and `trailingSlash: true` to `next.config.ts` (or `next.config.js`/`next.config.mjs` depending on Next.js version)
   - `cd frontend && next build`
   - Upload the `out/` directory to IPFS (Pinata, BuidlGuidl IPFS, or `ipfs add -r`)
   - CID changed = real deploy. Same CID = stale build — always `rm -rf .next out` before rebuilding.

6. **Pre-publish checklist:**
   - OG image (1200x630) with absolute production URL, not localhost
   - Page title/description updated
   - Favicon set
   - Production RPC configured (Alchemy/Infura)
   - No test/dev wallets or hardcoded addresses in frontend code
   - `.env.local` has all required environment variables

7. **ENS subdomain setup (optional):**
   - Create subdomain on app.ens.domains
   - Set IPFS content hash: `ipfs://<CID>`
   - Verify via `.limo` gateway (may cache 5-15 min)

8. **When things go wrong:**
   - "No signer available" → missing `--account` or `--private-key` flag
   - Deploy tx fails on mainnet → check gas estimate, ensure wallet has enough ETH + buffer
   - Verification fails → check Etherscan API key, ensure compiler version matches. For L2s, try `--verifier-url` explicitly.
   - Frontend can't reach contracts → check chain ID + address match between wagmi config and deployed contracts
   - IPFS routes 404 → missing `trailingSlash: true`

9. **Phase exit criteria:** Contracts verified on block explorer. Frontend loads on production URL. OG unfurls correctly. Tested with real wallet.

## Content Strategy

- **Extract and restructure** content from both existing projects (ethereum-wingman-main and ethskills-master)
- **Distill, don't copy-paste** — rewrite for maximum conciseness
- **Only include what LLMs get wrong** — if the LLM already knows it, skip it
- **Verified addresses** are NOT duplicated — agent fetches from `https://ethskills.com/addresses/SKILL.md`
- Sub-skill line targets: **150 lines** for testing/security/deploy, **200 lines** for contracts/frontend (these need more scaffolding detail). Hard max **250 lines** — if a file exceeds this, split it or cut content.

## Default Stack

| Concern | Tool | Why |
|---------|------|-----|
| Smart contracts | **Foundry** (forge, cast, anvil) | 10-100x faster than Hardhat, Solidity-native testing, built-in fuzzing |
| Contract library | **OpenZeppelin v5** | Battle-tested, industry standard |
| Local network | **Anvil fork of mainnet** | Real protocol state, real tokens, real liquidity |
| Frontend | **Next.js (TypeScript) + wagmi v2 + viem + RainbowKit** | Industry standard, LLMs already know it well, type-safe |
| Wallet connection | **RainbowKit** (requires free WalletConnect projectId) | One component, handles all wallets, chain switching built in |
| Frontend hosting | **Vercel** (default) or **IPFS** (decentralized) | Vercel is simplest; IPFS for censorship resistance |
| Contract verification | **forge verify-contract** | Native Foundry integration |
| Default chain | **Ethereum mainnet** (L2s supported: Base, Arbitrum, Optimism) | Deepest liquidity, cheapest it's ever been (~0.05 gwei). L2s for gas-sensitive apps |

## Agent Workflow Summary

### Full Build (default)
```
User: "Build me a staking dApp where users stake ETH and earn rewards"
                              │
                    ┌─────────▼──────────┐
                    │  Meta SKILL.md     │  (always loaded)
                    │  • Detect/setup    │
                    │  • Fork mainnet    │
                    │  • Plan first      │
                    └─────────┬──────────┘
                              │
                    ┌─────────▼──────────┐
                    │  Agent presents    │
                    │  architecture plan │
                    │  (contracts, APIs, │
                    │   deploy order,    │
                    │   frontend pages)  │
                    └─────────┬──────────┘
                              │ user approves
                    ┌─────────▼──────────┐
                    │  Phase 1: Contracts│  → load contracts/SKILL.md
                    │  Write + deploy    │
                    └─────────┬──────────┘
                              │
                    ┌─────────▼──────────┐
                    │  Phase 2: Testing  │  → load testing/SKILL.md
                    │  Foundry tests     │
                    └─────────┬──────────┘
                              │
                    ┌─────────▼──────────┐
                    │  Phase 3: Security │  → load security/SKILL.md
                    │  Audit checklist   │
                    └─────────┬──────────┘
                              │ (loop back to Phase 1 if issues found)
                    ┌─────────▼──────────┐
                    │  Phase 4: Frontend │  → load frontend/SKILL.md
                    │  Build UI + test   │
                    └─────────┬──────────┘
                              │
                    ┌─────────▼──────────┐
                    │  Phase 5: Deploy   │  → load deploy/SKILL.md
                    │  Go to production  │
                    └─────────┬──────────┘
                              │
                            Done.
```

### Partial Workflows
```
"Review my contract for security"              → Meta → security/SKILL.md only
"Build a frontend for my deployed contracts"   → Meta → frontend/SKILL.md only
"Write tests for my existing contracts"        → Meta → testing/SKILL.md only
"Build contracts only, no UI"                  → Meta → Phase 1 → 2 → 3 (skip 4, 5)
"Deploy to production"                         → Meta → deploy/SKILL.md only
"Fix a bug in my contract"                     → Meta → Phase 1 → 2 → 3 (fix, test, re-check)
"Add staking to my existing dApp"              → Meta → Phase 1 → 2 → 3 → 4 → 5 (skip setup)
```

## Error Recovery

Each sub-skill includes a brief "when things go wrong" section. Common failures:

| Phase | Common Failure | Fix |
|-------|---------------|-----|
| Contracts | `forge build` fails with import errors | Check `remappings.txt`: `@openzeppelin/contracts/=lib/openzeppelin-contracts/contracts/` |
| Contracts | "No signer available" on deploy | Add `--private-key <key>` or `--account <keystore>` to `forge script` |
| Contracts | Deploy script reverts | Check constructor args, verify Anvil fork is running, check `vm.startBroadcast()` |
| Testing | `forge test` cryptic revert | Run with `-vvvv` for full trace |
| Testing | Fork tests timeout | Check RPC URL is valid, try a different provider |
| Security | Checklist item fails | Loop back to contracts phase, fix, re-test, re-audit |
| Frontend | `WagmiProviderNotFoundError` | Provider wrapping missing — check providers.tsx and layout.tsx |
| Frontend | Hook not returning data | Check ABI matches deployed contract, check address and chainId |
| Frontend | Tx pending forever | Verify Anvil running + block mining enabled (`cast rpc anvil_setIntervalMining 1`) |
| Deploy | Verification fails | Check Etherscan API key, compiler version. For L2s try `--verifier-url` explicitly |
| Deploy | IPFS routes 404 | Add `trailingSlash: true` to next.config |

## Assumptions

- **Foundry for all contract work.** All contract paths reference Foundry's layout (`src/`, `test/`, `script/`, `out/`, `broadcast/`). This is intentional — Foundry is the best tool for the job.
- **Next.js + wagmi v2 + viem for frontend.** The agent scaffolds this stack directly. LLMs already know these tools well — the skill only corrects what they get wrong (provider setup, tx UX, approve flows, confirmation handling, chainId configuration).

## Non-Goals

- This skill does NOT teach Ethereum fundamentals (what is a blockchain, what is gas). LLMs know this.
- This skill does NOT include contract templates. LLMs can generate patterns; the skill corrects their mistakes.
- This skill does NOT duplicate verified contract addresses. It references ethskills for those.
- This skill is NOT a Solidity tutorial. It's a building playbook.
- This skill does NOT cover post-deployment maintenance (monitoring, upgrades, incident response). Scope ends at "it's live and working."

## Resolved Decisions

1. **No framework template.** The agent scaffolds the project itself using `forge init --no-git` + `create-next-app --yes`. This avoids framework-specific quirks and lets the agent create exactly what's needed.
2. **Size targets:** 150-line target for testing/security/deploy sub-skills, 200-line target for contracts/frontend (need scaffolding detail). 250-line hard max for all.
3. **Packaging:** Start local. Package for skills.sh distribution later (YAML frontmatter + metadata.json).
4. **Versioning:** Include a date stamp in the meta SKILL.md (e.g., "Last updated: February 2026"). Ethereum moves fast — stale skills give stale advice.
