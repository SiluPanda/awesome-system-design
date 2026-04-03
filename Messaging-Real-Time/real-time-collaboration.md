# Real-Time Collaboration (Google Docs)

## 1. Problem Statement & Requirements

### Functional Requirements
- **Concurrent editing** — multiple users edit the same document simultaneously
- **Real-time sync** — edits appear on all collaborators' screens within ~100 ms
- **Conflict resolution** — concurrent edits to the same section must merge correctly without data loss
- **Cursor & selection sync** — see where other users' cursors are and what they've selected
- **Presence** — show who is currently viewing/editing the document
- **Undo/redo** — per-user undo that doesn't revert other users' changes
- **Version history** — view and restore previous versions of the document
- **Comments & suggestions** — inline annotations anchored to document positions
- **Offline editing** — queue local changes and sync when reconnected
- **Rich text** — bold, italic, headings, lists, tables, images

### Non-Functional Requirements
- **Low latency** — local edits appear instantly (< 16 ms); remote edits visible within 100-200 ms
- **Consistency** — all users converge to the same document state (eventual consistency)
- **High availability** — documents always editable, even during partial outages
- **Scalability** — support millions of documents, thousands with 50+ concurrent editors
- **Durability** — no data loss; every accepted edit is persisted
- **Ordering** — causal ordering of operations preserved

### Out of Scope
- Spreadsheet-specific features (cell formulas, recalculation)
- Presentation/slides editing
- Drawing/whiteboard (requires different data model)
- Real-time voice/video (that's WebRTC)

---

## 2. Scale Estimations

### Users & Documents
| Metric | Value |
|--------|-------|
| Total users | 500M |
| Daily Active Users (DAU) | 50M |
| Total documents | 5B |
| Documents edited per day | 100M |
| Avg concurrent editors per document | 3-5 |
| Max concurrent editors (edge case) | 100+ |
| Avg edits per user per minute | 30 (typing) |

### Operations
| Metric | Value |
|--------|-------|
| Edits per second (global) | 50M DAU × 30 edits/min × (10 min active/session) / 86400 = **~175K ops/sec** |
| Peak edits per second | ~500K ops/sec |
| Cursor position updates per second | ~350K/sec (2× edit rate, includes mouse moves) |
| WebSocket connections (concurrent) | ~10M |

### Storage
| Metric | Value |
|--------|-------|
| Average document size | 50 KB (text only) |
| Average operation size | 100 bytes |
| Operations per document per day | ~1,000 |
| Operation log storage per day | 100M docs × 1,000 ops × 100 bytes = **~10 TB/day** |
| Document snapshots | 5B × 50 KB = **~250 TB** |
| Operation log (30-day retention before compaction) | ~300 TB |

### Network
| Metric | Value |
|--------|-------|
| Inbound (edits) | 175K × 100 bytes = **~17.5 MB/s** |
| Outbound (broadcast to collaborators) | 17.5 MB/s × avg 3 collaborators = **~52.5 MB/s** |
| WebSocket server count (20K conn/server) | ~500 servers |

---

## 3. The Core Problem: Concurrent Editing

```
+--------------------------------------------------------------+
|            The Concurrent Editing Problem                      |
|                                                               |
|  Initial document: "ABCD"                                    |
|                                                               |
|  User A (offline for a moment):                              |
|    Inserts "X" at position 1 → "AXBCD"                      |
|                                                               |
|  User B (offline for a moment):                              |
|    Deletes char at position 3 → "ABD"                       |
|                                                               |
|  Problem: How do we merge these?                             |
|                                                               |
|  Naive merge:                                                |
|    Apply A's op to B's state: Insert "X" at pos 1 → "AXBD" |
|    Apply B's op to A's state: Delete at pos 3 → "AXBD"     |
|    ✅ Same result (got lucky this time)                     |
|                                                               |
|  But consider:                                               |
|    User A: Insert "X" at position 2                         |
|    User B: Insert "Y" at position 2                         |
|                                                               |
|    A applies B's op: "AXYB..." or "AYXB..."?               |
|    B applies A's op: "AYXB..." or "AXYB..."?               |
|    ❌ Order depends on arrival → different results!         |
|                                                               |
|  We need an algorithm that guarantees convergence            |
|  regardless of operation ordering.                           |
+--------------------------------------------------------------+
```

---

## 4. Algorithm Deep Dive: OT vs CRDTs

### Option 1: Operational Transformation (OT)

```
+--------------------------------------------------------------+
|            Operational Transformation (OT)                    |
|                                                               |
|  Core idea: Transform operations against each other so      |
|  that applying them in any order produces the same result.   |
|                                                               |
|  Document: "ABCD"                                            |
|                                                               |
|  Op1 (User A): Insert("X", pos=1)  → "AXBCD"              |
|  Op2 (User B): Delete(pos=3)        → "ABD"                |
|                                                               |
|  Transform(Op1, Op2):                                        |
|    Op1 inserts at pos 1                                     |
|    Op2 deletes at pos 3                                     |
|    Since Op1 inserts BEFORE Op2's position,                 |
|    Op2's position must shift right by 1:                    |
|    Op2' = Delete(pos=4)                                     |
|                                                               |
|  Transform(Op2, Op1):                                        |
|    Op2 deletes at pos 3                                     |
|    Op1 inserts at pos 1                                     |
|    Since Op2 deletes AFTER Op1's position,                  |
|    Op1 stays the same:                                      |
|    Op1' = Insert("X", pos=1)                                |
|                                                               |
|  Results:                                                    |
|  User A: "ABCD" → apply Op1 → "AXBCD" → apply Op2' → "AXBD"|
|  User B: "ABCD" → apply Op2 → "ABD"  → apply Op1' → "AXBD"|
|  ✅ Both converge to "AXBD"                                 |
|                                                               |
|                                                               |
|  The Transform Function:                                    |
|  +--------------------------------------+                   |
|  |                                       |                   |
|  |  transform(op1, op2) → (op1', op2')  |                   |
|  |                                       |                   |
|  |  Property: apply(apply(S, op1), op2')|                   |
|  |          = apply(apply(S, op2), op1')|                   |
|  |                                       |                   |
|  |  This is called the "Transformation  |                   |
|  |  Property" or "TP1"                  |                   |
|  |                                       |                   |
|  +--------------------------------------+                   |
|                                                               |
|  Visualization (OT Diamond):                                |
|                                                               |
|           S (initial state)                                  |
|          / \                                                 |
|     Op1 /   \ Op2                                           |
|        /     \                                               |
|      S1       S2                                             |
|        \     /                                               |
|    Op2' \   / Op1'                                          |
|          \ /                                                 |
|           S'  (converged state)                              |
|                                                               |
+--------------------------------------------------------------+
```

### OT Transform Rules

```
+--------------------------------------------------------------+
|            OT Transform Rules (Text Operations)               |
|                                                               |
|  Operations: Insert(char, pos), Delete(pos)                 |
|                                                               |
|  transform(Insert(c1, p1), Insert(c2, p2)):                 |
|  +--------------------------------------+                   |
|  | if p1 < p2:                          |                   |
|  |   return (Insert(c1, p1),            |                   |
|  |           Insert(c2, p2+1))          |                   |
|  | elif p1 > p2:                        |                   |
|  |   return (Insert(c1, p1+1),          |                   |
|  |           Insert(c2, p2))            |                   |
|  | else: (same position)               |                   |
|  |   // tie-break by user ID           |                   |
|  |   if user1 < user2:                  |                   |
|  |     return (Insert(c1, p1),          |                   |
|  |             Insert(c2, p2+1))        |                   |
|  |   else:                              |                   |
|  |     return (Insert(c1, p1+1),        |                   |
|  |             Insert(c2, p2))          |                   |
|  +--------------------------------------+                   |
|                                                               |
|  transform(Insert(c, p1), Delete(p2)):                      |
|  +--------------------------------------+                   |
|  | if p1 <= p2:                         |                   |
|  |   return (Insert(c, p1),             |                   |
|  |           Delete(p2+1))              |                   |
|  | else:                                |                   |
|  |   return (Insert(c, p1-1),           |                   |
|  |           Delete(p2))                |                   |
|  +--------------------------------------+                   |
|                                                               |
|  transform(Delete(p1), Delete(p2)):                         |
|  +--------------------------------------+                   |
|  | if p1 < p2:                          |                   |
|  |   return (Delete(p1), Delete(p2-1))  |                   |
|  | elif p1 > p2:                        |                   |
|  |   return (Delete(p1-1), Delete(p2))  |                   |
|  | else: (same char deleted)            |                   |
|  |   return (NoOp, NoOp)                |                   |
|  +--------------------------------------+                   |
+--------------------------------------------------------------+
```

### OT Architecture (Google Docs Approach)

```
+--------------------------------------------------------------+
|         OT with Central Server (Google Docs Model)           |
|                                                               |
|  +--------+         +--------------+         +--------+    |
|  |Client A|====WS===|   OT Server   |====WS===|Client B|    |
|  |        |         |   (per doc)   |         |        |    |
|  +--------+         +--------------+         +--------+    |
|                                                               |
|  Server maintains:                                           |
|  - Canonical document state                                 |
|  - Global operation revision number                         |
|  - Operation history log                                    |
|                                                               |
|  Client maintains:                                           |
|  - Local document state                                     |
|  - Pending operations (sent but not yet ACKed)              |
|  - Buffer of unconfirmed local edits                        |
|                                                               |
|  Protocol:                                                   |
|  1. Client applies edit locally (instant feedback)          |
|  2. Client sends op to server with base revision number     |
|  3. Server transforms op against any concurrent ops         |
|     that happened since that revision                       |
|  4. Server applies transformed op, increments revision      |
|  5. Server broadcasts transformed op to all other clients  |
|  6. Server ACKs original client with new revision           |
|  7. Other clients transform received op against their      |
|     pending local ops, then apply                           |
|                                                               |
|  Key insight: Server is the single source of truth.         |
|  It serializes all operations into a total order.           |
|  This avoids the complexity of peer-to-peer OT.             |
+--------------------------------------------------------------+
```

### Option 2: CRDTs (Conflict-free Replicated Data Types)

```
+--------------------------------------------------------------+
|            CRDTs for Collaborative Editing                    |
|                                                               |
|  Core idea: Design the data structure so that concurrent    |
|  operations ALWAYS commute — no transformation needed.       |
|                                                               |
|  Instead of position-based operations (Insert at pos 3),    |
|  assign each character a globally unique, ordered ID.        |
|                                                               |
|                                                               |
|  Sequence CRDT (e.g., RGA, LSEQ, Yjs YATA):               |
|  +------------------------------------------+               |
|  |                                           |               |
|  |  Document: "ABC"                          |               |
|  |                                           |               |
|  |  Internal representation:                 |               |
|  |  +-----------+--------+----------------+ |               |
|  |  | Char      | ID     | Visible?       | |               |
|  |  +-----------+--------+----------------+ |               |
|  |  | 'A'       | (1,A)  | true           | |               |
|  |  | 'B'       | (2,A)  | true           | |               |
|  |  | 'C'       | (3,A)  | true           | |               |
|  |  +-----------+--------+----------------+ |               |
|  |                                           |               |
|  |  ID format: (lamport_timestamp, user_id) |               |
|  |  IDs are globally unique and totally     |               |
|  |  ordered.                                |               |
|  |                                           |               |
|  |  Insert 'X' between A and B:             |               |
|  |  → Create ID between (1,A) and (2,A)    |               |
|  |  → New ID: (1.5,B) or use fractional    |               |
|  |    indexing / tree-based IDs             |               |
|  |                                           |               |
|  |  Delete 'B':                              |               |
|  |  → Mark (2,A) as tombstone (visible=false)|               |
|  |  → Never remove from structure           |               |
|  |                                           |               |
|  +------------------------------------------+               |
|                                                               |
|  Yjs (most popular CRDT library):                           |
|  +------------------------------------------+               |
|  |                                           |               |
|  |  Uses YATA (Yet Another Transformation   |               |
|  |  Approach) algorithm:                    |               |
|  |                                           |               |
|  |  Each item has:                           |               |
|  |  - id: {client_id, clock}                |               |
|  |  - left: reference to left neighbor      |               |
|  |  - right: reference to right neighbor    |               |
|  |  - content: the actual character(s)      |               |
|  |  - deleted: boolean (tombstone)          |               |
|  |                                           |               |
|  |  Concurrent inserts at same position:    |               |
|  |  → Deterministic ordering by client_id   |               |
|  |  → No transformation needed!             |               |
|  |                                           |               |
|  |  Merging: Just integrate all items into  |               |
|  |  the linked list following YATA rules.   |               |
|  |  Always converges.                       |               |
|  |                                           |               |
|  +------------------------------------------+               |
+--------------------------------------------------------------+
```

### OT vs CRDT Comparison

```
+--------------------------------------------------------------------+
|                   OT vs CRDT Comparison                            |
|                                                                     |
|  +------------------+------------------+----------------------+   |
|  | Aspect           | OT               | CRDT                 |   |
|  +------------------+------------------+----------------------+   |
|  | Architecture     | Requires central | Peer-to-peer OK      |   |
|  |                  | server           | (no central server)  |   |
|  +------------------+------------------+----------------------+   |
|  | Correctness      | Hard to prove    | Mathematically       |   |
|  |                  | (many edge cases)| proven convergence   |   |
|  +------------------+------------------+----------------------+   |
|  | Complexity       | Transform funcs  | Data structure       |   |
|  |                  | for each op pair | design is complex    |   |
|  +------------------+------------------+----------------------+   |
|  | Memory overhead  | Low — positions  | Higher — unique IDs  |   |
|  |                  | are integers     | per character +      |   |
|  |                  |                  | tombstones           |   |
|  +------------------+------------------+----------------------+   |
|  | Offline support  | Difficult —      | Excellent —          |   |
|  |                  | transform chain  | merge anytime        |   |
|  |                  | grows long       |                      |   |
|  +------------------+------------------+----------------------+   |
|  | Latency          | Must wait for    | Can apply locally    |   |
|  |                  | server ACK       | and sync later       |   |
|  |                  | before confirming|                      |   |
|  +------------------+------------------+----------------------+   |
|  | Used by          | Google Docs,     | Figma, Apple Notes,  |   |
|  |                  | CKEditor 5       | Notion (partially),  |   |
|  |                  |                  | Yjs, Automerge       |   |
|  +------------------+------------------+----------------------+   |
|  | Rich text        | Well understood  | Growing support      |   |
|  |                  | (20+ years)      | (Yjs + ProseMirror)  |   |
|  +------------------+------------------+----------------------+   |
|  | Undo/Redo        | Complex — must   | Simpler — undo is   |   |
|  |                  | transform undo   | inverse operation    |   |
|  |                  | against others   | in CRDT              |   |
|  +------------------+------------------+----------------------+   |
|                                                                     |
|  Recommendation for this design:                                   |
|  +------------------------------------------+                     |
|  |  Use OT with central server for:         |                     |
|  |  ✅ Simpler server architecture          |                     |
|  |  ✅ Lower memory overhead                |                     |
|  |  ✅ Battle-tested (Google Docs uses this) |                     |
|  |                                           |                     |
|  |  Use CRDTs for:                           |                     |
|  |  ✅ Offline-first applications            |                     |
|  |  ✅ Peer-to-peer architectures            |                     |
|  |  ✅ Simpler correctness guarantees        |                     |
|  |                                           |                     |
|  |  Our design: OT with central server      |                     |
|  |  (Google Docs model) as primary, with     |                     |
|  |  discussion of CRDT alternative.          |                     |
|  +------------------------------------------+                     |
+--------------------------------------------------------------------+
```

---

## 5. High-Level Architecture

```
+--------------------------------------------------------------------------+
|                Real-Time Collaboration Architecture                       |
|                                                                           |
|  +------------+      +--------------+      +----------------------+     |
|  |   Clients   |=WS==|  WebSocket    |----->|  Collaboration       |     |
|  | (Browsers)  |      |  Gateway      |      |  Service (OT Engine) |     |
|  +------------+      +--------------+      +----------+-----------+     |
|                                                        |                  |
|                             +---------------------------+                  |
|                             |                           |                  |
|                             v                           v                  |
|                      +--------------+          +--------------+          |
|                      |  Operation    |          |  Document    |          |
|                      |  Log          |          |  Store       |          |
|                      |  (Kafka)      |          |  (Cloud DB)  |          |
|                      +--------------+          +--------------+          |
|                                                                           |
|                      +--------------+          +--------------+          |
|                      |  Presence     |          |  Version     |          |
|                      |  Service      |          |  History     |          |
|                      |  (Redis)      |          |  Service     |          |
|                      +--------------+          +--------------+          |
+--------------------------------------------------------------------------+
```

### Detailed Component Architecture

```
                    +-----------------------------------------+
                    |           Client (Browser)               |
                    |                                           |
                    |  +-------------------------------------+ |
                    |  |  Editor (ProseMirror / Quill / etc) | |
                    |  |  +---------+  +------------------+ | |
                    |  |  | Local   |  | OT Client Engine | | |
                    |  |  | State   |  |                   | | |
                    |  |  | (doc +  |  | • Apply local ops| | |
                    |  |  |  cursor)|  | • Send to server | | |
                    |  |  |         |  | • Transform      | | |
                    |  |  |         |  |   incoming ops   | | |
                    |  |  |         |  | • Buffer pending | | |
                    |  |  +---------+  +------------------+ | |
                    |  +-------------------------------------+ |
                    +------------------+----------------------+
                                       | WebSocket
                    +------------------v----------------------+
                    |        WebSocket Gateway                  |
                    |  (500 servers, 20K connections each)     |
                    |                                          |
                    |  • Authenticate connections              |
                    |  • Route to document session             |
                    |  • Fan-out ops to co-editors             |
                    +------------------+----------------------+
                                       |
                    +------------------v----------------------+
                    |     Collaboration Service (OT Engine)    |
                    |                                          |
                    |  +------------------------------------+ |
                    |  |  Document Session Manager           | |
                    |  |                                      | |
                    |  |  Per-document in-memory state:      | |
                    |  |  • Current document content         | |
                    |  |  • Revision counter                 | |
                    |  |  • Operation history (recent)       | |
                    |  |  • Connected clients list           | |
                    |  |  • Cursor positions                 | |
                    |  |                                      | |
                    |  |  OT logic:                          | |
                    |  |  • Receive op at revision R         | |
                    |  |  • Transform against ops R+1..HEAD  | |
                    |  |  • Apply to canonical state         | |
                    |  |  • Broadcast transformed op         | |
                    |  |  • Persist to operation log         | |
                    |  +------------------------------------+ |
                    +------------------+----------------------+
                                       |
           +---------------------------+-----------------------+
           v                           v                       v
  +--------------+          +--------------+         +--------------+
  | Operation Log |          | Document DB   |         |   Redis       |
  | (Append-only) |          | (Snapshots)   |         |  (Presence +  |
  |               |          |               |         |   Cursors)    |
  | Kafka or      |          | Cloud Spanner |         |               |
  | Cassandra     |          | or PostgreSQL |         | user→cursor   |
  |               |          |               |         | doc→editors   |
  | partition by  |          | doc_id → blob |         |               |
  | doc_id        |          | + metadata    |         |               |
  +--------------+          +--------------+         +--------------+
```

---

## 6. Detailed OT Client-Server Protocol

### Client State Machine

```
+--------------------------------------------------------------+
|            Client State Machine                               |
|                                                               |
|  The client is always in one of three states:               |
|                                                               |
|  +------------------------------------------------------+   |
|  |                                                       |   |
|  |  State 1: SYNCHRONIZED                               |   |
|  |  +-------------------------------------+             |   |
|  |  | No pending ops.                     |             |   |
|  |  | Client revision = server revision.  |             |   |
|  |  |                                      |             |   |
|  |  | On local edit:                       |             |   |
|  |  |   → Apply locally                   |             |   |
|  |  |   → Send op to server               |             |   |
|  |  |   → Transition to AWAITING_ACK      |             |   |
|  |  |                                      |             |   |
|  |  | On remote op:                        |             |   |
|  |  |   → Apply directly (no transform)   |             |   |
|  |  +-------------------------------------+             |   |
|  |         | local edit                                  |   |
|  |         v                                             |   |
|  |  State 2: AWAITING_ACK                               |   |
|  |  +-------------------------------------+             |   |
|  |  | One op sent to server, waiting ACK. |             |   |
|  |  | pending_op = sent op                |             |   |
|  |  | buffer = empty                      |             |   |
|  |  |                                      |             |   |
|  |  | On local edit:                       |             |   |
|  |  |   → Apply locally                   |             |   |
|  |  |   → Add to buffer (compose ops)    |             |   |
|  |  |   → Transition to AWAITING_ACK_     |             |   |
|  |  |     WITH_BUFFER                      |             |   |
|  |  |                                      |             |   |
|  |  | On server ACK:                       |             |   |
|  |  |   → Transition to SYNCHRONIZED      |             |   |
|  |  |                                      |             |   |
|  |  | On remote op:                        |             |   |
|  |  |   → Transform remote op against     |             |   |
|  |  |     pending_op                       |             |   |
|  |  |   → Apply transformed remote op     |             |   |
|  |  |   → Update pending_op (transformed) |             |   |
|  |  +-------------------------------------+             |   |
|  |         | local edit (while waiting)                  |   |
|  |         v                                             |   |
|  |  State 3: AWAITING_ACK_WITH_BUFFER                   |   |
|  |  +-------------------------------------+             |   |
|  |  | One op in flight + buffered local ops|             |   |
|  |  | pending_op = sent op                |             |   |
|  |  | buffer = composed local ops          |             |   |
|  |  |                                      |             |   |
|  |  | On local edit:                       |             |   |
|  |  |   → Apply locally                   |             |   |
|  |  |   → Compose with buffer             |             |   |
|  |  |                                      |             |   |
|  |  | On server ACK:                       |             |   |
|  |  |   → Send buffer as next op          |             |   |
|  |  |   → pending_op = buffer             |             |   |
|  |  |   → buffer = empty                  |             |   |
|  |  |   → Transition to AWAITING_ACK      |             |   |
|  |  |                                      |             |   |
|  |  | On remote op:                        |             |   |
|  |  |   → Transform against pending_op    |             |   |
|  |  |   → Transform against buffer       |             |   |
|  |  |   → Apply transformed remote op     |             |   |
|  |  +-------------------------------------+             |   |
|  |                                                       |   |
|  +------------------------------------------------------+   |
+--------------------------------------------------------------+
```

### Server-Side OT Processing

```
Client A          Server (OT Engine)          Client B
   |                    |                        |
   |                    | State: "ABCD" rev=5    |
   |                    |                        |
   |-- Op: Insert("X",1)                        |
   |   base_rev=5 ----->|                        |
   |                    |                        |
   |                    |-- Op: Delete(3)         |
   |                    |   base_rev=5 <---------|
   |                    |                        |
   |                    | Server processes in     |
   |                    | arrival order:          |
   |                    |                        |
   |                    | 1. A's op arrives first |
   |                    |    base_rev=5 = HEAD ✅|
   |                    |    Apply: "ABCD"→"AXBCD"|
   |                    |    rev=6                |
   |                    |                        |
   |<-- ACK rev=6 -----|                        |
   |                    |-- broadcast Op -------->|
   |                    |   Insert("X",1)        |
   |                    |                        |
   |                    | 2. B's op arrives       |
   |                    |    base_rev=5, HEAD=6  |
   |                    |    Need to transform!  |
   |                    |                        |
   |                    |    Transform Delete(3)  |
   |                    |    against Insert("X",1)|
   |                    |    → Delete(4)          |
   |                    |    (position shifted +1)|
   |                    |                        |
   |                    |    Apply: "AXBCD"→"AXBD"|
   |                    |    rev=7                |
   |                    |                        |
   |                    |-- ACK rev=7 ----------->|
   |<-- broadcast ------|                        |
   |    Delete(4)       |                        |
   |                    |                        |
   | Final: "AXBD" ✅  | Final: "AXBD" ✅       |
```

---

## 7. Cursor & Selection Sync

```
+--------------------------------------------------------------+
|              Cursor Synchronization                           |
|                                                               |
|  Each client broadcasts cursor position + selection range   |
|  to all collaborators.                                       |
|                                                               |
|  Cursor message:                                             |
|  {                                                           |
|    "type": "cursor",                                        |
|    "user_id": "u123",                                       |
|    "user_name": "Alice",                                    |
|    "color": "#FF5733",                                      |
|    "cursor": {                                               |
|      "position": 42,                                        |
|      "selection_start": 42,                                 |
|      "selection_end": 58                                    |
|    }                                                         |
|  }                                                           |
|                                                               |
|  Challenges:                                                |
|  +------------------------------------------+               |
|  |                                           |               |
|  |  1. Cursor positions shift when others   |               |
|  |     insert/delete before the cursor      |               |
|  |                                           |               |
|  |     Solution: Transform cursor position  |               |
|  |     against incoming operations          |               |
|  |     (same transform as OT ops)           |               |
|  |                                           |               |
|  |  2. High frequency updates (every mouse  |               |
|  |     move or keystroke)                   |               |
|  |                                           |               |
|  |     Solution: Throttle to every 50ms     |               |
|  |     (20 updates/sec per user)            |               |
|  |     Use unreliable delivery (no need for |               |
|  |     guaranteed delivery — next update    |               |
|  |     will correct any missed one)         |               |
|  |                                           |               |
|  |  3. Visual rendering                     |               |
|  |     - Show colored cursor with user name |               |
|  |     - Show colored selection highlight   |               |
|  |     - Fade out after 5s of inactivity   |               |
|  |                                           |               |
|  +------------------------------------------+               |
|                                                               |
|  Data flow:                                                 |
|  +------------------------------------------+               |
|  |                                           |               |
|  |  Cursor updates are sent via WebSocket   |               |
|  |  BUT NOT persisted (ephemeral)           |               |
|  |                                           |               |
|  |  Client → WS Gateway → broadcast to all |               |
|  |  editors of same doc (bypass OT engine)  |               |
|  |                                           |               |
|  |  Redis stores current cursor positions   |               |
|  |  for new joiners (so they see existing   |               |
|  |  cursors immediately)                    |               |
|  |  TTL: 30s (auto-expire stale cursors)   |               |
|  |                                           |               |
|  +------------------------------------------+               |
+--------------------------------------------------------------+
```

---

## 8. Document Storage & Versioning

### Snapshot + Operation Log Model

```
+--------------------------------------------------------------+
|         Document Storage Strategy                             |
|                                                               |
|  We store documents as:                                     |
|  1. Periodic snapshots (full document state)               |
|  2. Operation log between snapshots                         |
|                                                               |
|  +----------------------------------------------+           |
|  |                                               |           |
|  |  Snapshot at rev 0     Op Log                |           |
|  |  "Hello"               rev 1: Insert("!",5)  |           |
|  |                        rev 2: Insert(" ",5)   |           |
|  |                        rev 3: Insert("W",6)   |           |
|  |                        ... 997 more ops ...   |           |
|  |                                               |           |
|  |  Snapshot at rev 1000  Op Log                |           |
|  |  "Hello World!"        rev 1001: Delete(4)   |           |
|  |                        rev 1002: Insert("a",4)|           |
|  |                        ...                    |           |
|  |                                               |           |
|  |  Snapshot at rev 2000  Op Log                |           |
|  |  "Hella World! More"   rev 2001: ...         |           |
|  |                                               |           |
|  +----------------------------------------------+           |
|                                                               |
|  Snapshotting rules:                                        |
|  - Every 1,000 operations                                   |
|  - Every 5 minutes of editing                               |
|  - On document close (all editors leave)                    |
|  - On explicit "save" / version checkpoint                  |
|                                                               |
|  To reconstruct any revision:                               |
|  1. Find nearest snapshot before target revision            |
|  2. Replay operations from snapshot to target               |
|  3. O(ops_since_snapshot) — bounded by snapshot frequency   |
|                                                               |
|  Version history (user-facing):                             |
|  - Store named checkpoints ("Version 3: Alice's edits")    |
|  - Auto-checkpoint every 30 min of editing                  |
|  - User can view diff between any two versions             |
|  - User can restore to any version (creates new checkpoint) |
+--------------------------------------------------------------+
```

### Data Model

```
+--------------------------------------------------------------+
|                    documents table                             |
|                                                               |
|  +------------------+--------------+---------------------+  |
|  | Column           | Type         | Notes               |  |
|  +------------------+--------------+---------------------+  |
|  | doc_id           | UUID         | Primary key         |  |
|  | title            | TEXT         |                     |  |
|  | owner_id         | UUID         |                     |  |
|  | current_revision | BIGINT       | HEAD revision       |  |
|  | latest_snapshot  | BIGINT       | Revision of last    |  |
|  |                  |              | snapshot            |  |
|  | content_snapshot | BLOB         | Latest snapshot     |  |
|  | created_at       | TIMESTAMP    |                     |  |
|  | updated_at       | TIMESTAMP    |                     |  |
|  +------------------+--------------+---------------------+  |
|                                                               |
|                    operations table                           |
|                                                               |
|  Partition Key: doc_id                                       |
|  Clustering Key: revision ASC                                |
|                                                               |
|  +------------------+--------------+---------------------+  |
|  | Column           | Type         | Notes               |  |
|  +------------------+--------------+---------------------+  |
|  | doc_id           | UUID         | Partition key       |  |
|  | revision         | BIGINT       | Clustering key      |  |
|  | user_id          | UUID         | Who made this edit  |  |
|  | operation        | JSONB        | Serialized op       |  |
|  | timestamp        | TIMESTAMP    |                     |  |
|  +------------------+--------------+---------------------+  |
|                                                               |
|                    snapshots table                            |
|                                                               |
|  +------------------+--------------+---------------------+  |
|  | Column           | Type         | Notes               |  |
|  +------------------+--------------+---------------------+  |
|  | doc_id           | UUID         | Partition key       |  |
|  | revision         | BIGINT       | Clustering key      |  |
|  | content          | BLOB         | Full doc at this rev|  |
|  | created_at       | TIMESTAMP    |                     |  |
|  +------------------+--------------+---------------------+  |
+--------------------------------------------------------------+
```

---

## 9. Presence System

```
+--------------------------------------------------------------+
|              Document Presence                                |
|                                                               |
|  Shows who is currently viewing/editing a document.          |
|                                                               |
|  Redis data structures:                                     |
|  +------------------------------------------+               |
|  |                                           |               |
|  |  doc_editors:{doc_id} → Hash             |               |
|  |  {                                        |               |
|  |    "u123": {                              |               |
|  |      "name": "Alice",                    |               |
|  |      "color": "#FF5733",                 |               |
|  |      "cursor_pos": 42,                   |               |
|  |      "last_active": 1712160005,          |               |
|  |      "ws_server": "ws-gw-7"             |               |
|  |    },                                     |               |
|  |    "u456": {                              |               |
|  |      "name": "Bob",                      |               |
|  |      "color": "#3498DB",                 |               |
|  |      "cursor_pos": 108,                  |               |
|  |      "last_active": 1712160003,          |               |
|  |      "ws_server": "ws-gw-12"            |               |
|  |    }                                      |               |
|  |  }                                        |               |
|  |                                           |               |
|  +------------------------------------------+               |
|                                                               |
|  Join flow:                                                 |
|  1. Client opens document → WebSocket connect               |
|  2. Server adds user to doc_editors:{doc_id}               |
|  3. Server broadcasts "Alice joined" to all editors        |
|  4. Server sends current editor list + cursors to Alice    |
|                                                               |
|  Leave flow:                                                |
|  1. Client closes tab → WebSocket disconnect               |
|  2. Server removes user from doc_editors:{doc_id}          |
|  3. Server broadcasts "Alice left" to remaining editors    |
|                                                               |
|  Heartbeat: Refresh last_active every 10s                  |
|  Cleanup: Remove entries where last_active > 30s ago       |
|                                                               |
|  Color assignment:                                          |
|  Pre-defined palette of 12 distinct colors.                |
|  Assign round-robin as users join.                          |
|  Color persists for session (not globally).                 |
+--------------------------------------------------------------+
```

---

## 10. Offline Editing

```
+--------------------------------------------------------------+
|              Offline Editing Support                           |
|                                                               |
|  With OT (harder):                                          |
|  +------------------------------------------+               |
|  |                                           |               |
|  |  1. Client goes offline                  |               |
|  |  2. Client buffers all local operations  |               |
|  |     in IndexedDB                         |               |
|  |  3. Client tracks its base revision      |               |
|  |                                           |               |
|  |  On reconnect:                            |               |
|  |  4. Client sends all buffered ops with   |               |
|  |     their base revision                  |               |
|  |  5. Server transforms each op against    |               |
|  |     ALL ops that happened while client   |               |
|  |     was offline                          |               |
|  |  6. This can be expensive if offline     |               |
|  |     for a long time (transform chain)    |               |
|  |                                           |               |
|  |  Optimization: If offline > 5 min:       |               |
|  |  → Fetch latest snapshot from server     |               |
|  |  → Three-way merge (local, server, base) |               |
|  |  → Present conflicts to user if needed   |               |
|  |                                           |               |
|  +------------------------------------------+               |
|                                                               |
|  With CRDTs (natural fit):                                  |
|  +------------------------------------------+               |
|  |                                           |               |
|  |  1. Client goes offline                  |               |
|  |  2. Client continues editing locally     |               |
|  |     (CRDT state is local)                |               |
|  |  3. On reconnect: sync CRDT states      |               |
|  |     → Automatic merge, guaranteed        |               |
|  |       convergence                        |               |
|  |  4. No transformation needed             |               |
|  |  5. No conflicts (by design)             |               |
|  |                                           |               |
|  |  ✅ This is why CRDTs excel for         |               |
|  |     offline-first apps                   |               |
|  |                                           |               |
|  +------------------------------------------+               |
+--------------------------------------------------------------+
```

---

## 11. Undo/Redo in Collaborative Context

```
+--------------------------------------------------------------+
|         Undo/Redo in Collaborative Editing                    |
|                                                               |
|  Challenge: User A types "Hello". User B types "World".     |
|  A hits Ctrl+Z. Should "Hello" disappear while "World"     |
|  remains? Yes — undo should only affect YOUR operations.    |
|                                                               |
|  Naive approach (broken):                                   |
|  +------------------------------------------+               |
|  |  Just revert the last operation globally |               |
|  |  ❌ A's undo would delete B's text!      |               |
|  +------------------------------------------+               |
|                                                               |
|  Correct approach: Per-user undo stack                      |
|  +------------------------------------------+               |
|  |                                           |               |
|  |  Each client maintains its own undo stack|               |
|  |  containing only ITS operations.         |               |
|  |                                           |               |
|  |  A's undo stack: [Insert("Hello",0)]     |               |
|  |  B's undo stack: [Insert("World",5)]     |               |
|  |                                           |               |
|  |  When A hits Ctrl+Z:                     |               |
|  |  1. Pop A's top op: Insert("Hello",0)    |               |
|  |  2. Create inverse: Delete(0..4)         |               |
|  |  3. Transform inverse against all ops    |               |
|  |     that happened AFTER the original     |               |
|  |  4. Apply transformed inverse            |               |
|  |  5. "Hello" removed, "World" stays       |               |
|  |                                           |               |
|  |  The transform step is crucial:          |               |
|  |  Original: Insert("Hello",0) at rev 5    |               |
|  |  Since then: Insert("World",5) at rev 6  |               |
|  |  Inverse of original: Delete(0,5)        |               |
|  |  Transform against rev 6: Delete(0,5)    |               |
|  |  (position unchanged since World is      |               |
|  |   after Hello)                            |               |
|  |  Result: "World" (Hello removed)         |               |
|  |                                           |               |
|  +------------------------------------------+               |
+--------------------------------------------------------------+
```

---

## 12. Rich Text Document Model

```
+--------------------------------------------------------------+
|         Rich Text Operation Model                             |
|                                                               |
|  Plain text ops (Insert/Delete) aren't enough for rich text.|
|  We need formatting operations too.                         |
|                                                               |
|  Extended operation types:                                   |
|  +------------------------------------------+               |
|  |                                           |               |
|  |  1. Insert(text, position, attributes)   |               |
|  |     e.g., Insert("hello", 5, {bold: true})|               |
|  |                                           |               |
|  |  2. Delete(position, length)             |               |
|  |     e.g., Delete(5, 3)                   |               |
|  |                                           |               |
|  |  3. Retain(length, attributes?)          |               |
|  |     Skip N chars, optionally changing    |               |
|  |     their formatting                     |               |
|  |     e.g., Retain(5, {bold: true})        |               |
|  |     → Makes chars 0-4 bold              |               |
|  |                                           |               |
|  +------------------------------------------+               |
|                                                               |
|  Delta format (used by Quill editor):                       |
|  +------------------------------------------+               |
|  |                                           |               |
|  |  Document "Hello World" with "Hello" bold|               |
|  |                                           |               |
|  |  [                                        |               |
|  |    { insert: "Hello", attrs: {bold:true} }|               |
|  |    { insert: " World" }                   |               |
|  |    { insert: "\n" }                       |               |
|  |  ]                                        |               |
|  |                                           |               |
|  |  Operation "make 'World' italic":        |               |
|  |  [                                        |               |
|  |    { retain: 6 },     // skip "Hello "   |               |
|  |    { retain: 5,                           |               |
|  |      attrs: {italic: true} }             |               |
|  |  ]                                        |               |
|  |                                           |               |
|  +------------------------------------------+               |
|                                                               |
|  Tree-based model (ProseMirror / Google Docs):              |
|  +------------------------------------------+               |
|  |                                           |               |
|  |  Document as a tree:                      |               |
|  |                                           |               |
|  |  doc                                      |               |
|  |  +-- paragraph                            |               |
|  |  |   +-- text("Hello ", bold)            |               |
|  |  |   +-- text("World", italic)           |               |
|  |  +-- heading(level=2)                    |               |
|  |  |   +-- text("Subtitle")               |               |
|  |  +-- bullet_list                         |               |
|  |      +-- list_item                       |               |
|  |      |   +-- paragraph                   |               |
|  |      |       +-- text("Item 1")          |               |
|  |      +-- list_item                       |               |
|  |          +-- paragraph                   |               |
|  |              +-- text("Item 2")          |               |
|  |                                           |               |
|  |  Operations reference positions in the   |               |
|  |  flat token stream of the tree.          |               |
|  |                                           |               |
|  +------------------------------------------+               |
+--------------------------------------------------------------+
```

---

## 13. Comments & Suggestions

```
+--------------------------------------------------------------+
|         Comments Anchored to Document Positions               |
|                                                               |
|  Challenge: A comment anchored to "the quick brown fox"     |
|  must stay attached even when text around it changes.        |
|                                                               |
|  Solution: Marker-based anchoring                           |
|  +------------------------------------------+               |
|  |                                           |               |
|  |  Insert invisible markers at comment     |               |
|  |  start and end positions:                |               |
|  |                                           |               |
|  |  "The [COMMENT_START:c1]quick brown       |               |
|  |   fox[COMMENT_END:c1] jumped over..."    |               |
|  |                                           |               |
|  |  Markers participate in OT like regular  |               |
|  |  characters:                              |               |
|  |  - Insert before marker → marker shifts  |               |
|  |  - Delete text within markers → comment  |               |
|  |    range shrinks                         |               |
|  |  - Delete all text → comment becomes     |               |
|  |    "orphaned" (shown as floating)        |               |
|  |                                           |               |
|  |  Comment data stored separately:          |               |
|  |  {                                        |               |
|  |    comment_id: "c1",                      |               |
|  |    doc_id: "doc_abc",                    |               |
|  |    author: "Alice",                      |               |
|  |    text: "Should this be 'red' instead?",|               |
|  |    resolved: false,                      |               |
|  |    replies: [ ... ],                      |               |
|  |    created_at: "2026-04-03T..."           |               |
|  |  }                                        |               |
|  |                                           |               |
|  +------------------------------------------+               |
|                                                               |
|  Suggestions (track changes):                               |
|  +------------------------------------------+               |
|  |                                           |               |
|  |  A suggestion is a "proposed operation"  |               |
|  |  that isn't applied to the document yet. |               |
|  |                                           |               |
|  |  Rendered as strikethrough (deletion)    |               |
|  |  or colored underline (insertion).       |               |
|  |                                           |               |
|  |  Accept → apply the operation            |               |
|  |  Reject → discard the operation          |               |
|  |                                           |               |
|  |  Storage: separate suggestions table     |               |
|  |  with the proposed operation + metadata  |               |
|  |                                           |               |
|  +------------------------------------------+               |
+--------------------------------------------------------------+
```

---

## 14. Document Session Management

```
+--------------------------------------------------------------+
|         Document Session Lifecycle                            |
|                                                               |
|  "Session" = period when a document has active editors.     |
|  A document with no editors for 5 min = session closed.     |
|                                                               |
|  Session open:                                               |
|  +------------------------------------------+               |
|  |                                           |               |
|  |  1. First editor opens document          |               |
|  |  2. Load latest snapshot from DB         |               |
|  |  3. Replay operations since snapshot     |               |
|  |  4. Hold document state in memory        |               |
|  |     (on the Collaboration Service node)  |               |
|  |  5. Assign session to a specific node    |               |
|  |     (sticky routing by doc_id)           |               |
|  |                                           |               |
|  +------------------------------------------+               |
|                                                               |
|  Session active:                                             |
|  +------------------------------------------+               |
|  |                                           |               |
|  |  • All OT for this doc handled by same   |               |
|  |    server node (single-threaded per doc) |               |
|  |  • Operations applied and broadcast in   |               |
|  |    real-time                              |               |
|  |  • Periodic snapshots persisted to DB    |               |
|  |  • All operations appended to op log     |               |
|  |                                           |               |
|  +------------------------------------------+               |
|                                                               |
|  Session close:                                              |
|  +------------------------------------------+               |
|  |                                           |               |
|  |  1. All editors leave (last WS closes)   |               |
|  |  2. Wait 5 min (grace period for rejoins)|               |
|  |  3. Take final snapshot, persist to DB   |               |
|  |  4. Flush op log                         |               |
|  |  5. Release in-memory state              |               |
|  |                                           |               |
|  +------------------------------------------+               |
|                                                               |
|  Sticky routing:                                             |
|  +------------------------------------------+               |
|  |                                           |               |
|  |  All operations for doc_id "abc" must go |               |
|  |  to the SAME Collaboration Service node. |               |
|  |                                           |               |
|  |  Routing: consistent_hash(doc_id) →      |               |
|  |  collab_service_node_7                   |               |
|  |                                           |               |
|  |  If node_7 crashes:                      |               |
|  |  - Session migrates to node_8            |               |
|  |  - Rebuild state from snapshot + op log  |               |
|  |  - Clients reconnect, resync from their  |               |
|  |    last known revision                   |               |
|  |                                           |               |
|  +------------------------------------------+               |
+--------------------------------------------------------------+
```

---

## 15. Production Architecture

```
                    +-----------------------------------------+
                    |         Client (Browser/Mobile)          |
                    |                                           |
                    |  Editor Engine (ProseMirror)             |
                    |  + OT Client + Local Buffer              |
                    |  + IndexedDB (offline cache)             |
                    +------------------+----------------------+
                                       | WebSocket (TLS)
                    +------------------v----------------------+
                    |      Load Balancer (L4)                   |
                    |   Sticky by doc_id (for session routing) |
                    +------------------+----------------------+
                                       |
               +-----------------------+------------------------+
               v                       v                        v
      +-----------------+   +-----------------+   +-----------------+
      |  WS Gateway      |   |  WS Gateway      |   |  WS Gateway      |
      |  (500 servers)    |   |                   |   |                   |
      |  20K conn each    |   |  • Auth           |   |                   |
      |                   |   |  • Route to doc   |   |                   |
      |  Routes ops to    |   |    session        |   |                   |
      |  correct Collab   |   |  • Fan-out to     |   |                   |
      |  Service node     |   |    co-editors     |   |                   |
      +--------+----------+   +--------+----------+   +--------+----------+
               |                       |                        |
               +-----------------------+------------------------+
                                       |
                    +------------------v----------------------+
                    |   Collaboration Service (OT Engine)      |
                    |   (200 nodes, partitioned by doc_id)    |
                    |                                          |
                    |   Each node handles ~500-1000 active    |
                    |   document sessions in memory            |
                    |                                          |
                    |   +------------------------------------+ |
                    |   | Doc Session: doc_abc               | |
                    |   |  state: "Hello World..."           | |
                    |   |  revision: 4521                    | |
                    |   |  editors: [Alice, Bob]             | |
                    |   |  recent_ops: [...]                 | |
                    |   +------------------------------------+ |
                    |   +------------------------------------+ |
                    |   | Doc Session: doc_xyz               | |
                    |   |  ...                               | |
                    |   +------------------------------------+ |
                    +------------------+----------------------+
                                       |
              +------------------------+------------------------+
              v                        v                        v
     +--------------+       +--------------+        +--------------+
     | Operation Log |       | Document DB   |        |    Redis      |
     | (Cassandra)   |       | (Cloud Spanner|        |   Cluster     |
     |               |       |  or Postgres) |        |               |
     | Append-only   |       |               |        | • Presence    |
     | Partition by  |       | • Snapshots   |        | • Cursors     |
     | doc_id        |       | • Metadata    |        | • Doc→node    |
     |               |       | • Permissions |        |   routing map |
     | Compacted     |       | • Comments    |        | • Session     |
     | after snapshot|       |               |        |   locks       |
     +--------------+       +--------------+        +--------------+
                                    |
                             +------v-------+
                             | Blob Storage  |
                             | (S3/GCS)      |
                             |               |
                             | Images,       |
                             | Attachments,  |
                             | Export files   |
                             +--------------+

     +--------------------------------------------------+
     |             Supporting Services                    |
     |                                                    |
     |  +--------------+  +--------------+              |
     |  | Version       |  | Permission   |              |
     |  | History Svc   |  | Service      |              |
     |  |               |  |              |              |
     |  | Diff, restore |  | View, edit,  |              |
     |  | version list  |  | comment,     |              |
     |  |               |  | share        |              |
     |  +--------------+  +--------------+              |
     |                                                    |
     |  +--------------+  +--------------+              |
     |  | Export        |  | Monitoring   |              |
     |  | Service       |  |              |              |
     |  |               |  | • Op latency |              |
     |  | PDF, DOCX,    |  | • Session    |              |
     |  | Markdown      |  |   count      |              |
     |  |               |  | • Conflict   |              |
     |  |               |  |   rate       |              |
     |  +--------------+  +--------------+              |
     +--------------------------------------------------+
```

---

## 16. Tradeoffs & Design Decisions Summary

| Decision | Option A | Option B | Chosen | Why |
|----------|----------|----------|--------|-----|
| **Sync algorithm** | OT (Operational Transformation) | CRDT (Conflict-free Replicated Data Types) | OT (server-centric) | Lower memory overhead, battle-tested (Google Docs), simpler architecture with central server |
| **Architecture** | Peer-to-peer | Central server | Central server | Simpler OT (single serialization point), easier persistence, permissions enforcement |
| **Document model** | Flat text (positions) | Tree-based (ProseMirror) | Tree-based | Rich text requires structure (paragraphs, lists, tables) |
| **Storage** | Full document per edit | Snapshot + operation log | Snapshot + op log | Space-efficient, enables version history, supports audit trail |
| **Session routing** | Random (any node) | Sticky by doc_id | Sticky | Single-threaded OT per doc — all ops for a doc must go to same node |
| **Cursor sync** | Reliable (guaranteed delivery) | Unreliable (fire-and-forget) | Unreliable | Cursors are ephemeral; next update corrects any missed one |
| **Snapshot frequency** | Every operation | Every 1,000 ops | Every 1,000 ops | Balance between recovery speed and storage cost |
| **Offline** | Disallow | OT transform chain | OT + fallback to 3-way merge | OT works for short offline; 3-way merge for extended offline |
| **Undo model** | Global undo stack | Per-user undo stack | Per-user | User's undo should only revert their own operations |
| **Comments** | Position index | Marker-based anchoring | Markers | Markers move with text as document changes |
| **Op log storage** | MySQL | Cassandra | Cassandra | Append-heavy, partition by doc_id, linear scaling |

---

## 17. Interview Tips

1. **Start by stating the core challenge** — "The fundamental problem is: two users edit the same word at the same time. How do we merge without losing either edit?" This frames everything.

2. **Draw the OT diamond** — Show state S, two concurrent ops, and how transformation produces the same result S' regardless of order. This visual sells your understanding.

3. **Compare OT vs CRDT** — Even if you choose one, show you know the other. "OT requires a central server but uses less memory. CRDTs work peer-to-peer but carry tombstone overhead."

4. **Client state machine is a great deep dive** — Show SYNCHRONIZED → AWAITING_ACK → AWAITING_ACK_WITH_BUFFER. This proves you understand the practical implementation, not just theory.

5. **Don't forget cursors** — Interviewers love the cursor sync detail. "Cursor positions must be transformed against incoming operations. We throttle cursor updates to 20/sec and use unreliable delivery."

6. **Snapshot + op log for storage** — "We don't store the full document on every keystroke. We take snapshots every 1,000 ops and replay the op log to reconstruct any revision."

7. **Session stickiness is essential** — "All operations for a document must route to the same server node. This gives us single-threaded OT — no distributed consensus needed for operation ordering."

8. **Mention undo correctness** — "Per-user undo stack. When you hit Ctrl+Z, it reverts YOUR last operation, not someone else's. The inverse is transformed against all subsequent ops."

9. **Version history comes for free** — "Because we store the operation log, version history is just 'replay ops from snapshot to revision N.' Diff is computed from the op sequence."

10. **Name real implementations** — "Google Docs uses server-centric OT. Figma uses CRDTs. Yjs is the most popular open-source CRDT library." This shows industry awareness.
