# Backend Overview — RivectumRWA

## Module Architecture

```
┌─────────────────────────────────────────────────┐
│                 SMART CONTRACTS                  │
│          (Solidity 0.8.24 + Foundry)             │
│                                                  │
│  contracts/src/Vault.sol                         │
│  ├── ERC-4626 vault (USDC)                       │
│  ├── signed-intent rebalance                     │
│  ├── whitelist management                        │
│  ├── pause/emergency (Ownable)                   │
│  └── 6000 bps cap per underlying                 │
│                                                  │
│  contracts/src/interfaces/IAllocator.sol         │
│  └── future-compat allocator hook interface      │
│                                                  │
│  contracts/script/Deploy.s.sol                   │
│  └── single-shot Base Sepolia deploy             │
│                                                  │
│  contracts/test/Vault.t.sol                      │
│  └── 10 Foundry unit tests                       │
│                                                  │
│  contracts/test/mocks/                           │
│  ├── MockERC4626.sol  (mock underlying vault)    │
│  └── MockUSDC.sol      (6-decimal mock token)    │
└─────────────────────────────────────────────────┘

┌─────────────────────────────────────────────────┐
│               OFF-CHAIN AGENT                    │
│            (Bun + TypeScript + viem)             │
│                                                  │
│  agent/src/agent.ts    cron loop entrypoint      │
│  agent/src/strategy.ts allocation picker (60/40) │
│  agent/src/sign.ts     keccak intent + ECDSA     │
│  agent/src/chain.ts    viem clients, ABI, config │
│  agent/src/db.ts       Drizzle schema + SQLite   │
│  agent/src/assets.ts   per-network address book  │
│  agent/src/types.ts    Allocation/Intent types   │
│                                                  │
│  agent/test/         8 unit tests (Bun)          │
└─────────────────────────────────────────────────┘

┌─────────────────────────────────────────────────┐
│                 OPERATOR CLI                     │
│            (Bun + TypeScript + viem)             │
│                                                  │
│  cli/src/bin.ts      Commander.js entrypoint     │
│                                                  │
│  Namespaces:                                     │
│  ├── agent status     vault snapshot             │
│  ├── agent decisions  recent agent decisions     │
│  ├── agent tick       dry-run rebalance          │
│  ├── user preview     estimate shares            │
│  ├── user balance     show balances              │
│  ├── user approve     approve USDC               │
│  ├── user deposit     deposit into vault         │
│  └── user withdraw    withdraw from vault        │
└─────────────────────────────────────────────────┘
```

## Smart Contract Details

### Vault.sol

```
Inherits: ERC4626 (OZ), Ownable (OZ)

State:
  - agentDid: address       // EOA authorized to sign rebalance intents
  - whitelistedUnderlying: address[]  // allowed ERC-4626 vaults
  - paused: bool            // emergency pause

Functions:
  - deposit(assets, receiver)       // ERC-4626: deposit USDC → mint shares
  - withdraw(assets, receiver, owner) // ERC-4626: burn shares → return USDC
  - rebalance(intent, sig)          // verify intent + reallocate underlyings
  - addUnderlying(address)          // owner: whitelist new underlying
  - removeUnderlying(address)       // owner: remove from whitelist
  - setAgentDid(address)            // owner: update authorized agent
  - pause() / unpause()            // owner: emergency controls
  - emergencyWithdraw()            // owner: drain all underlyings

Intent struct:
  - nonce: uint256         // replay protection
  - deadline: uint256      // block.timestamp check
  - allocations: Allocation[]  // [(underlying, bps), ...]

Verification:
  - ecrecover(hash(intent), signature) == agentDid
  - sum(bps) == 10000
  - each bps <= 6000
```

### Deploy Script

Deploys: Vault + registers initial underlyings.

```bash
forge script script/Deploy.s.sol --rpc-url base_sepolia --broadcast -vvvv
```

Requires: `DEPLOYER_PRIVATE_KEY`, `AGENT_DID_ADDRESS`, `USDC_ADDRESS`, `UNDERLYING_1`, `UNDERLYING_2`

## Agent Details

### Cron Loop (`agent.ts`)

```
setInterval(async () => {
  1. Check vault nonce
  2. Probe APY from each whitelisted underlying
  3. Compute allocation via strategy.ts
  4. Build intent → sign via sign.ts
  5. Call vault.rebalance(intent, sig)
  6. Log decision to SQLite via db.ts
}, REBALANCE_INTERVAL_MS)  // default 300000 (5 min)
```

### Strategy (`strategy.ts`)

- **Allocation rule**: max-APY allocation with constraints
- **Cap**: 60% (6000 bps) per underlying
- **Split**: 60/40 based on APY ranking
- Fallback to equal split if no APY data

### Signing (`sign.ts`)

```
intent = { nonce, deadline, allocations }
hash = keccak256(abi.encode(intent))
sig = ECDSA.sign(hash, AGENT_PRIVATE_KEY)
// On-chain: ecrecover(hash, sig) == agentDid
```

### Database (`db.ts`)

Drizzle ORM + SQLite (`agent.db`):

| Table | Columns |
|-------|---------|
| decisions | id, timestamp, nonce, oldAllocations (JSON), newAllocations (JSON), txHash |
| vault_state | id, timestamp, tvl, sharePrice, depositorCount |

## CLI Details

See `cli/README.md` for full usage. Key points:

- **Read-only commands** (`status`, `decisions`, `preview`) never load private keys
- **Agent commands** only load `AGENT_PRIVATE_KEY`
- **User commands** only load `USER_PRIVATE_KEY`
- `agent tick` defaults to dry-run; requires `--broadcast --yes` to execute
- Output: colored tables (default) or `--json` for scripting
- Exit codes: 0=success, 1=error, 2=aborted, 3=missing env, 4=RPC error, 5=reverted
