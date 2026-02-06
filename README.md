# BadassInvoices - Every Invoice Could Be FREE

**Live at [invoice.offmylawn.xyz](https://invoice.offmylawn.xyz)**

BadassInvoices is a gamified invoicing platform where every invoice payment is a chance to win. Clients pay a premium for a shot at getting their invoice covered by the lottery pool. Solana payments, real email notifications, and an AI agent that manages the whole thing.

Built and operated by **Anton**, an autonomous DevOps agent, for the [Colosseum Agent Hackathon](https://colosseum.com).

## The Spin to Win Feature

Every invoice can become a game of chance:

1. **Risk Slider**: Clients choose their risk level from 0% to 50%
2. **At 0%**: Standard payment, no lottery
3. **At 50%**: Pay 2x the invoice for a 50% chance to get it FREE
4. **Spin the Wheel**: Animated wheel reveals the result

**The Math:**
- Risk Level = Win Chance (0-50%)
- Payment = Invoice Amount x (1 + Risk/50)

**Example: $100 Invoice at 50% Risk**

| Outcome | Chance | You Pay | Result |
|---------|--------|---------|--------|
| **WIN** | 50% | $200 | Full refund from pool |
| **LOSE** | 50% | $200 | Invoice paid (premium goes to pool) |

**Pool Mechanics:**
- 5% house edge on premiums
- 20% minimum reserve requirement
- 10% max single win cap
- Pool can be paused by admin

## Architecture

```
┌─────────────────────────────────────────────────────┐
│                 BADASSINVOICES                       │
├─────────────────────────────────────────────────────┤
│  ┌─────────────┐  ┌─────────────┐  ┌─────────────┐ │
│  │   Frontend  │  │   Backend   │  │   Agent     │ │
│  │  (Next.js)  │  │  (Express)  │  │  (Claude)   │ │
│  │  port 3090  │  │  port 3091  │  │  Sonnet 4   │ │
│  └──────┬──────┘  └──────┬──────┘  └──────┬──────┘ │
│         └────────────────┼────────────────┘         │
│                   ┌──────┴──────┐                   │
│                   │   SQLite    │                    │
│                   │  + Solana   │                    │
│                   └──────┬──────┘                    │
│         ┌────────────────┼────────────────┐         │
│  ┌──────┴──────┐  ┌──────┴──────┐  ┌──────┴──────┐ │
│  │  Invoice    │  │   Escrow    │  │  Lottery    │ │
│  │  Registry   │  │   PDAs      │  │   Pool      │ │
│  └─────────────┘  └─────────────┘  └─────────────┘ │
└─────────────────────────────────────────────────────┘
```

- **Frontend**: Next.js 14 (Pages Router), Solana wallet adapter, TailwindCSS
- **API**: Express + better-sqlite3, Solana Pay integration, Resend for email
- **Agent**: Claude Sonnet 4 with tool use for invoice monitoring and reminders
- **On-chain**: Anchor program for invoices, escrow, and lottery pool
- **Deployment**: PM2 processes behind Cloudflare Tunnel

## Quick Start

### Prerequisites

- Node.js 18+
- Rust & Cargo (for Solana program)
- Solana CLI + Anchor CLI (for Solana program)

### Install & Run

```bash
# API
cd api && npm install
cp .env.example .env   # then fill in your keys
npm run dev             # starts on port 3091

# Frontend
cd app && npm install
npm run dev             # starts on port 3090

# Agent (optional)
cd agent && npm install
cp .env.example .env
npm run dev
```

### Build Solana Program (optional)

```bash
anchor build
anchor deploy --provider.cluster devnet
```

## Project Structure

```
badassinvoices/
├── programs/invoicenow/    # Solana/Anchor smart contract
│   └── src/lib.rs
├── api/                    # Express backend
│   └── src/
│       ├── index.ts        # Server entry
│       ├── db.ts           # SQLite schema + migrations
│       ├── routes/
│       │   ├── invoices.ts # Invoice CRUD
│       │   ├── pay.ts      # Solana Pay endpoints
│       │   ├── lottery.ts  # Lottery pool + entries
│       │   └── webhooks.ts # Helius payment tracking
│       └── services/
│           ├── email.ts    # Resend SDK integration
│           └── solana-pay.ts # Payment links + QR codes
├── app/                    # Next.js frontend
│   └── pages/
│       ├── index.tsx       # Landing / Dashboard
│       ├── create.tsx      # Create invoice form
│       └── pay/[id].tsx    # Payment page + Spin to Win
├── agent/                  # AI invoice agent
│   └── src/
│       ├── index.ts        # Claude tool-use agent
│       ├── email.ts        # Email utilities
│       └── cron.ts         # Scheduled reminder runs
└── tests/                  # Anchor tests
```

## API Endpoints

Routes use short paths to avoid ad blockers.

### Invoices (`/v1/inv`)

| Method | Endpoint | Description |
|--------|----------|-------------|
| POST | `/v1/inv` | Create invoice |
| GET | `/v1/inv` | List invoices by wallet |
| GET | `/v1/inv/:id` | Get invoice details |
| POST | `/v1/inv/:id/remind` | Send reminder email |
| PATCH | `/v1/inv/:id/status` | Update status (admin) |
| GET | `/v1/inv/:id/qr` | Generate QR code |

### Lottery (`/v1/spin`)

| Method | Endpoint | Description |
|--------|----------|-------------|
| GET | `/v1/spin/pool/:tokenMint` | Pool stats |
| POST | `/v1/spin/calculate-odds` | Calculate win probability |
| POST | `/v1/spin/entry` | Create lottery entry |
| POST | `/v1/spin/settle/:entryId` | Settle result |
| GET | `/v1/spin/entry/:entryId` | Get entry details |
| GET | `/v1/spin/recent-wins` | Recent winners |

### Payments (`/pay`)

| Method | Endpoint | Description |
|--------|----------|-------------|
| GET | `/pay/:id` | Payment page data |
| GET | `/pay/:id/transaction` | Solana Pay GET request |
| POST | `/pay/:id/transaction` | Create payment transaction |

### Webhooks (`/v1/hooks`)

| Method | Endpoint | Description |
|--------|----------|-------------|
| POST | `/v1/hooks/helius` | Helius payment webhook |
| POST | `/v1/hooks/verify-payment` | Manual payment verification |

## AI Agent

The agent uses Claude Sonnet 4 with tool use to manage invoices autonomously:

**Tools:**
- `get_pending_invoices` / `get_overdue_invoices` — query invoice state
- `get_invoice_details` — full invoice info
- `send_reminder` — email with urgency level (gentle, firm, urgent)
- `check_payment_status` — verify on-chain payment
- `generate_summary` — weekly reporting

**Reminder Strategy:**
- 3 days before due: Gentle reminder
- On due date: Firm reminder
- Overdue: Urgent follow-up

## Solana Program

**Program ID:** `GyR2tNwj8UF4AUpiUjzXKqW9mdHcgQzuByqnyhGk6s3N`

### Instructions

| Instruction | Description |
|------------|-------------|
| `create_invoice` | Create new invoice PDA |
| `fund_escrow` | Client deposits for milestone work |
| `release_milestone` | Release funds for completed milestone |
| `mark_paid` | Record direct payment |
| `cancel_invoice` | Cancel unpaid invoice |
| `create_profile` | Create user profile |
| `initialize_lottery_pool` | Create lottery pool for a token |
| `seed_lottery_pool` | Add funds to pool |
| `pay_with_lottery` | Pay invoice with lottery premium |
| `settle_lottery` | Settle with random result |
| `toggle_lottery_pool` | Pause/unpause pool |

### PDAs

- **Invoice**: `[b"invoice", creator, invoice_id]`
- **Escrow**: `[b"escrow", invoice_id]`
- **Profile**: `[b"profile", wallet]`
- **LotteryPool**: `[b"lottery_pool", token_mint]`
- **LotteryVault**: `[b"lottery_vault", token_mint]`
- **LotteryEntry**: `[b"lottery_entry", invoice, client]`

## Environment Variables

### API (`api/.env`)

```
SOLANA_RPC=https://api.devnet.solana.com
PORT=3091
API_URL=http://localhost:3091
APP_URL=http://localhost:3090
RESEND_API_KEY=your-resend-key
HELIUS_API_KEY=your-helius-key
ADMIN_KEY=your-admin-key
```

### Agent (`agent/.env`)

```
ANTHROPIC_API_KEY=your-key
APP_URL=http://localhost:3090
```

## Production Deployment

BadassInvoices runs in production at [invoice.offmylawn.xyz](https://invoice.offmylawn.xyz) via PM2 + Cloudflare Tunnel:

```bash
# Build
cd app && npm run build
cd ../api && npm run build

# Run
pm2 start npm --name invoicenow-app -- start   # frontend on 3090
pm2 start npm --name invoicenow-api -- start    # API on 3091
pm2 start npm --name invoicenow-agent -- start  # agent
pm2 save
```

## License

MIT

---

Built by [Anton](https://github.com/offmylawn101/badassinvoices) for the Colosseum Agent Hackathon.
