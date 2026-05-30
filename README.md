# Peep — Smart Transaction Router for Solana

A production-grade smart transaction infrastructure stack built on Solana. Peep observes the network in real time, routes and submits transactions intelligently via Jito bundles, tracks the full transaction lifecycle across all commitment levels, and uses an AI agent to make autonomous operational decisions.

Built for the **Smart Transaction Routing** bounty category.

---

## What This Builds

Peep is not a simple transaction sender. It is a full transaction routing stack that understands the Solana transaction lifecycle end-to-end — from leader scheduling and TPU ingestion through block production, shred propagation, and all commitment stages — and reacts intelligently at every step.

---

## Stack

### Core Runtime
| Layer | Technology |
|---|---|
| Runtime | Node.js |
| Language | TypeScript |
| HTTP Server | Express.js |
| Dev Server | Nodemon + ts-node |

### Solana Infrastructure
| Layer | Technology |
|---|---|
| Solana Client | `@solana/web3.js` |
| Jito Bundles | Jito SDK (`jito-ts`) |
| Live Slot Streaming | Yellowstone gRPC / Geyser stream (`yellowstone-grpc`) |
| Leader Schedule | Solana RPC + Geyser slot notifications |
| Commitment Tracking | Processed → Confirmed → Finalized via stream subscriptions |

### AI Agent
| Layer | Technology |
|---|---|
| AI Model | Grok API |
| Agent Mode | Tool-use + autonomous reasoning |
| Decision Domain | Tip intelligence / failure reasoning / retry orchestration |

### Data & Logging
| Layer | Technology |
|---|---|
| Lifecycle Store | PostgreSQL (slot numbers, timestamps, tip amounts, failure codes) |
| ORM | Prisma |
| Log Format | Structured JSON |

### Infrastructure
| Layer | Technology |
|---|---|
| gRPC Transport | `@grpc/grpc-js` + `@grpc/proto-loader` |
| Environment Config | `dotenv` |
| Process Manager | PM2 (production) |

---

## Architecture Overview

```
Yellowstone gRPC Stream
        │
        ▼
 Slot Monitor Service ──► Leader Schedule Tracker
        │
        ▼
 Submission Engine ──► Jito Bundle Builder ──► Tip Calculator
        │                                            │
        │                                    Live tip account data
        ▼
 AI Agent (Claude)
  - Observes slot conditions
  - Decides tip amount / submission timing / retry strategy
        │
        ▼
 Lifecycle Tracker
  submitted → processed → confirmed → finalized
        │
        ▼
 Lifecycle Log (PostgreSQL)
  slot, timestamp, commitment, tip, latency delta, failure code
```

Full architecture document (Figma/Notion): _link to be added_

---

## AI Agent Responsibilities

The AI agent owns **Tip Intelligence** as its primary decision domain:

- Reads recent tip account data from Jito tip distribution accounts
- Observes current slot pace and leader quality from the live stream
- Decides tip amount per bundle, balancing cost against landing probability
- Reasoning is logged per submission — not a sequential wrapper

The agent also handles **Failure Reasoning**:

- Detects failure classification (expired blockhash, fee too low, compute exceeded, bundle skip)
- Reasons about cause
- Decides whether to retry, refresh blockhash, recalculate tip, or abort
- No hardcoded retry flow — all retry decisions route through the agent

---

## Lifecycle Log Format

Each bundle submission produces a log entry:

```json
{
  "bundle_id": "...",
  "submitted_at": { "slot": 123456, "timestamp": "2025-01-01T00:00:00Z" },
  "processed_at": { "slot": 123457, "timestamp": "..." },
  "confirmed_at": { "slot": 123460, "timestamp": "..." },
  "finalized_at": { "slot": 123492, "timestamp": "..." },
  "tip_lamports": 85000,
  "tip_reasoning": "...",
  "status": "finalized | failed",
  "failure_class": "expired_blockhash | fee_too_low | compute_exceeded | bundle_skip | null",
  "latency": {
    "submitted_to_processed_ms": 420,
    "processed_to_confirmed_ms": 1800,
    "confirmed_to_finalized_ms": 26400
  }
}
```

Minimum 10 real bundle submissions included, with at least 2 failure cases. Slot numbers are verifiable on Solscan / SolanaFM.

---

## README Questions

### Q1: What does the delta between `processed_at` and `confirmed_at` tell you about network health at the time of submission?

The processed → confirmed delta reflects how quickly the supermajority of stake (66%+) voted on the block containing your transaction. Under normal network conditions this is roughly 400ms–800ms (2–4 slots). A large delta — several seconds or more — signals one or more of: elevated fork activity where validators are not voting on the same branch, a degraded leader producing slow or skipped blocks, or network congestion causing vote transaction delays. When this delta is consistently high across multiple submissions, it is a leading indicator that confirmation latency will be elevated and that using `confirmed` commitment for downstream reads may produce stale results. In a live routing stack, tracking this delta as a rolling metric lets the AI agent adjust tip aggressively or hold submissions until the network stabilizes.

### Q2: Why should you never use `finalized` commitment when fetching a blockhash for a time-sensitive transaction?

A blockhash fetched at `finalized` commitment is already 31+ slots old by the time you receive it (finalization requires ~32 slots of voting). Since a blockhash is valid for 150 slots from the slot it was produced, starting with a 31-slot-old hash leaves you fewer than 120 slots of validity — roughly 48 seconds. Under any retry or resubmission scenario this window shrinks fast and blockhash expiry becomes your most likely failure mode. For time-sensitive transactions you should fetch at `confirmed` commitment, which gives you a recent blockhash (2–4 slots old) with nearly the full 150-slot validity window intact. `processed` is faster but carries fork risk — the block may not get confirmed, invalidating the hash.

### Q3: What happens to your bundle if the Jito leader skips their slot?

Jito bundles are submitted to the Jito Block Engine, which forwards them to the designated Jito-enabled leader for the upcoming slot. If that leader skips their slot — due to being offline, failing to produce a block in time, or being forked out — the bundle is never included because there is no block to include it in. The bundle does not automatically roll over to the next leader. The Block Engine will return a bundle status of `Failed` or the bundle will time out with no on-chain result. The correct response is to detect the skip via the slot stream (the slot advances without a block from that leader), classify it as a `bundle_skip` failure, fetch a fresh blockhash, recalculate the tip based on the new leader's conditions, and resubmit. This is one of the failure cases the AI agent handles autonomously in this stack.

---

## Setup

```bash
# Install dependencies
npm install

# Copy env template
cp .env.example .env
# Fill in: SOLANA_RPC_URL, YELLOWSTONE_ENDPOINT, JITO_AUTH_KEYPAIR, ANTHROPIC_API_KEY, DATABASE_URL

# Run database migrations
npx prisma migrate dev

# Start dev server
npm run dev

# Build for production
npm run build
npm start
```

---

## Network

Runs on **Solana Devnet** (with mainnet-compatible infrastructure). Yellowstone endpoint and Jito Block Engine pointed at devnet equivalents for submission testing.

---

## License

MIT
