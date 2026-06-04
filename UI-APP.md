# UI App Overview — RivectumRWA Dashboard

## Tech Stack

| Category | Technology |
|----------|-----------|
| Framework | Next.js 15 (App Router) |
| UI Library | React 18 |
| Styling | Tailwind CSS 4 |
| Charts | Recharts 3.8 |
| Icons | Lucide React |
| Wallet | Reown AppKit 1.7.19 + wagmi 2.15 |
| Data Fetching | SWR 2.4 + React Query 5.101 |
| State | React Query cache + wagmi hooks |

## Page Routes

| Route | Page | Description |
|-------|------|-------------|
| `/` | `app/page.tsx` | Dashboard home — KPI strip, vault overview, recent activity |
| `/vault` | `app/vault/page.tsx` | Vault details — TVL, share price, deposit/withdraw |
| `/allocations` | `app/allocations/page.tsx` | Current allocation across underlyings |
| `/activity` | `app/activity/page.tsx` | Agent decision feed (from SQLite via API) |
| `/strategy` | `app/strategy/page.tsx` | Strategy configuration — APY signals, allocation rules |
| `/analytics` | `app/analytics/page.tsx` | Historical performance charts |
| `/risk` | `app/risk/page.tsx` | Risk metrics — concentration, exposure |
| `/registry` | `app/registry/page.tsx` | Whitelisted underlying vaults |
| `/settings` | `app/settings/page.tsx` | Owner settings — pause, agent DID, emergency |
| `/demo` | `app/demo/page.tsx` | Demo mode seed data toggle |

## Component Hierarchy

```
layout.tsx
├── Navigation (Sidebar)
│   ├── logo / brand
│   ├── nav links (Vault, Allocations, Activity, Strategy, Analytics, Risk, Registry, Settings)
│   └── wallet connect button (Reown AppKit)
│
page.tsx (Dashboard Home)
├── KpiStrip (TVL, APY, Share Price, Depositors)
├── DepositCard / WithdrawCard
├── AllocationCard (pie chart — current allocation)
├── ActivityCard (recent decisions feed)
├── PerformanceCard (historical chart)
├── StrategyCard (current strategy signals)
├── UnderlyingsCard (underlying vault list)
├── ContractCard (contract addresses + status)
├── OwnerCard (pause state, agent DID, emergency)
├── OpsStrip (nonce, last rebalance, agent status)
└── ApyCard (APY comparison across underlyings)
```

## Key UI Components

### Deposit Flow
1. Connect wallet (Reown AppKit modal)
2. Approve USDC spending
3. Enter deposit amount
4. Preview shares to receive
5. Confirm transaction

### Withdraw Flow
1. Connected wallet with vault shares
2. Enter withdraw amount (USDC or shares)
3. Preview USDC to receive
4. Confirm transaction

### Activity Feed
- Reads from `/api/decisions` (SQLite agent.db)
- Each entry: timestamp, nonce, old allocation → new allocation, tx hash → BaseScan link
- Auto-refresh via SWR

### Allocation View
- Pie chart: current allocation across underlyings (Recharts)
- Table: per-underlying APY, TVL, share %

### Strategy View
- APY signal cards per underlying
- Current allocation rules (60/40, 60% cap)
- Agent status (running/paused, last tick timestamp)

## API Routes

| Route | Method | Description |
|-------|--------|-------------|
| `/api/decisions` | GET | Agent decision history from SQLite |
| `/api/copilot` | POST | Serra Copilot — OpenAI gpt-4o-mini chat |

## Auth Flow

```
User → Reown AppKit modal → Select wallet (MetaMask, WalletConnect, etc.)
  → Connect to Base Sepolia (chainId: 84532)
  → Dashboard detects wallet address + chain
  → wagmi hooks read vault contract data
  → Deposit/withdraw requires wallet signature + tx confirmation
```

## Demo Mode

When `NEXT_PUBLIC_DEMO=1`:
- Dashboard renders seed data (decisions, allocations, registry)
- Falls back to on-chain data when available
- Useful for screenshots and demos without deployed contracts
