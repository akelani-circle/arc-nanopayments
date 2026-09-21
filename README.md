# Arc Nanopayments

Demonstrate gasless USDC nanopayments using [Circle Nanopayments](https://www.circle.com/nanopayments) on Arc. A **payment agent script** acts as the buyer, paying for paywalled resources in a loop, while a **Next.js web app** acts as the seller, exposing x402-protected endpoints and providing a seller dashboard to monitor payments and withdraw earnings.

Circle Gateway batches many signed offchain authorizations into a single onchain settlement, enabling economically viable sub-cent payments.

<img alt="Arc Nanopayments dashboard" src="public/screenshot.png" />

## Table of Contents

- [Features](#features)
- [Prerequisites](#prerequisites)
- [Getting Started](#getting-started)
- [How It Works](#how-it-works)
- [Paywalled Endpoints](#paywalled-endpoints)
- [Upgrading](#upgrading)
- [Environment Variables](#environment-variables)
- [User Accounts](#user-accounts)
- [Available Scripts](#available-scripts)
- [Testing](#testing)
- [Security & Usage Model](#security--usage-model)

## Features

- **Paywalled API** (`/api/premium/*`) — Four x402-protected endpoints priced from $0.0003 to $0.03 in USDC. See [Paywalled Endpoints](#paywalled-endpoints).
- **Payment agent** (`agent.mts`) — Funds a fresh wallet from the buyer wallet, deposits USDC into Gateway, and pays the endpoints about once per second. Supports a spending limit.
- **Sign in** (`/`) — Demo login for the seller dashboard. See [User Accounts](#user-accounts).
- **Payments table** (`/dashboard`) — Real-time list of incoming nanopayments with filtering and sorting, linked to the [Arc Testnet Explorer](https://testnet.arcscan.app).
- **Gateway balance** (`TopBarGatewayControls`) — Top-bar badge with the seller's available Gateway balance, plus a dialog with total, withdrawing, withdrawable, and wallet USDC balances.
- **Withdraw** (`WithdrawDialog`) — Withdraw available USDC from Gateway to an address on any supported testnet (Arc Testnet, Base Sepolia, Ethereum Sepolia, Arbitrum Sepolia, Optimism Sepolia, Avalanche Fuji, Polygon Amoy).

## Prerequisites

- **Node.js v22+** — Install via [nvm](https://github.com/nvm-sh/nvm) (`nvm use` reads the `.nvmrc` file)
- **Docker Desktop** — [Install Docker Desktop](https://www.docker.com/products/docker-desktop/)

## Getting Started

1. Clone the repository and install dependencies:

   ```bash
   git clone git@github.com:akelani-circle/arc-nanopayments.git
   cd arc-nanopayments
   npm install
   ```

2. Set up environment variables:

   ```bash
   cp .env.example .env.local
   ```

   Then edit `.env.local` and fill in all required values (see [Environment Variables](#environment-variables) section below).

3. Generate seller and buyer wallets:

   ```bash
   npm run generate-wallets
   ```

   This creates two EVM wallets (seller and buyer) and writes the seller address and both private keys to `.env.local`. It also adds a random `SESSION_SECRET` (used to sign dashboard sessions) if you do not have one yet. Follow the on-screen instructions to fund the buyer wallet with testnet USDC via the [Circle faucet](https://faucet.circle.com/).

4. Set up the local Supabase database (requires Docker Desktop installed and running):

   ```bash
   npm run db:start
   ```

   This starts Supabase in Docker and applies the migrations in `supabase/migrations`. The output shows the Supabase URL and API keys needed for your `.env.local`; run `npm run db:status` to see them again.

5. Start the development server:

   ```bash
   npm run dev
   ```

   The app will be available at `http://localhost:3000`.

6. Run the payment agent:

   ```bash
   npm run agent
   ```

   The agent creates a throwaway wallet, funds it with gas and USDC from the buyer wallet, deposits `DEPOSIT_AMOUNT` USDC into Gateway, and then pays the x402-protected endpoints in turn, about once per second, on Arc Testnet. It tops up Gateway when the balance drops below 0.5 USDC. You can run several agents in parallel.

   To set a USDC spending limit, use the `--limit` flag. The agent will pause when the limit is reached and prompt for additional allowance:

   ```bash
   npm run agent -- --limit 0.5
   ```

## How It Works

- Built with [Next.js](https://nextjs.org/) App Router and [Supabase](https://supabase.com/)
- Uses the [x402 protocol](https://www.x402.org/) for HTTP 402 nanopayments with USDC on the [Arc Network](https://arc.circle.com/)
- Uses [Circle's x402 batching SDK](https://www.npmjs.com/package/@circle-fin/x402-batching) (`GatewayClient`) for gasless payment facilitation
- Includes a payment agent script that uses `GatewayClient` to deposit USDC into Gateway and pay x402-protected resources
- Seller dashboard with real-time payment monitoring, Gateway balance display, and cross-chain withdrawal support
- Payment events and withdrawals are persisted to Supabase with real-time subscriptions
- Styled with [Tailwind CSS](https://tailwindcss.com) and components from [shadcn/ui](https://ui.shadcn.com/)

## Paywalled Endpoints

The seller exposes several x402-protected API routes at different price points:

| Endpoint | Method | Price (USDC) | Description |
| --- | --- | --- | --- |
| `/api/premium/quote` | GET | $0.001 | Returns a premium inspirational quote |
| `/api/premium/dataset` | GET | $0.01 | Returns a small JSON analytics dataset |
| `/api/premium/compute` | POST | $0.0003 | Performs text analysis on submitted content |
| `/api/premium/agent-task` | GET | $0.03 | Returns a clue/step for a treasure hunt task |

Each endpoint returns `402 Payment Required` for unpaid requests. The buyer agent automatically signs the authorization and retries with the payment signature to receive the content.

## Upgrading

Changes that require action on an existing deployment:

- **Add `SESSION_SECRET`** (16+ random characters; `npm run generate-wallets` creates one). Dashboard sign-in is refused until it is set. Sessions used to be the fixed cookie value `authenticated`, which anyone could copy; they are now signed and expire after a day. Everyone is signed out once.
- **`/api/gateway/balance` and `/api/gateway/withdraw` now require a signed-in session.** They were reachable by anyone who knew the URL, and the withdraw endpoint sends the seller's Gateway balance to any address it is given.
- **Rename** `SUPABASE_SERVICE_ROLE_KEY` to `SUPABASE_SECRET_KEY`. The old name is no longer read.
- **Remove** `BUYER_ADDRESS` and `OPENAI_API_KEY`. They are no longer used.
- **Apply the new migration** (`npm run db:start` locally, `npm run supabase -- db push` on a hosted project). It drops the unused `pg_graphql` extension.
- `SELLER_ADDRESS` is now validated. A missing or malformed value makes the paywalled endpoints answer `500` instead of quoting a payment to `undefined`.

## Environment Variables

Copy `.env.example` to `.env.local` and fill in the required values:

```bash
# Supabase
NEXT_PUBLIC_SUPABASE_URL=your-project-url
NEXT_PUBLIC_SUPABASE_PUBLISHABLE_KEY=your-publishable-key
SUPABASE_SECRET_KEY=your-secret-key

# x402 / Circle Nanopayments
SELLER_ADDRESS=0xYourWalletAddress
SELLER_PRIVATE_KEY=0xYourSellerPrivateKey

# Signs dashboard sessions (16+ characters)
SESSION_SECRET=your-long-random-secret

# Buyer wallet (funds the payment agent)
BUYER_PRIVATE_KEY=0xYourBuyerPrivateKey

# Payment agent (optional)
# BASE_URL=http://localhost:3000
# DEPOSIT_AMOUNT=1
```

| Variable | Scope | Purpose |
| --- | --- | --- |
| `NEXT_PUBLIC_SUPABASE_URL` | Public | Supabase project URL. |
| `NEXT_PUBLIC_SUPABASE_PUBLISHABLE_KEY` | Public | Supabase publishable key. |
| `SUPABASE_SECRET_KEY` | Server-side | Supabase secret key, used to record payment events and withdrawals. |
| `SELLER_ADDRESS` | Server-side | EVM wallet address that receives USDC payments. Also used for Gateway balance queries. |
| `SELLER_PRIVATE_KEY` | Server-side | Seller wallet private key, used for withdrawals. |
| `SESSION_SECRET` | Server-side | Signs dashboard session cookies. Required to sign in; 16+ random characters. |
| `ADMIN_EMAIL` / `ADMIN_PASSWORD` | Server-side | Optional. Override the demo login (`admin@example.com` / `123456`). |
| `BUYER_PRIVATE_KEY` | Agent | Buyer wallet private key. The agent uses it to fund its throwaway wallet. |
| `BASE_URL` | Agent | Optional. Base URL of the seller app. Defaults to `http://localhost:3000`. |
| `DEPOSIT_AMOUNT` | Agent | Optional. USDC amount moved into Gateway on each deposit. Defaults to `1`. |
| `VERCEL_URL` | Server-side | Optional. Set automatically on Vercel. Used as the app's base URL for metadata; defaults to `http://localhost:3000`. |

> **Tip:** Run `npm run generate-wallets` to auto-generate `SELLER_ADDRESS`, `SELLER_PRIVATE_KEY`, `BUYER_PRIVATE_KEY` and `SESSION_SECRET`.

## User Accounts

### Demo Account

The app has a single demo account for local development (override it with `ADMIN_EMAIL` and `ADMIN_PASSWORD`; do not ship the default):

| Email | Password |
| --- | --- |
| `admin@example.com` | `123456` |

## Available Scripts

- `npm run dev` — Start the Next.js development server
- `npm run build` — Create a production build
- `npm run start` — Start the production server
- `npm run lint` — Run ESLint
- `npm test` — Run the unit tests (no services needed)
- `npm run test:integration` — Run database tests against the local Supabase (`npm run db:start` first)
- `npm run db:start` / `db:stop` / `db:status` / `db:reset` — Manage the local Supabase instance
- `npm run generate-wallets` — Generate seller and buyer wallets and write them to `.env.local`
- `npm run agent` — Run the payment agent against the local app

## Testing

- `npm test` runs the unit tests in `tests/unit`. They mock Supabase and the Circle SDKs, so they need no credentials or Docker. They cover session signing, the proxy, login, the withdraw and balance endpoints, and the x402 paywall.
- `npm run test:integration` runs `tests/integration` against the **local** Supabase stack: who can read and write `payment_events` and `withdrawals`, and that the GraphQL endpoint is gone. It reads connection settings from `.env.local`.

## Security & Usage Model

This sample application:
- Assumes testnet usage only
- Handles secrets via environment variables
- Signs dashboard sessions and requires one for the seller APIs
- Is not intended for production use without modification

Known limitations to address before any production use:
- **Payment history is public.** The dashboard reads `payment_events` and `withdrawals` in the browser with the publishable key, so anyone with that key can read every payer address, amount and withdrawal destination (they cannot write). Production code should serve the dashboard through authenticated server routes instead.
- **One shared demo login.** Replace it with real authentication (for example Supabase Auth) and make the withdraw endpoint require more than a session, such as re-authentication or a fixed withdrawal address.
- **Withdrawals are not serialized.** Two withdrawals submitted at once are both checked against the same balance. The one that cannot be covered is expected to fail at Gateway, but this app does not prevent the attempt (or its failed record).
- **Payment is taken before the handler runs.** If a paywalled handler throws after settlement, the buyer has paid and receives a `500`.

See `SECURITY.md` for vulnerability reporting guidelines. Please report issues privately via Circle's bug bounty program.
