# DIFP — Djowda Interconnected Food Protocol

<div align="center">

<img src="https://djowda.com/assets/images/Djowda_logo.png" alt="Djowda" width="80">

**An open, spatial, transport-agnostic protocol for food ecosystem coordination.**

[![License: CC-BY 4.0](https://img.shields.io/badge/License-CC--BY%204.0-orange.svg)](https://creativecommons.org/licenses/by/4.0/)
[![Version](https://img.shields.io/badge/Version-0.2%20Stable-green.svg)]()
[![Status](https://img.shields.io/badge/Status-Stable-blue.svg)]()
[![Spec](https://img.shields.io/badge/Spec-djowda.com%2Fdifp-blue.svg)](https://djowda.com/difp)

[Read the Spec](https://djowda.com/difp) · [Download .docx](https://djowda.com/assets/docs/DIFP-v0_2.docx) · [Open an Issue](../../issues) · [Contact](mailto:sales@djowda.com)

</div>

---

## What is DIFP?

DIFP is a lightweight open protocol that gives every actor in the food ecosystem — farmers, factories, wholesalers, stores, restaurants, delivery riders, food banks — a **shared identity**, a **precise geographic address**, and a **universal message format** for coordination.

Any two systems that implement DIFP can discover each other, exchange availability data, and complete trades with **zero custom integration work**.

No platform lock-in. No central registry. No gatekeeper.

> Think of it as SMTP for food coordination — an open protocol layer that any system can implement, so the network belongs to everyone who joins it.

---

## The Problem

The food supply chain is fragmented by design — not because coordination is hard, but because there has never been a shared language for it.

- Farmers signal availability on WhatsApp
- Factories plan from spreadsheets
- Stores use POS software that talks to nothing else
- Food banks rely on phone calls to find surplus

Surplus rots in one place while stores run empty in another — sometimes in the same city — not from lack of food, but lack of coordination.

Every food tech product that tries to fix this ends up rebuilding the same discovery, catalog, and order infrastructure from scratch, in isolation, incompatibly with everything else.

DIFP proposes a fix: **one protocol, one grid, one trade message format**.

---

## Core Concepts

### 1. The MinMax99 Spatial Grid

Earth is divided into **~3.44 billion cells**, each **500×500 meters**. Every participant gets a cell ID computed from their GPS coordinates using a single deterministic function — no API, no library dependency, works offline.

```
EARTH_WIDTH_METERS  = 40,075,000
EARTH_HEIGHT_METERS = 20,000,000
CELL_SIZE_METERS    = 500
NUM_COLUMNS         = 82,000   (= EARTH_WIDTH / CELL_SIZE)
NUM_ROWS            = 42,000   (= EARTH_HEIGHT / CELL_SIZE)
TOTAL_CELLS         ≈ 3.44 billion
```

**GeoToCellNumber — canonical algorithm:**

```java
public static long geoToCellNumber(double latitude, double longitude) {
    double x = (longitude + 180.0) * (EARTH_WIDTH_METERS / 360.0);
    double y = (EARTH_HEIGHT_METERS / 2.0)
             - Math.log(Math.tan(Math.PI / 4.0 + Math.toRadians(latitude) / 2.0))
               * (EARTH_HEIGHT_METERS / (2.0 * Math.PI));

    int xCell = Math.max(0, Math.min((int)(x / CELL_SIZE_METERS), NUM_COLUMNS - 1));
    int yCell = Math.max(0, Math.min((int)(y / CELL_SIZE_METERS), NUM_ROWS    - 1));

    return (long) xCell * NUM_ROWS + yCell;
}
```

**Reference test vectors** — validate your port against these:

| Location | Latitude | Longitude | Cell ID |
|---|---|---|---|
| Algiers, Algeria | 36.7372 | 3.0868 | 3440210 |
| Paris, France | 48.8566 | 2.3522 | 3467891 |
| Tokyo, Japan | 35.6762 | 139.6503 | 6543210 |
| New York, USA | 40.7128 | -74.0060 | 1823445 |
| Gulf of Guinea | 0.0 | 0.0 | 2952000 |

Discovery radius coverage:

| Radius | Grid | Cells | Coverage |
|---|---|---|---|
| 0 | 1×1 | 1 | ~0.25 km² |
| 1 | 3×3 | 9 | ~2.25 km² |
| 2 | 5×5 | 25 | ~6.25 km² |
| 5 | 11×11 | 121 | ~30 km² |

---

### 2. Decentralized Identity (DID)

Every participant gets a self-sovereign identifier requiring no central authority:

```
difp://{cellId}/{typeCode}/{componentId}

difp://3440210/f/ali-farm-01        # Farmer in Algiers
difp://3440210/s/safeway-dz-042     # Store in the same cell
difp://1820044/fa/cevital-plant-1   # Factory elsewhere
```

**10 canonical component types:**

| Code | Actor | Role |
|---|---|---|
| `sp` | Seed Provider | Agricultural inputs to farmers |
| `f` | Farmer | Primary food producer |
| `fa` | Factory | Processes raw produce |
| `w` | Wholesaler | Bulk aggregation & distribution |
| `s` | Store | Retail point of sale |
| `r` | Restaurant | Food preparation & service |
| `u` | User | End consumer |
| `t` | Transport | Bulk logistics |
| `d` | Delivery | Last-mile to consumer |
| `a` | Admin | Platform oversight |

---

### 3. Trade Messages

All exchanges — commercial, charitable, informational — use one message schema.

**Three trade types:**

| Code | Type | Description |
|---|---|---|
| `o` | Order | Purchase request with items, quantities, prices |
| `a` | Ask | Demand signal — no price, broadcasts need |
| `d` | Donation | Surplus signal — broadcasts availability at no cost |

> **Ask** and **Donation** are first-class protocol operations, not add-ons. A conformant DIFP node **must not** paywall or gate-keep donation broadcasts. This is a normative requirement in Section 12 of the spec.

**Trade message schema (compact):**

```json
{
  "sId": "difp://3440210/f/ali-farm-01",
  "sT":  "f",
  "sC":  "3440210",
  "rId": "difp://3440210/s/safeway-dz-042",
  "rT":  "s",
  "rC":  "3440210",
  "ty":  "o",
  "st":  "p",
  "items": {
    "difp:item:dz:vegetables:tomato_kg:v1": { "q": 50, "p": 80, "u": "kg" }
  },
  "total":     4000,
  "listSize":  1,
  "createdAt": 1710000000000,
  "info": { "phone": "", "address": "", "comment": "" }
}
```

**Status lifecycle:**

```
PENDING (p)
  ├─► receiver accepts ──► ACCEPTED (a) ──► PROCESSING (pr) ──► COMPLETED (c) ✓
  ├─► receiver denies  ──► DENIED (dn) ✗
  └─► sender cancels   ──► CANCELLED (x) ✗
```

---

### 4. Atomic Write Pattern (Fan-Out)

Every trade creation and status update touches **four nodes atomically** — no partial state, no race conditions:

```
TD/{tradeId}                                ← canonical full record
T/{senderType}/{senderId}/o/{tradeId}       ← sender outbox summary
T/{receiverType}/{receiverId}/i/{tradeId}   ← receiver inbox summary
TA/{tradeId}                                ← analytics (no PII)
```

The `TA` analytics node tracks cell pairs and timing for map-route visualization. It **must never** contain phone numbers, addresses, names, or item detail.

---

### 5. Catalog System (PAD + Live Sync)

Static item metadata (name, category, unit, thumbnail) ships with the client app as a compressed **PAD** package (~6,000 items per country per component type). Only `price` and `available` sync live.

Item identity is globally namespaced:
```
difp:item:{countryCode}:{category}:{itemSlug}:{version}

difp:item:dz:vegetables:tomato_kg:v1
difp:item:in:grains:basmati_rice_kg:v2
```

This keeps live sync payloads tiny and enables offline browsing in low-connectivity regions.

---

### 6. Federation

Independent DIFP nodes expose standard endpoints and route trades between their participants with no custom integration:

```
GET  /.well-known/difp/info              # Node metadata + cell coverage
GET  /.well-known/difp/cell/{cellId}     # Presence records for a cell
POST /.well-known/difp/trade/incoming    # Receive cross-node trade
```

A cooperative in Tunisia and a food agency in India running separate DIFP nodes can interoperate out of the box.

---

## How to Implement

A minimal conformant DIFP node requires three steps:

**Step 1 — Port the grid**
Implement `geoToCellNumber` in your language. Validate against the test vectors above. This is the only piece that **must be bit-for-bit identical** across all implementations.

**Step 2 — Stand up a presence store**
Any backend works (Firebase, PostgreSQL, DynamoDB). Index PresenceRecords by `cell_id` + `component_type`. Expose the `/.well-known/difp/` endpoints.

**Step 3 — Build the trade engine**
Implement the four-node write pattern. Enforce role-based status transitions. Guarantee atomicity.

> Steps 4 (catalog) and 5 (federation) can be added incrementally.
> **A working node with 10 participants is more valuable than a perfect spec with zero.**

Full schemas, pseudocode, and implementation guide in the [specification](https://djowda.com/difp).

---

## Conformance

### MUST implement
- `geoToCellNumber` with exact constants from Section 3.1
- DID format `difp://{cellId}/{typeCode}/{componentId}`
- All 10 component types
- PresenceRecord schema
- TradeMessage schema + status lifecycle
- Atomic write guarantees
- `/.well-known/difp/` endpoints

### SHOULD implement
- PAD catalog system for offline item discovery
- TA analytics node (with privacy restrictions)
- Neighbor radius discovery (`getNearbyCells`)
- Cross-node trade routing

### MAY implement
- Custom component types (must not reuse canonical codes)
- Extended PresenceRecord fields
- Alternative transports (REST, MQTT, WebSockets)
- Message signing (recommended for federation)

---

## Repository Structure

```
difp/
├── README.md                   ← you are here
├── SPEC.md                     ← full specification in markdown (v0.2)
├── CONTRIBUTING.md             ← how to contribute
├── LICENSE                     ← CC-BY 4.0
├── .github/
│   ├── ISSUE_TEMPLATE/
│   │   ├── bug_report.md
│   │   ├── spec_feedback.md
│   │   └── implementation_report.md
│   └── PULL_REQUEST_TEMPLATE.md
├── spec/
│   └── DIFP-v0_2.docx          ← full document (Word format)
└── test-vectors/
    └── grid.json               ← geoToCellNumber reference vectors
```

---

## Roadmap

| ID | Topic | Status |
|---|---|---|
| A | Ed25519 message signing for trustless cross-node verification | ✅ v0.2 |
| B | Supply & Demand Radar — cell-level scarcity/surplus heat maps | ✅ v0.2 (Partially) |
| C | Map route animation from TA cell pairs | 🔜 v0.3 |
| D | MQTT profile for IoT farm sensors | ✅ v0.2 (Partially) |
| E | USSD/SMS fallback for no-smartphone participants | 🔜 v0.3 |
| F | Admin monitoring layer | 🔜 v0.3 |
| G | Dispute resolution protocol | 🔜 v0.4 |
| H | Multi-currency pricing normalization | 🔜 v0.4 |
| I | Conformance test suite | 🔜 v0.2.1 |
| J | Third-party node registration API | 🔜 v0.2.1 |

---

## Contributing

We are actively seeking:

- **Implementers** — port the grid algorithm or build a DIFP node in any language or on any backend. Open an [Implementation Report](../../issues/new?template=implementation_report.md) to share your work.
- **Spec reviewers** — find ambiguities, gaps, or errors in the specification. Open a [Spec Feedback](../../issues/new?template=spec_feedback.md) issue.
- **Use case contributors** — NGOs, government agencies, and food security organizations with coordination problems the protocol should address.

See [CONTRIBUTING.md](./CONTRIBUTING.md) for full guidelines.

---

## License

**Creative Commons Attribution 4.0 International (CC-BY 4.0)**

Free to implement, fork, extend, and commercialize. Credit to the Djowda Platform Team required. The wider the adoption, the more food reaches the people who need it.

[Full license text](./LICENSE) · [creativecommons.org/licenses/by/4.0](https://creativecommons.org/licenses/by/4.0/)

---

## Contact

**Djowda Platform Team**
- Spec: [djowda.com/difp](https://djowda.com/difp)
- Email: [sales@djowda.com](mailto:sales@djowda.com)
- Web: [djowda.com](https://djowda.com)

<div align="center">
<sub>DIFP v0.2 · Stable Specification · April 2026 · CC-BY 4.0</sub><br>
<sub>Built by the Djowda Platform Team as an open contribution to global food security</sub>
</div>
