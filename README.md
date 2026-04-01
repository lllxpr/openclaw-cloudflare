# OpenClaw on Cloudflare Containers

Deploy [OpenClaw](https://github.com/anthropics/openclaw) — an open-source AI coding gateway — on [Cloudflare Containers](https://developers.cloudflare.com/containers/) with Workers AI integration, persistent device pairing via KV, and a built-in admin dashboard.

[![Deploy to Cloudflare](https://deploy.workers.cloudflare.com/button)](https://deploy.workers.cloudflare.com/?url=https://github.com/lllxpr/openclaw-cloudflare)

## Features

- **One-click deploy** — OpenClaw running on Cloudflare's global network
- **Workers AI** — Powered by Kimi K2.5 via AI Gateway (no external API keys needed)
- **Persistent pairing** — Device approvals and chat history survive container restarts (KV-backed)
- **Admin dashboard** — Overview stats, device management, CLI terminal (Cloudflare-themed light UI)
- **Auto-onboarding** — Container self-configures on startup with restore from KV

## Architecture

```
Browser ──► Cloudflare Worker ──► Container (OpenClaw Gateway)
                │                       │
                ├── /admin/        Admin Dashboard (HTML)
                ├── /admin/api/*   Management Server (:18700)
                ├── /persist/*     KV Persistence Endpoints
                ├── /openai/v1/*   AI Gateway Proxy → Workers AI
                └── /*             OpenClaw Control UI (:18789)
```

## Prerequisites

- [Node.js](https://nodejs.org/) v18+
- [Cloudflare account](https://dash.cloudflare.com/sign-up) with Containers access (beta)
- [Wrangler CLI](https://developers.cloudflare.com/workers/wrangler/) v4+

## Step-by-Step Setup

### 1. Clone this repo

```bash
git clone https://github.com/lllxpr/openclaw-cloudflare.git
cd openclaw-cloudflare
npm install
```

### 2. Login to Cloudflare

```bash
npx wrangler login
```

### 3. Create a KV namespace

```bash
npx wrangler kv namespace create OPENCLAW_KV
```

Copy the output `id` value and update `wrangler.toml`:

```toml
[[kv_namespaces]]
binding = "OPENCLAW_KV"
id = "<your-kv-namespace-id>"   # ← paste here
```

### 4. Configure Secrets

Copy the example secrets file and fill in your values:

```bash
cp .dev.vars.example .dev.vars
```

Edit `.dev.vars`:

```env
GATEWAY_AUTH_TOKEN=your-secret-token
AI_GATEWAY_ACCOUNT_ID=your-account-id
AI_GATEWAY_ID=your-gateway-id
AI_GATEWAY_AUTH_TOKEN=your-api-token
```

- **`GATEWAY_AUTH_TOKEN`** — A secret string for authenticating with the OpenClaw Chat UI. Generate one with `openssl rand -hex 16`.
- **`AI_GATEWAY_ACCOUNT_ID`** — Your Cloudflare Account ID. Find it at `dash.cloudflare.com` → any zone → Overview → right sidebar.
- **`AI_GATEWAY_ID`** — Your AI Gateway name. Create one at `dash.cloudflare.com` → AI → AI Gateway.
- **`AI_GATEWAY_AUTH_TOKEN`** — Cloudflare API token. Create at My Profile → API Tokens → Create Token (with AI Gateway permission).

For production deployment, set them as secrets:

```bash
npx wrangler secret put GATEWAY_AUTH_TOKEN
npx wrangler secret put AI_GATEWAY_ACCOUNT_ID
npx wrangler secret put AI_GATEWAY_ID
npx wrangler secret put AI_GATEWAY_AUTH_TOKEN
```

> **Note**: `WORKER_URL` is auto-detected from the first incoming request — no manual configuration needed.

### 5. Update `wrangler.toml`

Change the worker name:

```toml
name = "your-worker-name"
```

### 6. Deploy

```bash
npm run deploy
```

First deploy will take a few minutes to pull the container image (~784 MB).

### 7. Access your deployment

- **Admin Dashboard**: `https://<your-worker>.workers.dev/admin/`
- **Chat UI**: `https://<your-worker>.workers.dev/#token=<your-token>`
- **Health check**: `https://<your-worker>.workers.dev/healthz`

### 8. Approve your first device

1. Open the Chat UI link above
2. Open the Admin Dashboard → **Devices** tab
3. Click **Approve** on the pending device
4. Return to Chat UI — it should connect automatically

The approval persists in KV, so you won't need to re-approve after container restarts.

## Admin Dashboard

The admin dashboard at `/admin/` uses a light theme consistent with the [Cloudflare design language](https://eab4f323.moltworker-workshop.pages.dev/) (orange accents, white cards, clean typography). It has three tabs:

| Tab | Description |
|-----|-------------|
| **Overview** | Container stats (uptime, CPU, memory, KV snapshot), container info (image, instance type, model, AI Gateway) |
| **Devices** | Pending pairing requests with approve buttons, paired devices list |
| **CLI** | Run commands inside the container with quick-access buttons and command history |

The header bar also includes **Open Chat** (links to the OpenClaw Control UI) and **Restart** buttons.

## How Persistence Works

OpenClaw stores pairing data and chat sessions in `/home/node/.openclaw/`. Since container filesystems are ephemeral, this project uses Cloudflare KV to persist:

1. **On startup**: Container fetches a base64-encoded tarball from KV and extracts it
2. **On approve**: Snapshot is saved to KV immediately

Persisted directories: `devices/`, `identity/`, `agents/` (chat sessions).

> **Note**: KV has a 25 MB value size limit. For heavy image-based chat usage, consider migrating to R2.

## Project Structure

```
├── src/
│   ├── index.ts          # Worker + Container class + routing + mgmt server
│   └── admin-html.ts     # Admin dashboard HTML (Tailwind CSS)
├── .dev.vars.example      # Secrets template (GATEWAY_AUTH_TOKEN, AI_GATEWAY_AUTH_TOKEN)
├── wrangler.toml          # Cloudflare configuration
├── package.json
├── tsconfig.json
└── README.md
```

## Troubleshooting

**Container won't start?**
- Check `npx wrangler containers list --json` for container state
- Use the Admin Dashboard → **CLI** tab to run diagnostic commands inside the container

**Devices not showing?**
- The devices list caches every 30s. Click **Refresh** in the Devices tab for a fresh read.

**Chat UI shows "pairing required"?**
- Go to Admin → Devices → Approve the pending device

**KV save failing?**
- Check `/persist/status` endpoint
- Verify KV namespace ID in `wrangler.toml` matches your actual namespace

## License

MIT
