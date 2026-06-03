# Chat Notifications App — Full Technical Documentation

A real-time chat application built to test and demonstrate notification systems,
modeled after WhatsApp / Messenger style instant messaging.

---

## Table of Contents

1. [Tech Stack](#1-tech-stack)
2. [System Architecture](#2-system-architecture)
3. [Project Structure](#3-project-structure)
4. [Database Schema](#4-database-schema)
5. [Authentication Flow](#5-authentication-flow)
6. [Real-Time Communication — Socket.io Events](#6-real-time-communication--socketio-events)
7. [REST API Reference](#7-rest-api-reference)
8. [Notification System](#8-notification-system)
9. [Environment Variables](#9-environment-variables)
10. [Setup & Installation Guide](#10-setup--installation-guide)
11. [How to Test Notifications](#11-how-to-test-notifications)
12. [Folder & File Explanations](#12-folder--file-explanations)

---

## 1. Tech Stack

### Frontend
| Technology | Version | Purpose |
|---|---|---|
| **Next.js** | 14.x | Full-stack React framework (pages + API routes) |
| **React** | 18.x | UI component library |
| **Tailwind CSS** | 3.x | Utility-first CSS styling |
| **Socket.io-client** | 4.x | WebSocket client for real-time events |

### Backend
| Technology | Version | Purpose |
|---|---|---|
| **Node.js** | 18+ | JavaScript runtime |
| **Next.js API Routes** | 14.x | REST API endpoints (runs inside Next.js) |
| **Socket.io** | 4.x | WebSocket server for real-time bidirectional communication |
| **Custom HTTP Server** | — | `server.js` wraps Next.js to attach Socket.io on same port |

### Database
| Technology | Version | Purpose |
|---|---|---|
| **PostgreSQL** | 14+ | Relational database — stores users and messages |
| **Prisma ORM** | 5.x | Database client, schema management, migrations |

### Authentication
| Technology | Purpose |
|---|---|
| **bcryptjs** | Password hashing (never store plain text passwords) |
| **jsonwebtoken (JWT)** | Stateless auth tokens — user stays logged in |
| **HTTP-only Cookies** | Secure token storage — not accessible from JavaScript |

### Language
- **JavaScript (JSX)** — No TypeScript, plain JS throughout

---

## 2. System Architecture

```
┌─────────────────────────────────────────────────────────────────┐
│                        CLIENT (Browser)                          │
│                                                                   │
│   React Components (JSX)                                         │
│   ┌──────────────┐  ┌──────────────┐  ┌─────────────────────┐  │
│   │  UserList    │  │  ChatWindow  │  │  NotificationToast  │  │
│   └──────────────┘  └──────────────┘  └─────────────────────┘  │
│           │                │                       │             │
│           └────────────────┴───────────────────────┘             │
│                            │                                      │
│                    chat/page.jsx (main controller)                │
│                        │           │                             │
│               REST API (fetch)   WebSocket (socket.io-client)    │
└───────────────────┬───────────────┬──────────────────────────────┘
                    │               │
                    │  HTTP / WS    │   (same port 3000)
                    │               │
┌───────────────────┴───────────────┴──────────────────────────────┐
│                       SERVER (Node.js)                            │
│                                                                   │
│   server.js  ──  HTTP Server  ──  Socket.io Server               │
│                        │                  │                       │
│               Next.js Handler      Real-time Events              │
│                        │                                          │
│              ┌─────────┴──────────┐                              │
│              │   Next.js App      │                              │
│              │  ┌──────────────┐  │                              │
│              │  │  API Routes  │  │  /api/auth/*                 │
│              │  │  (server-    │  │  /api/messages               │
│              │  │   side JS)   │  │  /api/users                  │
│              │  └──────┬───────┘  │                              │
│              └─────────┼──────────┘                              │
│                        │                                          │
│                   Prisma ORM                                      │
│                        │                                          │
└────────────────────────┼──────────────────────────────────────────┘
                         │
┌────────────────────────┴──────────────────────────────────────────┐
│                     PostgreSQL Database                            │
│                                                                    │
│   Table: User          Table: Message                             │
│   ─────────────        ──────────────────                         │
│   id (PK)              id (PK)                                    │
│   username             content                                    │
│   email                senderId (FK → User)                       │
│   password             receiverId (FK → User)                     │
│   createdAt            read (boolean)                             │
│   updatedAt            createdAt                                  │
└───────────────────────────────────────────────────────────────────┘
```

### Why This Architecture?

- **Same port for HTTP + WebSocket:** Socket.io attaches to the Node.js HTTP server, so both REST API calls and WebSocket connections go through port 3000. This is how production chat apps like WhatsApp Web work.
- **Next.js API Routes for REST:** User registration, login, fetching messages — these don't need real-time. REST is simpler and cacheable.
- **Socket.io for real-time:** Sending messages, typing indicators, online status — these need instant delivery without polling.

---

## 3. Project Structure

```
NextJS_Notifications/
│
├── server.js                    ← Custom Node.js server (Socket.io + Next.js)
├── next.config.js               ← Next.js configuration
├── tailwind.config.js           ← Tailwind CSS configuration
├── postcss.config.mjs           ← PostCSS (required by Tailwind)
├── jsconfig.json                ← Path aliases (@/* → src/*)
├── package.json                 ← Dependencies and npm scripts
├── .env                         ← Your local secrets (never commit this)
├── .env.example                 ← Template for .env
├── .gitignore
│
├── prisma/
│   └── schema.prisma            ← Database schema (tables, relations)
│
└── src/
    ├── lib/
    │   ├── prisma.js            ← Prisma client singleton (DB connection)
    │   └── auth.js              ← JWT sign/verify + getCurrentUser()
    │
    ├── app/                     ← Next.js App Router
    │   ├── globals.css          ← Global styles + Tailwind directives
    │   ├── layout.jsx           ← Root HTML layout (wraps every page)
    │   ├── page.jsx             ← Root route "/" → redirects to /chat or /login
    │   │
    │   ├── (auth)/              ← Route group (no layout change, just grouping)
    │   │   ├── login/
    │   │   │   └── page.jsx     ← Login form
    │   │   └── register/
    │   │       └── page.jsx     ← Registration form
    │   │
    │   ├── chat/
    │   │   └── page.jsx         ← Main chat UI (all real-time logic lives here)
    │   │
    │   └── api/                 ← REST API (server-side, runs in Node.js)
    │       ├── auth/
    │       │   ├── register/route.js   ← POST /api/auth/register
    │       │   ├── login/route.js      ← POST /api/auth/login
    │       │   ├── logout/route.js     ← POST /api/auth/logout
    │       │   └── me/route.js         ← GET /api/auth/me
    │       ├── messages/
    │       │   └── route.js            ← GET / POST / PATCH /api/messages
    │       └── users/
    │           └── route.js            ← GET /api/users
    │
    └── components/
        ├── UserList.jsx         ← Sidebar list of users with badges
        ├── ChatWindow.jsx       ← Message area + input bar
        └── NotificationToast.jsx ← Slide-in notification popups
```

---

## 4. Database Schema

Defined in `prisma/schema.prisma`. Prisma reads this file and creates/updates the actual PostgreSQL tables.

### User Table

```
id          String    Primary key, auto-generated (cuid format)
username    String    Unique — displayed in UI
email       String    Unique — used for login
password    String    Bcrypt hash (never plain text)
createdAt   DateTime  Auto-set on creation
updatedAt   DateTime  Auto-updated on any change
```

### Message Table

```
id          String    Primary key, auto-generated
content     String    The message text
senderId    String    Foreign key → User.id (who sent it)
receiverId  String    Foreign key → User.id (who receives it)
read        Boolean   false = unread, true = read (default: false)
createdAt   DateTime  When the message was sent
```

### Relations

- One User can send many Messages (`sentMessages`)
- One User can receive many Messages (`receivedMessages`)
- Deleting a User cascades — their messages are also deleted

### Indexes (for performance)

```
@@index([senderId, receiverId])   ← Fast conversation lookup
@@index([receiverId, read])       ← Fast unread count lookup
```

---

## 5. Authentication Flow

### Registration

```
User fills form → POST /api/auth/register
  → Validate: username, email, password present
  → Check: username and email not already taken (DB query)
  → bcrypt.hash(password, 10) — creates secure hash
  → prisma.user.create() — saves to DB
  → Return: user object (no password)
  → Redirect to /login
```

### Login

```
User fills form → POST /api/auth/login
  → Find user by email in DB
  → bcrypt.compare(inputPassword, storedHash)
  → If match: jwt.sign({ userId, username, email }, JWT_SECRET, { expiresIn: '7d' })
  → Set cookie: auth_token = <JWT>
      httpOnly: true       ← JS cannot read this cookie (XSS protection)
      secure: true (prod)  ← HTTPS only in production
      sameSite: lax        ← CSRF protection
      maxAge: 7 days
  → Return: user object
  → Redirect to /chat
```

### Authenticated Requests

```
Every API route that needs auth calls: getCurrentUser()
  → reads cookies() from request headers
  → gets auth_token cookie value
  → jwt.verify(token, JWT_SECRET)
  → returns { userId, username, email } or null
  → If null → return 401 Unauthorized
```

### Logout

```
POST /api/auth/logout
  → response.cookies.delete('auth_token')
  → Cookie is cleared, user must log in again
```

---

## 6. Real-Time Communication — Socket.io Events

Socket.io provides bidirectional, event-based communication between browser and server.

### How the Connection Works

```
1. Chat page loads → io(window.location.origin) called
2. Browser connects via WebSocket (falls back to polling if blocked)
3. On connect → client emits "user_connected" with their userId
4. Server maps: userId → socketId in a Map()
5. Server emits "online_users" to ALL clients with current online list
```

### Server-Side Event Handlers (`server.js`)

```
Server tracks: onlineUsers = Map { userId → socketId }
```

| Event Received | From | What Server Does |
|---|---|---|
| `user_connected(userId)` | Client | Store userId→socketId in Map, emit `online_users` to all |
| `send_message({ message })` | Client | Forward `receive_message` to receiver's socket, send `new_notification` to receiver, confirm `message_delivered` to sender |
| `typing({ receiverId, senderName, isTyping })` | Client | Forward `user_typing` to receiver's socket |
| `messages_read({ senderId })` | Client | Send `messages_read_ack` to sender's socket |
| `disconnect` | Socket.io | Remove user from Map, emit updated `online_users` to all |

### Client-Side Events (in `chat/page.jsx`)

| Event Emitted | When | Payload |
|---|---|---|
| `user_connected` | On socket connect | `userId` |
| `send_message` | After message saved to DB | `{ message: { id, content, senderId, receiverId, senderName, ... } }` |
| `typing` | While typing (throttled 2s) | `{ receiverId, senderName, isTyping: true/false }` |
| `messages_read` | When user opens a chat | `{ senderId }` |

| Event Received | What Client Does |
|---|---|
| `online_users` | Update green dot on each user in sidebar |
| `receive_message` | If viewing that chat → add to messages list. If not → increment unread badge |
| `new_notification` | Show toast, play beep, OS notification, flash page title |
| `user_typing` | Show/hide typing dots in chat header |
| `messages_read_ack` | Update tick marks from single (sent) to double (read) |
| `message_delivered` | Confirm message reached server |

### Online Users Map

```javascript
// In server.js — in-memory (resets on server restart)
const onlineUsers = new Map()
// Example: { "clxyz123": "socket_abc", "clxyz456": "socket_def" }

// When user connects:
onlineUsers.set(userId, socket.id)

// When user disconnects:
onlineUsers.delete(userId)

// To send to a specific user:
const socketId = onlineUsers.get(receiverId)
io.to(socketId).emit('event', data)
```

---

## 7. REST API Reference

Base URL: `http://localhost:3000/api`

All routes returning user data never include the password field.

---

### Auth Routes

#### `POST /api/auth/register`

Create a new user account.

**Request Body:**
```json
{
  "username": "johndoe",
  "email": "john@example.com",
  "password": "mypassword"
}
```

**Success Response (201):**
```json
{
  "user": { "id": "clxyz", "username": "johndoe", "email": "john@example.com", "createdAt": "..." },
  "message": "Account created successfully"
}
```

**Error Responses:**
- `400` — Missing fields or password < 6 chars
- `409` — Email or username already taken
- `500` — Server error

---

#### `POST /api/auth/login`

Login with email and password.

**Request Body:**
```json
{
  "email": "john@example.com",
  "password": "mypassword"
}
```

**Success Response (200):**
Sets `auth_token` cookie. Returns:
```json
{
  "user": { "id": "clxyz", "username": "johndoe", "email": "john@example.com", "createdAt": "..." }
}
```

**Error Responses:**
- `400` — Missing fields
- `401` — Wrong email or password

---

#### `POST /api/auth/logout`

Clear the auth cookie.

**Success Response (200):**
```json
{ "message": "Logged out" }
```

---

#### `GET /api/auth/me`

Get the currently logged-in user (requires auth cookie).

**Success Response (200):**
```json
{
  "user": { "id": "clxyz", "username": "johndoe", "email": "john@example.com", "createdAt": "..." }
}
```

**Error:** `401` if not logged in.

---

### Users Route

#### `GET /api/users`

Get all users except yourself, with unread counts and last message preview.

**Requires:** Auth cookie

**Success Response (200):**
```json
{
  "users": [
    {
      "id": "clxyz456",
      "username": "janedoe",
      "email": "jane@example.com",
      "createdAt": "...",
      "unreadCount": 3,
      "isOnline": false,
      "lastMessage": "Hey how are you?",
      "lastMessageAt": "2024-01-15T10:30:00Z"
    }
  ]
}
```

Note: `isOnline` is always `false` from the API — the client updates it in real-time from the `online_users` socket event.

---

### Messages Routes

#### `GET /api/messages?userId=<id>`

Fetch all messages in a conversation between you and `userId`, ordered oldest first.

**Requires:** Auth cookie + `userId` query param

**Success Response (200):**
```json
{
  "messages": [
    {
      "id": "msg_abc",
      "content": "Hello!",
      "senderId": "clxyz123",
      "receiverId": "clxyz456",
      "read": true,
      "createdAt": "2024-01-15T10:00:00Z",
      "sender": { "id": "clxyz123", "username": "johndoe" }
    }
  ]
}
```

---

#### `POST /api/messages`

Send a new message. Saves to DB — then client emits via socket for real-time delivery.

**Requires:** Auth cookie

**Request Body:**
```json
{
  "content": "Hello!",
  "receiverId": "clxyz456"
}
```

**Success Response (201):**
```json
{
  "message": { "id": "msg_abc", "content": "Hello!", "senderId": "...", "receiverId": "...", "read": false, "createdAt": "...", "sender": { ... } }
}
```

---

#### `PATCH /api/messages`

Mark all unread messages from a sender as read.

**Requires:** Auth cookie

**Request Body:**
```json
{
  "senderId": "clxyz456"
}
```

**Success Response (200):**
```json
{ "success": true }
```

---

## 8. Notification System

When a user receives a message, up to 5 notification types fire simultaneously:

### 1. In-App Toast Notification

- Slides in from the right side of the screen
- Shows sender's name (first letter as avatar) + message preview
- Auto-dismisses after **5 seconds** with a shrinking blue progress bar
- Click it → opens that chat directly
- X button → manual dismiss
- Up to 5 toasts stack at once

**Code location:** `src/components/NotificationToast.jsx`

### 2. Browser OS Notification (Web Notifications API)

- Only fires when the **browser tab is not active** (minimised or in background)
- Shows as a native Windows/Mac/Android notification
- Requires user permission — permission is requested when the chat page first loads
- Click it → brings the tab to focus

**Requires:** User must click "Allow" on the browser permission popup

```javascript
// Request permission (runs once on chat page load)
Notification.requestPermission()

// Show notification
new Notification('New message from John', { body: 'Hey how are you?' })
```

### 3. Notification Sound (Web Audio API)

- Plays a short 880Hz sine wave beep (0.3 seconds)
- Generated programmatically — no audio file needed
- Works in all modern browsers

```javascript
// Generated with Web Audio API, no MP3/WAV file needed
const ctx = new AudioContext()
const osc = ctx.createOscillator()
osc.frequency.value = 880
// ... fades out over 0.3 seconds
```

### 4. Unread Badge Counter

- Each user in the sidebar shows a blue badge with unread count
- Increments for every message received while NOT viewing that chat
- Resets to 0 when you click that user to open the chat
- Total unread across all users shown in sidebar header

### 5. Page Title Flash

- Document title changes to `(New!) Chat Notifications Test` for 3 seconds
- Useful when the tab is open but the user is looking elsewhere
- Reverts automatically

---

## 9. Environment Variables

File: `.env` (copy from `.env.example`)

| Variable | Required | Example | Description |
|---|---|---|---|
| `DATABASE_URL` | Yes | `postgresql://postgres:pass@localhost:5432/chat_notifications` | PostgreSQL connection string |
| `JWT_SECRET` | Yes | `my-super-secret-key-123` | Secret for signing JWT tokens. Use a long random string in production. |
| `PORT` | No | `3000` | Port the server listens on (default: 3000) |
| `NODE_ENV` | No | `development` | Set to `production` for production builds |

**DATABASE_URL format:**
```
postgresql://USERNAME:PASSWORD@HOST:PORT/DATABASE_NAME
```

---

## 10. Setup & Installation Guide

### Prerequisites

- Node.js 18 or higher → https://nodejs.org
- PostgreSQL 14 or higher → https://www.postgresql.org/download/
- npm (comes with Node.js)

### Step 1 — Clone / Download the project

```bash
cd your-folder
# project files should be here
```

### Step 2 — Install dependencies

```bash
npm install
```

This installs: Next.js, React, Socket.io, Prisma, bcryptjs, jsonwebtoken, Tailwind CSS.

### Step 3 — Create the database in PostgreSQL

Open your PostgreSQL client (psql or pgAdmin) and run:

```sql
CREATE DATABASE chat_notifications;
```

### Step 4 — Configure environment variables

```bash
copy .env.example .env
```

Open `.env` and fill in:
```
DATABASE_URL="postgresql://postgres:YOUR_PASSWORD@localhost:5432/chat_notifications"
JWT_SECRET="any-long-random-string-here"
```

### Step 5 — Push database schema

This creates the `User` and `Message` tables in your database:

```bash
npm run db:push
```

Expected output:
```
✔ Your database is now in sync with your Prisma schema.
```

### Step 6 — Generate Prisma client

```bash
npm run db:generate
```

### Step 7 — Start the development server

```bash
npm run dev
```

Expected output:
```
> Local:   http://localhost:3000
> Network: http://YOUR_IP:3000
> Socket.io attached on the same port
> Environment: development
```

### Step 8 — Open the app

Visit: `http://localhost:3000`

You will be redirected to `/login`. Register a user first.

---

### Available npm Scripts

| Script | Command | What it does |
|---|---|---|
| `npm run dev` | `node server.js` | Start development server |
| `npm run build` | `next build` | Build for production |
| `npm start` | `NODE_ENV=production node server.js` | Run production build |
| `npm run db:push` | `npx prisma db push` | Sync schema to database |
| `npm run db:generate` | `npx prisma generate` | Regenerate Prisma client |
| `npm run db:studio` | `npx prisma studio` | Open Prisma visual database browser |
| `npm run lint` | `next lint` | Run ESLint |

---

## 11. How to Test Notifications

### Basic Test (2 browser tabs)

1. Open `http://localhost:3000` in **Tab 1**
2. Register as **User A** (e.g., username: alice, email: alice@test.com)
3. Login as User A
4. Open `http://localhost:3000` in a **new Incognito window**
5. Register as **User B** (e.g., username: bob, email: bob@test.com)
6. Login as User B
7. In User A's window → click User B in the sidebar → send a message

**What you should see in User B's window:**
- Toast notification slides in from the right
- Beep sound plays
- Unread badge appears on User A in the sidebar

### Testing OS Browser Notifications

1. After logging in, the browser will ask for notification permission — click **Allow**
2. Minimise the browser tab
3. Have another user send you a message
4. You should see a Windows/system notification pop up

### Testing from Another Device (same Wi-Fi)

1. Run `npm run dev` on your machine
2. Find your IP: run `ipconfig` in terminal (look for IPv4 address)
3. Open firewall for port 3000 (run as Administrator):
   ```
   netsh advfirewall firewall add rule name="Chat App 3000" protocol=TCP dir=in localport=3000 action=allow
   ```
4. On other device: open `http://YOUR_IP:3000`
5. Register a different user on each device and chat

### Testing with ngrok (public internet)

```bash
# Install ngrok from https://ngrok.com
ngrok http 3000
# Copy the https://xxxx.ngrok.io URL
# Share with anyone — works from anywhere
```

---

## 12. Folder & File Explanations

### `server.js`

The entry point. Instead of using `next dev` or `next start`, we run `node server.js` which:
1. Creates a standard Node.js HTTP server
2. Attaches Socket.io to that HTTP server
3. Tells Next.js to handle all regular HTTP requests

This is necessary because Next.js doesn't expose its internal HTTP server, so we create our own and give it to both Next.js and Socket.io.

### `src/lib/prisma.js`

Exports a singleton Prisma client. In Next.js development mode, the code hot-reloads frequently. Without the singleton pattern, each reload would create a new database connection, quickly exhausting PostgreSQL's connection limit. The `globalThis` trick prevents this.

### `src/lib/auth.js`

Three functions:
- `signToken(payload)` — creates a JWT string
- `verifyToken(token)` — validates and decodes a JWT
- `getCurrentUser()` — reads the `auth_token` cookie from the current request and returns the decoded user, or null

### `src/app/chat/page.jsx`

The most important file — the entire real-time chat logic lives here:
- Fetches current user on mount
- Fetches user list
- Initialises Socket.io connection
- Handles all socket events (receive_message, new_notification, user_typing, messages_read_ack)
- Manages state: messages, users, toasts, typingUsers, onlineUsers
- Uses `useRef` for values that socket event handlers need access to without stale closures (selectedUserId, currentUser)

### `src/components/ChatWindow.jsx`

Handles the right panel:
- Renders message bubbles (blue = yours, grey = theirs)
- Single tick SVG = sent, double tick SVG = read
- Textarea with Enter-to-send, Shift+Enter for newline
- Typing indicator: emits `typing` event with 2s debounce
- Bouncing dots animation when the other user is typing

### `src/components/UserList.jsx`

The left sidebar list:
- Shows avatar (first 2 letters of username)
- Green dot = online (updated via socket)
- Blue badge = unread message count
- Last message preview
- Highlights the currently selected conversation

### `src/components/NotificationToast.jsx`

Stack of toast notifications in the top-right corner:
- Each toast animates in using CSS translate + opacity transition
- Auto-dismisses after 5 seconds
- Blue progress bar shrinks over 5 seconds (CSS animation)
- Clicking the toast calls `handleSelectUser()` to open that conversation

### `prisma/schema.prisma`

The single source of truth for the database structure. When you run `prisma db push`, Prisma reads this file and creates/modifies the actual PostgreSQL tables to match. You never write raw SQL to change the schema — you edit this file and push.

---

## Key Design Decisions

### Why Socket.io instead of native WebSockets?

Socket.io adds:
- Automatic reconnection
- Fallback to HTTP long-polling (when WebSockets are blocked by firewalls/proxies)
- Room/namespace support
- Event-based API (cleaner than raw `ws.send(JSON.stringify(...))`)

### Why save messages to DB before emitting via socket?

1. **Persistence** — if the receiver is offline, they see the message when they log in
2. **Source of truth** — the DB is authoritative. The socket is just a delivery mechanism.
3. **Order guarantee** — messages have a `createdAt` timestamp in the DB

### Why use httpOnly cookies for the JWT?

If the JWT were stored in `localStorage`, any JavaScript on the page (including malicious scripts from XSS attacks) could read it. httpOnly cookies cannot be accessed by JavaScript at all — only sent automatically with every HTTP request by the browser.

### Why JWT instead of sessions?

Sessions require the server to store session state (in memory or Redis). JWT is stateless — the server just verifies the signature. Simpler for a single-server setup.

---

*Built with Next.js 14 + Socket.io + PostgreSQL + Prisma + Tailwind CSS*

