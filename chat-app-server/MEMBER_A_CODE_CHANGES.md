# Member A - Code Changes Reference

## File 1: src/models/Message.js

### BEFORE:
```javascript
const messageSchema = new mongoose.Schema(
  {
    conversationId: {
      type: mongoose.Schema.Types.ObjectId,
      ref: "Conversation",
      required: true,
    },
    sender: {
      type: mongoose.Schema.Types.ObjectId,
      ref: "User",
      required: true,
    },
    text: {
      type: String,
      required: true,
      trim: true,
    },
    status: {
      type: String,
      enum: ["sent", "delivered", "read"],
      default: "sent",
    },
  },
  { timestamps: true },
);

// Fast paginated history queries: fetch messages for a conversation sorted by time
messageSchema.index({ conversationId: 1, createdAt: -1 });

module.exports = mongoose.model("Message", messageSchema);
```

### AFTER:
```javascript
const messageSchema = new mongoose.Schema(
  {
    conversationId: {
      type: mongoose.Schema.Types.ObjectId,
      ref: "Conversation",
      required: true,
    },
    sender: {
      type: mongoose.Schema.Types.ObjectId,
      ref: "User",
      required: true,
    },
    receiverId: {
      type: mongoose.Schema.Types.ObjectId,
      ref: "User",
      required: true,
    },
    text: {
      type: String,
      required: true,
      trim: true,
    },
    status: {
      type: String,
      enum: ["sent", "delivered", "read"],
      default: "sent",
    },
    deliveredAt: {
      type: Date,
      default: null,
    },
    seenAt: {
      type: Date,
      default: null,
    },
  },
  { timestamps: true },
);

// Fast paginated history queries: fetch messages for a conversation sorted by time
messageSchema.index({ conversationId: 1, createdAt: -1 });

// Index for bulk updates: find all messages for a conversation up to a specific message
messageSchema.index({ conversationId: 1, _id: 1 });

// Index for delivered/seen status queries: find undelivered or unseen messages
messageSchema.index({ receiverId: 1, status: 1, createdAt: -1 });

module.exports = mongoose.model("Message", messageSchema);
```

**Changes**:
- ✅ Added `receiverId` field (ObjectId, required)
- ✅ Added `deliveredAt` field (Date, nullable)
- ✅ Added `seenAt` field (Date, nullable)
- ✅ Added two new indexes for bulk updates and status queries

---

## File 2: src/routes/chat.routes.js

### BEFORE:
```javascript
const express = require("express");
const router = express.Router();
const auth = require("../middleware/auth.middleware");
const {
  getConversations,
  getMessages,
  createConversation,
  searchUsers,
  getLastSeen,
  getLastSeenBatch,
} = require("../controllers/chat.controller");

// All routes require authentication
router.use(auth);

// ... existing routes ...

router.post("/last-seen", getLastSeenBatch);

module.exports = router;
```

### AFTER:
```javascript
const express = require("express");
const router = express.Router();
const auth = require("../middleware/auth.middleware");
const {
  getConversations,
  getMessages,
  createConversation,
  searchUsers,
  getLastSeen,
  getLastSeenBatch,
  markConversationSeen,
} = require("../controllers/chat.controller");

// All routes require authentication
router.use(auth);

// ... existing routes ...

router.post("/last-seen", getLastSeenBatch);

// @route   POST /api/chat/:conversationId/seen
// @desc    Mark messages in a conversation as seen (REST fallback for socket)
// @body    { lastSeenMessageId: ObjectId }
router.post("/:conversationId/seen", markConversationSeen);

module.exports = router;
```

**Changes**:
- ✅ Imported `markConversationSeen` controller
- ✅ Added new route: `POST /:conversationId/seen`

---

## File 3: src/controllers/chat.controller.js

### ADDED (at end of file):
```javascript
// @desc    Mark messages in a conversation as seen (REST fallback)
// @route   POST /api/chat/:conversationId/seen
// @body    { lastSeenMessageId: ObjectId }
exports.markConversationSeen = async (req, res) => {
  try {
    const userId = req.user.id;
    const { conversationId } = req.params;
    const { lastSeenMessageId } = req.body;

    if (!lastSeenMessageId) {
      return res.status(400).json({ message: "lastSeenMessageId is required" });
    }

    // Verify the requesting user is a participant
    const conversation = await Conversation.findOne({
      _id: conversationId,
      participants: userId,
    });

    if (!conversation) {
      return res
        .status(403)
        .json({ message: "Access denied to this conversation" });
    }

    // Find the last seen message to ensure it exists in this conversation
    const lastSeenMessage = await Message.findOne({
      _id: lastSeenMessageId,
      conversationId,
    });

    if (!lastSeenMessage) {
      return res.status(404).json({ message: "Message not found in this conversation" });
    }

    // Bulk update: mark all messages in conversation up to and including lastSeenMessageId as seen
    // Only update messages received by the requesting user that aren't already "read"
    const result = await Message.updateMany(
      {
        conversationId,
        receiverId: userId,
        status: { $ne: "read" },
        createdAt: { $lte: lastSeenMessage.createdAt },
      },
      {
        $set: {
          status: "read",
          seenAt: new Date(),
        },
      }
    );

    res.json({
      message: "Messages marked as seen",
      modifiedCount: result.modifiedCount,
      upToMessageId: lastSeenMessageId,
    });
  } catch (err) {
    console.error("markConversationSeen error:", err.message);
    res.status(500).json({ message: "Server error" });
  }
};
```

**Changes**:
- ✅ New controller function implementing REST fallback
- ✅ Validates user is participant in conversation
- ✅ Validates message exists in conversation
- ✅ Bulk updates all messages up to the specified message
- ✅ Returns proper error codes and response format

---

## Summary

| Component | Change |
|-----------|--------|
| **Schema** | Added 3 new fields (`receiverId`, `deliveredAt`, `seenAt`) |
| **Indexes** | Added 2 new compound indexes for performance |
| **Routes** | Added 1 new route: `POST /:conversationId/seen` |
| **Controller** | Added 1 new handler: `markConversationSeen` |
| **Contracts** | Created `CONTRACTS.md` with full specification |

All changes follow the specification in `CONTRACTS.md` and are ready for Member B and Member C to implement socket handlers and frontend logic.

