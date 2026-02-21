# Member A - Completed Tasks: DB Schema & Event Contracts

## ✅ Completed Items

### 1. Updated Message Schema
**File**: `src/models/Message.js`

**Added Fields**:
- ✅ `receiverId` (ObjectId, required) - Who receives this message
- ✅ `deliveredAt` (Date, nullable) - When message was delivered to receiver
- ✅ `seenAt` (Date, nullable) - When message was read by receiver
- ✅ `status` enum already existed: `"sent"` → `"delivered"` → `"read"`

**Schema**:
```javascript
{
  conversationId: ObjectId (required),
  sender: ObjectId (required),
  receiverId: ObjectId (required),  // NEW
  text: String (required),
  status: String (enum: ["sent", "delivered", "read"], default: "sent"),
  deliveredAt: Date (default: null),        // NEW
  seenAt: Date (default: null),             // NEW
  createdAt: Date (auto),
  updatedAt: Date (auto)
}
```

---

### 2. Added Database Indexes
**File**: `src/models/Message.js`

Three indexes created for optimal query performance:

```javascript
// 1. For paginated message history retrieval
messageSchema.index({ conversationId: 1, createdAt: -1 });

// 2. For bulk updates (mark messages up to X as read)
messageSchema.index({ conversationId: 1, _id: 1 });

// 3. For querying undelivered/unseen messages
messageSchema.index({ receiverId: 1, status: 1, createdAt: -1 });
```

**Why these indexes**:
- `(conversationId, createdAt)`: Fast pagination when fetching chat history
- `(conversationId, _id)`: Enables efficient bulk updates like "mark all messages ≤ messageId as read"
- `(receiverId, status, createdAt)`: Quickly find all undelivered/unseen messages for a user (used on reconnection)

---

### 3. Defined Event Payload Contracts
**File**: `CONTRACTS.md` (New)

Comprehensive specification of all socket events and REST endpoints:

#### Socket Events (Bi-directional):
- ✅ `message:send` (Client → Server) - Send a message
- ✅ `message:delivered` (Server → Client) - Ack with real message object
- ✅ `message:new` (Server → Client) - New message received notification
- ✅ `message:status` (Server → Both) - Status update with two variants:
  - **Single ID**: For individual message status changes
  - **Bulk Up-To ID**: For reading multiple messages at once
- ✅ `conversation:seen` (Client → Server) - Mark conversation as read

#### REST Fallback:
- ✅ `POST /api/chat/:conversationId/seen` - HTTP fallback when socket unavailable

All payloads include exact JSON structure with example data.

---

### 4. Implemented REST Fallback Endpoint
**Files Modified**:
- `src/routes/chat.routes.js` - Added route definition
- `src/controllers/chat.controller.js` - Added handler logic

**Endpoint**: `POST /api/chat/:conversationId/seen`

**Request**:
```json
{
  "lastSeenMessageId": "507f1f77bcf86cd799439020"
}
```

**Server Action**:
1. Verify user is participant in conversation
2. Find the target message to validate it exists in this conversation
3. Bulk update all messages: `{ conversationId, receiverId: userId, status: { $ne: "read" }, createdAt: { $lte: targetMessage.createdAt } }`
4. Set `status: "read"` and `seenAt: now()`
5. Return count of modified messages

**Response**:
```json
{
  "message": "Messages marked as seen",
  "modifiedCount": 5,
  "upToMessageId": "507f1f77bcf86cd799439020"
}
```

**Error Handling**:
- 400: `lastSeenMessageId is required`
- 403: User not a participant
- 404: Message not found in conversation
- 500: Server error

---

## 📋 What Member B (Socket Implementation) Needs to Do

Using the `CONTRACTS.md` specification:

1. **message:send handler**:
   - Save message with `status: "sent"` and `receiverId`
   - If receiver online: update to `"delivered"`, emit `message:status` event
   - If offline: leave as `"sent"` (will be auto-updated on reconnection)

2. **On receiver connection**:
   - Load all `status: "sent"` messages for this user
   - Bulk update to `"delivered"` with `deliveredAt`
   - Emit `message:new` for each missed message
   - Emit `message:status` for delivered notifications

3. **conversation:seen handler**:
   - Bulk update messages to `"read"` status
   - Set `seenAt: now()`
   - Emit `message:status` back to both users

4. **Multi-tab support**:
   - Maintain `userId → Set<socketId>` in Redis
   - When message status changes, emit to all sockets for that user

---

## 📋 What Member C (Frontend) Needs to Do

Using the `CONTRACTS.md` specification:

1. **Listen for events**:
   - `message:new` → Add to chat, auto-emit `conversation:seen`
   - `message:status` → Update message status in store

2. **Emit events**:
   - `message:send` → Send new message
   - `conversation:seen` → On mount/focus (with `lastSeenMessageId`)

3. **Compute lastSeenMessageId**:
   - Get the latest received message currently visible in DOM
   - Extract its `_id` and send in `conversation:seen` event

4. **Render status ticks**:
   - No tick: `status: "sent"`
   - 1 tick: `status: "delivered"` + `deliveredAt`
   - 2 ticks: `status: "read"` + `seenAt`

5. **Fallback**:
   - If socket unavailable, call `POST /api/chat/:conversationId/seen` via REST

---

## 📂 Updated Files

```
chat-app-server/
├── src/
│   ├── models/
│   │   └── Message.js               ✅ Added fields & indexes
│   ├── routes/
│   │   └── chat.routes.js           ✅ Added /seen route
│   └── controllers/
│       └── chat.controller.js       ✅ Added markConversationSeen handler
│
└── CONTRACTS.md                      ✅ NEW - Complete event specifications
```

---

## 🚀 Next Steps

1. **Member B**: Implement socket handlers using `CONTRACTS.md`
2. **Member C**: Wire frontend UI using exact event payloads from `CONTRACTS.md`
3. **Team**: Test end-to-end flow:
   - Send message (online receiver) → Verify "delivered" status appears
   - Send message (offline) → Come online → Verify "delivered" status updates
   - Open conversation → Verify "read" status applied to visible messages
   - Multi-device → Verify status syncs across tabs

---

## 📝 Notes

- All three members reference `CONTRACTS.md` for exact event structure
- Database is ready; socket/frontend can proceed independently once they follow the spec
- REST fallback provides safety for unreliable networks
- Indexes are optimized for the expected query patterns (history fetch, bulk updates, status queries)

