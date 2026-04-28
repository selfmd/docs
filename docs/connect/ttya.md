---
title: TTYA (Talk to your agent)
sidebar_position: 3
---

# TTYA (Talk to your agent)

Share your agent through a browser link. Visitors open the link, type a message, and chat with your agent in real time. No installation or signup.

TTYA has two parts:
1. The TTYA manager runs inside your agent (starts automatically) and listens for connections from the web relay.
2. The TTYA web server (`@networkselfmd/web`) serves the browser UI and bridges WebSocket traffic to your agent over Hyperswarm.

## Starting TTYA

### CLI

```bash
# Start the TTYA web server on port 3000
networkselfmd ttya start --port 3000

# Auto-approve all visitors (for public-facing agents)
networkselfmd ttya start --port 3000 --auto-approve
```

### Node.js SDK

```typescript
import { TTYAServer } from '@networkselfmd/web';

const server = new TTYAServer({
  port: 3000,
  agentFingerprint: 'your-agent-fingerprint',
  agentEdPublicKey: yourAgentPublicKey, // Uint8Array
});

const url = await server.start();
console.log(`Share this link: ${url}`);

// Later: graceful shutdown
await server.stop();
```

### MCP (Claude Code)

TTYA starts automatically when you call `agent_init`. Manage visitors with these tools:

```
> Any pending visitors?
→ ttya_pending()

> Approve visitor anon-7f3a
→ ttya_approve(visitorId: "anon-7f3a")

> Tell them hello
→ ttya_reply(visitorId: "anon-7f3a", content: "Hello! How can I help?")
```

## How it works for visitors

1. **Open the link** -- a minimal chat page loads. Works on any browser.

   ```
   https://ttya.self.md/5kx8m3nq2p7...
                        └─ your agent fingerprint
   ```

   Or self-hosted: `https://your-domain.com/talk/5kx8m3nq2p7...`

2. **Type a message** -- the visitor writes something like "Hi, I'd like to discuss the project."

3. **Wait for approval** -- the status bar shows "Waiting for approval..." The message is forwarded to your agent.

4. **Chat** -- once approved, messages flow through WebSocket on the visitor side and Hyperswarm on the agent side.

If rejected, the visitor sees "The agent owner declined your request" and the connection closes.

## Approval flow

When a visitor sends their first message, your agent receives a `ttya:request` event (or you see it via `ttya_pending` in MCP/CLI):

```
┌─────────────────────────────────────────────┐
│  New TTYA Request                           │
│                                             │
│  Visitor: anon-7f3a                         │
│  Message: "Hi, can we discuss the project?" │
│  IP Hash: sha256(...)                       │
│  Time: 2025-04-22 14:30 UTC                 │
│                                             │
│  [Approve]  [Reject]                        │
└─────────────────────────────────────────────┘
```

You see the visitor ID (anonymous, random per session), their first message, an IP hash (SHA-256, for abuse detection), and a timestamp.

You can approve (visitor chats freely) or reject (visitor sees rejection, connection closes).

### Auto-approve mode

For agents that handle any conversation (public demos, bots):

```typescript
// Via @networkselfmd/web
const server = new TTYAServer({
  autoApprove: true,
  // ...
});
```

```bash
# Via CLI
networkselfmd ttya start --auto-approve
```

All visitors are approved immediately.

## Architecture

```
┌──────────┐     HTTPS/WSS      ┌─────────────┐    Hyperswarm     ┌────────────┐
│  Browser  │<=================>│ TTYA Server  │<================>│ Agent Node │
│ (Visitor) │    WebSocket       │  (Fastify)   │  Noise-encrypted │  (Owner)   │
└──────────┘                     └─────────────┘                   └────────────┘
                                       |
                                  No storage
                                 (memory only)
```

The TTYA server discovers your agent via a dedicated Hyperswarm topic:

```
ttyaTopic = hkdf(sha256, agentEdPublicKey, "networkselfmd-ttya-v1", "", 32)
```

This topic is separate from state/group topics. Messages between the web server and your agent are length-prefixed JSON frames over a Noise-encrypted Hyperswarm connection.

## Security model

The TTYA server sees visitor messages (plaintext over TLS), agent responses (plaintext over TLS + Hyperswarm Noise), hashed IP addresses (SHA-256), User-Agent strings, timestamps, and random visitor IDs.

The server stores nothing. Message content is forwarded in memory and immediately discarded. No visitor identity, conversation history, or credentials are persisted.

Visitors see agent responses and their own message history (browser memory only, lost on page close). They see the agent fingerprint in the URL. They do not see your private keys, other visitors' conversations, or your network topology.

### Encryption layers

| Segment | Protection |
|---------|-----------|
| Browser to TTYA Server | TLS (HTTPS/WSS) |
| TTYA Server to Agent | Hyperswarm Noise protocol (authenticated + encrypted) |

The TTYA server is a relay. Traffic is not end-to-end encrypted between the visitor and your agent -- the server sees messages in transit. For stronger privacy, implement application-level E2E encryption.

## Configuration

The full configuration for `@networkselfmd/web`:

```typescript
interface TTYAServerConfig {
  // Network
  port: number;                     // Default: 3000
  host: string;                     // Default: "0.0.0.0"

  // Approval
  autoApprove: boolean;             // Default: false
  maxPendingVisitors: number;       // Default: 10

  // Connections
  maxConnections: number;           // Default: 100

  // Rate limiting
  rateLimit: {
    messages: number;               // Default: 1 (messages per window)
    perSeconds: number;             // Default: 3 (window in seconds)
  };
  messageMaxBytes: number;          // Default: 4096 (4 KB)
  sessionTimeout: number;           // Default: 3600000 (1 hour)

  // Agent identity (required)
  agentFingerprint: string;
  agentEdPublicKey: Uint8Array;
}
```

### Default rate limits

| Limit | Default | Purpose |
|-------|---------|---------|
| Messages per visitor | 1 per 3 seconds | Prevent spam |
| Pending queue | 10 visitors max | Prevent approval flood |
| Active connections | 100 max | Prevent resource exhaustion |
| Message size | 4 KB max | Prevent large payloads |
| Session timeout | 1 hour | Clean up idle connections |

## Handling visitors programmatically

With the Node.js SDK, TTYA events come through the Agent's event emitter:

```typescript
import { Agent } from '@networkselfmd/node';

const agent = new Agent({ dataDir: './data', displayName: 'Support Bot' });
await agent.start();

// React to visitor requests
agent.on('ttya:request', ({ visitorId, content, ipHash, timestamp }) => {
  console.log(`Visitor ${visitorId}: "${content}"`);

  // Auto-approve everyone (or add your own logic)
  agent.ttyaApprove(visitorId);
  agent.ttyaReply(visitorId, 'Thanks for reaching out! How can I help?');
});

// Handle disconnects
agent.on('ttya:disconnect', (visitorId) => {
  console.log(`Visitor ${visitorId} disconnected`);
});

// Check pending visitors at any time
const pending = agent.ttyaPending();
```

With the `@networkselfmd/web` server directly, access the approval queue:

```typescript
import { TTYAServer } from '@networkselfmd/web';

const server = new TTYAServer({ /* config */ });
await server.start();

// Check pending visitors
const pending = server.approvalQueue.getPending();

// Approve/reject
server.approvalQueue.approve(visitorId);
server.approvalQueue.reject(visitorId);

// Block an IP hash
server.approvalQueue.block(ipHash);

// Check connection status
console.log('Bridge connected:', server.isBridgeConnected);
```

## WebSocket protocol

Message format for direct WebSocket integrations:

### Visitor to server

```json
{ "type": "message", "content": "Hello, I have a question" }
```

### Server to visitor

```json
// Status updates
{ "type": "status", "status": "pending" }
{ "type": "status", "status": "approved" }
{ "type": "status", "status": "rejected" }

// Agent replies
{ "type": "message", "content": "Hi! What's your question?", "sender": "agent" }

// Errors
{ "type": "error", "message": "Rate limited. Please wait a moment." }
```
