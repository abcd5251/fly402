# Fly402

**An AI travel agent that holds your seat before the good flights are gone — and pays for
everything itself, over HTTP, on Hedera.**

You describe the trip once. The agent watches real fares, and when one fits your budget it doesn't
just alert you — it takes the seat off the market, parks your ticket money in an escrow contract,
and emails you a link. You confirm whenever you get to it. The price is still there.

| | |
|---|---|
| **Network** | Hedera testnet |
| **Payments** | [x402 protocol](https://x402.org), settled through the Blocky402 facilitator |
| **Escrow contract** | [`0x5EC8953b1f2EacCe062f715027106b4896F6B87d`](https://hashscan.io/testnet/contract/0x5EC8953b1f2EacCe062f715027106b4896F6B87d) |
| **Frontend** | React 19 · TanStack Start · Vite |
| **Backend** | Express 5 · TypeScript |

---

## Why this exists

Web3 doesn't meet in one place — ETHGlobal, Devconnect, Token2049, EthCC, a different country
every time. You know about the trip weeks ahead and you book it days ahead, not because you forgot
but because watching a fare is a job and you already have one. By the time you look, the good
flights are gone.

A price hold would fix that, except today a hold is either free and therefore worthless, or it
doesn't exist at all. **Fly402 makes the hold a real product: an option on a seat, with a price
and a lifecycle.** A small fee buys the time and is gone. The ticket money goes into a contract
instead of the seller's pocket, and leaves by exactly one of three doors — you book and it pays
for the seat, you pass and it comes back, you forget and it comes back anyway.

That last door is why this needs a chain. *"We'll refund you"* is a promise; a contract isn't.

---

## Hedera integration

Every payment in Fly402 settles on Hedera testnet: HBAR moves over x402 through the Blocky402
facilitator, and deposits sit in `HoldEscrow.sol`, deployed at
[`0x5EC8953b1f2EacCe062f715027106b4896F6B87d`](https://hashscan.io/testnet/contract/0x5EC8953b1f2EacCe062f715027106b4896F6B87d).
The Hedera code is in `backend/seller/src/index.ts` (x402 scheme on `hedera:testnet`),
`backend/seller/src/lib/hedera-client.ts` (`@hiero-ledger/sdk` contract calls), and
`frontend/src/lib/x402-client.ts` (the agent's ECDSA signer).

| File | What it does |
|------|--------------|
| `backend/seller/src/index.ts` | Registers `ExactHederaScheme` on `hedera:testnet`; gates `/holds` and `/booking` behind a real `402` |
| `backend/seller/src/lib/hedera-client.ts` | `@hiero-ledger/sdk` client; `ContractExecuteTransaction` for `open` / `settle` / `refund` |
| `backend/seller/src/lib/pricing.ts` | Every amount the buyer is charged, in one place |
| `contracts/src/HoldEscrow.sol` | The escrow itself — no owner, no pause, no upgrade path |
| `contracts/scripts/deploy.cjs` | Deploys the contract to Hedera |
| `frontend/src/lib/x402-client.ts` | The browser agent's Hedera ECDSA signer and x402 `fetch` wrapper |
| `frontend/src/lib/hedera-account.ts` | Live HBAR balance reads from the Hedera mirror node |

> **Why Hedera specifically.** A hold only works if the deposit is provably locked before the fare
> moves. Hedera's ~3-second finality and flat, predictable fees are what make a short, low-value
> option viable at all — on a chain with volatile gas, settlement would eat the product.

---

## How a seat hold works

```
   ┌── agent · unattended ──────────────────┐   ┌── you · whenever ─────────┐
   │                                        │   │                           │
   │   POST /holds                          │   │                           │
 ──┼─→ 402 Payment Required                 │   │                           │
   │   wallet signs → Blocky402 → ~3s       │   │                           │
   │        │                               │   │                      ┌──→ book    settle()
   │        ├── hold fee  → seller (spent)  │   │   you open the link  │
   │        └── deposit   → HoldEscrow ─────┼───┼─→ fare still locked ─┼──→ pass    refund()
   │                                        │   │   countdown running  │
   │   email → agent@fly402.ai ─────────────┼───┤                      └──→ expire  refund()
   └────────────────────────────────────────┘   └───────────────────────────┘
```

One x402 call carries two kinds of money. The **fee** is non-refundable and stays with the seller —
it pays for the seat being off the market. The **deposit** is forwarded into `HoldEscrow.sol` and
can only leave through `settle()` or `refund()`.

### The contract

`contracts/src/HoldEscrow.sol` — no owner, no pause, no admin key, no upgrade path.

| Function | Who can call it | When |
|----------|-----------------|------|
| `open(id, seller, expiresAt)` | anyone, payable | locks a deposit against a hold id |
| `settle(id)` | the payer only | before expiry — deposit goes to the seller |
| `refund(id)` | the payer, **or anyone after expiry** | returns the deposit |

That last row is the point: once the hold expires the refund is **permissionless**. A seller that
goes offline cannot strand a traveller's money, and nobody can change the rules after a deposit is
already in.

---

## Running it

**Prerequisites:** Node.js 20+, and two Hedera testnet accounts from
[portal.hedera.com](https://portal.hedera.com) — one seller, one buyer. Both must use **ECDSA**
keys; ED25519 is not compatible with x402.

```bash
# 1 · backend
cd backend/seller
npm install
cp .env.example .env      # fill in HEDERA_ACCOUNT_ID
npm run dev               # → http://localhost:4021

# 2 · frontend (new terminal)
cd frontend
npm install
cp .env.example .env      # fill in the buyer's account + ECDSA key
npm run dev               # → http://localhost:8080
```

### Environment

`backend/seller/.env` — the seller:

```env
HEDERA_ACCOUNT_ID=0.0.xxxxx        # receives x402 payments
PORT=4021
HEDERA_NETWORK=testnet

# escrow — without these, holds fall back to local tracking and the UI says so
HEDERA_ESCROW_ADDRESS=0x...
HEDERA_OPERATOR_ID=0.0.xxxxx
HEDERA_OPERATOR_KEY=0x...          # ECDSA

FLIGHT_SERPAPI=...                 # live Google Flights fares
RESEND_API_KEY=re_...              # match emails
EMAIL_FROM=agent@fly402.ai
APP_URL=http://localhost:8080      # deep links in emails
```

`frontend/.env` — the buyer:

```env
VITE_API_URL=http://localhost:4021
VITE_HEDERA_ACCOUNT_ID=0.0.xxxxx
VITE_HEDERA_PRIVATE_KEY=0x...      # ECDSA; hex or DER both work
```

> ⚠️ `VITE_*` values are compiled into the browser bundle. Use a **throwaway testnet key** here —
> never a funded or mainnet one.

### Endpoints

| Endpoint | Method | Auth | Description |
|----------|--------|------|-------------|
| `/flights/search` | GET | free | Demo inventory |
| `/flights/catalog` | GET | free | Live Google Flights + demo rows |
| `/flights/:id` | GET | free | Flight details |
| `/holds` | POST | **paid** | Open a seat hold; deposit → escrow |
| `/holds/:id` | GET | free | Hold status |
| `/holds/:id/release` | POST | free | Release and refund the deposit |
| `/escrow` | GET | free | Vault snapshot |
| `/notify/match` | POST | free | Send the match email |
| `/booking` | POST | **paid** | Confirm the booking |
| `/health` | GET | free | Health check |

Prices live in `backend/seller/src/lib/pricing.ts` — that file is the single source of truth, and
`frontend/src/lib/hold.ts` mirrors it.

See the `402` for yourself:

```bash
curl -i -X POST http://localhost:4021/booking

curl -i -X POST http://localhost:4021/holds \
  -H "Content-Type: application/json" \
  -d '{"flightId":"nh853","passengers":1,"hours":24}'
```

Both return `402 Payment Required` with the payment terms in the `PAYMENT-REQUIRED` header.

---

## Deploying the escrow contract

Holds work without it — they fall back to local tracking. To run them on-chain:

```bash
cd contracts
npm install
cp .env.example .env      # HEDERA_OPERATOR_ID + HEDERA_OPERATOR_KEY
npm run deploy:hedera
```

Then put the deployed address into `backend/seller/.env` as `HEDERA_ESCROW_ADDRESS`.

---

## Project structure

```
fly-with-ai/
├── backend/seller/              # Express + x402 middleware — the paid API
│   └── src/
│       ├── index.ts             # x402 scheme, routes, hold logic
│       ├── data/flights.ts      # demo inventory
│       └── lib/
│           ├── pricing.ts       # every amount charged
│           ├── hedera-client.ts # Hedera SDK + contract calls
│           ├── escrow.ts        # hold ledger
│           ├── serpapi.ts       # live fares · 12h cache · stale-on-fail
│           └── email.ts         # Resend
├── frontend/                    # React 19 + TanStack Start
│   └── src/
│       ├── lib/
│       │   ├── x402-client.ts    # Hedera signer + paid fetch
│       │   ├── hedera-account.ts # mirror-node balance
│       │   ├── api.ts            # API client
│       │   ├── hold.ts           # hold types + quote
│       │   └── trip.ts           # trip types + spending policy
│       └── routes/              # trip · flight · booking · escrow · activity
├── contracts/
│   ├── src/HoldEscrow.sol
│   └── scripts/deploy.cjs
├── DESCRIPTION.md               # what it is and why
└── HOW_ITS_MADE.md              # how it was built
```

---

## Email notifications

The agent emails you the moment it finds a match, with a deep link straight into the flight page
(`/flight?flightId=...&tripData=...`) so the trip context survives the round trip. Set
`RESEND_API_KEY` and `EMAIL_FROM` in `backend/seller/.env`. Resend's test sender
(`onboarding@resend.dev`) only delivers to your own Resend signup address — verify a domain to
reach anyone else.

## Resources

- [x402 documentation](https://docs.x402.org)
- [Hedera documentation](https://docs.hedera.com) · [portal](https://portal.hedera.com) ·
  [HashScan](https://hashscan.io/testnet)

## License

MIT
