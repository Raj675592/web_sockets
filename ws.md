# WebSockets A–Z: The Complete Node.js Tutorial

> A thorough, end-to-end guide to WebSockets using Node.js — from the protocol fundamentals to production-ready patterns.

---

## Table of Contents

1. [What Are WebSockets?](#1-what-are-websockets)
2. [HTTP vs WebSockets vs Other Techniques](#2-http-vs-websockets-vs-other-techniques)
3. [The WebSocket Protocol Deep Dive](#3-the-websocket-protocol-deep-dive)
4. [Environment Setup](#4-environment-setup)
5. [Raw WebSocket Server with Node.js `net` Module](#5-raw-websocket-server-with-nodejs-net-module)
6. [Using the `ws` Library](#6-using-the-ws-library)
7. [WebSocket Client API in the Browser](#7-websocket-client-api-in-the-browser)
8. [WebSocket Lifecycle & Events](#8-websocket-lifecycle--events)
9. [Sending & Receiving Data](#9-sending--receiving-data)
10. [Building a Chat Application](#10-building-a-chat-application)
11. [Broadcasting & Rooms](#11-broadcasting--rooms)
12. [Heartbeats, Ping/Pong & Keep-Alive](#12-heartbeats-pingpong--keep-alive)
13. [Reconnection Logic on the Client](#13-reconnection-logic-on-the-client)
14. [Authentication & Authorization](#14-authentication--authorization)
15. [WebSocket with Express.js](#15-websocket-with-expressjs)
16. [Using Socket.IO](#16-using-socketio)
17. [Binary Data & Streams](#17-binary-data--streams)
18. [Rate Limiting & Backpressure](#18-rate-limiting--backpressure)
19. [Scaling WebSockets (Redis Pub/Sub)](#19-scaling-websockets-redispubsub)
20. [Error Handling & Debugging](#20-error-handling--debugging)
21. [Security Best Practices](#21-security-best-practices)
22. [Testing WebSocket Servers](#22-testing-websocket-servers)
23. [Production Deployment (Nginx, PM2, Docker)](#23-production-deployment-nginx-pm2-docker)
24. [Performance Tuning](#24-performance-tuning)
25. [Quick Reference Cheat Sheet](#25-quick-reference-cheat-sheet)

---

## 1. What Are WebSockets?

**WebSocket** is a standardized communication protocol (RFC 6455) that provides a **full-duplex, persistent channel** over a single TCP connection. Both the client and server can send data to each other **at any time**, without waiting for a request.

### Why Were WebSockets Invented?

Before WebSockets, developers used hacks like:
- **Polling** — client asks server every N seconds ("Any news?")
- **Long-polling** — client sends a request, server holds it until new data arrives
- **Server-Sent Events** — server can push, but client cannot respond on the same channel

WebSockets solve all of this with a **true two-way channel**.

### Core Properties

| Property | Description |
|---|---|
| **Persistent** | Connection stays open; no repeated TCP handshakes |
| **Full-duplex** | Client and server talk simultaneously |
| **Low latency** | No HTTP overhead after the initial handshake |
| **Lightweight frames** | 2–14 bytes of overhead per message vs hundreds for HTTP |
| **Text & Binary** | Supports UTF-8 text, JSON, Buffers, ArrayBuffers |

### Real-World Use Cases

- Live chat (WhatsApp Web, Slack)
- Collaborative documents (Google Docs-style)
- Online multiplayer games
- Real-time dashboards & analytics
- Financial tickers (stocks, crypto)
- Live sports scores & commentary
- IoT sensor data streaming
- Live notifications & alerts
- Video/audio signaling (WebRTC signaling layer)

---

## 2. HTTP vs WebSockets vs Other Techniques

### HTTP Request–Response

Every interaction requires the client to initiate:

```
Client ──── GET /messages ────► Server
Client ◄─── 200 OK ───────────── Server   (connection closed)

Client ──── GET /messages ────► Server   (must ask again!)
Client ◄─── 200 OK ───────────── Server
```

**Problem:** Server can never push data unprompted. Inefficient for real-time needs.

---

### Polling

```
Client ──── GET /updates ────► Server  (every 2 seconds)
Client ◄─── 200 [] ──────────── Server  (empty most of the time)
Client ──── GET /updates ────► Server
Client ◄─── 200 [] ──────────── Server
Client ──── GET /updates ────► Server
Client ◄─── 200 [new msg!] ─── Server
```

**Problem:** Wastes bandwidth. Latency equals the polling interval.

---

### Long-Polling

```
Client ──── GET /updates ─────────────► Server
                                         (server holds the connection...)
Client ◄─── 200 [new msg!] ─────────── Server  (when something happens)

Client ──── GET /updates ─────────────► Server  (immediately reconnects)
```

**Better**, but still HTTP overhead per cycle. Server must manage many hanging connections.

---

### Server-Sent Events (SSE)

```
Client ──── GET /events ────► Server
Client ◄──────────────────── Server: event: msg\ndata: hello\n\n
Client ◄──────────────────── Server: event: msg\ndata: world\n\n
```

**One-directional push** from server to client only. Client cannot send back on the same channel.

---

### WebSockets ✅

```
Client ──── HTTP Upgrade ────► Server
Client ◄─── 101 Switching ──── Server  (handshake complete)

Client ◄──────────────────────► Server  (full-duplex, persistent)
         ← any side sends anytime →
```

**Best for:** Bidirectional, low-latency, real-time communication.

### Comparison Table

| Feature | Polling | Long-Poll | SSE | WebSocket |
|---|---|---|---|---|
| Bidirectional | ❌ | ❌ | ❌ | ✅ |
| Server Push | ❌ | ✅ | ✅ | ✅ |
| Low Latency | ❌ | ⚠️ | ✅ | ✅ |
| Low Overhead | ❌ | ❌ | ✅ | ✅ |
| Native Browser | ✅ | ✅ | ✅ | ✅ |
| HTTP/2 Compat | ✅ | ✅ | ✅ | ⚠️ |
| Complexity | Low | Medium | Low | Medium |

---

## 3. The WebSocket Protocol Deep Dive

### The Opening Handshake

WebSocket begins as a standard HTTP/1.1 request and upgrades:

**Client sends:**
```http
GET /chat HTTP/1.1
Host: example.com
Upgrade: websocket
Connection: Upgrade
Sec-WebSocket-Key: dGhlIHNhbXBsZSBub25jZQ==
Sec-WebSocket-Version: 13
Origin: http://example.com
```

**Server responds:**
```http
HTTP/1.1 101 Switching Protocols
Upgrade: websocket
Connection: Upgrade
Sec-WebSocket-Accept: s3pPLMBiTxaQ9kYGzzhZRbK+xOo=
```

- `Sec-WebSocket-Key` — random base64 nonce from client
- `Sec-WebSocket-Accept` — server hashes the key with a magic GUID using SHA-1, then base64-encodes it
- After `101`, the connection is no longer HTTP — it's a raw WebSocket channel

### Key Derivation (How the Accept Header Is Computed)

```
accept = base64(SHA1(key + "258EAFA5-E914-47DA-95CA-C5AB0DC85B11"))
```

### WebSocket Frame Format

After the handshake, data travels in **frames**:

```
 0                   1                   2                   3
 0 1 2 3 4 5 6 7 8 9 0 1 2 3 4 5 6 7 8 9 0 1 2 3 4 5 6 7 8 9 0 1
+-+-+-+-+-------+-+-------------+-------------------------------+
|F|R|R|R| opcode|M| Payload len |    Extended payload length    |
|I|S|S|S|  (4)  |A|     (7)     |             (16/64)           |
|N|V|V|V|       |S|             |   (if payload len==126/127)   |
| |1|2|3|       |K|             |                               |
+-+-+-+-+-------+-+-------------+ - - - - - - - - - - - - - - -+
|     Extended payload length continued, if payload len == 127  |
+ - - - - - - - - - - - - - - -+-------------------------------+
|                               |Masking-key, if MASK set to 1  |
+-------------------------------+-------------------------------+
| Masking-key (continued)       |          Payload Data         |
+-------------------------------- - - - - - - - - - - - - - - -+
:                     Payload Data continued ...                :
+---------------------------------------------------------------+
```

### Opcodes

| Opcode | Meaning |
|---|---|
| `0x0` | Continuation frame |
| `0x1` | Text frame (UTF-8) |
| `0x2` | Binary frame |
| `0x8` | Connection close |
| `0x9` | Ping |
| `0xA` | Pong |

### Close Handshake

Either side sends a **Close frame** (opcode `0x8`) with an optional 2-byte status code and reason string. The other side echoes a Close frame and the TCP connection is terminated.

### Common Close Codes

| Code | Meaning |
|---|---|
| 1000 | Normal closure |
| 1001 | Going away (page navigating away) |
| 1002 | Protocol error |
| 1003 | Unsupported data |
| 1006 | Abnormal closure (no close frame) |
| 1007 | Invalid frame payload |
| 1008 | Policy violation |
| 1009 | Message too big |
| 1011 | Internal server error |

---

## 4. Environment Setup

### Prerequisites

- Node.js v18+ (LTS recommended)
- npm or yarn
- A terminal
- A modern browser (Chrome, Firefox, Edge)

### Project Initialization

```bash
mkdir websockets-tutorial
cd websockets-tutorial
npm init -y
```

### Install Core Dependencies

```bash
# Minimal WebSocket library
npm install ws

# For HTTP server integration
npm install express

# Socket.IO (full-featured, covered later)
npm install socket.io socket.io-client

# For auth examples
npm install jsonwebtoken

# Dev tools
npm install --save-dev nodemon
```

### Recommended Project Structure

```
websockets-tutorial/
├── server/
│   ├── index.js           # Main server entry
│   ├── chat.js            # Chat server example
│   ├── rooms.js           # Rooms/broadcast example
│   ├── auth.js            # Auth example
│   └── scaling.js         # Redis pub/sub example
├── client/
│   ├── index.html         # Browser client
│   ├── chat.html          # Chat UI
│   └── client.js          # Reconnect logic
├── tests/
│   └── ws.test.js
├── package.json
└── README.md
```

### Add a dev script to `package.json`

```json
{
  "scripts": {
    "dev": "nodemon server/index.js",
    "start": "node server/index.js"
  }
}
```

---

## 5. Raw WebSocket Server with Node.js `net` Module

Before using a library, let's understand what happens under the hood by manually implementing the handshake. This is **educational only** — use `ws` for real projects.

```js
// server/raw.js
const net = require('net');
const crypto = require('crypto');

const MAGIC = '258EAFA5-E914-47DA-95CA-C5AB0DC85B11';

function generateAcceptKey(wsKey) {
  return crypto
    .createHash('sha1')
    .update(wsKey + MAGIC)
    .digest('base64');
}

function parseHandshake(data) {
  const headers = {};
  const lines = data.toString().split('\r\n');
  lines.slice(1).forEach(line => {
    const [key, ...rest] = line.split(': ');
    if (key) headers[key.toLowerCase()] = rest.join(': ');
  });
  return headers;
}

function decodeFrame(buffer) {
  const firstByte = buffer[0];
  const secondByte = buffer[1];
  const opcode = firstByte & 0x0f;
  const isMasked = !!(secondByte & 0x80);

  let payloadLength = secondByte & 0x7f;
  let maskStart = 2;

  if (payloadLength === 126) {
    payloadLength = buffer.readUInt16BE(2);
    maskStart = 4;
  } else if (payloadLength === 127) {
    payloadLength = buffer.readBigUInt64BE(2);
    maskStart = 10;
  }

  const mask = isMasked ? buffer.slice(maskStart, maskStart + 4) : null;
  const dataStart = isMasked ? maskStart + 4 : maskStart;
  const payload = buffer.slice(dataStart, dataStart + payloadLength);

  if (isMasked) {
    for (let i = 0; i < payload.length; i++) {
      payload[i] ^= mask[i % 4];
    }
  }

  return { opcode, payload: payload.toString('utf8') };
}

function encodeFrame(message) {
  const payload = Buffer.from(message, 'utf8');
  const length = payload.length;
  let frame;

  if (length <= 125) {
    frame = Buffer.allocUnsafe(2 + length);
    frame[1] = length;
  } else if (length <= 65535) {
    frame = Buffer.allocUnsafe(4 + length);
    frame[1] = 126;
    frame.writeUInt16BE(length, 2);
  } else {
    frame = Buffer.allocUnsafe(10 + length);
    frame[1] = 127;
    frame.writeBigUInt64BE(BigInt(length), 2);
  }

  frame[0] = 0x81; // FIN + text opcode
  payload.copy(frame, frame.length - length);
  return frame;
}

const server = net.createServer(socket => {
  let upgraded = false;

  socket.on('data', data => {
    if (!upgraded) {
      // Perform WebSocket handshake
      const headers = parseHandshake(data);
      const wsKey = headers['sec-websocket-key'];
      const acceptKey = generateAcceptKey(wsKey);

      const response = [
        'HTTP/1.1 101 Switching Protocols',
        'Upgrade: websocket',
        'Connection: Upgrade',
        `Sec-WebSocket-Accept: ${acceptKey}`,
        '\r\n'
      ].join('\r\n');

      socket.write(response);
      upgraded = true;
      console.log('Client connected via raw WebSocket!');
    } else {
      // Decode and echo back
      const { opcode, payload } = decodeFrame(data);
      if (opcode === 0x8) {
        socket.end(); // close frame
      } else {
        console.log('Received:', payload);
        socket.write(encodeFrame(`Echo: ${payload}`));
      }
    }
  });

  socket.on('end', () => console.log('Client disconnected'));
  socket.on('error', err => console.error('Socket error:', err.message));
});

server.listen(8080, () => {
  console.log('Raw WebSocket server listening on ws://localhost:8080');
});
```

> **Note:** This is intentionally minimal. A real WebSocket implementation handles fragmented messages, all opcodes, extension negotiation, and more. Use the `ws` library.

---

## 6. Using the `ws` Library

The `ws` package is the de-facto standard low-level WebSocket library for Node.js.

### Basic Echo Server

```js
// server/index.js
const WebSocket = require('ws');

const wss = new WebSocket.Server({ port: 8080 });

wss.on('connection', (ws, req) => {
  const ip = req.socket.remoteAddress;
  console.log(`Client connected from ${ip}`);

  // Send a welcome message
  ws.send(JSON.stringify({ type: 'welcome', message: 'Connected to WebSocket server!' }));

  // Handle incoming messages
  ws.on('message', (data, isBinary) => {
    const message = isBinary ? data : data.toString();
    console.log(`Received: ${message}`);

    // Echo back
    ws.send(JSON.stringify({ type: 'echo', payload: message }));
  });

  // Handle disconnection
  ws.on('close', (code, reason) => {
    console.log(`Client disconnected. Code: ${code}, Reason: ${reason}`);
  });

  // Handle errors
  ws.on('error', err => {
    console.error('WebSocket error:', err.message);
  });
});

wss.on('listening', () => {
  console.log('WebSocket server running on ws://localhost:8080');
});

wss.on('error', err => {
  console.error('Server error:', err);
});
```

### `WebSocketServer` Constructor Options

```js
const wss = new WebSocket.Server({
  port: 8080,              // Port to listen on
  host: '0.0.0.0',        // Bind to all interfaces
  path: '/ws',            // Only handle connections on this path
  maxPayload: 1048576,    // Max message size in bytes (1 MB)
  clientTracking: true,   // Track connected clients in wss.clients
  perMessageDeflate: {    // Enable per-message compression
    zlibDeflateOptions: { chunkSize: 1024, level: 3 },
    zlibInflateOptions: { chunkSize: 10 * 1024 },
    threshold: 1024       // Only compress messages > 1 KB
  }
});
```

### Attaching to an Existing HTTP Server

```js
const http = require('http');
const WebSocket = require('ws');

const server = http.createServer((req, res) => {
  res.writeHead(200, { 'Content-Type': 'text/plain' });
  res.end('HTTP server running');
});

const wss = new WebSocket.Server({ server }); // shares port with HTTP

wss.on('connection', ws => {
  ws.send('Hello from shared server!');
});

server.listen(3000, () => {
  console.log('HTTP + WS server on http://localhost:3000');
});
```

### Checking Client State

```js
// ws.readyState values:
WebSocket.CONNECTING  // 0 - handshake in progress
WebSocket.OPEN        // 1 - connection established
WebSocket.CLOSING     // 2 - close handshake started
WebSocket.CLOSED      // 3 - connection closed

// Always check before sending:
if (ws.readyState === WebSocket.OPEN) {
  ws.send('safe to send');
}
```

---

## 7. WebSocket Client API in the Browser

### Connecting

```js
// Unsecure (development)
const ws = new WebSocket('ws://localhost:8080');

// Secure (production — always use wss:// in production)
const ws = new WebSocket('wss://example.com/ws');

// With sub-protocols
const ws = new WebSocket('wss://example.com/ws', ['json', 'v2']);
```

### Listening for Events

```js
ws.onopen = (event) => {
  console.log('Connection established');
  ws.send(JSON.stringify({ type: 'hello', from: 'browser' }));
};

ws.onmessage = (event) => {
  const data = JSON.parse(event.data);
  console.log('Message received:', data);
};

ws.onclose = (event) => {
  // event.wasClean — true if the close was clean
  // event.code    — close code (e.g. 1000)
  // event.reason  — human-readable reason
  console.log(`Closed: ${event.code} ${event.reason}`);
};

ws.onerror = (event) => {
  // Note: the browser gives very little error info for security reasons
  console.error('WebSocket error. Check the Network tab.');
};
```

### EventListener syntax (alternative)

```js
ws.addEventListener('open', () => { /* ... */ });
ws.addEventListener('message', (e) => { /* ... */ });
ws.addEventListener('close', (e) => { /* ... */ });
ws.addEventListener('error', (e) => { /* ... */ });
```

### Sending Data

```js
// Text
ws.send('Hello!');

// JSON
ws.send(JSON.stringify({ action: 'subscribe', channel: 'prices' }));

// Binary: ArrayBuffer
const buffer = new ArrayBuffer(4);
const view = new Int32Array(buffer);
view[0] = 42;
ws.send(buffer);

// Binary: Blob
const blob = new Blob(['hello'], { type: 'text/plain' });
ws.send(blob);
```

### Closing the Connection

```js
ws.close();               // Code 1000, no reason
ws.close(1000, 'Done');  // Normal closure with reason
```

### Checking State

```js
ws.readyState === WebSocket.CONNECTING  // 0
ws.readyState === WebSocket.OPEN        // 1
ws.readyState === WebSocket.CLOSING     // 2
ws.readyState === WebSocket.CLOSED      // 3
```

---

## 8. WebSocket Lifecycle & Events

### Full Lifecycle Diagram

```
Browser                          Node.js Server
   |                                    |
   |--- TCP SYN -------------------->   |
   |<-- TCP SYN-ACK -----------------  |
   |--- TCP ACK -------------------->   |
   |                                    |
   |--- HTTP GET /ws (Upgrade) ----->   |
   |<-- HTTP 101 Switching ----------   |   [connection event fires]
   |                                    |
   |--- WS Frame (text) ------------>   |   [message event fires]
   |<-- WS Frame (text) -------------  |
   |                                    |
   |<-- WS Ping ---------------------  |   [ping event fires]
   |--- WS Pong -------------------->   |   [pong event fires]
   |                                    |
   |--- WS Close (1000) ------------>   |   [close event fires]
   |<-- WS Close (1000) -------------  |
   |                                    |
   |--- TCP FIN -------------------->   |
   |<-- TCP FIN-ACK -----------------  |
```

### Server-Side Events (ws library)

```js
wss.on('connection', (ws, req) => {
  // Fires when a client completes the handshake

  ws.on('message', (data, isBinary) => {
    // Fires for each incoming message
  });

  ws.on('close', (code, reason) => {
    // Fires when the connection closes
  });

  ws.on('error', (err) => {
    // Fires on socket-level errors
  });

  ws.on('ping', (data) => {
    // Fires when a ping frame is received
    // ws auto-responds with pong by default
  });

  ws.on('pong', (data) => {
    // Fires when a pong frame is received
  });

  ws.on('unexpected-response', (req, res) => {
    // Fires if server doesn't respond with 101
  });

  ws.on('upgrade', (response) => {
    // Fires when the upgrade handshake completes
  });
});
```

---

## 9. Sending & Receiving Data

### Sending with Callbacks and Promises

```js
// With callback
ws.send('message', (err) => {
  if (err) console.error('Send failed:', err);
});

// Promisified helper
function sendAsync(ws, data) {
  return new Promise((resolve, reject) => {
    if (ws.readyState !== WebSocket.OPEN) {
      return reject(new Error('Connection is not open'));
    }
    const payload = typeof data === 'object' ? JSON.stringify(data) : data;
    ws.send(payload, (err) => {
      if (err) reject(err);
      else resolve();
    });
  });
}

// Usage
await sendAsync(ws, { type: 'msg', text: 'Hello!' });
```

### Message Protocol Design

Use a structured envelope for all messages:

```js
// Define message types
const MessageType = {
  CHAT:         'chat',
  JOIN:         'join',
  LEAVE:        'leave',
  TYPING:       'typing',
  PING:         'ping',
  PONG:         'pong',
  ERROR:        'error',
  SUBSCRIBE:    'subscribe',
  UNSUBSCRIBE:  'unsubscribe',
};

// Message envelope
function createMessage(type, payload, meta = {}) {
  return JSON.stringify({
    type,
    payload,
    timestamp: Date.now(),
    ...meta,
  });
}

// Parse incoming messages safely
function parseMessage(raw) {
  try {
    const data = JSON.parse(raw.toString());
    if (!data.type) throw new Error('Missing type field');
    return data;
  } catch (err) {
    return null; // Malformed message
  }
}

// Server-side router
ws.on('message', (raw) => {
  const msg = parseMessage(raw);
  if (!msg) return ws.send(createMessage('error', 'Invalid JSON'));

  switch (msg.type) {
    case MessageType.CHAT:    handleChat(ws, msg.payload);    break;
    case MessageType.JOIN:    handleJoin(ws, msg.payload);    break;
    case MessageType.TYPING:  handleTyping(ws, msg.payload);  break;
    case MessageType.PING:    ws.send(createMessage('pong', null)); break;
    default:
      ws.send(createMessage('error', `Unknown type: ${msg.type}`));
  }
});
```

### Receiving Binary Data (Node.js server)

```js
ws.on('message', (data, isBinary) => {
  if (isBinary) {
    // data is a Buffer
    console.log('Binary length:', data.length);
    console.log('First byte:', data[0]);
    // Process: image upload, audio, etc.
  } else {
    // data is a Buffer of UTF-8 text
    console.log('Text:', data.toString());
  }
});
```

---

## 10. Building a Chat Application

### Server — `server/chat.js`

```js
const http = require('http');
const WebSocket = require('ws');
const { v4: uuidv4 } = require('uuid'); // npm install uuid

const server = http.createServer();
const wss = new WebSocket.Server({ server });

// Store connected clients with metadata
const clients = new Map(); // ws -> { id, username, joinedAt }

function broadcast(data, excludeWs = null) {
  const msg = JSON.stringify(data);
  wss.clients.forEach(client => {
    if (client !== excludeWs && client.readyState === WebSocket.OPEN) {
      client.send(msg);
    }
  });
}

function sendTo(ws, data) {
  if (ws.readyState === WebSocket.OPEN) {
    ws.send(JSON.stringify(data));
  }
}

wss.on('connection', (ws) => {
  const clientId = uuidv4();
  clients.set(ws, { id: clientId, username: null, joinedAt: Date.now() });

  // Request username
  sendTo(ws, { type: 'request_username', message: 'Please send your username' });

  ws.on('message', (raw) => {
    let msg;
    try {
      msg = JSON.parse(raw.toString());
    } catch {
      return sendTo(ws, { type: 'error', message: 'Invalid JSON' });
    }

    const client = clients.get(ws);

    switch (msg.type) {
      case 'set_username': {
        const username = msg.username?.trim();
        if (!username || username.length < 2) {
          return sendTo(ws, { type: 'error', message: 'Username must be at least 2 characters' });
        }

        // Check for duplicate usernames
        const taken = [...clients.values()].some(c => c.username === username);
        if (taken) {
          return sendTo(ws, { type: 'error', message: 'Username already taken' });
        }

        clients.set(ws, { ...client, username });

        sendTo(ws, {
          type: 'joined',
          message: `Welcome, ${username}! There are ${wss.clients.size} user(s) online.`,
          onlineCount: wss.clients.size,
        });

        broadcast({ type: 'user_joined', username, onlineCount: wss.clients.size }, ws);
        console.log(`${username} joined the chat`);
        break;
      }

      case 'chat': {
        if (!client.username) {
          return sendTo(ws, { type: 'error', message: 'Set a username first' });
        }
        if (!msg.text?.trim()) return;

        const chatMsg = {
          type: 'chat',
          id: uuidv4(),
          username: client.username,
          text: msg.text.trim().slice(0, 1000), // Truncate long messages
          timestamp: Date.now(),
        };

        broadcast(chatMsg); // Send to everyone including sender
        break;
      }

      case 'typing': {
        if (client.username) {
          broadcast({ type: 'typing', username: client.username, isTyping: msg.isTyping }, ws);
        }
        break;
      }

      default:
        sendTo(ws, { type: 'error', message: 'Unknown message type' });
    }
  });

  ws.on('close', () => {
    const client = clients.get(ws);
    if (client?.username) {
      broadcast({ type: 'user_left', username: client.username, onlineCount: wss.clients.size - 1 });
      console.log(`${client.username} left the chat`);
    }
    clients.delete(ws);
  });

  ws.on('error', err => console.error(`Client error [${clientId}]:`, err.message));
});

server.listen(8080, () => {
  console.log('Chat server running on ws://localhost:8080');
});
```

### Client — `client/chat.html`

```html
<!DOCTYPE html>
<html lang="en">
<head>
  <meta charset="UTF-8" />
  <title>WebSocket Chat</title>
  <style>
    * { box-sizing: border-box; margin: 0; padding: 0; }
    body { font-family: sans-serif; background: #f0f2f5; height: 100vh; display: flex; flex-direction: column; align-items: center; justify-content: center; }
    #app { width: 480px; background: #fff; border-radius: 12px; box-shadow: 0 4px 20px rgba(0,0,0,.1); overflow: hidden; display: flex; flex-direction: column; height: 600px; }
    #header { background: #4f46e5; color: white; padding: 16px; font-size: 18px; font-weight: bold; }
    #status { font-size: 12px; opacity: .8; margin-top: 4px; }
    #messages { flex: 1; overflow-y: auto; padding: 16px; display: flex; flex-direction: column; gap: 8px; }
    .msg { max-width: 80%; padding: 8px 12px; border-radius: 12px; font-size: 14px; }
    .msg.me { align-self: flex-end; background: #4f46e5; color: white; }
    .msg.other { align-self: flex-start; background: #e5e7eb; color: #111; }
    .msg.system { align-self: center; background: none; color: #888; font-size: 12px; font-style: italic; }
    .msg-meta { font-size: 11px; opacity: .6; margin-bottom: 2px; }
    #typing { color: #888; font-size: 12px; height: 20px; padding: 0 16px; }
    #footer { display: flex; padding: 12px; border-top: 1px solid #eee; gap: 8px; }
    #input { flex: 1; padding: 10px; border: 1px solid #ddd; border-radius: 8px; font-size: 14px; }
    #sendBtn { padding: 10px 20px; background: #4f46e5; color: white; border: none; border-radius: 8px; cursor: pointer; font-size: 14px; }
    #sendBtn:disabled { opacity: .5; cursor: not-allowed; }
  </style>
</head>
<body>
  <div id="app">
    <div id="header">
      💬 WebSocket Chat
      <div id="status">Connecting...</div>
    </div>
    <div id="messages"></div>
    <div id="typing"></div>
    <div id="footer">
      <input id="input" type="text" placeholder="Type a message..." disabled maxlength="1000" />
      <button id="sendBtn" disabled>Send</button>
    </div>
  </div>

  <script>
    const messagesEl = document.getElementById('messages');
    const inputEl = document.getElementById('input');
    const sendBtn = document.getElementById('sendBtn');
    const statusEl = document.getElementById('status');
    const typingEl = document.getElementById('typing');

    let username = null;
    let typingTimer = null;
    let isTyping = false;

    const ws = new WebSocket('ws://localhost:8080');

    ws.onopen = () => {
      statusEl.textContent = 'Connected';
    };

    ws.onclose = () => {
      statusEl.textContent = 'Disconnected';
      inputEl.disabled = true;
      sendBtn.disabled = true;
      addSystem('Disconnected from server');
    };

    ws.onerror = () => {
      addSystem('Connection error');
    };

    ws.onmessage = (event) => {
      const msg = JSON.parse(event.data);

      switch (msg.type) {
        case 'request_username': {
          const name = prompt('Enter a username (min 2 characters):');
          ws.send(JSON.stringify({ type: 'set_username', username: name }));
          break;
        }
        case 'joined': {
          username = msg.message.split(',')[0].split(' ')[1];
          statusEl.textContent = `${msg.onlineCount} online`;
          inputEl.disabled = false;
          sendBtn.disabled = false;
          addSystem(msg.message);
          break;
        }
        case 'chat': {
          addMessage(msg);
          break;
        }
        case 'user_joined': {
          statusEl.textContent = `${msg.onlineCount} online`;
          addSystem(`${msg.username} joined the chat`);
          break;
        }
        case 'user_left': {
          statusEl.textContent = `${msg.onlineCount} online`;
          addSystem(`${msg.username} left the chat`);
          break;
        }
        case 'typing': {
          if (msg.isTyping) {
            typingEl.textContent = `${msg.username} is typing...`;
          } else {
            typingEl.textContent = '';
          }
          break;
        }
        case 'error': {
          addSystem(`Error: ${msg.message}`);
          break;
        }
      }
    };

    function addMessage({ username: user, text, timestamp }) {
      const isMe = user === username;
      const div = document.createElement('div');
      div.className = `msg ${isMe ? 'me' : 'other'}`;
      div.innerHTML = `
        <div class="msg-meta">${isMe ? 'You' : user} · ${new Date(timestamp).toLocaleTimeString()}</div>
        <div>${escapeHtml(text)}</div>
      `;
      messagesEl.appendChild(div);
      messagesEl.scrollTop = messagesEl.scrollHeight;
    }

    function addSystem(text) {
      const div = document.createElement('div');
      div.className = 'msg system';
      div.textContent = text;
      messagesEl.appendChild(div);
      messagesEl.scrollTop = messagesEl.scrollHeight;
    }

    function escapeHtml(text) {
      return text.replace(/&/g,'&amp;').replace(/</g,'&lt;').replace(/>/g,'&gt;');
    }

    function sendMessage() {
      const text = inputEl.value.trim();
      if (!text || ws.readyState !== WebSocket.OPEN) return;
      ws.send(JSON.stringify({ type: 'chat', text }));
      inputEl.value = '';
      sendTyping(false);
    }

    function sendTyping(typing) {
      if (isTyping !== typing) {
        isTyping = typing;
        ws.send(JSON.stringify({ type: 'typing', isTyping: typing }));
      }
    }

    sendBtn.onclick = sendMessage;

    inputEl.addEventListener('keydown', (e) => {
      if (e.key === 'Enter') return sendMessage();
      sendTyping(true);
      clearTimeout(typingTimer);
      typingTimer = setTimeout(() => sendTyping(false), 2000);
    });
  </script>
</body>
</html>
```

---

## 11. Broadcasting & Rooms

### Broadcasting Patterns

```js
// 1. Broadcast to ALL connected clients
function broadcastAll(wss, data) {
  const msg = JSON.stringify(data);
  wss.clients.forEach(client => {
    if (client.readyState === WebSocket.OPEN) {
      client.send(msg);
    }
  });
}

// 2. Broadcast to all EXCEPT the sender
function broadcastOthers(wss, senderWs, data) {
  const msg = JSON.stringify(data);
  wss.clients.forEach(client => {
    if (client !== senderWs && client.readyState === WebSocket.OPEN) {
      client.send(msg);
    }
  });
}

// 3. Send to a specific client by ID
function sendToClient(clients, targetId, data) {
  for (const [ws, meta] of clients.entries()) {
    if (meta.id === targetId && ws.readyState === WebSocket.OPEN) {
      ws.send(JSON.stringify(data));
      return true;
    }
  }
  return false;
}
```

### Room Implementation

```js
// server/rooms.js
const WebSocket = require('ws');

const wss = new WebSocket.Server({ port: 8080 });

// Room structure: Map<roomName, Set<WebSocket>>
const rooms = new Map();

function joinRoom(ws, roomName) {
  if (!rooms.has(roomName)) {
    rooms.set(roomName, new Set());
  }
  rooms.get(roomName).add(ws);

  // Track which rooms the client is in
  if (!ws.rooms) ws.rooms = new Set();
  ws.rooms.add(roomName);

  console.log(`Client joined room: ${roomName} (${rooms.get(roomName).size} members)`);
}

function leaveRoom(ws, roomName) {
  const room = rooms.get(roomName);
  if (!room) return;

  room.delete(ws);
  ws.rooms?.delete(roomName);

  if (room.size === 0) {
    rooms.delete(roomName); // Clean up empty rooms
  }
}

function leaveAllRooms(ws) {
  if (!ws.rooms) return;
  for (const room of ws.rooms) {
    leaveRoom(ws, room);
  }
}

function broadcastToRoom(roomName, data, excludeWs = null) {
  const room = rooms.get(roomName);
  if (!room) return;

  const msg = JSON.stringify(data);
  room.forEach(client => {
    if (client !== excludeWs && client.readyState === WebSocket.OPEN) {
      client.send(msg);
    }
  });
}

function getRoomList() {
  const list = {};
  rooms.forEach((members, name) => {
    list[name] = members.size;
  });
  return list;
}

wss.on('connection', (ws) => {
  ws.rooms = new Set();

  ws.on('message', (raw) => {
    let msg;
    try { msg = JSON.parse(raw.toString()); } catch { return; }

    switch (msg.type) {
      case 'join_room':
        joinRoom(ws, msg.room);
        ws.send(JSON.stringify({
          type: 'room_joined',
          room: msg.room,
          members: rooms.get(msg.room)?.size
        }));
        broadcastToRoom(msg.room, {
          type: 'user_joined_room',
          room: msg.room,
          members: rooms.get(msg.room)?.size
        }, ws);
        break;

      case 'leave_room':
        leaveRoom(ws, msg.room);
        broadcastToRoom(msg.room, {
          type: 'user_left_room',
          room: msg.room,
        });
        break;

      case 'room_message':
        if (!ws.rooms.has(msg.room)) {
          ws.send(JSON.stringify({ type: 'error', message: 'You are not in that room' }));
          return;
        }
        broadcastToRoom(msg.room, {
          type: 'room_message',
          room: msg.room,
          text: msg.text,
          from: ws.username || 'Anonymous',
          timestamp: Date.now(),
        });
        break;

      case 'list_rooms':
        ws.send(JSON.stringify({ type: 'room_list', rooms: getRoomList() }));
        break;
    }
  });

  ws.on('close', () => {
    leaveAllRooms(ws);
  });
});

console.log('Room server on ws://localhost:8080');
```

---

## 12. Heartbeats, Ping/Pong & Keep-Alive

Without periodic checks, dead connections can accumulate (e.g., client crashed without sending a close frame). Implement heartbeats.

### Server-Side Heartbeat

```js
const WebSocket = require('ws');

const wss = new WebSocket.Server({ port: 8080 });

const HEARTBEAT_INTERVAL = 30000; // 30 seconds
const HEARTBEAT_TIMEOUT  = 35000; // Consider dead after 35s

wss.on('connection', (ws) => {
  ws.isAlive = true;
  ws.lastPong = Date.now();

  // Reset alive flag when pong is received
  ws.on('pong', () => {
    ws.isAlive = true;
    ws.lastPong = Date.now();
  });

  // You can also implement application-level ping
  ws.on('message', (raw) => {
    const msg = JSON.parse(raw.toString());
    if (msg.type === 'ping') {
      ws.send(JSON.stringify({ type: 'pong', timestamp: Date.now() }));
    }
  });
});

// Heartbeat interval — ping all clients
const heartbeatInterval = setInterval(() => {
  wss.clients.forEach(ws => {
    if (!ws.isAlive) {
      console.log('Terminating dead connection');
      ws.terminate(); // Force-close without close frame
      return;
    }

    ws.isAlive = false; // Will be reset on pong
    ws.ping(); // Send WebSocket-level ping
  });
}, HEARTBEAT_INTERVAL);

wss.on('close', () => {
  clearInterval(heartbeatInterval);
});

console.log('Heartbeat server on ws://localhost:8080');
```

### Per-Connection Timeout (alternative approach)

```js
wss.on('connection', (ws) => {
  let timeout;

  function resetTimeout() {
    clearTimeout(timeout);
    timeout = setTimeout(() => {
      console.log('Connection timed out');
      ws.terminate();
    }, 60000); // 60 second idle timeout
  }

  resetTimeout();

  ws.on('message', () => resetTimeout()); // Reset on any activity
  ws.on('pong', () => resetTimeout());
  ws.on('close', () => clearTimeout(timeout));
});
```

---

## 13. Reconnection Logic on the Client

WebSockets do not automatically reconnect. Implement this in the browser client.

```js
// client/reconnect.js

class ReconnectingWebSocket {
  constructor(url, options = {}) {
    this.url = url;
    this.options = {
      maxReconnectDelay: 30000,   // 30s max delay
      minReconnectDelay: 1000,    // 1s initial delay
      reconnectDecay: 1.5,        // Exponential backoff factor
      maxRetries: Infinity,       // Retry forever
      debug: false,
      ...options,
    };

    this.retryCount = 0;
    this.ws = null;
    this.shouldReconnect = true;
    this.listeners = { open: [], message: [], close: [], error: [] };

    this.connect();
  }

  connect() {
    this.ws = new WebSocket(this.url);

    this.ws.onopen = (e) => {
      this.retryCount = 0;
      this.log('Connected');
      this.listeners.open.forEach(fn => fn(e));
    };

    this.ws.onmessage = (e) => {
      this.listeners.message.forEach(fn => fn(e));
    };

    this.ws.onerror = (e) => {
      this.log('Error');
      this.listeners.error.forEach(fn => fn(e));
    };

    this.ws.onclose = (e) => {
      this.log(`Closed: ${e.code}`);
      this.listeners.close.forEach(fn => fn(e));

      if (this.shouldReconnect && this.retryCount < this.options.maxRetries) {
        const delay = Math.min(
          this.options.minReconnectDelay * Math.pow(this.options.reconnectDecay, this.retryCount),
          this.options.maxReconnectDelay
        );
        this.retryCount++;
        this.log(`Reconnecting in ${Math.round(delay)}ms (attempt ${this.retryCount})`);
        setTimeout(() => this.connect(), delay);
      }
    };
  }

  send(data) {
    if (this.ws?.readyState === WebSocket.OPEN) {
      this.ws.send(typeof data === 'object' ? JSON.stringify(data) : data);
    } else {
      this.log('Cannot send — not connected');
    }
  }

  close(code = 1000, reason = '') {
    this.shouldReconnect = false;
    this.ws?.close(code, reason);
  }

  on(event, callback) {
    this.listeners[event]?.push(callback);
    return this; // Chainable
  }

  log(msg) {
    if (this.options.debug) console.log(`[RWS] ${msg}`);
  }
}

// Usage
const ws = new ReconnectingWebSocket('ws://localhost:8080', { debug: true });

ws.on('open', () => console.log('Connected!'))
  .on('message', (e) => console.log('Received:', e.data))
  .on('close', (e) => console.log('Closed:', e.code));

ws.send({ type: 'hello' });
```

---

## 14. Authentication & Authorization

### Method 1: Token in the URL Query String

```js
// Client (browser)
const token = localStorage.getItem('authToken');
const ws = new WebSocket(`wss://example.com/ws?token=${token}`);
```

```js
// Server (ws library)
const url = require('url');
const jwt = require('jsonwebtoken');

const JWT_SECRET = process.env.JWT_SECRET || 'your-secret-key';

wss.on('connection', (ws, req) => {
  const params = new url.URL(req.url, 'http://localhost').searchParams;
  const token = params.get('token');

  try {
    const user = jwt.verify(token, JWT_SECRET);
    ws.user = user;
    ws.send(JSON.stringify({ type: 'auth_success', username: user.name }));
  } catch (err) {
    ws.send(JSON.stringify({ type: 'auth_error', message: 'Invalid token' }));
    ws.close(1008, 'Unauthorized'); // Policy violation close code
  }
});
```

> ⚠️ **Note:** Query string tokens appear in server logs. Prefer cookies or the first-message method below.

### Method 2: Cookie-Based Auth (Recommended for Browsers)

```js
// Client: Just connect — cookies are sent automatically
const ws = new WebSocket('wss://example.com/ws');
```

```js
// Server: Read cookies from the handshake request
const cookie = require('cookie');
const jwt = require('jsonwebtoken');

wss.on('connection', (ws, req) => {
  const cookies = cookie.parse(req.headers.cookie || '');
  const token = cookies.session;

  if (!token) {
    ws.close(1008, 'No session cookie');
    return;
  }

  try {
    ws.user = jwt.verify(token, process.env.JWT_SECRET);
  } catch {
    ws.close(1008, 'Invalid session');
  }
});
```

### Method 3: First-Message Authentication

The client sends credentials as the first message after connecting:

```js
// Client
ws.onopen = () => {
  ws.send(JSON.stringify({
    type: 'auth',
    token: localStorage.getItem('authToken'),
  }));
};
```

```js
// Server
wss.on('connection', (ws) => {
  ws.authenticated = false;

  const authTimeout = setTimeout(() => {
    if (!ws.authenticated) {
      ws.close(1008, 'Authentication timeout');
    }
  }, 5000); // Must authenticate within 5 seconds

  ws.on('message', (raw) => {
    const msg = JSON.parse(raw.toString());

    if (!ws.authenticated) {
      if (msg.type === 'auth') {
        try {
          ws.user = jwt.verify(msg.token, JWT_SECRET);
          ws.authenticated = true;
          clearTimeout(authTimeout);
          ws.send(JSON.stringify({ type: 'auth_success' }));
        } catch {
          ws.close(1008, 'Invalid token');
        }
      } else {
        ws.send(JSON.stringify({ type: 'error', message: 'Authenticate first' }));
      }
      return;
    }

    // Handle authenticated messages
    handleMessage(ws, msg);
  });
});
```

### Middleware-Style Auth with Express

```js
const express = require('express');
const http = require('http');
const WebSocket = require('ws');
const jwt = require('jsonwebtoken');

const app = express();
const server = http.createServer(app);
const wss = new WebSocket.Server({ noServer: true }); // noServer: handle upgrade manually

server.on('upgrade', (req, socket, head) => {
  // Parse token from cookie or query
  const params = new URL(req.url, 'http://localhost').searchParams;
  const token = params.get('token');

  jwt.verify(token, process.env.JWT_SECRET, (err, user) => {
    if (err) {
      socket.write('HTTP/1.1 401 Unauthorized\r\n\r\n');
      socket.destroy();
      return;
    }

    req.user = user;
    wss.handleUpgrade(req, socket, head, (ws) => {
      wss.emit('connection', ws, req);
    });
  });
});

wss.on('connection', (ws, req) => {
  console.log(`Authenticated user: ${req.user.name}`);
  ws.user = req.user;
});

server.listen(3000);
```

---

## 15. WebSocket with Express.js

### Integrating `ws` with Express on One Port

```js
// server/express-ws.js
const express = require('express');
const http = require('http');
const WebSocket = require('ws');
const path = require('path');

const app = express();
const server = http.createServer(app);
const wss = new WebSocket.Server({ server, path: '/ws' });

// Express routes
app.use(express.static(path.join(__dirname, '../client')));
app.use(express.json());

app.get('/api/status', (req, res) => {
  res.json({
    connectedClients: wss.clients.size,
    uptime: process.uptime(),
  });
});

app.post('/api/broadcast', (req, res) => {
  const { message } = req.body;
  let sent = 0;

  wss.clients.forEach(client => {
    if (client.readyState === WebSocket.OPEN) {
      client.send(JSON.stringify({ type: 'broadcast', message }));
      sent++;
    }
  });

  res.json({ sent });
});

// WebSocket handling
wss.on('connection', (ws, req) => {
  console.log('New WS connection from', req.socket.remoteAddress);

  ws.on('message', (data) => {
    console.log('Received:', data.toString());
    ws.send(JSON.stringify({ type: 'ack', received: true }));
  });
});

server.listen(3000, () => {
  console.log('Server: http://localhost:3000');
  console.log('WebSocket: ws://localhost:3000/ws');
});
```

### Using `express-ws` Package

```bash
npm install express-ws
```

```js
const express = require('express');
const expressWs = require('express-ws');

const app = express();
expressWs(app); // Patches app with .ws() method

// WebSocket route
app.ws('/chat', (ws, req) => {
  ws.on('message', (msg) => {
    ws.send(`Echo: ${msg}`);
  });
});

// Regular HTTP routes work alongside WebSocket routes
app.get('/', (req, res) => res.send('Hello HTTP!'));

app.listen(3000, () => console.log('Listening on :3000'));
```

---

## 16. Using Socket.IO

**Socket.IO** builds on WebSockets and adds: automatic reconnection, rooms, namespaces, event-based API, and fallback to long-polling.

### Installation

```bash
npm install socket.io        # Server
npm install socket.io-client # Node.js client
```

### Socket.IO Server

```js
// server/socketio.js
const http = require('http');
const { Server } = require('socket.io');

const server = http.createServer();
const io = new Server(server, {
  cors: {
    origin: '*',     // In production, restrict this
    methods: ['GET', 'POST'],
  },
  transports: ['websocket', 'polling'], // Try WS first, fallback to polling
});

// Middleware
io.use((socket, next) => {
  const token = socket.handshake.auth.token;
  if (!token) return next(new Error('Authentication required'));

  try {
    socket.user = jwt.verify(token, process.env.JWT_SECRET);
    next();
  } catch {
    next(new Error('Invalid token'));
  }
});

io.on('connection', (socket) => {
  console.log(`Connected: ${socket.id}`);

  // Emit to this client only
  socket.emit('welcome', { message: 'Hello!' });

  // Custom events
  socket.on('join_room', (room) => {
    socket.join(room);
    socket.to(room).emit('user_joined', { id: socket.id });
    io.to(room).emit('room_count', io.sockets.adapter.rooms.get(room)?.size);
  });

  socket.on('chat', ({ room, text }) => {
    io.to(room).emit('chat', {
      text,
      from: socket.id,
      timestamp: Date.now(),
    });
  });

  socket.on('disconnect', (reason) => {
    console.log(`Disconnected: ${socket.id} (${reason})`);
  });
});

// Namespaces — separate "virtual" servers
const adminNS = io.of('/admin');
adminNS.on('connection', (socket) => {
  console.log('Admin connected:', socket.id);
});

server.listen(3000, () => console.log('Socket.IO server on :3000'));
```

### Socket.IO Browser Client

```html
<script src="https://cdn.socket.io/4.7.2/socket.io.min.js"></script>
<script>
  const socket = io('http://localhost:3000', {
    auth: { token: localStorage.getItem('token') },
    transports: ['websocket'],
  });

  socket.on('connect', () => {
    console.log('Connected:', socket.id);
    socket.emit('join_room', 'general');
  });

  socket.on('welcome', (data) => console.log(data.message));

  socket.on('chat', ({ text, from, timestamp }) => {
    console.log(`${from}: ${text}`);
  });

  socket.on('connect_error', (err) => {
    console.error('Connection failed:', err.message);
  });

  // Send a message
  socket.emit('chat', { room: 'general', text: 'Hello everyone!' });
</script>
```

### Socket.IO Key Emission Patterns

```js
// To the sender only
socket.emit('event', data);

// To everyone except the sender
socket.broadcast.emit('event', data);

// To a specific room
io.to('room1').emit('event', data);

// To multiple rooms
io.to('room1').to('room2').emit('event', data);

// To a specific socket by ID
io.to(socketId).emit('event', data);

// To everyone in a room except sender
socket.to('room1').emit('event', data);

// To everyone on the server
io.emit('event', data);
```

### ws vs Socket.IO

| Feature | `ws` | Socket.IO |
|---|---|---|
| Protocol | Pure WebSocket | WS + custom protocol |
| Bundle size | Tiny | ~45KB (client) |
| Auto-reconnect | ❌ Manual | ✅ Built-in |
| Rooms | ❌ Manual | ✅ Built-in |
| Namespaces | ❌ | ✅ |
| Fallback transports | ❌ | ✅ (polling) |
| Acknowledgements | ❌ | ✅ |
| Broadcast | Manual | Built-in |
| Binary | ✅ | ✅ |
| Learning curve | Low | Medium |
| Use when... | Full control | Feature-rich apps |

---

## 17. Binary Data & Streams

### Sending Binary from Server

```js
wss.on('connection', (ws) => {
  // Send a Buffer
  const buf = Buffer.from([0x01, 0x02, 0x03, 0x04]);
  ws.send(buf);

  // Send an image file
  const fs = require('fs');
  const image = fs.readFileSync('./photo.jpg');
  ws.send(image);
});
```

### Receiving Binary on Server

```js
ws.on('message', (data, isBinary) => {
  if (isBinary) {
    // data is a Buffer
    const fs = require('fs');
    fs.writeFileSync(`./uploads/${Date.now()}.bin`, data);
    console.log(`Saved ${data.length} bytes`);
  }
});
```

### Streaming Large Files in Chunks

Avoid sending huge files in one frame. Break them into chunks:

```js
// Server: send a file in chunks
async function sendFileInChunks(ws, filePath, chunkSize = 64 * 1024) {
  const fs = require('fs');
  const stat = fs.statSync(filePath);
  const totalChunks = Math.ceil(stat.size / chunkSize);
  const stream = fs.createReadStream(filePath, { highWaterMark: chunkSize });

  let chunkIndex = 0;

  // Send file metadata first
  ws.send(JSON.stringify({
    type: 'file_start',
    name: require('path').basename(filePath),
    size: stat.size,
    chunks: totalChunks,
  }));

  for await (const chunk of stream) {
    // Send chunk index as first 4 bytes, then the data
    const frame = Buffer.allocUnsafe(4 + chunk.length);
    frame.writeUInt32BE(chunkIndex, 0);
    chunk.copy(frame, 4);
    ws.send(frame);
    chunkIndex++;
  }

  ws.send(JSON.stringify({ type: 'file_end' }));
}
```

```js
// Client: reassemble chunks
const chunks = [];
let totalChunks = 0;

ws.onmessage = (event) => {
  if (typeof event.data === 'string') {
    const msg = JSON.parse(event.data);
    if (msg.type === 'file_start') {
      totalChunks = msg.chunks;
      console.log(`Receiving ${msg.name} (${msg.size} bytes, ${totalChunks} chunks)`);
    } else if (msg.type === 'file_end') {
      const blob = new Blob(chunks.sort((a, b) => a.index - b.index).map(c => c.data));
      const url = URL.createObjectURL(blob);
      console.log('File ready:', url);
    }
  } else {
    // Binary chunk
    event.data.arrayBuffer().then(buf => {
      const view = new DataView(buf);
      const index = view.getUint32(0);
      const data = buf.slice(4);
      chunks.push({ index, data });
    });
  }
};
```

---

## 18. Rate Limiting & Backpressure

### Message Rate Limiting per Client

```js
wss.on('connection', (ws) => {
  const RATE_LIMIT = 10;   // Max messages
  const WINDOW_MS = 1000;  // Per second

  let messageCount = 0;
  let windowStart = Date.now();

  ws.on('message', (data) => {
    const now = Date.now();

    // Reset window if expired
    if (now - windowStart > WINDOW_MS) {
      messageCount = 0;
      windowStart = now;
    }

    messageCount++;

    if (messageCount > RATE_LIMIT) {
      ws.send(JSON.stringify({ type: 'error', message: 'Rate limit exceeded. Slow down.' }));
      // Optional: disconnect repeat offenders
      // ws.close(1008, 'Rate limit exceeded');
      return;
    }

    handleMessage(ws, data);
  });
});
```

### Handling Backpressure on the Server

When sending fast, the server's write buffer can fill up. Handle this with `ws.bufferedAmount`:

```js
function safeSend(ws, data) {
  // Check if the socket's internal buffer is too full
  if (ws.bufferedAmount > 1024 * 1024) { // 1MB threshold
    console.warn('Client buffer full, skipping message');
    return false;
  }
  ws.send(data);
  return true;
}

// For streaming scenarios, use the drain event
ws.on('drain', () => {
  console.log('Buffer drained, can send more');
});
```

---

## 19. Scaling WebSockets (Redis Pub/Sub)

A single Node.js process can't share WebSocket state across multiple instances. Use **Redis Pub/Sub** as a message bus.

```bash
npm install ioredis
```

### Architecture

```
Client A → Node Instance 1 → Redis PUBLISH → Node Instance 2 → Client B
                           → Redis PUBLISH → Node Instance 3 → Client C
```

### Server with Redis Pub/Sub

```js
// server/scaling.js
const WebSocket = require('ws');
const Redis = require('ioredis');

const CHANNEL = 'ws_broadcast';

const publisher  = new Redis({ host: 'localhost', port: 6379 });
const subscriber = new Redis({ host: 'localhost', port: 6379 });

subscriber.subscribe(CHANNEL);

const wss = new WebSocket.Server({ port: process.env.PORT || 8080 });

// When Redis receives a message, broadcast to all local clients
subscriber.on('message', (channel, message) => {
  if (channel === CHANNEL) {
    wss.clients.forEach(client => {
      if (client.readyState === WebSocket.OPEN) {
        client.send(message);
      }
    });
  }
});

wss.on('connection', (ws) => {
  ws.on('message', async (data) => {
    const msg = data.toString();
    // Publish to Redis so ALL instances broadcast it
    await publisher.publish(CHANNEL, msg);
  });

  ws.on('close', () => {
    console.log('Client disconnected');
  });
});

console.log(`WebSocket server on port ${process.env.PORT || 8080}`);
```

### Socket.IO with Redis Adapter

```bash
npm install @socket.io/redis-adapter ioredis
```

```js
const { createServer } = require('http');
const { Server } = require('socket.io');
const { createAdapter } = require('@socket.io/redis-adapter');
const { createClient } = require('redis');

const httpServer = createServer();
const io = new Server(httpServer);

const pubClient = createClient({ url: 'redis://localhost:6379' });
const subClient = pubClient.duplicate();

Promise.all([pubClient.connect(), subClient.connect()]).then(() => {
  io.adapter(createAdapter(pubClient, subClient));

  io.on('connection', (socket) => {
    socket.on('chat', (msg) => {
      // Broadcasts across ALL Socket.IO instances via Redis
      io.emit('chat', msg);
    });
  });

  httpServer.listen(3000);
  console.log('Clustered Socket.IO on :3000');
});
```

---

## 20. Error Handling & Debugging

### Comprehensive Error Handling

```js
// server/error-handling.js
const WebSocket = require('ws');

const wss = new WebSocket.Server({ port: 8080 });

// Server-level errors
wss.on('error', (err) => {
  console.error('[Server Error]', err.message);
  // Could restart or alert monitoring
});

wss.on('connection', (ws, req) => {
  // Per-connection errors
  ws.on('error', (err) => {
    // Common errors:
    // ECONNRESET — client forcibly closed the TCP connection
    // EPIPE      — tried to write to a closed socket
    if (err.code !== 'ECONNRESET' && err.code !== 'EPIPE') {
      console.error('[WebSocket Error]', err.code, err.message);
    }
    // The 'close' event will fire after 'error'
  });

  ws.on('message', (data) => {
    try {
      const msg = JSON.parse(data.toString());
      processMessage(ws, msg);
    } catch (err) {
      console.error('[Message Error]', err.message);
      safeClose(ws, 1007, 'Invalid payload');
    }
  });
});

function safeClose(ws, code = 1000, reason = '') {
  try {
    if (ws.readyState === WebSocket.OPEN) {
      ws.close(code, reason);
    } else {
      ws.terminate();
    }
  } catch (err) {
    console.error('Error during close:', err.message);
  }
}

// Graceful shutdown
process.on('SIGINT', () => {
  console.log('Shutting down gracefully...');
  wss.clients.forEach(client => client.close(1001, 'Server shutting down'));
  wss.close(() => {
    console.log('Server closed');
    process.exit(0);
  });
});
```

### Debugging in the Browser

Open DevTools → **Network tab** → Filter by **WS**:
- Click a WebSocket connection to inspect it
- Go to **Messages** sub-tab to see all frames sent and received
- Green = sent by client, White/grey = received from server

### Debugging with wscat (CLI tool)

```bash
# Install globally
npm install -g wscat

# Connect to a WS server
wscat -c ws://localhost:8080

# Connect with headers
wscat -c ws://localhost:8080 -H "Authorization: Bearer TOKEN"

# Connect to secure server
wscat -c wss://example.com/ws
```

### Logging Middleware

```js
function withLogging(wss) {
  wss.on('connection', (ws, req) => {
    const ip = req.socket.remoteAddress;
    const id = Math.random().toString(36).slice(2, 9);
    console.log(`[${id}] CONNECT from ${ip}`);

    const originalSend = ws.send.bind(ws);
    ws.send = (data, ...args) => {
      console.log(`[${id}] SEND:`, typeof data === 'string' ? data.slice(0, 200) : `<binary ${data.length} bytes>`);
      return originalSend(data, ...args);
    };

    ws.on('message', (data) => {
      console.log(`[${id}] RECV:`, data.toString().slice(0, 200));
    });

    ws.on('close', (code, reason) => {
      console.log(`[${id}] CLOSE: ${code} ${reason}`);
    });
  });
}

withLogging(wss);
```

---

## 21. Security Best Practices

### 1. Always Use `wss://` in Production

```js
// ❌ Development only
const ws = new WebSocket('ws://example.com');

// ✅ Production (TLS-encrypted)
const ws = new WebSocket('wss://example.com');
```

Configure TLS on your Node.js server:

```js
const https = require('https');
const fs = require('fs');
const WebSocket = require('ws');

const server = https.createServer({
  cert: fs.readFileSync('/etc/letsencrypt/live/example.com/fullchain.pem'),
  key:  fs.readFileSync('/etc/letsencrypt/live/example.com/privkey.pem'),
});

const wss = new WebSocket.Server({ server });
server.listen(443);
```

### 2. Validate the Origin Header

Prevent Cross-Site WebSocket Hijacking (CSWSH):

```js
const wss = new WebSocket.Server({
  port: 8080,
  verifyClient: (info) => {
    const origin = info.origin || info.req.headers.origin;
    const allowedOrigins = ['https://myapp.com', 'https://www.myapp.com'];

    if (!allowedOrigins.includes(origin)) {
      console.warn(`Rejected connection from origin: ${origin}`);
      return false; // Reject
    }
    return true; // Accept
  },
});
```

### 3. Sanitize All Input

```js
function sanitize(str, maxLen = 1000) {
  if (typeof str !== 'string') return '';
  return str.trim().slice(0, maxLen);
}

ws.on('message', (raw) => {
  let msg;
  try {
    msg = JSON.parse(raw.toString());
  } catch {
    return ws.close(1007, 'Invalid JSON');
  }

  // Validate each field
  if (msg.type !== 'chat' && msg.type !== 'join') {
    return ws.close(1008, 'Unknown type');
  }

  msg.text = sanitize(msg.text);
  processMessage(ws, msg);
});
```

### 4. Limit Message Size

```js
const wss = new WebSocket.Server({
  port: 8080,
  maxPayload: 100 * 1024, // 100 KB max message size
});
```

### 5. Rate Limit Connections

```js
const connectionCounts = new Map(); // IP -> count

wss.on('connection', (ws, req) => {
  const ip = req.socket.remoteAddress;
  const MAX_CONNECTIONS_PER_IP = 5;
  const current = connectionCounts.get(ip) || 0;

  if (current >= MAX_CONNECTIONS_PER_IP) {
    ws.close(1008, 'Too many connections');
    return;
  }

  connectionCounts.set(ip, current + 1);

  ws.on('close', () => {
    const count = connectionCounts.get(ip) - 1;
    if (count <= 0) connectionCounts.delete(ip);
    else connectionCounts.set(ip, count);
  });
});
```

### 6. Use Helmet with Express (HTTP Headers)

```bash
npm install helmet
```

```js
const helmet = require('helmet');
app.use(helmet()); // Sets secure HTTP headers
```

### 7. Never Trust Client Data

```js
// ❌ Never do this
ws.on('message', (data) => {
  const { userId } = JSON.parse(data.toString());
  // Using userId from client — client can send ANY userId!
  deleteUser(userId);
});

// ✅ Always use server-side session data
ws.on('message', (data) => {
  const { action } = JSON.parse(data.toString());
  const userId = ws.user.id; // From verified JWT/session
  if (action === 'delete') deleteUser(userId);
});
```

### Security Checklist

- [ ] `wss://` (TLS) in production
- [ ] Origin header validation
- [ ] JWT / session authentication
- [ ] Input validation and sanitization
- [ ] Max payload size configured
- [ ] Rate limiting (messages & connections)
- [ ] No sensitive data in close reason strings
- [ ] Dependency audit (`npm audit`)
- [ ] Server-side state only for authorization

---

## 22. Testing WebSocket Servers

### Unit Testing with Jest + `ws`

```bash
npm install --save-dev jest ws
```

```js
// tests/ws.test.js
const WebSocket = require('ws');

// Import the server factory (not auto-started)
const { createServer } = require('../server/chat'); // export wss separately

let wss;
let port;

beforeEach((done) => {
  wss = createServer(0); // port 0 = random available port
  wss.on('listening', () => {
    port = wss.address().port;
    done();
  });
});

afterEach((done) => {
  wss.close(done);
});

function connectClient() {
  return new Promise((resolve, reject) => {
    const ws = new WebSocket(`ws://localhost:${port}`);
    ws.on('open', () => resolve(ws));
    ws.on('error', reject);
  });
}

function waitForMessage(ws) {
  return new Promise((resolve) => {
    ws.once('message', (data) => resolve(JSON.parse(data.toString())));
  });
}

test('Server sends welcome on connect', async () => {
  const ws = await connectClient();
  const msg = await waitForMessage(ws);
  expect(msg.type).toBe('welcome');
  ws.close();
});

test('Echo works correctly', async () => {
  const ws = await connectClient();
  await waitForMessage(ws); // Consume welcome

  ws.send(JSON.stringify({ type: 'echo', text: 'hello' }));
  const response = await waitForMessage(ws);

  expect(response.type).toBe('echo');
  expect(response.text).toBe('hello');
  ws.close();
});

test('Two clients receive broadcast', async () => {
  const ws1 = await connectClient();
  const ws2 = await connectClient();

  await waitForMessage(ws1); // welcome
  await waitForMessage(ws2); // welcome

  const received = waitForMessage(ws2); // ws2 waits for broadcast

  ws1.send(JSON.stringify({ type: 'broadcast', text: 'hi from ws1' }));

  const msg = await received;
  expect(msg.text).toBe('hi from ws1');

  ws1.close();
  ws2.close();
});
```

### Exporting the Server for Testing

```js
// server/chat.js — testable structure
function createServer(port = 8080) {
  const http = require('http');
  const httpServer = http.createServer();
  const wss = new WebSocket.Server({ server: httpServer });

  // ... setup handlers ...

  httpServer.listen(port);
  return wss; // expose for testing
}

module.exports = { createServer };
```

---

## 23. Production Deployment (Nginx, PM2, Docker)

### Nginx Reverse Proxy Configuration

WebSocket connections need specific headers to be proxied correctly:

```nginx
# /etc/nginx/sites-available/myapp
server {
    listen 80;
    server_name example.com;
    return 301 https://$host$request_uri;
}

server {
    listen 443 ssl;
    server_name example.com;

    ssl_certificate     /etc/letsencrypt/live/example.com/fullchain.pem;
    ssl_certificate_key /etc/letsencrypt/live/example.com/privkey.pem;

    # HTTP routes
    location /api {
        proxy_pass http://localhost:3000;
        proxy_http_version 1.1;
        proxy_set_header Host $host;
        proxy_set_header X-Real-IP $remote_addr;
    }

    # WebSocket routes
    location /ws {
        proxy_pass http://localhost:3000;
        proxy_http_version 1.1;

        # Critical headers for WebSocket upgrade
        proxy_set_header Upgrade $http_upgrade;
        proxy_set_header Connection "upgrade";

        proxy_set_header Host $host;
        proxy_set_header X-Real-IP $remote_addr;
        proxy_set_header X-Forwarded-For $proxy_add_x_forwarded_for;

        # WebSocket timeout settings
        proxy_read_timeout 86400s;  # 24 hours — prevent nginx from closing idle connections
        proxy_send_timeout 86400s;
    }
}
```

### Running with PM2

```bash
npm install -g pm2
```

```js
// ecosystem.config.js
module.exports = {
  apps: [
    {
      name: 'ws-server',
      script: './server/index.js',
      instances: 'max',        // Use all CPU cores
      exec_mode: 'cluster',    // Node.js cluster mode
      watch: false,
      env: {
        NODE_ENV: 'production',
        PORT: 3000,
        JWT_SECRET: 'your-production-secret',
        REDIS_URL: 'redis://localhost:6379',
      },
      max_memory_restart: '512M',
      error_file: './logs/err.log',
      out_file: './logs/out.log',
    },
  ],
};
```

```bash
pm2 start ecosystem.config.js
pm2 save
pm2 startup  # Auto-start on reboot
```

> **Note:** With PM2 cluster mode and WebSockets, use a Redis adapter (Socket.IO) or sticky sessions in Nginx to ensure a client always hits the same instance.

### Sticky Sessions in Nginx (for Cluster)

```nginx
upstream ws_backend {
    ip_hash; # Sticky sessions based on client IP
    server localhost:3001;
    server localhost:3002;
    server localhost:3003;
    server localhost:3004;
}

server {
    location /ws {
        proxy_pass http://ws_backend;
        proxy_http_version 1.1;
        proxy_set_header Upgrade $http_upgrade;
        proxy_set_header Connection "upgrade";
    }
}
```

### Dockerfile

```dockerfile
# Dockerfile
FROM node:20-alpine

WORKDIR /app

COPY package*.json ./
RUN npm ci --only=production

COPY . .

EXPOSE 3000

ENV NODE_ENV=production

CMD ["node", "server/index.js"]
```

```yaml
# docker-compose.yml
version: '3.8'

services:
  app:
    build: .
    ports:
      - "3000:3000"
    environment:
      - NODE_ENV=production
      - REDIS_URL=redis://redis:6379
      - JWT_SECRET=${JWT_SECRET}
    depends_on:
      - redis
    restart: unless-stopped

  redis:
    image: redis:7-alpine
    ports:
      - "6379:6379"
    volumes:
      - redis_data:/data
    restart: unless-stopped

  nginx:
    image: nginx:alpine
    ports:
      - "80:80"
      - "443:443"
    volumes:
      - ./nginx.conf:/etc/nginx/conf.d/default.conf:ro
      - /etc/letsencrypt:/etc/letsencrypt:ro
    depends_on:
      - app
    restart: unless-stopped

volumes:
  redis_data:
```

---

## 24. Performance Tuning

### Node.js System Tuning

```bash
# Increase max file descriptors (each WS connection uses one)
ulimit -n 100000

# Add to /etc/security/limits.conf for persistence:
# * soft nofile 100000
# * hard nofile 100000
```

### Node.js Application Tuning

```js
// Tune the HTTP server
const server = http.createServer();
server.maxHeadersCount = 50;
server.setTimeout(0); // Disable default timeout for WS connections

// Increase TCP socket buffer sizes
server.on('connection', (socket) => {
  socket.setNoDelay(true);       // Disable Nagle's algorithm for low latency
  socket.setKeepAlive(true, 60000); // TCP keepalive every 60s
});

// Use per-message deflate only when beneficial
const wss = new WebSocket.Server({
  server,
  perMessageDeflate: {
    threshold: 1024, // Only compress if payload > 1KB
  },
});
```

### JSON Performance

For high-throughput servers, use faster JSON serializers:

```bash
npm install fast-json-stringify ajv
```

```js
const fastJson = require('fast-json-stringify');

// Pre-compile schemas for frequently sent message types
const stringifyChatMsg = fastJson({
  type: 'object',
  properties: {
    type: { type: 'string' },
    text: { type: 'string' },
    username: { type: 'string' },
    timestamp: { type: 'integer' },
  },
});

// ~2-5x faster than JSON.stringify for this schema
ws.send(stringifyChatMsg({ type: 'chat', text: 'hi', username: 'alice', timestamp: Date.now() }));
```

### Measure Connection Stats

```js
setInterval(() => {
  let open = 0, closing = 0, closed = 0;
  wss.clients.forEach(ws => {
    if (ws.readyState === WebSocket.OPEN)    open++;
    if (ws.readyState === WebSocket.CLOSING) closing++;
    if (ws.readyState === WebSocket.CLOSED)  closed++;
  });

  console.log(`[Stats] Open: ${open} | Closing: ${closing} | Closed: ${closed} | Mem: ${
    Math.round(process.memoryUsage().heapUsed / 1024 / 1024)
  }MB`);
}, 10000);
```

---

## 25. Quick Reference Cheat Sheet

### Server (ws Library)

```js
const WebSocket = require('ws');
const wss = new WebSocket.Server({ port: 8080 });

wss.on('connection', (ws, req) => {
  ws.send('hello');                          // Send text
  ws.send(Buffer.from([1,2,3]));             // Send binary

  ws.on('message', (data, isBinary) => {});  // Receive
  ws.on('close', (code, reason) => {});      // Disconnected
  ws.on('error', (err) => {});               // Error

  ws.close(1000, 'Bye');                     // Close gracefully
  ws.terminate();                            // Force close

  ws.readyState; // 0=CONNECTING, 1=OPEN, 2=CLOSING, 3=CLOSED
  ws.ping();     // Send ping frame
});

// Broadcast to all
wss.clients.forEach(c => c.readyState === WebSocket.OPEN && c.send(msg));
```

### Browser Client

```js
const ws = new WebSocket('wss://example.com/ws');

ws.onopen    = (e) => {};                  // Connected
ws.onmessage = (e) => { e.data; };        // Message received
ws.onclose   = (e) => { e.code; };        // Disconnected
ws.onerror   = (e) => {};                 // Error

ws.send('text');                           // Send string
ws.send(JSON.stringify({ type: 'msg' })); // Send JSON
ws.send(new ArrayBuffer(8));              // Send binary

ws.close();                               // Close (code 1000)
ws.readyState;                            // 0–3
```

### Socket.IO Emit Cheatsheet

```js
// Server
socket.emit('ev', data);                  // → sender
socket.broadcast.emit('ev', data);        // → all except sender
io.emit('ev', data);                      // → everyone
io.to('room').emit('ev', data);           // → room
io.to(socketId).emit('ev', data);         // → specific client
socket.to('room').emit('ev', data);       // → room except sender

// Client
socket.emit('ev', data);
socket.on('ev', callback);
```

### WebSocket Close Codes

| Code | Meaning |
|---|---|
| 1000 | Normal closure |
| 1001 | Going away |
| 1002 | Protocol error |
| 1003 | Unsupported data |
| 1006 | Abnormal closure |
| 1007 | Invalid payload |
| 1008 | Policy violation |
| 1009 | Message too big |
| 1011 | Server error |

### Key npm Packages

| Package | Purpose |
|---|---|
| `ws` | Low-level WebSocket server/client |
| `socket.io` | Full-featured WS framework |
| `express-ws` | Add WS routes to Express |
| `@socket.io/redis-adapter` | Scale Socket.IO across instances |
| `ioredis` | Redis client for Node.js |
| `jsonwebtoken` | JWT auth |
| `wscat` | CLI tool to test WS servers |

---

## Further Reading

- [RFC 6455 — The WebSocket Protocol](https://tools.ietf.org/html/rfc6455)
- [MDN — WebSocket API](https://developer.mozilla.org/en-US/docs/Web/API/WebSocket)
- [ws Library GitHub](https://github.com/websockets/ws)
- [Socket.IO Documentation](https://socket.io/docs/v4/)
- [WebSocket Security (OWASP)](https://owasp.org/www-community/attacks/WebSocket_Attack)
- [High Performance Browser Networking — WebSocket chapter](https://hpbn.co/websocket/)

---

*Happy hacking! WebSockets open the door to a whole class of real-time applications — now go build something awesome.* 🚀