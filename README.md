# Peep: Smart Transaction Router for Solana

A production-grade smart transaction infrastructure stack built on Solana. Peep observes the network in real time, routes and submits transactions intelligently via Jito bundles, tracks the full transaction lifecycle across all commitment levels, and uses an AI agent to make autonomous operational decisions.

Built for the **Smart Transaction Routing** bounty category.

---

## What This Builds

Peep is not a simple transaction sender. It is a full transaction routing stack that understands the Solana transaction lifecycle end-to-end, from leader scheduling and TPU ingestion through block production, shred propagation, and all commitment stages, and reacts intelligently at every step.

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
 AI Agent (Grok)
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
- Reasoning is logged per submission (not a sequential wrapper)

The agent also handles **Failure Reasoning**:

- Detects failure classification (expired blockhash, fee too low, compute exceeded, bundle skip)
- Reasons about cause
- Decides whether to retry, refresh blockhash, recalculate tip, or abort
- No hardcoded retry flow; all retry decisions route through the agent

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

## Network

Runs on **Solana Devnet** (with mainnet-compatible infrastructure). Yellowstone endpoint and Jito Block Engine pointed at devnet equivalents for submission testing.

---

## License

MIT
