---
title: TTYA (deferred)
sidebar_position: 3
---

# TTYA — deferred

TTYA is deferred and is not part of the supported product offering. No hosted browser-chat service is advertised, and TTYA tools are not exposed by the MCP server. Existing implementation details below are retained as an archival reference, not an onboarding guide.

## Archival implementation reference

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
