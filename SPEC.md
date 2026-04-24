# DIFP — Djowda Interconnected Food Protocol
## Open Protocol Specification · v0.2

**Status:** STABLE — v0.2 Message Envelope
**Specification:** DIFP-CORE-0.2
**Issued:** April 2026
**License:** [CC-BY 4.0](https://creativecommons.org/licenses/by/4.0/)
**Contact:** [sales@djowda.com](mailto:sales@djowda.com)

---

### Preamble

Almost one billion people face food insecurity every year — not because the world doesn't produce enough food, but because the coordination infrastructure between producers and consumers is broken. Farmers don't know who needs what. Stores don't know what's available nearby. Surplus rots while scarcity persists — sometimes in the same city.

DIFP is a proposed solution to this coordination problem. It defines a common language for every actor in the food ecosystem — from seed providers to delivery riders — to discover each other, signal availability, and complete exchanges in real time, anchored to a precise geographic cell on Earth's surface.

This specification is intentionally open. Any developer, cooperative, government agency, or NGO may implement DIFP. No royalties. No gatekeeper. A single open standard means a Tunisian marketplace and an Indian cooperative can interoperate with zero extra integration work — because they speak the same protocol.

**🌍 Mission:** Reduce global food insecurity by eliminating coordination friction across the food supply chain.

---

### Abstract

DIFP is a lightweight, open, spatial food coordination protocol. It specifies:

| Module | Description |
|---|---|
| **Participant identity** | How food ecosystem actors identify themselves globally without a central registry |
| **Spatial addressing** | How Earth's surface is divided into ~500m × 500m cells, each acting as a coordination zone |
| **Presence and discovery** | How participants announce and find each other within and across cells |
| **Trade message format** | A universal structure for orders, asks, and donations between any two participants |
| **Protocol federation** | How independent DIFP implementations discover and interoperate with each other |

DIFP is transport-agnostic. The reference implementation uses Firebase Realtime Database. Conformant implementations may use REST, WebSockets, MQTT, or any equivalent transport.

---

### 01 · Terminology

The key words **MUST**, **MUST NOT**, **SHOULD**, **SHOULD NOT**, and **MAY** in this document are to be interpreted as described in RFC 2119.

| Term | Definition |
|---|---|
| **Component** | Any participant registered on a DIFP network (farmer, store, user, etc.) |
| **Cell** | A ~500 × 500 m geographic zone identified by a numeric cell ID |
| **Cell ID** | A 64-bit integer encoding the position of a cell in the global grid |
| **DID** | Decentralized Identifier — a self-sovereign identity string of the form `difp://{cellId}/{type}/{id}` |
| **Trade** | A structured message representing an order, ask, or donation between two components |
| **Interactor** | A software module handling the full discovery + trade lifecycle for one component pair |
| **Fan-out** | An atomic multi-path write ensuring consistency across all affected data nodes |
| **PAD** | Pre-loaded Application Data — static catalog content shipped with the app |
| **DIFP Node** | A server or service implementing this specification |
| **Federation** | The mechanism by which distinct DIFP Nodes discover and exchange data with each other |

---

### 02 · Ecosystem Actors

DIFP defines ten canonical component types. Each type maps to a distinct role within the food chain. Implementations **MUST** support all ten types; they **MAY** add custom types using the extension mechanism defined in Section 11.

| Code | Actor | Role in Food Chain |
|---|---|---|
| `sp` | Seed Provider | Supplies seeds and agricultural inputs to farmers |
| `f` | Farmer | Primary producer — grows and harvests food |
| `fa` | Factory | Processes raw produce into packaged goods |
| `w` | Wholesaler | Aggregates and distributes in bulk to stores and restaurants |
| `s` | Store | Retail point of sale to end consumers |
| `r` | Restaurant | Prepares and serves food to consumers |
| `u` | User | End consumer — orders from stores and restaurants |
| `t` | Transport | Bulk logistics between any two non-consumer nodes |
| `d` | Delivery | Last-mile delivery from store or restaurant to user |
| `a` | Admin | Platform oversight — read access across all node types |

#### 2.1 Supply Chain Flow

`Seed Provider` → `Farmer` → `Factory` → `Wholesaler` → `Store` / `Restaurant` → `User`

*Supporting roles (connect at any layer): Transport · Delivery · Admin*

---

### 03 · Spatial Addressing — The MinMax99 Grid

DIFP uses a deterministic global grid to assign every participant to a precise geographic zone. This is the foundational innovation of the protocol: proximity-first coordination replaces search-radius queries.

#### 3.1 Grid Constants

All conformant implementations **MUST** use the following constants without modification:

```python
EARTH_WIDTH_METERS  = 40,075,000   # equatorial circumference
EARTH_HEIGHT_METERS = 20,000,000   # meridian half-circumference
CELL_SIZE_METERS    = 500          # cell edge length
NUM_COLUMNS         = 82,000       # EARTH_WIDTH / CELL_SIZE
NUM_ROWS            = 42,000       # EARTH_HEIGHT / CELL_SIZE
TOTAL_CELLS         ≈ 3.44 × 10⁹  # 82,000 × 42,000
```

These constants are fixed across all versions of the protocol. Any implementation that changes `CELL_SIZE_METERS` is not DIFP-conformant and will produce incompatible cell IDs.

#### 3.2 GeoToCellNumber — Canonical Conversion

Any implementation **MUST** produce identical cell IDs from identical (latitude, longitude) inputs using the following algorithm:

```javascript
function geoToCellNumber(latitude, longitude) {
    // Step 1: longitude → x in meters (linear)
    let x = (longitude + 180.0) * (EARTH_WIDTH_METERS / 360.0);

    // Step 2: latitude → y in meters (Mercator, origin top-left)
    let y = (EARTH_HEIGHT_METERS / 2.0)
          - Math.log(Math.tan(Math.PI / 4.0 + (latitude * Math.PI / 180.0) / 2.0))
            * (EARTH_HEIGHT_METERS / (2.0 * Math.PI));

    // Step 3: meters → integer cell indices
    let xCell = Math.floor(x / CELL_SIZE_METERS);
    let yCell = Math.floor(y / CELL_SIZE_METERS);

    // Step 4: clamp to grid bounds
    xCell = Math.max(0, Math.min(xCell, NUM_COLUMNS - 1));
    yCell = Math.max(0, Math.min(yCell, NUM_ROWS - 1));

    // Step 5: encode as single int64
    return BigInt(xCell) * BigInt(NUM_ROWS) + BigInt(yCell);
}
```

#### 3.3 Reverse Lookup

```javascript
function cellNumberToXY(cellId) {
    let xCell = Math.floor(Number(cellId) / NUM_ROWS);
    let yCell = Number(cellId) % NUM_ROWS;
    return { xCell, yCell };
}
```

#### 3.4 Neighbor Resolution

A component with discovery radius `r` **MUST** query cells in a `(2r+1) × (2r+1)` square centered on its own cell:

```javascript
function getNearbyCells(centerCellId, radius) {
    let { xC, yC } = cellNumberToXY(centerCellId);
    let result = [];

    for (let dx = -radius; dx <= radius; dx++) {
        for (let dy = -radius; dy <= radius; dy++) {
            let x = Math.max(0, Math.min(xC + dx, NUM_COLUMNS - 1));
            let y = Math.max(0, Math.min(yC + dy, NUM_ROWS - 1));
            result.push(BigInt(x) * BigInt(NUM_ROWS) + BigInt(y));
        }
    }
    return result;
}
```

| Radius | Grid Size | Cell Count | Coverage |
|---|---|---|---|
| 0 | 1×1 | 1 | ~0.25 km² |
| 1 | 3×3 | 9 | ~2.25 km² |
| 2 | 5×5 | 25 | ~6.25 km² |
| 5 | 11×11 | 121 | ~30 km² |

---

### 04 · Component Identity

DIFP uses a Decentralized Identifier (DID) scheme that requires no central authority. Any conformant implementation can generate and verify identities offline.

#### 4.1 DID Format

```
// Schema
difp://{cellId}/{typeCode}/{componentId}

// Examples
difp://3440210/f/ali-farm-01        // Farmer in cell 3,440,210 (Algiers)
difp://3440210/s/safeway-dz-042     // Store in the same cell
difp://1820044/fa/cevital-plant-1   // Factory in cell 1,820,044
```

#### 4.2 Identity Registration

On first registration a DIFP Node **MUST**: (1) accept the participant's GPS coordinates, (2) compute the cell ID using `geoToCellNumber`, (3) assign a `componentId` unique within the node, (4) construct and store the full DID, (5) issue a signed identity token to the client.

#### 4.3 Legacy Encoding (Firebase Reference Implementation)

Implementations using Firebase Auth **MAY** encode identity in the `displayName` field as a compact string:

```
// Format: "{cellId}_{typeCode}"
"3440210_f"   // Farmer in cell 3,440,210
"3440210_s"   // Store in the same cell
"1820044_fa"  // Factory in cell 1,820,044
```

This encoding is specific to the Firebase reference implementation. All other inter-node communication **MUST** use the full DID format.

---

### 05 · Presence & Discovery

Every registered component **MUST** publish a presence record. This record is the atomic unit of discovery — it lets any other participant find and evaluate a potential counterpart.

#### 5.1 Presence Record Schema

```typescript
interface PresenceRecord {
    did:             string;   // Full DIFP DID (required)
    component_name:  string;   // Display name (required)
    phone_number:    string;   // Contact number (required)
    cell_id:         number;   // Numeric cell ID (required)
    component_type:  string;   // Type code from Section 2 (required)
    avatar_id?:      string;   // Reference to avatar asset (optional)
    status:          'open' | 'closed' | 'busy'; // (required)
    working_time?:   string;   // "HH:MM-HH:MM" local time (optional)
    last_update:     number;   // Unix timestamp ms (required)
    user_id:         string;   // Auth provider user ID (required)
    is_asking?:      boolean;  // Currently broadcasting an Ask (optional)
    is_donating?:    boolean;  // Currently broadcasting a Donation (optional)
}
```

#### 5.2 Firebase Reference Node Layout

`C/{typeCode}/{componentId}/`   ← presence records

Example:
```
C/f/ali-farm-01/
    did:            "difp://3440210/f/ali-farm-01"
    component_name: "Ali's Farm"
    cell_id:        3440210
    status:         "open"
    ...
```

**Implementation Note:** Records **MUST** be kept flat (no nested objects beyond the `PresenceRecord` fields). Deep nesting prevents efficient `cell_id` index queries on Firebase.

---

### 06 · Catalog System

DIFP separates item metadata (static) from item availability and pricing (live). This split dramatically reduces bandwidth and enables offline operation.

#### 6.1 Item Identity

```
// Format: difp:item:{countryCode}:{category}:{itemSlug}:{version}
difp:item:dz:vegetables:tomato_kg:v1
difp:item:in:grains:basmati_rice_kg:v2
difp:item:fr:dairy:camembert_250g:v1
```

#### 6.2 Static Item Record (PAD)

```typescript
interface CatalogItem {
    item_id:   string;             // namespaced identifier
    name:      Record<string, string>; // {"ar": "طماطم", "fr": "Tomate", "en": "Tomato"}
    category:  string;             // vegetables | grains | dairy | meat | ...
    unit:      string;             // kg | g | l | ml | piece | box | ...
    thumbnail: string;             // asset reference (.webp)
    gtin?:     string;             // optional GS1 barcode for interoperability
}
```

Static records are distributed as a **PAD** — a compressed package (~6,000 items per country per component type) downloaded once at installation. The PAD **MUST NOT** change at runtime; updates arrive only via app updates.

#### 6.3 Live Availability Record

```typescript
interface LiveItem {
    item_id:     string;  // references CatalogItem.item_id
    price:       number;  // in local currency units
    available:   boolean;
    last_update: number;  // Unix timestamp ms
}
```

Only `LiveItem` records travel over the wire during normal usage. Implementations **MUST NOT** include item names or thumbnails in live sync payloads.

---

### 07 · The Trade Protocol

A Trade is the fundamental coordination primitive of DIFP. All exchanges — commercial, charitable, or informational — are modeled as Trades between exactly two participants.

#### 7.1 Trade Types

| Code | Type | Description |
|---|---|---|
| `o` | **Order** | A purchase request — includes items, quantities, prices, and total |
| `a` | **Ask** | A resource availability inquiry — no price; signals demand to nearby suppliers |
| `d` | **Donation** | An offer to transfer resources at no cost — signals surplus to nearby receivers |

#### 7.2 Trade Message Schema

```typescript
interface TradeMessage {
    // Routing
    sId: string;    // Sender DID
    sT:  string;    // Sender type code
    sC:  string;    // Sender cell ID
    rId: string;    // Receiver DID
    rT:  string;    // Receiver type code
    rC:  string;    // Receiver cell ID

    // Trade metadata
    ty:  'o' | 'a' | 'd'; // Trade type
    st:  'p' | 'a' | 'pr' | 'c' | 'x' | 'dn'; // Status

    // Payload
    items: Record<string, OrderLine | 1>;
    // OrderLine = { q: number, p: number, u: string }
    // For Ask/Donation: flat map { item_id: 1 }

    total?:    number;   // Order total — order type only
    listSize:  number;   // Count of distinct items

    // Timestamps
    createdAt:   number;
    lastUpdated: number;

    // Contact
    info: { phone: string; address: string; comment: string };
    dCause?: string;      // Denial reason (if status = dn)
}
```

#### 7.3 Status Lifecycle

`PENDING (p)`
├─► receiver accepts ──► `ACCEPTED (a)` ──► `PROCESSING (pr)` ──► `COMPLETED (c)` ✓
├─► receiver denies ──► `DENIED (dn)` ✗
└─► sender cancels ──► `CANCELLED (x)` ✗

#### 7.4 Role-Based Transition Rules

| Actor | From | To |
|---|---|---|
| Sender | PENDING | CANCELLED |
| Receiver | PENDING | ACCEPTED |
| Receiver | PENDING | DENIED |
| Either | ACCEPTED | PROCESSING |
| Either | PROCESSING | COMPLETED |

---

### 08 · Data Node Layout

This section describes the Firebase Realtime Database node structure used in the reference implementation. Other transports **MUST** map to equivalent structures.

#### 8.1 Four-Node Pattern

| Node | Description |
|---|---|
| `TD/{tradeId}` | Canonical full record — source of truth for all trade data |
| `T/{typeCode}/{componentId}/i/{tradeId}` | Receiver inbox summary |
| `T/{typeCode}/{componentId}/o/{tradeId}` | Sender outbox summary |
| `TA/{tradeId}` | Analytics — no PII, map-route data only |

#### 8.2 Inbox / Outbox Summary Schema

```typescript
interface TradeSummary {
    fId: string;   // counterpart DID
    fT:  string;   // counterpart type code
    ty:  string;   // trade type
    st:  string;   // current status
    pv:  string;   // human-readable preview string
    ls:  number;   // list size (item count)
    lu:  number;   // last update timestamp
}
```

#### 8.3 Analytics Node Schema

```typescript
interface TradeAnalytics {
    createdAt:          number;
    acceptedAt?:        number;
    processingAt?:      number;
    completedAt?:       number;
    finalStatus:        string;
    durationToAccept:   number;   // seconds
    durationToComplete: number;   // seconds
    senderCell:         string;   // for map route animation
    receiverCell:       string;   // for map route animation
}
```

**🔒 Privacy Rule:** The `TA` (analytics) node **MUST NEVER** contain phone numbers, postal addresses, personal names, or item-level detail. It is used only for aggregate metrics and map-route animation.

---

### 09 · Atomic Write Guarantees (Fan-Out)

Every state-changing operation in DIFP **MUST** update all affected nodes atomically. No intermediate state should be observable by other clients.

#### 9.1 Trade Creation — Required Writes

// Atomic write set for `createTrade(trade)`
- `TD/{tradeId}` ← full canonical record
- `T/{senderType}/{senderId}/o/{tradeId}` ← sender outbox summary
- `T/{receiverType}/{receiverId}/i/{tradeId}` ← receiver inbox summary
- `TA/{tradeId}/createdAt` ← analytics seed
- `TA/{tradeId}/senderCell` ← routing cell
- `TA/{tradeId}/receiverCell` ← routing cell

#### 9.2 Status Update — Required Writes

// Atomic write set for `updateStatus(tradeId, newStatus)`
- `TD/{tradeId}/st` ← canonical status
- `TD/{tradeId}/lastUpdated` ← timestamp
- `T/{senderType}/{senderId}/o/{tradeId}/st` ← mirror to sender
- `T/{senderType}/{senderId}/o/{tradeId}/lu` ← mirror timestamp
- `T/{receiverType}/{receiverId}/i/{tradeId}/st` ← mirror to receiver
- `T/{receiverType}/{receiverId}/i/{tradeId}/lu` ← mirror timestamp
- `TA/{tradeId}/{statusTimestampKey}` ← analytics event

Implementations **MUST** use their transport's equivalent of Firebase's multi-path `updateChildren()` — a batch write, database transaction, or two-phase commit — to guarantee these writes succeed or fail together.

---

### 10 · Protocol Federation

DIFP is designed to be adopted by independent operators — NGOs, national food agencies, regional cooperatives. Federation enables these distinct nodes to interoperate without a central broker.

#### 10.1 Well-Known Endpoint

`GET https://{host}/.well-known/difp/info`

**Response:**
```json
{
    "protocol":  "DIFP",
    "version":   "0.2",
    "nodeId":    "string",
    "coverage":  ["cellId", "cellId", "..."],
    "contact":   "email",
    "federates": ["https://node2.example.com", "..."]
}
```

#### 10.2 Cell Presence Lookup

`GET https://{host}/.well-known/difp/cell/{cellId}`

**Response:** list of `PresenceRecord` for that cell

#### 10.3 Cross-Node Trade Routing

When a sender on Node A initiates a trade with a receiver on Node B: Node A validates the receiver DID → packages the TradeMessage → POSTs to Node B's trade inbox endpoint → Node B delivers to receiver's inbox → all subsequent status updates are mirrored to both nodes.

`POST https://{host}/.well-known/difp/trade/incoming`
**Body:** `TradeMessage` (signed with sender node's key)

**Response:**
```json
{
    "accepted": true,
    "tradeId":  "string"
}
```

---

### 11 · Conformance Requirements

#### 11.1 MUST Implement
- **Grid algorithm:** `geoToCellNumber` with exact constants from Section 3.1
- **DID format:** `difp://{cellId}/{typeCode}/{componentId}` as canonical identity scheme
- **All 10 component types** from Section 2
- **PresenceRecord schema** from Section 5.1
- **TradeMessage schema** from Section 7.2
- **Status lifecycle** from Section 7.3, including role-based transition rules
- **Atomic write guarantees** from Section 9
- **Well-known endpoints** from Section 10.1 and 10.2

#### 11.2 SHOULD Implement
- **PAD catalog system** from Section 6 for offline item discovery
- **TA analytics node** from Section 8.3 with privacy restrictions enforced
- **Neighbor radius discovery** using `getNearbyCells` from Section 3.4
- **Cross-node trade routing** from Section 10.3

#### 11.3 MAY Implement
- **Custom component types** using codes not in the canonical set — **MUST NOT** reuse canonical codes
- **Extended PresenceRecord fields** — **MUST** preserve all required fields
- **Alternative transport layers** (REST, MQTT, WebSockets) — **MUST** preserve message schemas
- **Message signing** — **RECOMMENDED** for cross-node federation

---

### 12 · Food Security Mandate

This section is normative for implementations that wish to display the **DIFP Food Security Compliant** badge.

#### 12.1 Ask & Donation Support
Implementations **MUST** support the **Ask (a)** and **Donation (d)** trade types. These are the primary mechanism by which DIFP addresses food insecurity — they allow participants to signal demand and surplus without a commercial transaction.

#### 12.2 No Gatekeeping on Food Donation
A DIFP Node **MUST NOT** require payment, subscription, or identity verification beyond basic registration before a component may broadcast a Donation trade. Donations are a human right, not a premium feature.

#### 12.3 Offline Resilience
Implementations operating in low-connectivity regions **SHOULD** implement the PAD catalog system and local caching of PresenceRecords so that participants can at minimum browse nearby suppliers without an active connection.

#### 12.4 Open Interoperability
Implementations **MUST NOT** introduce proprietary barriers that prevent a DIFP-conformant client from another node from discovering, contacting, or trading with their participants. Closed walled gardens are incompatible with the mission of DIFP.

**🌱 Intent:** Every design decision in DIFP should be evaluated against the question: does this reduce the time and friction between a food surplus and a food need? If yes, it belongs in the protocol.

---

### 15 · Message Envelope

DIFP v0.2 introduces a universal message envelope — a fixed outer structure that wraps every message in the network, regardless of its content type, sender role, or transport layer. Every client, node, service, and IoT device speaks the same envelope. The payload is typed and varies; the envelope never does.

**Design principle:** `envelope = network behavior` · `payload = business logic`. The envelope answers who, what, when, and how. The payload answers what specifically.

#### 15.1 Envelope Schema (normative)

```json
{
  "id":        "string",         // unique message ID — deduplication + tracing
  "type":      "string",         // message type (e.g. "trade.ask") — determines payload schema
  "version":   "0.2",            // protocol version — backward compatibility

  "from": {
    "did":       "string",         // full DIFP DID — sender identity
    "node":      "string",         // originating node ID — routing context
    "publicKey": "string",         // Ed25519 public key — signature verification
    "role":      "client | node | service | device"
  },

  "target": {
    "type":  "cell | node | broadcast | direct",
    "value": "string"          // cell ID, node ID, or DID depending on type
  },
  "mode":      "event | request | response",
  "cell":      "string",         // sender's spatial cell — geographic routing anchor

  "timestamp": "string",         // ISO 8601 UTC
  "ttl":       300,              // seconds — prevents infinite propagation
  "nonce":     1024,             // monotonic per DID — prevents replay attacks

  "hash":      "string",         // SHA-256 of canonical(envelope_without_sig + payload)
  "signature": "string",         // Ed25519 sign(hash, privateKey)

  "context": {
    "traceId":  "string",         // cross-message trace chain
    "parentId": "string"          // parent message ID (for request→response linking)
  },

  "payload": {}
}
```

#### 15.2 Field Reference

| Field | Required | Purpose |
|---|---|---|
| `id` | ✓ | Unique message ID for deduplication and tracing. Format: `msg-{timestamp}-{random}` |
| `type` | ✓ | Dot-namespaced message type (e.g. `trade.ask`). Determines payload schema and node processing logic. |
| `version` | ✓ | Protocol version. Nodes reject messages with incompatible versions. |
| `from.did` | ✓ | Full DIFP DID of sender. Used for identity and routing. |
| `from.node` | ✓ | Originating node ID. Used for federation routing and failover. |
| `from.publicKey` | ✓ | Ed25519 public key for signature verification. Nodes cache this per DID. |
| `from.role` | ✓ | Sender role. Determines processing rules and trust level. See Section 17. |
| `target.type` | ✓ | Routing scope: `cell`, `node`, `broadcast`, or `direct`. |
| `target.value` | ✓ | Routing destination: cell ID, node ID, or DID depending on `target.type`. |
| `mode` | ✓ | Message intent: `event` (broadcast), `request` (expects reply), `response` (reply to request). |
| `cell` | ✓ | Sender's geographic cell ID — the spatial routing anchor. All messages carry this regardless of target. |
| `timestamp` | ✓ | ISO 8601 UTC. Nodes reject messages where `timestamp > now + tolerance`. |
| `ttl` | ✓ | Seconds before message expires. Nodes **MUST NOT** forward messages with `ttl ≤ 0`. |
| `nonce` | ✓ | Monotonically increasing integer per DID. Prevents replay attacks. |
| `hash` | ✓ | SHA-256 of canonical JSON of (`envelope_without_hash_and_sig` + `payload`). Detects tampering. |
| `signature` | ✓ | Ed25519 signature over hash. Guarantees authenticity. |
| `context` | — | Optional. Enables message chain tracing for debugging and analytics. |
| `payload` | ✓ | Type-specific business data. Schema defined by `type` value. See Section 16. |

#### 15.3 Canonical Hashing (normative)

Nodes **MUST** compute hash identically. Any deviation produces an invalid signature. The canonical form is defined as:

1. **Step 1:** construct the signable object
   ```javascript
   signable = {
     ...envelope_fields_except_hash_and_signature,
     payload: payload
   }
   ```
2. **Step 2:** canonical JSON (keys sorted, no whitespace, UTF-8)
   ```javascript
   canonical = JSON.stringify(signable, sortedKeys)
   ```
3. **Step 3:** hash
   ```javascript
   hash = SHA256(canonical)
   ```
4. **Step 4:** sign
   ```javascript
   signature = Ed25519.sign(hash, privateKey)
   ```

Keys **MUST** be sorted lexicographically. Whitespace **MUST** be removed. Floating point numbers **MUST** use their minimal decimal representation. Any deviation produces a different hash and an invalid signature across implementations.

#### 15.4 Full Message Example — `trade.ask` (Farmer → Cell)

```json
{
  "id":      "msg-20260417-9f3a2b7c",
  "type":    "trade.ask",
  "version": "0.2",

  "from": {
    "did":       "difp://3440210/f/ali-farm-01",
    "node":      "node-oran-01",
    "publicKey": "ed25519:7f8a9cABC...",
    "role":      "client"
  },

  "target": { "type": "cell", "value": "3440210" },
  "mode":     "event",
  "cell":     "3440210",

  "timestamp": "2026-04-17T14:32:10Z",
  "ttl":       300,
  "nonce":     1024,

  "hash":      "sha256:98fa3c...",
  "signature": "sig:ab34f9c...",

  "context": { "traceId": "trace-abc123" },

  "payload": {
    // trade.ask payload — schema defined in Section 16
  }
}
```

---

### 16 · Message Type System

All DIFP message types follow a dot-namespaced format: `<domain>.<action>`. The domain groups related messages; the action describes the intent. The `type` field determines which payload schema applies and how nodes process the message.

#### 16.1 Message Classes

| Class | Behavior | Examples |
|---|---|---|
| **Command** | Triggers an action on the receiving node | `trade.accept`, `trade.cancel` |
| **Event** | Broadcasts a state change — no response expected | `presence.announce`, `trade.ask` |
| **Query** | Expects a structured response | `query.cell`, `query.resource` |
| **System** | Node-to-node protocol messages | `node.sync`, `node.ping` |

#### 16.2 Full Type Registry

| Domain | Type | Class | Description |
|---|---|---|---|
| **identity** | `identity.register` | Command | Register a new DID on the network |
| | `identity.update` | Command | Update DID metadata |
| | `identity.revoke` | Command | Revoke a DID (e.g. decommission a node) |
| **presence** | `presence.announce` | Event | Participant announces availability in a cell |
| | `presence.update` | Event | Update status or capabilities |
| | `presence.leave` | Event | Participant leaves the cell or goes offline |
| **trade** | `trade.ask` | Event | Broadcast demand signal — I need this resource |
| | `trade.offer` | Event | Broadcast supply signal — I have this resource |
| | `trade.donate` | Event | Broadcast surplus at no cost |
| | `trade.accept` | Command | Accept a pending trade |
| | `trade.reject` | Command | Deny a pending trade |
| | `trade.complete` | Command | Mark trade as completed |
| | `trade.cancel` | Command | Cancel a pending trade (sender only) |
| **query** | `query.cell` | Query | Discover participants in a cell or radius |
| | `query.resource` | Query | Find who has a specific resource available |
| | `query.actor` | Query | Look up a specific DID's presence record |
| **radar** | `radar.snapshot` | Query | Request supply/demand aggregate for a cell |
| | `radar.update` | Event | Node publishes updated radar data for a cell |
| **logistics** | `logistics.request` | Event | Request transport or delivery capacity |
| | `logistics.offer` | Event | Offer available transport or delivery capacity |
| | `logistics.update` | Event | Update logistics status |
| **node** | `node.announce` | System | Node announces itself to the network |
| | `node.sync` | System | Request state sync from another node |
| | `node.ping` | System | Health check between nodes |
| | `node.failover` | System | Signal node going offline, transfer responsibility |
| **reputation** | `reputation.update` | Event | Update reputation score after completed trade |
| | `dispute.open` | Command | Open a dispute on a trade |
| | `dispute.resolve` | Command | Resolve a dispute |
| **sensor** | `sensor.report` | Event | IoT device reports a reading |
| | `automation.trigger`| Command | Automated action triggered by sensor threshold |

Implementations **MUST NOT** use undocumented type strings in production federation. Custom types **MUST** use a prefixed namespace (e.g. `custom.myapp.action`) and **MUST NOT** collide with any type in this registry.

---

### 17 · Roles & Routing

The three new envelope fields — `from.role`, `target`, and `mode` — together define how a message flows through the network and how nodes decide to process it.

#### 17.1 from.role — Sender Role

| Role | Actor | Trust Level | Typical Message Types |
|---|---|---|---|
| **client** | User-facing app | Standard | `trade.*`, `presence.*`, `query.*` |
| **node** | DIFP server | Elevated | `node.*`, `radar.*`, `query.*` |
| **service** | Third-party service | Standard | `query.*`, `radar.*` |
| **device** | IoT sensor | Standard | `sensor.*`, `automation.*`, `trade.donate` |

#### 17.2 target.type — Routing Scope

| Type | Value field contains | Propagation | Use case |
|---|---|---|---|
| **cell** | Cell ID | Node serves all participants in that cell + neighbors | Geographic broadcast — `trade.ask`, `presence.announce` |
| **node** | Node ID string | Routed directly to the named node only | Federation — `node.sync`, `node.ping` |
| **broadcast**| "*" | All reachable nodes (TTL-limited) | Network-wide events — `node.announce` |
| **direct** | Recipient DID | Routed to the specific DID's home node | Private messages — `trade.accept`, `query.response` |

#### 17.3 mode — Message Intent

| Mode | Behavior | Response expected | Examples |
|---|---|---|---|
| **event** | Fire-and-forget broadcast. | No | `trade.ask`, `presence.announce` |
| **request** | Expects a response message. | Yes | `query.cell`, `node.sync`, `node.ping` |
| **response**| Reply to a prior request. | No | `query.response`, `node.sync reply` |

---

### 18 · Node Processing Pipeline

Every DIFP node **MUST** implement the following processing pipeline for every received message.

1. **Structural Validation:** Verify the message is valid JSON. Check all required envelope fields are present and correctly typed. Reject if version is not supported.
2. **Semantic Validation:** Verify: `ttl > 0` · `timestamp ≤ now + tolerance` · `cell` is a valid grid cell ID · `nonce` is greater than last seen for this DID · `type` is known.
3. **Cryptographic Validation:** Recompute hash. Compare with `hash` field. Verify `signature` using `from.publicKey`.
4. **Payload Validation:** Validate payload against the JSON Schema for the message type. Unknown types with valid envelopes **MUST** be forwarded but not processed.
5. **Processing:** Execute type-specific handler (store records, match trades, update radar, etc.).
6. **Propagation:** Decrement `ttl`. Forward based on `target.type`. Messages with `ttl ≤ 0` **MUST NOT** be forwarded.

---

### 13 · Expansion Roadmap (Post-v0.1)

| ID | Topic | Status |
|---|---|---|
| A | **Message Signing** | ✅ v0.2 |
| B | **Supply & Demand Radar** | ✅ v0.2 (Partially) |
| C | **Map Route Animation** | 🔜 v0.3 |
| D | **IoT Integration** | ✅ v0.2 (Partially) |
| E | **USSD / SMS Fallback** | 🔜 v0.3 |
| F | **Admin Monitoring Layer** | 🔜 v0.3 |
| G | **Dispute Resolution** | 🔜 v0.4 |
| H | **Multi-currency Pricing** | 🔜 v0.4 |
| I | **Conformance Test Suite** | 🔜 v0.2.1 |
| J | **Third-Party Registration** | 🔜 v0.2.1 |

---

### 14 · How to Implement DIFP

1. **Implement the Grid:** Port `geoToCellNumber` (Section 3.2) and validate against reference test vectors.
2. **Stand Up a Presence Store:** Create a table/collection for `PresenceRecords` indexed by `cell_id` and `component_type`. Expose well-known endpoints.
3. **Implement the Trade Engine:** Build the four-node write pattern adapted to your storage. Enforce role-based status transitions.
4. **Add the Catalog:** Compile PAD packages for supported regions. Wire up live availability sync.
5. **Federate:** Register your node and implement the federation handshake.

---

### License & Acknowledgements

DIFP was developed by the Djowda Platform team as an open contribution to the global food security challenge.

**Creative Commons Attribution 4.0 International (CC-BY 4.0).** You are free to share and adapt this specification for any purpose, including commercial, provided you give appropriate credit to the Djowda Platform.

To contribute, open an issue or pull request at [djowda.com/difp](https://djowda.com/difp) or contact [sales@djowda.com](mailto:sales@djowda.com).

**DIFP — Djowda Interconnected Food Protocol**
Specification v0.2 · April 2026
[CC-BY 4.0](https://creativecommons.org/licenses/by/4.0/) · [sales@djowda.com](mailto:sales@djowda.com) · [djowda.com](https://djowda.com)
