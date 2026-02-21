# 🛡️ Arcjet Security Guide

> **Arcjet** হলো একটি developer-first security layer যা আপনার Node.js / Express অ্যাপকে **Bot Detection**, **Rate Limiting**, এবং **Shield Protection** দিয়ে সুরক্ষিত রাখে।  
> মাত্র কয়েক লাইন কোডেই আপনার HTTP API এবং WebSocket server উভয়কে secure করা যায়।

---

## 📌 সূচিপত্র

- [Arcjet কী করে?](#-arcjet-কী-করে)
- [Installation](#-installation)
- [Environment Setup](#-environment-setup)
- [Project Structure](#-project-structure)
- [arcjet.js — Core Configuration](#-arcjetjs--core-configuration)
- [server.js — Express Integration](#-serverjs--express-integration)
- [ws/server.js — WebSocket Integration](#-wsserverjs--websocket-integration)
- [Rules বিস্তারিত](#-rules-বিস্তারিত)
- [Terminal-এ Test করো](#-terminal-এ-test-করো)
- [DRY_RUN vs LIVE Mode](#-dry_run-vs-live-mode)
- [Error Codes Reference](#-error-codes-reference)

---

## 🔐 Arcjet কী করে?

```
Incoming Request (HTTP / WebSocket)
          ↓
    ┌─────────────┐
    │   Arcjet    │
    │  Security   │
    │   Layer     │
    └──────┬──────┘
           │
    ┌──────▼──────────────────────────┐
    │  🛡️  Shield      — Injection/XSS attack block  │
    │  🤖  detectBot   — Bot & crawler detect        │
    │  ⏱️  slidingWindow — Rate limiting             │
    └──────┬──────────────────────────┘
           │
    ✅ Allow → আপনার App
    ❌ Deny  → 429 / 403 Response
```

---

## 📦 Installation

```bash
# npm
npm install @arcjet/node

# yarn
yarn add @arcjet/node

# pnpm
pnpm add @arcjet/node
```

> Arcjet ব্যবহার করতে একটি **API Key** লাগবে।  
> Free account: [https://arcjet.com](https://arcjet.com) → Dashboard → New Site → API Key কপি করো।

---

## ⚙️ Environment Setup

Project root-এ `.env` ফাইল তৈরি করো:

```env
# Arcjet API Key — arcjet.com থেকে নাও
ARCJET_KEY=ajkey_your_actual_key_here

# Mode: DRY_RUN (শুধু log করে, block করে না) | LIVE (সত্যিই block করে)
ARCJET_ENV=DRY_RUN
```

`.env.example` ফাইলও রাখো (team-এর জন্য):

```env
ARCJET_KEY=
ARCJET_ENV=DRY_RUN
```

> ⚠️ `.env` ফাইল কখনো GitHub-এ push করো না। `.gitignore`-এ add করো:

```bash
echo ".env" >> .gitignore
```

---

## 🗂️ Project Structure

```
your-project/
├── src/
│   ├── arcjet.js          ← Arcjet config & middleware
│   ├── server.js          ← Express + HTTP server
│   └── ws/
│       └── server.js      ← WebSocket server
├── .env
├── .env.example
└── package.json
```

---

## 🔧 arcjet.js — Core Configuration

```js
import arcjet, { detectBot, shield, slidingWindow } from '@arcjet/node';

const arcjetKey = process.env.ARCJET_KEY;
const arcjetMode = process.env.ARCJET_ENV === 'DRY_RUN' ? 'DRY_RUN' : 'LIVE';

// Key না থাকলে server চালুই হবে না
if (!arcjetKey) throw new Error('ARCJET_KEY environment variable is missing.');

// ─── HTTP API-এর জন্য Arcjet Instance ───────────────────────────────────────
export const httpArcjet = arcjetKey
  ? arcjet({
      key: arcjetKey,
      rules: [
        // সব ধরনের injection / XSS attack থেকে রক্ষা
        shield({ mode: arcjetMode }),

        // Bot detect করো, কিন্তু Google/Bing-কে allow করো
        detectBot({
          mode: arcjetMode,
          allow: ['CATEGORY:SEARCH_ENGINE', 'CATEGORY:PREVIEW'],
        }),

        // প্রতি 10 সেকেন্ডে সর্বোচ্চ 50 request
        slidingWindow({ mode: arcjetMode, interval: '10s', max: 50 }),
      ],
    })
  : null;

// ─── WebSocket-এর জন্য Arcjet Instance ──────────────────────────────────────
export const wsArcjet = arcjetKey
  ? arcjet({
      key: arcjetKey,
      rules: [
        shield({ mode: arcjetMode }),

        detectBot({
          mode: arcjetMode,
          allow: ['CATEGORY:SEARCH_ENGINE', 'CATEGORY:PREVIEW'],
        }),

        // WS-এ tight limit — প্রতি 2 সেকেন্ডে সর্বোচ্চ 5 connection
        slidingWindow({ mode: arcjetMode, interval: '2s', max: 5 }),
      ],
    })
  : null;

// ─── Express Middleware ──────────────────────────────────────────────────────
export function securityMiddleware() {
  return async (req, res, next) => {
    // Arcjet key না থাকলে middleware skip করো
    if (!httpArcjet) return next();

    try {
      const decision = await httpArcjet.protect(req);

      if (decision.isDenied()) {
        // Rate limit হলে 429
        if (decision.reason.isRateLimit()) {
          return res.status(429).json({ error: 'Too many requests.' });
        }

        // Bot বা shield block হলে 403
        return res.status(403).json({ error: 'Forbidden.' });
      }
    } catch (e) {
      console.error('Arcjet middleware error', e);
      return res.status(503).json({ error: 'Service Unavailable' });
    }

    next();
  };
}
```

### কোন rule কী করে?

| Rule            | কাজ                                          | HTTP Limit   | WS Limit    |
| --------------- | -------------------------------------------- | ------------ | ----------- |
| `shield`        | SQL injection, XSS, path traversal block করে | ✅           | ✅          |
| `detectBot`     | Automated bot/scraper block করে              | ✅           | ✅          |
| `slidingWindow` | Time-based rate limiting                     | 50 req / 10s | 5 conn / 2s |

---

## 🚀 server.js — Express Integration

```js
import express from 'express';
import http from 'http';
import { matchRouter } from './routes/matches.js';
import { attachWebSocketServer } from './ws/server.js';
import { securityMiddleware } from './arcjet.js';

const PORT = Number(process.env.PORT || 8000);
const HOST = process.env.HOST || '0.0.0.0';

const app = express();
const server = http.createServer(app);

app.use(express.json());

// Health check — security middleware-এর আগে (unprotected)
app.get('/', (req, res) => {
  res.send('Hello from Express server!');
});

// ✅ Arcjet security middleware — এর পরের সব route protected
app.use(securityMiddleware());

// Protected routes
app.use('/matches', matchRouter);

// WebSocket server attach করো
const { broadcastMatchCreated } = attachWebSocketServer(server);
app.locals.broadcastMatchCreated = broadcastMatchCreated;

server.listen(PORT, HOST, () => {
  const baseUrl =
    HOST === '0.0.0.0' ? `http://localhost:${PORT}` : `http://${HOST}:${PORT}`;

  console.log(`Server is running on ${baseUrl}`);
  console.log(
    `WebSocket Server is running on ${baseUrl.replace('http', 'ws')}/ws`,
  );
});
```

> **গুরুত্বপূর্ণ:** `app.use(securityMiddleware())` এর **উপরে** রাখা routes গুলো protected হবে না।  
> `/` health check route উদ্দেশ্যমূলকভাবে middleware-এর আগে রাখা হয়েছে।

---

## 🔌 ws/server.js — WebSocket Integration

```js
import { WebSocket, WebSocketServer } from 'ws';
import { wsArcjet } from '../arcjet.js';

// Helper: JSON পাঠাও (connection open থাকলে)
function sendJson(socket, payload) {
  if (socket.readyState !== WebSocket.OPEN) return;
  socket.send(JSON.stringify(payload));
}

// Helper: সব connected client-এ broadcast করো
function broadcast(wss, payload) {
  for (const client of wss.clients) {
    if (client.readyState !== WebSocket.OPEN) continue;
    client.send(JSON.stringify(payload));
  }
}

export function attachWebSocketServer(server) {
  const wss = new WebSocketServer({
    server,
    path: '/ws',
    maxPayload: 1024 * 1024, // 1MB max message size
  });

  wss.on('connection', async (socket, req) => {
    // ─── Arcjet WebSocket Protection ─────────────────────────────────────
    if (wsArcjet) {
      try {
        const decision = await wsArcjet.protect(req);

        if (decision.isDenied()) {
          // WebSocket close codes:
          // 1013 = Try Again Later (rate limit)
          // 1008 = Policy Violation (bot / shield)
          const code = decision.reason.isRateLimit() ? 1013 : 1008;
          const reason = decision.reason.isRateLimit()
            ? 'Rate limit exceeded'
            : 'Access denied';

          socket.close(code, reason);
          return;
        }
      } catch (e) {
        console.error('WS connection error', e);
        socket.close(1011, 'Server security error');
        return;
      }
    }

    // ─── Heartbeat (ping/pong) ────────────────────────────────────────────
    // Dead connection detect করতে ping পাঠানো হয়
    socket.isAlive = true;
    socket.on('pong', () => {
      socket.isAlive = true;
    });

    // Welcome message পাঠাও
    sendJson(socket, { type: 'welcome' });

    socket.on('error', console.error);
  });

  // প্রতি 30 সেকেন্ডে dead connection check ও clean করো
  const interval = setInterval(() => {
    wss.clients.forEach((ws) => {
      if (ws.isAlive === false) return ws.terminate(); // dead → terminate

      ws.isAlive = false;
      ws.ping(); // pong না আসলে পরের cycle-এ terminate হবে
    });
  }, 30000);

  wss.on('close', () => clearInterval(interval));

  // নতুন match তৈরি হলে সব client-কে জানাও
  function broadcastMatchCreated(match) {
    broadcast(wss, { type: 'match_created', data: match });
  }

  return { broadcastMatchCreated };
}
```

### WebSocket Close Codes

| Code   | নাম              | কখন ব্যবহার হয়              |
| ------ | ---------------- | ---------------------------- |
| `1008` | Policy Violation | Bot detect / Shield block    |
| `1011` | Internal Error   | Server-side unexpected error |
| `1013` | Try Again Later  | Rate limit exceeded          |

---

## 🧪 Terminal-এ Test করো

### ১. WebSocket Connection Test (wscat)

```bash
# wscat install না থাকলে
npm install -g wscat

# WebSocket-এ connect করো
wscat -c ws://localhost:8000/ws
```

সফল হলে এরকম দেখাবে:

```
Connected (press CTRL+C to quit)
< {"type":"welcome"}
>
```

Rate limit-এ hit করলে:

```
Disconnected (code: 1013, reason: "Rate limit exceeded")
```

---

### ২. HTTP Rate Limit Test — Bash Loop

> প্রতি 10 সেকেন্ডে **max 50 request** সেট করা আছে। নিচের command 60টি request পাঠাবে এবং কোনগুলো block হয় দেখাবে।

```bash
# comment this line then test

# detectBot({
#           mode: arcjetMode,
#           allow: ['CATEGORY:SEARCH_ENGINE', 'CATEGORY:PREVIEW'],
#         }),
✅✅
for i in {1..60}; do curl -s -o /dev/null -w "%{http_code}\n" http://localhost:8000/matches; done
```

**Expected Output:**

```
200   ← ✅ Normal request
200
200
...
200   ← 50 পর্যন্ত সব 200
429   ← ❌ Rate limit hit! (51তম request থেকে)
429
429
...
429
```

### test web socket in browser

```bash
# open http://localhost:8000
# past this in console

for (let i = 0; i < 10; i++) {
  const ws = new WebSocket("ws://localhost:8000/ws");
  ws.onopen = () => console.log(`Socket ${i} opened`);
  ws.onclose = (e) => console.log(`Socket ${i} closed: ${e.code} ${e.reason}`);
}

# output

# Socket 0 opened
# VM38:3 Socket 1 opened
# VM38:3 Socket 2 opened
# VM38:3 Socket 3 opened
# VM38:3 Socket 4 opened
# VM38:3 Socket 5 opened
# VM38:3 Socket 6 opened
# VM38:3 Socket 7 opened
# VM38:3 Socket 8 opened
# VM38:3 Socket 9 opened
# VM38:4 Socket 5 closed: 1013 Rate limit exceeded
# VM38:4 Socket 7 closed: 1013 Rate limit exceeded
# VM38:4 Socket 6 closed: 1013 Rate limit exceeded
# VM38:4 Socket 8 closed: 1013 Rate limit exceeded
# VM38:4 Socket 9 closed: 1013 Rate limit exceeded
```

---

### ৩. আরও বিস্তারিত Output দেখতে

```bash
# Response time ও status একসাথে দেখো
for i in {1..60}; do
  curl -s -o /dev/null -w "Request $i: %{http_code} (time: %{time_total}s)\n" \
  http://localhost:8000/matches
done
```

```
Request 1:  200 (time: 0.012s)
Request 2:  200 (time: 0.008s)
...
Request 50: 200 (time: 0.009s)
Request 51: 429 (time: 0.003s)   ← Arcjet block করেছে (দ্রুত respond করে)
Request 52: 429 (time: 0.002s)
```

---

### ৪. Concurrent Request Test (parallel)

```bash
# ৬০টি request একসাথে পাঠাও (parallel)
for i in {1..60}; do
  curl -s -o /dev/null -w "%{http_code}\n" http://localhost:8000/matches &
done
wait
```

---

### ৫. WebSocket Rapid Connection Test

```bash
# দ্রুত ৫টির বেশি WS connection চেষ্টা করো (2s-এ max 5 সেট আছে)
for i in {1..8}; do
  wscat -c ws://localhost:8000/ws --wait 1 &
done
```

শেষের connections এরকম দেখাবে:

```
Connected ...
< {"type":"welcome"}
Disconnected (code: 1013, reason: "Rate limit exceeded")
```

---

## 🔄 DRY_RUN vs LIVE Mode

| Mode      | আচরণ                                                  | কখন ব্যবহার করবো      |
| --------- | ----------------------------------------------------- | --------------------- |
| `DRY_RUN` | Request block করে না, শুধু Arcjet Dashboard-এ log করে | Development / Testing |
| `LIVE`    | সত্যিই block করে (429/403 পাঠায়)                     | Production            |

**.env পরিবর্তন করে switch করো:**

```env
# Testing-এ
ARCJET_ENV=DRY_RUN

# Production-এ
ARCJET_ENV=LIVE
```

> 💡 **Best Practice:** সবসময় `DRY_RUN` দিয়ে শুরু করো। Arcjet Dashboard-এ দেখো কোন requests block হচ্ছে, তারপর `LIVE` করো।

---

## 📋 Error Codes Reference

### HTTP Errors

| Status Code               | কারণ                         | Arcjet Rule            |
| ------------------------- | ---------------------------- | ---------------------- |
| `403 Forbidden`           | Bot detect অথবা Shield block | `detectBot` / `shield` |
| `429 Too Many Requests`   | Rate limit exceed            | `slidingWindow`        |
| `503 Service Unavailable` | Arcjet নিজেই error করলে      | Internal error         |

### WebSocket Close Codes

| Code                    | কারণ               | Arcjet Rule            |
| ----------------------- | ------------------ | ---------------------- |
| `1008 Policy Violation` | Bot / Shield block | `detectBot` / `shield` |
| `1011 Internal Error`   | Arcjet-এ exception | Error handling         |
| `1013 Try Again Later`  | Rate limit exceed  | `slidingWindow`        |

---

## 🎯 Interview Ready Answer

> **"Arcjet হলো একটি developer-first security middleware যা Node.js অ্যাপে Shield Protection, Bot Detection এবং Rate Limiting একসাথে প্রদান করে।**
>
> **HTTP API-তে Express middleware হিসেবে এবং WebSocket connection-এ `wsArcjet.protect(req)` দিয়ে integrate করা যায়। DRY_RUN mode-এ log করে এবং LIVE mode-এ সত্যিকারের block করে।"**

---

## 📦 Quick Reference

```bash
# Install
npm install @arcjet/node

# .env
ARCJET_KEY=ajkey_xxxxx
ARCJET_ENV=DRY_RUN

# Rate limit test
for i in {1..60}; do curl -s -o /dev/null -w "%{http_code}\n" http://localhost:8000/matches; done

# WebSocket test
wscat -c ws://localhost:8000/ws
```

---

_Made with ❤️ | Happy Coding!_
