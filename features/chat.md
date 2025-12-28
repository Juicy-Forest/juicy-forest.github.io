# Chat Feature

This document provides a comprehensive overview of the Chat feature in Juicy Forest, covering both the backend and frontend implementations.

---

## Table of Contents

1. [Architecture Overview](#architecture-overview)
2. [Backend](#backend)
   - [Microservice Structure](#microservice-structure)
   - [Database Models](#database-models)
   - [WebSocket Protocol](#websocket-protocol)
   - [REST API Endpoints](#rest-api-endpoints)
   - [Services](#services)
3. [Frontend](#frontend)
   - [Component Structure](#component-structure)
   - [ChatService](#chatservice)
   - [Real-time Communication](#real-time-communication)
4. [Message Flow Diagrams](#message-flow-diagrams)
5. [WebSocket Message Types](#websocket-message-types)

---

## Architecture Overview

The Chat feature follows a **microservices architecture** with real-time WebSocket communication:

```
┌─────────────────┐     ┌─────────────────┐     ┌─────────────────┐
│   SvelteKit     │────▶│   API Gateway   │────▶│  Chat Service   │
│   Frontend      │     │   (Port 3030)   │     │  (Port 3033)    │
│   (Port 5173)   │     └─────────────────┘     └────────┬────────┘
│                 │                                      │
│                 │◀────── WebSocket (ws://localhost:3033) ──────▶│
└─────────────────┘                                      │
                                                         ▼
                                               ┌─────────────────┐
                                               │    MongoDB      │
                                               │  (juicy-forest) │
                                               └─────────────────┘
```

- **Frontend**: SvelteKit application with reactive state management using Svelte 5 runes
- **API Gateway**: Express.js proxy routing REST requests to appropriate microservices
- **Chat Service**: Dedicated microservice handling both REST API and WebSocket connections
- **Database**: MongoDB for persistent storage of channels and messages

---

## Backend

### Microservice Structure

The chat microservice (`backend/chat/`) is organized as follows:

```
backend/chat/
├── index.js                 # Entry point, Express + WebSocket server
├── routes.js                # REST route definitions
├── controllers/
│   ├── channelController.js # REST endpoints for channels
│   └── socketController.js  # WebSocket connection handler
├── models/
│   ├── Channel.js           # Mongoose channel schema
│   └── Message.js           # Mongoose message schema
├── services/
│   ├── channelService.js    # Channel business logic
│   └── messageService.js    # Message business logic
└── utils/
    └── initDatabase.js      # MongoDB connection setup
```

#### Entry Point (`index.js`)

The chat service runs on port `3033` and initializes:
- Express.js server for REST endpoints
- WebSocket server (using `ws` library) for real-time communication
- MongoDB connection via Mongoose

```javascript
const wss = new WebSocketServer({server});
wss.on('connection', async (ws, req) => await handleConnection(wss, ws, req));
```

---

### Database Models

#### Channel Model

Represents a chat channel within a garden.

| Field     | Type       | Description                          |
|-----------|------------|--------------------------------------|
| `name`    | String     | Channel name (required, trimmed)     |
| `gardenId`| ObjectId   | Reference to the parent garden       |
| `createdAt`| Date      | Auto-generated timestamp             |
| `updatedAt`| Date      | Auto-generated timestamp             |

**Indexes:**
- `{ gardenId: 1 }` - for querying channels by garden
- `{ gardenId: 1, name: 1 }` - unique compound index

#### Message Model

Represents a chat message within a channel.

| Field              | Type       | Description                              |
|--------------------|------------|------------------------------------------|
| `author._id`       | ObjectId   | Author's user ID                         |
| `author.username`  | String     | Author's username at time of message     |
| `author.avatarColor`| String    | Author's avatar color (hex or CSS color) |
| `content`          | String     | Message content (max 3000 chars)         |
| `channel`          | ObjectId   | Reference to Channel                     |
| `createdAt`        | Date       | Auto-generated timestamp                 |
| `updatedAt`        | Date       | Auto-generated timestamp                 |

**Indexes:**
- `{ channel: 1, createdAt: -1 }` - for fetching channel messages
- `{ gardenId: 1, createdAt: -1 }` - for garden-wide queries
- `{ author: 1, createdAt: -1 }` - for user message history

---

### WebSocket Protocol

#### Authentication

WebSocket connections are authenticated using JWT tokens from cookies:

1. Client connects to `ws://localhost:3033`
2. Server parses the `auth-token` cookie from the request headers
3. JWT is verified using the shared secret (`JWT_SECRET`)
4. On success, user data is attached to the WebSocket instance (`ws.user`, `ws.id`)
5. On failure, connection is closed with code `1008` (Policy Violation)

```javascript
const cookies = cookie.parse(req.headers.cookie || '');
const token = cookies['auth-token'];
const decoded = jwt.verify(token, JWT_SECRET);
ws.user = decoded;
ws.id = decoded._id;
```

#### Initial Load

Upon successful connection, the server sends all existing data:

```json
{
  "type": "initialLoad",
  "messages": [...],
  "channels": [...]
}
```

---

### REST API Endpoints

All channel endpoints are proxied through the API Gateway (`/channel` → Chat Service).

| Method | Endpoint   | Description              | Request Body               |
|--------|------------|--------------------------|----------------------------|
| GET    | `/channel` | List all channels        | -                          |
| POST   | `/channel` | Create a new channel     | `{ name, gardenId }`       |

**Response format (Channel):**
```json
{
  "_id": "64abc123...",
  "name": "general"
}
```

---

### Services

#### Channel Service (`channelService.js`)

| Function                | Description                           |
|-------------------------|---------------------------------------|
| `saveChannel(name, gardenId)` | Creates a new channel          |
| `getChannels()`         | Retrieves all channels                |
| `getFormattedChannels()`| Returns channels in client format     |
| `formatChannel(channel)`| Transforms channel for client         |

#### Message Service (`messageService.js`)

| Function                          | Description                                  |
|-----------------------------------|----------------------------------------------|
| `saveMessage(senderId, username, message)` | Persists a new message              |
| `editMessage(messageId, content, userId)`  | Updates message content (author only)|
| `deleteMessage(messageId, userId)`         | Removes a message (author only)     |
| `getMessages()`                   | Retrieves all messages with channel data     |
| `getFormattedMessages()`          | Returns messages in client format            |
| `broadcastMessage(wss, sender, message)`   | Sends message to all connected clients|
| `broadcastEditedMessage(wss, message)`     | Broadcasts edit to all clients      |
| `broadcastDeletedMessage(wss, message)`    | Broadcasts deletion to all clients  |
| `broadcastActivity(wss, ws, channelId, avatarColor)` | Broadcasts typing indicator |

---

## Frontend

### Component Structure

The chat UI is built with SvelteKit and organized as follows:

```
client/src/
├── routes/chat/
│   └── +page.svelte           # Main chat page
├── lib/
│   ├── services/
│   │   └── chat.svelte.ts     # ChatService class (reactive state)
│   └── components/Chat/
│       ├── Avatar.svelte           # User avatar display
│       ├── ChannelItem.svelte      # Single channel in sidebar
│       ├── ChatHeader.svelte       # Active channel header
│       ├── ChatInput.svelte        # Message input form
│       ├── ChatMessages.svelte     # Message list container
│       ├── ChatSidebar.svelte      # Channel list sidebar
│       ├── CreateChannelModal.svelte # Channel creation modal
│       ├── MessageContent.svelte   # Message text display
│       ├── MessageEditForm.svelte  # Inline message editor
│       ├── MessageItem.svelte      # Individual message bubble
│       ├── MessageMenuButton.svelte # Message options trigger
│       ├── MessageMenuPopup.svelte  # Edit/delete menu
│       ├── MessageUnsendModal.svelte # Delete confirmation
│       ├── TypingDots.svelte       # Animated typing dots
│       ├── TypingIndicator.svelte  # Typing users display
│       └── types.ts                # TypeScript interfaces
```

### ChatService

The `ChatService` class (`chat.svelte.ts`) manages all chat state using Svelte 5 runes:

#### Reactive State Properties

| Property         | Type          | Description                          |
|------------------|---------------|--------------------------------------|
| `channels`       | `any[]`       | List of available channels           |
| `activeChannelId`| `string`      | Currently selected channel ID        |
| `messages`       | `any[]`       | All loaded messages                  |
| `peopleTyping`   | `any[]`       | Users currently typing               |
| `ws`             | `WebSocket`   | WebSocket connection instance        |
| `userData`       | `any`         | Current user's data                  |

#### Key Methods

| Method                            | Description                              |
|-----------------------------------|------------------------------------------|
| `socketsSetup()`                  | Initializes WebSocket connection         |
| `setActiveChannel(channelId)`     | Switches the current channel             |
| `createChannel(name, gardenId)`   | Creates new channel via REST API         |
| `sendMessage(content)`            | Sends a new message                      |
| `sendEditedMessage(messageId, content)` | Sends message edit               |
| `deleteMessage(messageId)`        | Sends message deletion request           |
| `sendActivity()`                  | Broadcasts typing indicator              |
| `processInitialLoad(data)`        | Handles initial data from server         |
| `processActivity(data)`           | Updates typing indicators                |
| `processEditedMessage(payload)`   | Updates local message on edit            |
| `processDeletedMessage(payload)`  | Removes local message on delete          |

#### Context Usage

The ChatService is instantiated in the chat page and shared via Svelte context:

```svelte
<!-- +page.svelte -->
<script lang="ts">
  const chat = new ChatService();
  setContext("chatService", chat);
</script>
```

Child components access it via:

```svelte
<script lang="ts">
  const chat: ChatService = getContext("chatService");
</script>
```

---

### Real-time Communication

#### WebSocket Connection Flow

1. **Connection**: `ChatService` constructor calls `socketsSetup()`
2. **Receive Initial Data**: Server sends `initialLoad` with all messages and channels
3. **Message Handling**: `onmessage` handler routes to appropriate processor
4. **Cleanup**: WebSocket closes when component unmounts

#### Typing Indicator

When a user types in the input:
1. `oninput` triggers `chat.sendActivity()`
2. Server broadcasts `activity` event to other clients
3. Recipients show typing indicator for 1 second (auto-cleared via timeout)

---

## Message Flow Diagrams

### Sending a Message

```
┌─────────┐           ┌─────────┐           ┌─────────┐           ┌─────────┐
│ Client  │           │   WS    │           │ Service │           │ MongoDB │
│   UI    │           │ Server  │           │  Layer  │           │         │
└────┬────┘           └────┬────┘           └────┬────┘           └────┬────┘
     │                     │                     │                     │
     │ sendMessage()       │                     │                     │
     │────────────────────▶│                     │                     │
     │  {type: "message",  │                     │                     │
     │   content, channelId}                     │                     │
     │                     │ saveMessage()       │                     │
     │                     │────────────────────▶│                     │
     │                     │                     │ Message.create()    │
     │                     │                     │────────────────────▶│
     │                     │                     │                     │
     │                     │                     │◀────────────────────│
     │                     │◀────────────────────│                     │
     │                     │                     │                     │
     │                     │ broadcastMessage()  │                     │
     │◀────────────────────│ (to all clients)    │                     │
     │  {type: "text",     │                     │                     │
     │   payload: {...}}   │                     │                     │
     │                     │                     │                     │
```

### Editing a Message

```
┌─────────┐           ┌─────────┐           ┌─────────┐
│ Client  │           │   WS    │           │ MongoDB │
└────┬────┘           │ Server  │           └────┬────┘
     │                └────┬────┘                │
     │ editMessage         │                     │
     │ {type:"editMessage",│                     │
     │  messageId, content}│                     │
     │────────────────────▶│                     │
     │                     │ Verify ownership    │
     │                     │────────────────────▶│
     │                     │ Update content      │
     │                     │────────────────────▶│
     │                     │                     │
     │ {type:"editMessage",│◀────────────────────│
     │  payload:{_id,      │                     │
     │   content, timestamp}}                    │
     │◀────────────────────│                     │
```

---

## WebSocket Message Types

### Client → Server

| Type          | Payload                                    | Description              |
|---------------|--------------------------------------------|--------------------------|
| `message`     | `{ content, channelId, avatarColor }`      | Send new message         |
| `editMessage` | `{ messageId, newContent }`                | Edit existing message    |
| `deleteMessage`| `{ messageId }`                           | Delete a message         |
| `activity`    | `{ channelId, avatarColor }`               | Typing indicator         |

### Server → Client

| Type          | Payload                                    | Description              |
|---------------|--------------------------------------------|--------------------------|
| `initialLoad` | `{ messages[], channels[] }`               | Initial data on connect  |
| `text`        | `{ _id, content, channelId, channelName, author, timestamp }` | New message |
| `editMessage` | `{ _id, content, timestamp }`              | Message was edited       |
| `deleteMessage`| `{ _id }`                                 | Message was deleted      |
| `activity`    | `{ channelId, payload: {username, avatarColor} }` | User is typing    |

---

## Security Considerations

1. **Authentication**: All WebSocket connections require valid JWT in cookies
2. **Authorization**: Users can only edit/delete their own messages
3. **Input Validation**: Message content is trimmed and limited to 3000 characters
4. **CORS**: API Gateway restricts origins to the configured client URL

---

## Configuration

### Environment Variables

| Variable        | Default                           | Description                    |
|-----------------|-----------------------------------|--------------------------------|
| `PORT`          | `3033`                            | Chat service port              |
| `MONGO_URI`     | `mongodb://localhost:27017/juicy-forest` | MongoDB connection string |
| `JWT_SECRET`    | `JWT-SECRET-TOKEN`                | JWT signing secret             |

### Gateway Configuration

The API Gateway (`backend/gateway/index.js`) proxies `/channel` routes to the chat service:

```javascript
services: {
  chat: {
    url: process.env.CHAT_SERVICE_URL || 'http://localhost:3033',
    routes: ['/channel']
  }
}
```

---

## Future Improvements

- [ ] Channel-based message filtering at database level (currently loads all messages)
- [ ] User presence indicators (online/offline status)
- [ ] File/image attachments
- [ ] Message reactions
- [ ] Channel permissions and moderation tools

