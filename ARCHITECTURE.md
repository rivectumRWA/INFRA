# Architecture — RivectumRWA

## System Overview

```
┌──────────────────────────────────────────────────────────────────────┐
│                        RivectumRWA System                            │
│                                                                      │
│  ┌──────────────────┐     ┌──────────────────┐     ┌──────────────┐ │
│  │   USER WALLET    │     │   WEB DASHBOARD  │     │   OPERATOR   │ │
│  │  (MetaMask/Any)  │     │  (Next.js 15)    │     │   (CLI)      │ │
│  └────────┬─────────┘     └────────┬─────────┘     └──────┬───────┘ │
│           │ deposit/withdraw      │ reads on-chain       │ admin    │
│           │ ERC-4626              │ state + agent DB     │ ops      │
│           ▼                       ▼                      ▼          │
│  ┌──────────────────────────────────────────────────────────────┐   │
│  │                    BASE SEPOLIA NETWORK                       │   │
│  │  ┌───────────────────────────────────────────────────────┐   │   │
│  │  │  VAULT.sol (ERC-4626)                                 │   │   │
│  │  │  - deposit/withdraw USDC                              │   │   │
│  │  │  - rebalance(intent, sig) → allocate across underlyings│   │   │
│  │  │  - whitelist underlying vaults                         │   │   │
│  │  │  - pause/emergency owner controls                      │   │   │
│  │  └───────────────────────────────────────────────────────┘   │   │
│  │                              │                                │   │
│  │         ┌────────────────────┼────────────────────┐          │   │
│  │         ▼                    ▼                    ▼          │   │
│  │  ┌─────────────┐    ┌─────────────┐    ┌─────────────┐      │   │
│  │  │ USDC Token  │    │ Underlying1 │    │ Underlying2 │      │   │
│  │  │ (6 dec)     │    │ (ERC-4626)  │    │ (ERC-4626)  │      │   │
│  │  └─────────────┘    └─────────────┘    └─────────────┘      │   │
│  └──────────────────────────────────────────────────────────────┘   │
│                                                                      │
│  ┌──────────────────────────────────────────────────────────────┐   │
│  │                  OFF-CHAIN AGENT (Bun)                        │   │
│  │   ┌──────────┐  ┌──────────┐  ┌──────────┐  ┌──────────┐   │   │
│  │   │ APY      │  │ Strategy │  │ Intent   │  │ Decision │   │   │
│  │   │ Probe    │─▶│ Picker   │─▶│ Signer   │─▶│ Logger   │   │   │
│  │   │          │  │ (60/40)  │  │ (ECDSA)  │  │ (SQLite) │   │   │
│  │   └──────────┘  └──────────┘  └──────────┘  └──────────┘   │   │
│  │                                      │                        │   │
│  │                                      ▼                        │   │
│  │                         ┌────────────────────┐               │   │
│  │                         │  VAULT.rebalance() │               │   │
│  │                         │  (signed intent)   │               │   │
│  │                         └────────────────────┘               │   │
│  └──────────────────────────────────────────────────────────────┘   │
│                                                                      │
│  ┌──────────────────────────────────────────────────────────────┐   │
│  │                   WEB API (Next.js)                           │   │
│  │   /api/decisions  → reads agent SQLite DB (activity feed)     │   │
│  │   /api/copilot    → OpenAI gpt-4o-mini (Serra Copilot)        │   │
│  └──────────────────────────────────────────────────────────────┘   │
│                                                                      │
│  ┌──────────────────────────────────────────────────────────────┐   │
│  │                   VPS INFRASTRUCTURE                          │   │
│  │   ┌──────────┐    ┌──────────────┐    ┌──────────────┐      │   │
│  │   │ Nginx    │───▶│ PM2: web     │    │ PM2: agent   │      │   │
│  │   │ :80→:3000│    │ (port 3000)  │    │ (cron loop)  │      │   │
│  │   └──────────┘    └──────────────┘    └──────────────┘      │   │
│  └──────────────────────────────────────────────────────────────┘   │
└──────────────────────────────────────────────────────────────────────┘
```

## Data Flow — Rebalance Cycle

```
 1. Agent cron triggers (REBALANCE_INTERVAL_MS, default 5 min)
 2. APY Probe reads totalAssets() from each whitelisted underlying
 3. Strategy Picker selects allocation (60/40 split, 60% cap per asset)
 4. Compute new allocations in bps (e.g., 6000 | 4000)
 5. Build Intent { nonce, deadline, allocations[] }
 6. Hash intent via keccak256 → ECDSA sign with AGENT_PRIVATE_KEY
 7. Call Vault.rebalance(intent, sig)
 8. Vault verifies: signer == agentDid, nonce valid, deadline not expired
 9. Vault executes: redeemAll underlyings → deposit to new allocations
10. Log decision to SQLite (agent.db) — visible in web /api/decisions
```

## Data Flow — User Deposit/Withdraw

```
 1. User connects wallet via Reown AppKit (Base Sepolia)
 2. User approves USDC spending for Vault
 3. User calls Vault.deposit(assets, receiver)
 4. Vault mints ERC-4626 shares to receiver
 5. Next agent tick detects new balance → triggers rebalance
 6. Dashboard reads on-chain (wagmi) + off-chain (SQLite via API)
```

## Key Design Decisions

| Decision | Rationale |
|----------|-----------|
| ECDSA (secp256k1) for intents | Single `ecrecover` on-chain — native EVM, no precompile |
| SQLite for agent decisions | Zero infra, file-based, read by Next.js API route |
| PM2 process management | Simple, auto-restart, systemd integration on boot |
| Nginx reverse proxy | SSL termination (future), port 80→3000 proxy |
| Bun runtime for agent/CLI | Fast startup, native TS, no build step |
| Foundry for contracts | Fast compile/test cycle, fuzzing, Solidity scripting |
