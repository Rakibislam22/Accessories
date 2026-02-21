# Read Receipts & Delivery Status - Event & REST Contracts

## Overview
This document defines the exact event payloads and REST API contracts for implementing read receipts and message delivery status tracking. It serves as the interface specification between:
- **Member A**: Database schema & contracts (you)
- **Member B**: Socket implementation
- **Member C**: Frontend state + UI

---

## Message Schema
```javascript
{
  _id: ObjectId,
  conversationId: ObjectId,        // Reference to conversation
  sender: ObjectId,                 // User who sent the message
  receiverId: ObjectId,             // User who receives the message
  text: String,
  status: String,                   // "sent" | "delivered" | "read"
  deliveredAt: Date | null,         // When message was delivered to receiver
  seenAt: Date | null,              // When message was read by receiver
  createdAt: Date,
  updatedAt: Date
}
```

### Indexes
```javascript
// For paginated history retrieval
{ conversationId: 1, createdAt: -1 }

// For bulk updates (mark all messages up to message X as read)
{ conversationId: 1, _id: 1 }

// For querying undelivered/unseen messages
{ receiverId: 1, status: 1, createdAt: -1 }
```

---

## Status Values
- `"sent"`: Message saved in DB, sent to receiver (but receiver may be offline)
- `"delivered"`: Message received by receiver's client (receiver came online and received it)
- `"read"`: Message seen by receiver (opened conversation and message is visible)

---

## Socket Events (Bi-directional)

### 1. message:send (Client → Server)
**When**: User sends a message
**Emitted by**: Frontend component
**Expected payload**:
```json
{
  "conversationId": "507f1f77bcf86cd799439011",
  "receiverId": "507f1f77bcf86cd799439012",
  "text": "Hello world",
  "tempId": "temp-msg-1"
}
```

**Response** (message:delivered):
Sent back to sender with the full saved message object
```json
{
  "_id": "507f1f77bcf86cd799439013",
  "tempId": "temp-msg-1",
  "conversationId": "507f1f77bcf86cd799439011",
  "sender": {
    "_id": "507f1f77bcf86cd799439010",
    "name": "Alice",
    "avatar": "https://..."
  },
  "text": "Hello world",
  "status": "sent",
  "createdAt": "2025-02-21T10:30:00Z"
}
```

---

### 2. message:new (Server → Client - Receiver)
**When**: Receiver is online and message is ready
**Emitted to**: Receiver (io.to(receiverSocketId))
**Payload**:
```json
{
  "_id": "507f1f77bcf86cd799439013",
  "conversationId": "507f1f77bcf86cd799439011",
  "sender": {
    "_id": "507f1f77bcf86cd799439010",
    "name": "Alice",
    "avatar": "https://..."
  },
  "text": "Hello world",
  "status": "sent",
  "createdAt": "2025-02-21T10:30:00Z"
}
```

---

### 3. message:status (Server → Both Users)
**When**: Message status changes (sent → delivered → read)

#### Option A: Single Message ID (RECOMMENDED)
Most efficient for individual status updates
```json
{
  "messageId": "507f1f77bcf86cd799439013",
  "status": "delivered",
  "deliveredAt": "2025-02-21T10:31:00Z"
}
```

#### Option B: Bulk Up-To ID (For Read Events)
When marking multiple messages as read in bulk
```json
{
  "conversationId": "507f1f77bcf86cd799439011",
  "status": "read",
  "upToMessageId": "507f1f77bcf86cd799439020",
  "seenAt": "2025-02-21T10:32:00Z",
  "modifiedCount": 5
}
```

**Frontend Action**:
- Single ID: Update that specific message in the store
- Up-to ID: Update all messages up to and including that ID with status "read"

---

### 4. conversation:seen (Client → Server)
**When**: User opens/focuses a conversation and has visible messages
**Emitted by**: Frontend component
**Payload**:
```json
{
  "conversationId": "507f1f77bcf86cd799439011",
  "lastSeenMessageId": "507f1f77bcf86cd799439020"
}
```

**Server Action**:
Your REST endpoint handles this, OR Member B implements socket handler.
- Query: Find all messages in this conversation received by the user with `createdAt <= lastSeenMessage.createdAt` and `status != "read"`
- Update: Bulk update to `status: "read"` and set `seenAt: now()`
- Emit: `message:status` event back to both users (sender sees their message was read)

---

## REST API Fallback

### Mark Conversation as Seen
**Endpoint**: `POST /api/chat/:conversationId/seen`  
**Auth**: Required (Bearer token)  
**Purpose**: TCP/network fallback when socket is unavailable

**Request Body**:
```json
{
  "lastSeenMessageId": "507f1f77bcf86cd799439020"
}
```

**Response (200 OK)**:
```json
{
  "message": "Messages marked as seen",
  "modifiedCount": 5,
  "upToMessageId": "507f1f77bcf86cd799439020"
}
```

**Possible Errors**:
- `400`: `lastSeenMessageId is required`
- `403`: User is not a participant in this conversation
- `404`: Message not found in this conversation
- `500`: Server error

---

## Implementation Sequence for Member B (Socket Events)

1. **On message:send**:
   - Save message with `status: "sent"` and `receiverId`
   - Emit `message:delivered` back to sender (with real `_id` + `createdAt`)
   - Check if receiver is online (Redis lookup)
   - If online: Emit `message:new` to receiver AND update message to `status: "delivered"` with `deliveredAt: now()` → Emit `message:status(messageId, "delivered")`
   - If offline: Message stays `"sent"`, will transition to `"delivered"` when receiver comes online

2. **On receiver connection**:
   - Query messages with `receiverId: userId` and `status: "sent"`
   - For each: update to `"delivered"` and emit `message:status` events to the conversation
   - Emit all `message:new` events for messages they missed while offline

3. **On conversation:seen**:
   - Bulk update messages with `status: "read"`, set `seenAt: now()`
   - Emit `message:status` with `upToMessageId` to both sender and receiver
   - Emit to sender so they see their messages were read

4. **Handle multi-tab support**:
   - Maintain `userId → Set<socketId>` mapping in Redis
   - When updating message status, emit to ALL socket IDs for that user
   - When user connects, load all "sent" messages and update to "delivered"

---

## Implementation Notes for Member C (Frontend)

1. **Listen for message:new**:
   - Add message to chat window
   - Auto-emit `conversation:seen` with the newly received message's ID
   - Frontend should compute `lastSeenMessageId` = "the latest message I can see in the DOM"

2. **Listen for message:status**:
   - Single ID: Update that specific message's status and timestamps
   - Up-to ID: Update all messages up to that ID

3. **Emit conversation:seen**:
   - On mount/focus: Scroll to latest message, compute `lastSeenMessageId`, emit event
   - On receiving new message: Emit conversation:seen immediately
   - Optional: Debounce if many messages arrive at once

4. **Render ticks**:
   - None: `status: "sent"` (only sender can see this on outgoing)
   - Single tick: `status: "delivered"` + `deliveredAt`
   - Double tick: `status: "read"` + `seenAt`

5. **Fallback**:
   - If socket fails, call `POST /api/chat/:conversationId/seen` via HTTP when conversation is opened

---

## Field Nullability

- `deliveredAt`: Null until message transitions to "delivered"
- `seenAt`: Null until message transitions to "read"
- `receiverId`: Always populated (required field)
- `status` is never null (defaults to "sent")

---

## Edge Cases Handled

1. **Message sent to offline user**:
   - Saved as `status: "sent"`
   - When receiver comes online, Member B updates to "delivered" + `deliveredAt`
   - Member A REST fallback can be used if socket times out

2. **User with multiple tabs**:
   - One tab emits `conversation:seen` with `lastSeenMessageId: X`
   - Member B should use `userId → Set<socketId>` to reach all tabs of that user
   - All tabs receive `message:status` events

3. **Offline receiver**:
   - Socket event emission queued in Redis or just attempted on reconnection
   - REST fallback available for critical updates

4. **Network interruption**:
   - If `conversation:seen` socket timesout (>5s), frontend calls REST endpoint
   - REST endpoint performs bulk update and returns count of modified messages

---

## Summary Table

| Event | Direction | Purpose | Payload Size |
|-------|-----------|---------|--------------|
| message:send | Client → Server | Send message | ~300 bytes |
| message:delivered | Server → Client | Ack with real ID | ~400 bytes |
| message:new | Server → Client | New message received | ~400 bytes |
| message:status | Server → Both | Status update | ~200 bytes (single) or ~300 bytes (bulk) |
| conversation:seen | Client → Server | Mark as read | ~150 bytes |
| POST /api/chat/:cid/seen | Client → Server | REST fallback | ~150 bytes |

