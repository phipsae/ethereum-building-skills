---
name: ethereum-building/frontend
description: Phase 4 — Next.js + wagmi v2 + viem frontend with correct transaction UX.
---

# Phase 4: Frontend

Build the frontend. The contracts are deployed on your local fork. The ABIs are in `frontend/contracts/deployedContracts.ts`. Now wire them to a UI.

## Provider Setup

You MUST set up providers correctly or every hook fails with `WagmiProviderNotFoundError`. This is the #1 scaffolding failure.

**1. Create `frontend/lib/wagmi.ts`:**
```typescript
import { getDefaultConfig } from "@rainbow-me/rainbowkit";
import { foundry } from "viem/chains";

export const config = getDefaultConfig({
  appName: "My dApp",
  projectId: process.env.NEXT_PUBLIC_WALLETCONNECT_PROJECT_ID!,
  chains: [foundry], // chainId 31337 — matches your Anvil fork
  ssr: true,
});
```

**2. Create `frontend/app/providers.tsx`:**
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
        <RainbowKitProvider>{children}</RainbowKitProvider>
      </QueryClientProvider>
    </WagmiProvider>
  );
}
```

**3. Wrap in `frontend/app/layout.tsx`:**
```typescript
import { Providers } from "./providers";

export default function RootLayout({ children }: { children: React.ReactNode }) {
  return (
    <html lang="en">
      <body>
        <Providers>{children}</Providers>
      </body>
    </html>
  );
}
```

**4. Add wallet connect button** anywhere in a page:
```typescript
import { ConnectButton } from "@rainbow-me/rainbowkit";
// Then in JSX: <ConnectButton />
```

## ABI + Address Wiring

Import from the `deployedContracts.ts` you created in Phase 1:
```typescript
import { deployedContracts } from "@/contracts/deployedContracts";

const contract = deployedContracts[31337].MyContract;
// contract.address and contract.abi are fully typed
```

For external protocols (tokens, Uniswap, etc.), add their ABI + address to the same directory with the same `as const` shape.

## Transaction UX — Hard Rules

These are not suggestions. Every onchain interaction must follow these rules:

### 1. Always Wait for Confirmation
Never show "Success" after signing. Only after the tx is mined:

```typescript
import { useWriteContract, useWaitForTransactionReceipt } from "wagmi";

const { writeContractAsync } = useWriteContract();
const [txHash, setTxHash] = useState<`0x${string}` | undefined>();
const { isLoading: isConfirming, isSuccess } = useWaitForTransactionReceipt({ hash: txHash });

async function handleAction() {
  const hash = await writeContractAsync({
    address: contract.address,
    abi: contract.abi,
    functionName: "stake",
    args: [amount],
  });
  setTxHash(hash);
}
// isConfirming = true while mining, isSuccess = true when done
```

### 2. Button States
Every onchain button: **disable on click → show loader → stay disabled until confirmation.**
```typescript
<button onClick={handleAction} disabled={isConfirming}>
  {isConfirming ? "Confirming..." : "Stake"}
</button>
```

Separate loading state PER BUTTON. Never a shared `isLoading`.

### 3. Approve Flow
Two-step, one button at a time:
1. Check `allowance()` — if insufficient, show "Approve" button
2. After approval confirms, show the action button ("Stake", "Swap", etc.)
3. Never show both buttons simultaneously

### 4. Error Handling
Reject in wallet → catch the error, re-enable the button:
```typescript
try {
  const hash = await writeContractAsync({ ... });
  setTxHash(hash);
} catch (err) {
  // User rejected or tx failed — button re-enables automatically
  console.error(err);
}
```

### 5. Display Rules
- Addresses: use `useEnsName` for ENS resolution where possible
- Token/ETH amounts: always `formatUnits(amount, decimals)` from `viem` — never hardcode 18
- Show USD values next to all amounts

## Common Patterns

**Read contract state:**
```typescript
const { data } = useReadContract({
  address: contract.address,
  abi: contract.abi,
  functionName: "totalStaked",
});
```

**Watch for events:**
```typescript
useWatchContractEvent({
  address: contract.address,
  abi: contract.abi,
  eventName: "Staked",
  onLogs(logs) { /* update UI */ },
});
```

**Format amounts:**
```typescript
import { formatUnits, parseUnits } from "viem";
const display = formatUnits(rawAmount, 6); // USDC: 6 decimals
const onchain = parseUnits(userInput, 6);
```

## Development Chain Config

- Anvil fork = chainId 31337. Your wagmi config uses `foundry` from `viem/chains`.
- For production, swap to the real chain. Use an env var:
  ```typescript
  import { foundry, mainnet, base } from "viem/chains";
  const chain = process.env.NEXT_PUBLIC_CHAIN === "mainnet" ? mainnet
    : process.env.NEXT_PUBLIC_CHAIN === "base" ? base
    : foundry;
  ```

## Browser Testing Checklist

Before moving to deploy, verify manually:

- [ ] App loads at localhost:3000
- [ ] Wallet connects (RainbowKit modal opens, account shows)
- [ ] Wrong network → prompt to switch (RainbowKit handles this)
- [ ] Approve flow → "Approving..." state, then transitions to action button
- [ ] Main action → button disables, loader shows, state updates after confirmation
- [ ] Reject in wallet → UI recovers, button re-enables

## When Things Go Wrong

| Problem | Fix |
|---------|-----|
| `WagmiProviderNotFoundError` | Provider wrapping missing — check `providers.tsx` is `"use client"` and `layout.tsx` wraps children |
| Hook returns undefined | Check ABI matches deployed contract, check address, check chainId matches (31337 for fork) |
| Tx pending forever | Anvil not running or block mining not enabled — `cast rpc anvil_setIntervalMining 1` |
| "Chain not configured" | Wagmi config missing the chain — use `foundry` from `viem/chains` for local dev |
| Hydration errors | Wrap wallet-dependent UI in a client component with `useEffect` mount guard |

## Exit Criteria

- [ ] Full user journey works on localhost with wallet connected
- [ ] All buttons have loading states (no raw clicks)
- [ ] Approve flow works (two-step)
- [ ] Success only shown after tx confirmation, not after signing

Next → load `deploy/SKILL.md`
