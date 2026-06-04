# Environment Variables — RivectumRWA

## Overview

Four modules require environment configuration:

| Module | Env File | Runtime |
|--------|----------|---------|
| [`contracts/`](#contracts) | `contracts/.env` | Foundry (deploy only) |
| [`agent/`](#agent) | `agent/.env` | Bun (server) |
| [`web/`](#web) | `web/.env.local` | Next.js (server + client) |
| [`cli/`](#cli) | `cli/.env` | Bun (local) |

> **Prefix convention**: `NEXT_PUBLIC_*` variables are exposed to browser JS. Never put private keys here.

---

## Contracts

**File**: `contracts/.env` (copy from `contracts/.env.example`)

| Variable | Required | Description |
|----------|----------|-------------|
| `BASE_SEPOLIA_RPC_URL` | Yes | RPC endpoint for Base Sepolia (e.g., `https://sepolia.base.org`) |
| `BASESCAN_API_KEY` | Yes | API key for contract verification on BaseScan |
| `DEPLOYER_PRIVATE_KEY` | Yes | EOA private key for deploying contracts (0x-prefixed) |
| `AGENT_DID_ADDRESS` | Yes | Agent EOA address — authorized signer for rebalance intents |
| `USDC_ADDRESS` | Yes | USDC token address on Base Sepolia |
| `UNDERLYING_1` | Yes | First whitelisted ERC-4626 underlying vault address |
| `UNDERLYING_2` | Yes | Second whitelisted ERC-4626 underlying vault address |

---

## Agent

**File**: `agent/.env` (copy from `agent/.env.example`)

| Variable | Required | Description |
|----------|----------|-------------|
| `RPC_URL` | Yes | Base Sepolia RPC endpoint |
| `VAULT_ADDRESS` | Yes | Deployed Vault.sol address |
| `USDC_ADDRESS` | Yes | USDC token address on Base Sepolia |
| `UNDERLYING_1` | Yes | First underlying vault address |
| `UNDERLYING_2` | Yes | Second underlying vault address |
| `AGENT_PRIVATE_KEY` | Yes | Agent EOA private key (must match `agentDid` on-chain) |
| `DB_PATH` | No | SQLite database path (default: `./agent.db`) |
| `REBALANCE_INTERVAL_MS` | No | Cron interval in ms (default: `300000` = 5 min) |

> **Security**: `AGENT_PRIVATE_KEY` is server-side only. Fund this EOA with Base Sepolia ETH for gas.

---

## Web (Dashboard)

**File**: `web/.env.local` (copy from `web/.env.example`)

### Public (Client-side)

| Variable | Required | Description |
|----------|----------|-------------|
| `NEXT_PUBLIC_WC_PROJECT_ID` | Yes | Reown (WalletConnect) project ID for wallet auth |
| `NEXT_PUBLIC_VAULT_ADDRESS` | Yes | Deployed Vault.sol address |
| `NEXT_PUBLIC_USDC_ADDRESS` | Yes | USDC token address on Base Sepolia |
| `NEXT_PUBLIC_RPC_URL` | Yes | Base Sepolia RPC endpoint |
| `NEXT_PUBLIC_UNDERLYING_1` | Yes | First underlying vault address |
| `NEXT_PUBLIC_UNDERLYING_2` | Yes | Second underlying vault address |
| `NEXT_PUBLIC_DEMO` | No | Set to `"1"` for demo seed data (screenshots/demos) |

### Server-side

| Variable | Required | Description |
|----------|----------|-------------|
| `AGENT_DB_PATH` | No | Path to agent SQLite DB (default: `../agent/agent.db`) |
| `OPENAI_API_KEY` | No | OpenAI API key for Serra Copilot (gpt-4o-mini) |

> **Security**: `OPENAI_API_KEY` is server-side only. Never prefix with `NEXT_PUBLIC_`.

---

## CLI

**File**: `cli/.env` (copy from `cli/.env.example`)

### Shared

| Variable | Required For | Description |
|----------|-------------|-------------|
| `RPC_URL` | all | Base Sepolia RPC endpoint |
| `VAULT_ADDRESS` | all | Deployed Vault.sol address |
| `USDC_ADDRESS` | user commands | USDC token address on Base Sepolia |

### Agent Namespace

| Variable | Required For | Description |
|----------|-------------|-------------|
| `AGENT_PRIVATE_KEY` | `agent tick` | Agent signer private key |
| `UNDERLYING_1` | `agent tick` | First underlying vault address |
| `UNDERLYING_2` | `agent tick` | Second underlying vault address |
| `DB_PATH` | `agent decisions`, `agent status` | Path to agent.db (default: `../agent/agent.db`) |

### User Namespace

| Variable | Required For | Description |
|----------|-------------|-------------|
| `USER_PRIVATE_KEY` | `user deposit`, `user withdraw`, `user approve` | User wallet private key |

> **Key Safety**: Read-only commands never load private keys. Agent commands only load `AGENT_PRIVATE_KEY`. User commands only load `USER_PRIVATE_KEY`.

---

## VPS Environment

On the VPS at `109.199.103.135`, the same `agent/.env` and `web/.env.local` files must be populated before running `deploy-vps.sh`.

See [DEPLOYMENT.md](./DEPLOYMENT.md) for the full deploy flow.
