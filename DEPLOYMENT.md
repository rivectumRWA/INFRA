# Deployment — RivectumRWA

## VPS Infrastructure

### Server

| Item | Value |
|------|-------|
| Provider | VPS (Linux) |
| IP | `109.199.103.135` |
| Domain | `app.rivectum.xyz` |
| OS | Ubuntu / Debian (root access) |
| Repo Path | `/root/rivectum/` |

### Process Architecture

```
Internet → :80 (Nginx) → :3000 (Next.js via PM2)
                        → agent (Bun via PM2, no HTTP port)
```

### Nginx Configuration

```nginx
# /etc/nginx/sites-available/rivectum

server {
    listen 80;
    server_name app.rivectum.xyz 109.199.103.135;

    location / {
        proxy_pass http://127.0.0.1:3000;
        proxy_http_version 1.1;
        proxy_set_header Upgrade $http_upgrade;
        proxy_set_header Connection "upgrade";
        proxy_set_header Host $host;
        proxy_set_header X-Real-IP $remote_addr;
        proxy_set_header X-Forwarded-For $proxy_add_x_forwarded_for;
        proxy_set_header X-Forwarded-Proto $scheme;
        proxy_read_timeout 86400;
    }
}
```

### PM2 Processes

| Process | Command | Port | Auto-restart |
|---------|---------|------|--------------|
| `rivectum-web` | `npx next start -p 3000` | 3000 | Yes |
| `rivectum-agent` | `bun run src/agent.ts` | — | Yes |

### Deployment Script (`deploy-vps.sh`)

Located at `/root/rivectum/deploy-vps.sh`. Run on VPS after git push:

```bash
#!/bin/bash
set -e
echo "=== RivectumRWA VPS Deploy ==="
cd /root/rivectum

# 1. Copy env files (preserve existing)
cp -n agent/.env.example agent/.env 2>/dev/null || true
cp -n web/.env.example web/.env.local 2>/dev/null || true

# 2. Install agent deps
cd agent && bun install && cd ..

# 3. Install web deps
cd web && npm install --legacy-peer-deps && cd ..

# 4. Build web
cd web && rm -rf .next && npx next build && cd ..

# 5. Setup PM2
pm2 delete all 2>/dev/null || true
cd web && pm2 start npx --name "rivectum-web" -- start -p 3000 && cd ..
cd agent && pm2 start ~/.bun/bin/bun --name "rivectum-agent" -- run src/agent.ts && cd ..
pm2 save
pm2 startup systemd -u root --hp /root 2>/dev/null || true
```

### Rebuild Script (`rebuild.sh`)

For code updates only (no dependency changes):

```bash
cd /root/rivectum/web
rm -rf .next node_modules/.cache
npx next build >> /root/rivectum/web/build-screen.log 2>&1
pm2 restart rivectum-web
```

## Environment Setup

### VPS Prerequisites

```bash
# Node.js 20+
curl -fsSL https://deb.nodesource.com/setup_20.x | bash -
apt-get install -y nodejs

# Bun
curl -fsSL https://bun.sh/install | bash
export PATH=$HOME/.bun/bin:$PATH

# PM2
npm install -g pm2

# Nginx
apt-get install -y nginx
```

### Required Environment Files

| File | Location | Purpose |
|------|----------|---------|
| `agent/.env` | `/root/rivectum/agent/.env` | Agent config, RPC, keys |
| `web/.env.local` | `/root/rivectum/web/.env.local` | Dashboard config, Reown project ID |

See [ENVIRONMENT.md](./ENVIRONMENT.md) for complete variable reference.

## Deploy Flow

```
Local (dev machine)                    VPS (109.199.103.135)
─────────────────                      ──────────────────────
1. git push origin main         ──▶    2. ssh root@109.199.103.135
                                       3. cd /root/rivectum && git pull
                                       4. Fill agent/.env + web/.env.local
                                       5. bash deploy-vps.sh
                                       6. Verify: curl http://localhost:3000
                                       7. Verify: curl http://app.rivectum.xyz
```

## Monitoring Commands

```bash
# PM2 status
pm2 status
pm2 logs rivectum-web
pm2 logs rivectum-agent

# Check web running
curl -s http://localhost:3000 | head -5

# Check agent DB
sqlite3 /root/rivectum/agent/agent.db "SELECT * FROM decisions ORDER BY timestamp DESC LIMIT 5;"

# Check Nginx
nginx -t
systemctl status nginx
```

## Common Issues

| Issue | Fix |
|-------|-----|
| PM2 processes dead | `pm2 resurrect` or re-run deploy script |
| Build fails | Clear `.next/` + `node_modules/.cache`, check Node version ≥ 20 |
| Agent not rebalancing | Check `AGENT_PRIVATE_KEY` has ETH for gas, check RPC_URL accessible |
| Nginx 502 | PM2 web process not running on port 3000 |
| Wallet connect fails | Verify `NEXT_PUBLIC_WC_PROJECT_ID` set in `.env.local` |
