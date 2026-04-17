# MintSeat

**Mint your seat. Own the chain.**

MintSeat is a B2B escrow-coordination platform for event ticketing. Venues embed a two-line API into their existing checkout to enable escrow-protected transactions, automatic artist and rights holder royalty routing, and resale price controls — without replacing any existing system.

Live at [mintseat.co](https://mintseat.co) · Built by [Solmare Group LLC](https://elevatewellth.co) · Chicago, IL

---

## The Problem

Ticketmaster takes 29% in fees on the average ticket. Artists earn nothing when their tickets resell on the secondary market. Season ticket holders spend hours listing tickets one by one and chase payment after every game. Fans get surprised by fees at the last checkout screen.

On April 16, 2026, a federal jury found Ticketmaster guilty of monopolizing the live entertainment ticketing market. The alternative is being built.

---

## How MintSeat Works

MintSeat sits on top of any ticketing system as a trust layer. No replacement required.

**For venues and sports teams** — embed two lines of code into your checkout. Every ticket sold gets escrow protection, resale rules you control, and automatic artist/rights holder royalties on every secondary sale.

**For season ticket holders** — connect your account, see all your games at once, set one pricing rule, list everything in under two minutes. Get paid automatically when each ticket sells. No Venmo. No chasing anyone.

**For fans** — see the real total price before you click. Your money is held in escrow until your ticket is confirmed delivered. Automatic full refund on non-delivery.

---

## Architecture

```
┌─────────────────────────────────────────────────────────────┐
│                        mintseat.co                          │
│                   Cloudflare Pages (static)                 │
└─────────────────────────────────────────────────────────────┘
                              │
              ┌───────────────┼───────────────┐
              ▼               ▼               ▼
   ┌──────────────┐  ┌──────────────┐  ┌──────────────┐
   │   Registry   │  │    Escrow    │  │     Risk     │
   │   Worker     │  │   Worker     │  │   Worker     │
   │              │  │              │  │              │
   │ Venue onboard│  │ XRPL escrow  │  │ Dispute mon. │
   │ Event registry│ │ EscrowCreate │  │ Seller reserve│
   │ Ticket reg.  │  │ EscrowFinish │  │ 2FA / TOTP   │
   │ Artist reg.  │  │ EscrowCancel │  │ Dup. detect. │
   │ Stripe V2    │  │ Payment split│  │ Fraud response│
   │ Subscriptions│  │              │  │              │
   └──────┬───────┘  └──────┬───────┘  └──────┬───────┘
          │                 │                  │
          └─────────────────┼──────────────────┘
                            ▼
                  ┌──────────────────┐
                  │  MINTSEAT_DATA   │
                  │  Cloudflare KV   │
                  └──────────────────┘
                            │
              ┌─────────────┴─────────────┐
              ▼                           ▼
   ┌──────────────────┐        ┌──────────────────┐
   │   Stripe Connect │        │   XRPL Ledger    │
   │   V2 Platform    │        │   Testnet/Main   │
   │                  │        │                  │
   │ Connected accts  │        │ EscrowCreate     │
   │ Express onboard  │        │ EscrowFinish     │
   │ Direct charges   │        │ EscrowCancel     │
   │ App fee routing  │        │ Split payments   │
   │ Subscriptions    │        │ Artist royalties │
   └──────────────────┘        └──────────────────┘
```

---

## Stack

| Layer | Technology |
|---|---|
| Frontend | Single-file HTML, Cloudflare Pages |
| Workers | Cloudflare Workers (3 workers) |
| Storage | Cloudflare KV (`MINTSEAT_DATA`) |
| Payments | Stripe Connect V2 — Platform model |
| Blockchain | XRPL (XRP Ledger) — Escrow + royalty routing |
| Email | Microsoft 365 — hello@mintseat.co |
| Domain | Cloudflare — mintseat.co |

---

## Workers

### `mintseat-registry-worker`
The spine of the platform. Handles venue onboarding, event registry, ticket registry, artist/rights holder registry, Stripe Connect V2 account creation, subscription management, and webhook processing.

**Key endpoints:**
- `POST /api/venue/register` — Create venue + Stripe V2 connected account
- `POST /api/venue/onboard-link` — Generate Stripe onboarding link
- `GET /api/venue/onboard-status` — Live status from Stripe API
- `POST /api/event/create` — Register event with resale rules
- `POST /api/ticket/register` — Register ticket at point of sale (Partner Verified tier)
- `POST /api/artist/register` — Register artist/rights holder with XRPL payout address
- `POST /api/products/checkout` — Direct charge checkout with platform fee
- `POST /api/subscription/checkout` — Venue SaaS subscription checkout
- `POST /api/webhooks/stripe` — V1 standard events (subscriptions)
- `POST /api/webhooks/stripe/v2` — V2 thin events (connected account changes)

### `mintseat-escrow-worker`
Handles all XRPL escrow logic. Creates escrow on buyer payment, routes splits on delivery confirmation, cancels and refunds on non-delivery.

**Key endpoints:**
- `POST /api/escrow/create` — Lock buyer funds in XRPL escrow
- `POST /api/escrow/finish` — Release funds on delivery confirmation
- `POST /api/escrow/cancel` — Auto-refund buyer on non-delivery
- `POST /api/listing/create` — Create resale listing with cancellation deposit
- `GET /api/listing/lookup` — Fetch listing with trust tier and MintScore

### `mintseat-risk-worker`
Implements all six Stripe risk management requirements. Payout schedules, dispute monitoring, seller reserves, TOTP 2FA, duplicate account detection, and fraud response.

**Key endpoints:**
- `POST /api/risk/seller/register` — Create seller risk profile + duplicate check
- `GET /api/risk/seller/payout-tier` — Current payout schedule and upgrade criteria
- `POST /api/risk/dispute/record` — Record dispute, auto-adjust payout schedule
- `GET /api/risk/reserve/status` — Seller reserve status and release progress
- `POST /api/risk/2fa/setup` — Generate TOTP secret for seller account
- `POST /api/risk/2fa/validate` — Validate TOTP code, issue action token
- `POST /api/risk/fraud/report` — Freeze payouts, forfeit reserve, alert admin

---

## MintScore

Every seller has a MintScore (0–10,000) — a proprietary trust algorithm built on five on-chain-verified pillars.

| Weight | Pillar | Signal |
|---|---|---|
| 30% | Delivery Rate | Completed transactions / total. On-chain verified. |
| 25% | Delivery Speed | XRPL timestamp from escrow create to delivery confirm. |
| 20% | Peer Ratings | Post-transaction buyer ratings. Recency weighted. |
| 15% | Cancellation Rate | Penalty cancellations as % of total listings. |
| 10% | On-Chain History | Total verified transaction volume. Cannot be fabricated. |

Recency weights: last 90 days = 50%, prior year = 35%, older = 15%. New sellers start at 500.

---

## Trust Tiers

| Tier | How | Buyer Protection |
|---|---|---|
| Partner Verified | Registered at original purchase through MintSeat-integrated venue | Highest — venue rules auto-enforce, royalties auto-route |
| Transfer Verified | Seller connected Ticketmaster/team app account | Ticket confirmed in seller's account |
| Seller Declared | Seller uploaded proof of ticket | Escrow protects funds until delivery |

---

## Payment Flow

```
Buyer pays (card or USDC)
        │
        ▼
  Funds lock in XRPL escrow
  (never in MintSeat's account)
        │
        ▼
  Ticket transfers
        │
        ▼
  Delivery confirmed
        │
        ▼
  Auto-split payment:
  ├── Artist / rights holder: 7% (configurable per event)
  ├── Seller: ~91.5%
  └── MintSeat platform fee: 1.5%
        │
        ▼
  If delivery NOT confirmed:
  └── Auto-refund to buyer (no manual action required)
```

---

## Pricing

### For Venues and Sports Teams

| Tier | Price | Events | Key Features |
|---|---|---|---|
| Starter | $49/mo | Up to 5/month | Escrow protection, basic dashboard |
| Pro | $149/mo | Unlimited | Artist royalties, resale controls, MintScore, STH bulk listing |
| Enterprise | $299/mo | Unlimited | Multi-venue, white-label, dedicated account manager |

### For Season Ticket Holders and Fans

**$0/month. Always.**

Season ticket holders pay 1.5% only when a ticket sells. Fans pay 1.5% shown upfront before checkout. No subscription. No hidden fees.

---

## Environment Variables

All three workers require:

```
STRIPE_SECRET_KEY                    # sk_live_xxx
STRIPE_PUBLISHABLE_KEY               # pk_live_xxx
STRIPE_WEBHOOK_SECRET                # whsec_xxx (V1 events)
STRIPE_WEBHOOK_SECRET_V2             # whsec_xxx (V2 thin events)
COORDINATOR_WALLET_ADDRESS           # XRPL coordinator wallet address
COORDINATOR_WALLET_SEED              # XRPL coordinator wallet seed (secret)
```

Registry worker also requires:

```
STRIPE_SUBSCRIPTION_PRICE_STARTER    # price_xxx
STRIPE_SUBSCRIPTION_PRICE_PRO        # price_xxx
STRIPE_SUBSCRIPTION_PRICE_ENTERPRISE # price_xxx
```

---

## Stripe Connect Configuration

- **Model:** Platform (sellers collect payments directly)
- **Account type:** V2 connected accounts
- **Fees collector:** Stripe
- **Losses collector:** Stripe
- **Onboarding:** Hosted by Stripe (Express)
- **Account management:** Stripe Dashboard

---

## Webhook Endpoints

| Endpoint | Type | Events |
|---|---|---|
| `/api/webhooks/stripe` | V1 Standard | `customer.subscription.updated`, `customer.subscription.deleted`, `payment_method.attached`, `payment_method.detached`, `customer.updated`, `invoice.paid`, `invoice.payment_failed`, `payment_intent.succeeded` |
| `/api/webhooks/stripe/v2` | V2 Thin | `v2.core.account[requirements].updated`, `v2.core.account[configuration.merchant].capability_status_updated`, `v2.core.account[configuration.customer].capability_status_updated` |

---

## KV Schema

```
venue:{venueId}                      VenueRecord
venue:slug:{slug}                    venueId
venue:apikey:{hashedKey}             venueId
venue:stripe:{stripeAccountId}       venueId
venue:{venueId}:events               string[]
venue:{venueId}:stats                VenueStats
venue:{venueId}:subscription         SubscriptionRecord

event:{eventId}                      EventRecord
event:slug:{slug}                    eventId
event:{eventId}:tickets              string[]
event:{eventId}:listings             string[]
event:{eventId}:stats                EventStats

ticket:{ticketId}                    TicketRecord
ticket:ext:{externalRef}             ticketId

artist:{artistId}                    ArtistRecord
artist:slug:{slug}                   artistId

seller:{sellerId}                    SellerRiskProfile
seller:{sellerId}:reserve            ReserveRecord
seller:{sellerId}:disputes           DisputeRecord[]
seller:{sellerId}:2fa                TOTPRecord
seller:{sellerId}:subscription       SubscriptionRecord

alert:queue                          AlertRecord[]
```

---

## Chicago Pilot

MintSeat is launching with Chicago venues first — The Riv, Metro, Schubas. If you're a venue, sports team, season ticket holder, or fan who's done with 30% fees and empty promises, get on the list at [mintseat.co](https://mintseat.co).

---

## Built By

**Guillermo Zamorano** — Founder, Solmare Group LLC  
Chicago, IL · 773-740-5917 · hello@mintseat.co  
Background in futures trading, crypto, and Web3 infrastructure.

---

*© 2026 Solmare Group LLC. All rights reserved.*
