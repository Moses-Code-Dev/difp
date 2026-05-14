# DIFP — Djowda Interconnected Food Protocol
## Open Protocol Specification · v0.4

**Status:** STABLE — v0.4 Lobby & Registry
**Specification:** DIFP-CORE-0.4
**Issued:** May 2026
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
difp://1711767603/f/ali-farm-01        // Farmer in cell 1,711,767,603 (Algiers)
difp://1711767603/s/safeway-dz-042     // Store in the same cell
difp://1820044/fa/cevital-plant-1      // Factory in cell 1,820,044
```

#### 4.2 Identity Registration

On first registration a DIFP Node **MUST**: (1) accept the participant's GPS coordinates, (2) compute the cell ID using `geoToCellNumber`, (3) assign a `componentId` unique within the node, (4) construct and store the full DID, (5) issue a signed identity token to the client.

#### 4.3 Legacy Encoding (Firebase Reference Implementation)

Implementations using Firebase Auth **MAY** encode identity in the `displayName` field as a compact string:

```
// Format: "{cellId}_{typeCode}"
"1711767603_f"   // Farmer in cell 1,711,767,603
"1711767603_s"   // Store in the same cell
"1820044_fa"     // Factory in cell 1,820,044
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
    did:            "difp://1711767603/f/ali-farm-01"
    component_name: "Ali's Farm"
    cell_id:        1711767603
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
    "version":   "0.3",
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

DIFP v0.2 introduced a universal message envelope — a fixed outer structure that wraps every message in the network, regardless of its content type, sender role, or transport layer. Every client, node, service, and IoT device speaks the same envelope. The payload is typed and varies; the envelope never does.

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
    "did":       "difp://1711767603/f/ali-farm-01",
    "node":      "node-oran-01",
    "publicKey": "ed25519:7f8a9cABC...",
    "role":      "client"
  },

  "target": { "type": "cell", "value": "1711767603" },
  "mode":     "event",
  "cell":     "1711767603",

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

### 19 · PAD System Overview

The PAD — Preloaded Assets Delivery — is the catalog layer of DIFP. It is the mechanism that makes real-time food coordination lightweight, offline-capable, and scalable to millions of participants simultaneously. Rather than transmitting full item records over the wire, DIFP pre-loads a complete food database onto every participant's device at installation time. During live coordination, only a minimal delta — item ID, price, and availability — is ever transmitted.

⚡ **Core insight:** A farmer's name, a product's photo, and a category label never change during a trade. Only the price and availability do. By separating static data (PAD) from live data (delta), DIFP reduces live message payload size by 99% compared to transmitting full item records.

#### 19.1 The Fundamental Split

| Layer | Name | Contents | When transmitted | Typical size |
|---|---|---|---|---|
| **Static** | PAD (`difp_pad`) | Full item metadata — name, brand, description, category, image, unit | Once, at app installation or update | ~5–15 MB compressed per country |
| **Live** | Delta record | Item ID + availability + price only | Every trade message, presence update, or catalog sync | ~20–30 bytes per item |

#### 19.2 Why This Architecture Wins

Consider a store broadcasting its current inventory of 200 items. Without PAD, each item requires transmitting its name, description, category, image URL, and unit — easily 500 bytes per item, or 100KB per broadcast. With PAD, the same broadcast transmits only item IDs, prices, and availability flags — approximately 6KB total. At 10,000 stores in a single city, the difference between network-crushing and network-invisible is the PAD system.

#### 19.3 PAD Scope

Each PAD package is scoped by two dimensions: country code and component type.
**PAD ID format:** `pad_{countryCode}_{typeCode}`

| Example | Coverage |
|---|---|
| `pad_dz_f` | Algeria — Farmer catalog (~6,000 items) |
| `pad_in_s` | India — Store catalog (~6,000 items) |
| `pad_fr_r` | France — Restaurant catalog (~6,000 items) |

---

### 20 · Item Schema — difp_pad

The `difp_pad` format defines the full static item record.

#### 20.1 Full Item Record (difp_pad)

```typescript
interface DifpPadItem {
  id:           number;         // unique item identifier within this PAD package
  name:         string;          // item display name — localized per country
  brand?:       string;         // brand name (optional)
  description?: string;         // short item description (optional)
  category:     string;          // top-level category
  sub_category: string;          // sub-category within category
  image:        string;          // asset reference ID
  unit:         string;          // base unit: kg | g | l | ml | piece | box | ...
  score:        number;          // quality/popularity score 0.0–5.0
  gtin?:        string;         // GS1 barcode
  item_uid:     string;          // globally namespaced DIFP identifier
  tags?:        string[];       // searchable tags (optional)
  season?:      string;         // availability season hint (optional)
}
```

#### 20.2 CSV File Format

PAD packages are distributed as compressed CSV files.
**Header row:** `id,name,brand,description,price,is_available,category,sub_category,image,unit,score`

#### 20.3 Category Taxonomy (normative)

| Category | Code |
|---|---|
| Vegetables | `vegetables` |
| Fruits | `fruits` |
| Grains & Cereals | `grains` |
| Legumes | `legumes` |
| Meat & Poultry | `meat` |
| Fish & Seafood | `fish` |
| Dairy & Eggs | `dairy` |
| Oils & Fats | `oils` |
| Spices & Herbs | `spices` |
| Beverages | `beverages` |
| Packaged & Processed| `packaged` |
| Bakery | `bakery` |
| Agricultural Inputs | `inputs` |
| Packaging & Supply | `supply` |

---

### 21 · Live Delta Sync

The live delta is the only catalog data that ever travels over the wire during active coordination.

#### 21.1 Delta Record Schema

```typescript
interface DeltaRecord {
  id: number;       // references DifpPadItem.id in the local PAD
  a:  boolean;      // availability (true = in stock, false = out of stock)
  p:  number;        // current price in local currency units
}
```

#### 21.2 Wire Format Example

```json
{
  "catalog": [
    { "id": 1,   "a": true,  "p": 120  },
    { "id": 2,   "a": true,  "p": 680  }
  ]
}
```

#### 21.3 How the Client Resolves a Delta

On receiving a delta record, the client looks up the `id` in the `localPAD` and merges the live price and availability.

**PAD version mismatch:** If a delta references an item ID not found in the local PAD, the client **MUST** still process the trade using the item ID as a fallback identifier and schedule a PAD update.

---

### 22 · PAD Distribution & Versioning

#### 22.1 PAD Version String
**Format:** `pad_{countryCode}_{typeCode}_v{n}`

#### 22.2 PAD Package Structure
- `items.csv`
- `manifest.json`
- `assets/` (webp images)

#### 22.3 Distribution Channels
1. **Bundled at Installation:** Ensures first-time users can coordinate immediately.
2. **Background OTA Update:** Downloads newer versions atomically.
3. **Node-Served PAD:** Served via `GET /.well-known/difp/pad/{padId}`.

---

### 23 · PAD Conformance Requirements

#### 23.1 MUST Implement
- PAD package bundled at installation.
- PAD item schema from Section 20.1.
- Live delta format from Section 21.1.
- `pad_version` field in catalog-bearing message payloads.
- Checksum verification before applying updates.
- Stable item IDs across versions.

#### 23.2 MUST NOT
- PAD CSV files **MUST NOT** contain price or availability data.
- Images **MUST NOT** be embedded or base64-encoded inside the CSV.
- Item IDs **MUST NOT** be reused or reassigned.

---

### 13 · Expansion Roadmap

| ID | Topic | Status |
|---|---|---|
| A | **Message Signing** | ✅ v0.2 |
| B | **Supply & Demand Radar** | ✅ v0.2 (Partially) |
| C | **Map Route Animation** | ✅ v0.4 |
| D | **IoT Integration** | ✅ v0.2 (Partially) |
| E | **USSD / SMS Fallback** | 🔜 v0.5 |
| F | **Admin Monitoring Layer** | 🔜 v0.5 |
| G | **Dispute Resolution** | 🔜 v0.5 |
| H | **Multi-currency Pricing** | 🔜 v0.5 |
| I | **Conformance Test Suite** | ✅ v0.4 (Partial) |
| J | **Third-Party Node Registration** | ✅ v0.4 |
| K | **Lobby Layer & Federated Discovery** | ✅ v0.4 |

---

### 14 · How to Implement DIFP

1. **Implement the Grid:** Port `geoToCellNumber` (Section 3.2). Validate against `test-vectors/grid.json`.
2. **Stand Up a Presence Store:** Expose well-known endpoints.
3. **Implement the Trade Engine:** Build the four-node write pattern.
4. **Add the Catalog:** Compile PAD packages and wire up live availability sync.
5. **Federate:** Implement the federation handshake (Section 10).
6. **Register via Lobby Layer:** Compute your node's lobbies with `cellIdToLobbyId`, announce to a registry with `registry.announce`, and expose the registry query endpoints (Sections 24–28).

---

### License & Acknowledgements

DIFP was developed by the Djowda Platform team as an open contribution to the global food security challenge.

**Creative Commons Attribution 4.0 International (CC-BY 4.0).** You are free to share and adapt this specification for any purpose, including commercial, provided you give appropriate credit to the Djowda Platform.

To contribute, open an issue or pull request at [djowda.com/difp](https://djowda.com/difp) or contact [sales@djowda.com](mailto:sales@djowda.com).

**DIFP — Djowda Interconnected Food Protocol**
Specification v0.4 · May 2026
[CC-BY 4.0](https://creativecommons.org/licenses/by/4.0/) · [sales@djowda.com](mailto:sales@djowda.com) · [djowda.com](https://djowda.com)

---

### 24 · Lobby Layer

The Lobby layer is the second spatial abstraction in DIFP, sitting directly above the MinMax99 cell grid. It answers a fundamental network question that cells alone cannot: given a location anywhere on Earth, which node should I contact to find participants there?

> **Cells handle precision. Lobbies handle routing.** A cell tells you where a participant is. A lobby tells you which node knows about participants in that region.

#### 24.1 Lobby Constants (normative)

All conformant implementations **MUST** use the following constants without modification. These values are derived directly from the Layer 0 grid constants and are **not** independently configurable.

```python
# Lobby grid constants — derived from MinMax99, MUST NOT be changed independently
LOBBY_SIZE        = 41          # cells per lobby on each axis (41 × 41 = 1,681 cells)
NUM_LOBBY_COLUMNS = 2000        # 82,000 / 41 — exact, 82,000 is divisible by 41
NUM_LOBBY_ROWS    = 1025        # ceil(42,000 / 41) — ceiling; 42,000 mod 41 = 16 (partial strip)
TOTAL_LOBBIES     = 2050000     # 2,000 × 1,025
```

#### 24.2 Physical Coverage

Each lobby covers approximately **20.5 km × 20.5 km** (41 cells × 500 m per cell). This is intentionally sized to match a typical urban district, rural municipality, or regional market catchment area — making it a natural unit for node responsibility partitioning.

#### 24.3 CellToLobbyNumber — Canonical Algorithm

The mapping from cell to lobby is deterministic and requires no network call. Any implementation **MUST** produce identical lobby IDs from identical cell IDs.

```javascript
// Layer 1 encoding — same row-major philosophy as Layer 0
function cellIdToLobbyId(cellId) {
    let { xCell, yCell } = cellNumberToXY(cellId);    // Layer 0 reverse lookup
    let lobbyX = Math.floor(xCell / LOBBY_SIZE);      // 0..1999
    let lobbyY = Math.floor(yCell / LOBBY_SIZE);      // 0..1024
    return BigInt(lobbyX) * BigInt(NUM_LOBBY_ROWS) + BigInt(lobbyY);
}

// Reverse: lobbyId → [lobbyX, lobbyY]
function lobbyIdToXY(lobbyId) {
    let lobbyX = Math.floor(Number(lobbyId) / NUM_LOBBY_ROWS);
    let lobbyY = Number(lobbyId) % NUM_LOBBY_ROWS;
    return { lobbyX, lobbyY };
}

// Local position within a lobby
function cellIdToLocalXY(cellId) {
    let { xCell, yCell } = cellNumberToXY(cellId);
    return {
        localX: xCell % LOBBY_SIZE,  // 0..40
        localY: yCell % LOBBY_SIZE   // 0..40
    };
}
```

#### 24.4 Reference Test Vectors

Validate your Layer 1 implementation against these vectors (see [`test-vectors/lobby.json`](./test-vectors/lobby.json) for the full machine-readable set).

| Location | Cell ID | Lobby ID |
|---|---|---|
| Algiers, Algeria | 1,711,767,603 | 1,988,574 |
| Paris, France | 1,705,129,761 | 1,980,620 |
| Tokyo, Japan | 2,989,365,749 | 3,474,340 |
| New York, USA | 991,131,039 | 1,151,116 |
| Gulf of Guinea (0,0) | 1,683,170,000 | 1,954,832 |

#### 24.5 Node Lobby Computation

A node determines which lobbies it is responsible for by mapping all stored cell IDs through `cellIdToLobbyId` and collecting the unique set.

```javascript
function computeOwnedLobbies(storedCellIds) {
    let lobbies = new Set();
    for (let cellId of storedCellIds) {
        lobbies.add(cellIdToLobbyId(cellId));
    }
    return lobbies; // deduplicated — typically far fewer entries than cells
}
```

---

### 25 · Node Registry

The Node Registry is a publicly queryable index that maps lobby IDs to the DIFP nodes that hold data for those lobbies. It is the routing backbone of the federated network — without it, a client has no way to discover which node to contact for a given location.

In DIFP v0.4, the registry is intentionally kept simple: it is a pure spatial routing index with no health scoring, no node roles, and no replication semantics. Those capabilities are deferred to v0.5.

#### 25.1 Registry Data Model

```json
// Conceptual model — implementation storage is not prescribed
// Registry: Map<lobbyId, Set<string>>
// lobbyId → set of node endpoint URLs
{
  "83907": ["https://node-oran-01.difp", "https://node-algiers-02.difp"],
  "84582": ["https://node-paris-01.difp"],
  "76786": ["https://node-lagos-01.difp", "https://node-lagos-02.difp"]
}
```

#### 25.2 Registry Architecture — v0.4

DIFP v0.4 supports two registry deployment patterns. Both expose the same query interface (Section 27), so clients are unaffected by which pattern a network uses:

- **Centralized registry:** A single globally accessible service (e.g. `registry-global.difp`). Suitable for early deployments.
- **Federated registries:** Multiple regional registries that exchange lobby announcements. Each registry exposes a `/.well-known/difp/registry/peers` endpoint listing peer registries.

#### 25.3 Registry Well-Known Endpoints

Any service acting as a DIFP registry **MUST** expose the following endpoints:

```
# Query which nodes serve a lobby
GET /.well-known/difp/registry/lobby/{lobbyId}
Response: { "lobbyId": 83907, "nodes": ["https://node-oran-01.difp", ...] }

# Bulk query — resolve multiple lobbies at once
POST /.well-known/difp/registry/lobby/batch
Body:     { "lobbyIds": [83907, 83908, 84582] }
Response: { "results": { "83907": [...], "83908": [...], "84582": [...] } }

# List all known peer registries (for federation bootstrapping)
GET /.well-known/difp/registry/peers
Response: { "registries": ["https://registry-global.difp", "https://registry-africa.difp"] }
```

#### 25.4 Node Registration at Startup

When a DIFP node starts, it **MUST** announce its lobbies to at least one registry. The sequence is:

1. Call `computeOwnedLobbies(storedCellIds)` — compute the set of lobby IDs the node currently holds data for.
2. Send a `registry.announce` message (Section 27.1) to the registry with the lobby set and node endpoint.
3. Re-announce whenever the lobby set changes (new participants registered, participants migrated out).

---

### 26 · Discovery Flow

The lobby and registry layers exist to enable a single end-to-end use case: given a GPS location, find the DIFP nodes that hold participant data for that location.

#### 26.1 Full Discovery Sequence

```javascript
// Step 1 — client computes spatial address from GPS
const cellId  = geoToCellNumber(latitude, longitude);
const lobbyId = cellIdToLobbyId(cellId);

// Step 2 — client queries registry for nodes serving that lobby
const nodes = await GET(`/.well-known/difp/registry/lobby/${lobbyId}`);
// → ["https://node-oran-01.difp", "https://node-algiers-02.difp"]

// Step 3 — client queries nodes for presence data
for (const nodeEndpoint of nodes) {
    const presenceRecords = await GET(`${nodeEndpoint}/.well-known/difp/cell/${cellId}`);
    if (presenceRecords.length > 0) break; // or merge results from multiple nodes
}

// Step 4 — client uses presence records for trade initiation (Sections 5–7)
```

#### 26.2 Neighbor Discovery

For radius-based discovery, the client resolves lobby IDs for all neighbor cells and batches the registry lookup:

```javascript
const nearbyCells  = getNearbyCells(centerCellId, radius);
const nearbyLobbies = [...new Set(nearbyCells.map(cellIdToLobbyId))];

// Single batched registry call instead of N individual calls
const results = await POST('/.well-known/difp/registry/lobby/batch', { lobbyIds: nearbyLobbies });

// Then query unique nodes from the merged result
const uniqueNodes = [...new Set(Object.values(results.results).flat())];
for (const nodeEndpoint of uniqueNodes) {
    presenceRecords.push(...await GET(`${nodeEndpoint}/.well-known/difp/cell/${cellId}`));
}
```

#### 26.3 No Registry Available

If a client cannot reach any registry, it **MUST** fall back to direct node queries using hardcoded or cached node endpoints from prior sessions. A client **MUST NOT** fail silently — it **SHOULD** notify the user that discovery may be incomplete.

#### 26.4 Multiple Nodes for the Same Lobby

The registry may return multiple nodes for a single lobby. Clients **MUST** try nodes in order and **MAY** merge results from multiple nodes. Clients **MUST** deduplicate participants across node responses using their DID.

---

### 27 · Registry Messages

DIFP v0.4 defines three canonical message types for the registry system. These extend the message type registry from Section 16 and follow the same dot-namespaced format and envelope structure from Section 15.

#### 27.1 registry.announce

Sent by a node to a registry to declare which lobbies it serves. This is the only write operation in the v0.4 registry protocol.

```json
{
  "id":      "msg-20260510-la-9f3a",
  "type":    "registry.announce",
  "version": "0.4",
  "from": {
    "did":  "difp://3440210/a/node-oran-01",
    "node": "node-oran-01",
    "role": "node"
  },
  "target": { "type": "direct", "value": "difp://registry/global" },
  "mode":  "event",
  "cell":  "3440210",
  "timestamp": "2026-05-10T08:00:00Z",
  "ttl": 3600,
  "payload": {
    "nodeEndpoint": "https://node-oran-01.difp",
    "lobbies": [83907, 83908, 83935, 84000]
  }
}
```

#### 27.2 registry.query

Sent by a client or node to a registry to discover which nodes serve a given lobby. This is a `request` mode message — a `registry.response` **MUST** be returned.

```json
{
  "id":      "msg-20260510-rq-4b2c",
  "type":    "registry.query",
  "version": "0.4",
  "from": {
    "did":  "difp://3440210/u/consumer-42",
    "node": "node-oran-01",
    "role": "client"
  },
  "target": { "type": "direct", "value": "difp://registry/global" },
  "mode":  "request",
  "cell":  "3440210",
  "timestamp": "2026-05-10T09:32:00Z",
  "ttl": 30,
  "payload": {
    "lobbyId": 83907
  }
}
```

> For bulk queries use `"lobbyIds": [83907, 83908, ...]` in the payload.

#### 27.3 registry.response

Returned by a registry in reply to a `registry.query`. The `context.parentId` **MUST** match the originating query message `id`.

```json
{
  "id":      "msg-20260510-rr-7f9d",
  "type":    "registry.response",
  "version": "0.4",
  "from": {
    "did":  "difp://0/a/registry-global",
    "node": "registry-global",
    "role": "node"
  },
  "target": { "type": "direct", "value": "difp://3440210/u/consumer-42" },
  "mode":  "response",
  "cell":  "0",
  "timestamp": "2026-05-10T09:32:00Z",
  "ttl": 60,
  "context": { "parentId": "msg-20260510-rq-4b2c" },
  "payload": {
    "lobbyId": 83907,
    "nodes": [
      "https://node-oran-01.difp",
      "https://node-algiers-02.difp"
    ]
  }
}
```

#### 27.4 Updated Message Type Registry

The following types are added to the registry from Section 16.2:

| Domain | Type | Class | Description |
|---|---|---|---|
| **registry** | `registry.announce` | System | Node announces which lobbies it serves to a registry |
| | `registry.query` | Query | Client or node queries which nodes serve a lobby |
| | `registry.response` | Query | Registry replies with the node list for a lobby |

---

### 28 · v0.4 Conformance Requirements

This section is normative. Conformant DIFP v0.4 implementations **MUST** satisfy all requirements below in addition to all prior conformance requirements from Sections 11 and 23.

#### 28.1 MUST Implement (Nodes)

- `cellIdToLobbyId` with exact constants from Section 24.1.
- `computeOwnedLobbies` — compute lobby set from all stored cell IDs.
- Send `registry.announce` to at least one registry on startup and on lobby set change.
- Expose `/.well-known/difp/cell/{cellId}` responding with current `PresenceRecord` list.

#### 28.2 MUST Implement (Registries)

- Accept `registry.announce` messages and update the lobby→node index.
- `GET /.well-known/difp/registry/lobby/{lobbyId}` — single lobby query.
- `POST /.well-known/difp/registry/lobby/batch` — bulk lobby query.

#### 28.3 MUST Implement (Clients)

- Compute `lobbyId = cellIdToLobbyId(cellId)` before initiating discovery.
- Query the registry for node endpoints before querying cell presence.
- Fall back to cached or hardcoded node endpoints if no registry is reachable (Section 26.3).
- Deduplicate participants by DID when merging responses from multiple nodes.

#### 28.4 SHOULD Implement

- `GET /.well-known/difp/registry/peers` — expose known peer registries for federation bootstrapping.
- Re-announce lobbies to registry on a scheduled interval (e.g. every 24 hours) to handle registry restarts.
- Batch neighbor lobby lookups using `POST /.well-known/difp/registry/lobby/batch` to minimize round trips.
