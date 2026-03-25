# Sportz Live Match Service ⚽

![Node.js](https://img.shields.io/badge/Node.js-22%2B-339933?logo=nodedotjs&logoColor=white)
![Express](https://img.shields.io/badge/Express-5.x-000000?logo=express&logoColor=white)
![PostgreSQL](https://img.shields.io/badge/PostgreSQL-16%2B-4169E1?logo=postgresql&logoColor=white)
![WebSocket](https://img.shields.io/badge/WebSocket-ws-010101?logo=socketdotio&logoColor=white)
![Drizzle ORM](https://img.shields.io/badge/Drizzle-ORM-C5F74F)
![Zod](https://img.shields.io/badge/Zod-Validation-3E67B1)
![Arcjet](https://img.shields.io/badge/Arcjet-Security-2B2B2B)

A real-time sports backend for match creation, commentary feeds, and live fan updates.

You get:
- REST APIs to create and list matches/commentary
- WebSocket broadcasting for live updates
- PostgreSQL persistence with Drizzle ORM
- Request validation using Zod
- HTTP + WebSocket protection with Arcjet

---

## Table of Contents 📚

- [Overview](#overview-)
- [Architecture](#architecture-)
- [Project Structure](#project-structure-)
- [Tech Stack](#tech-stack-)
- [Quick Start](#quick-start-)
- [Environment Variables](#environment-variables-)
- [Database & Migrations](#database--migrations-)
- [REST API](#rest-api-)
- [WebSocket Protocol](#websocket-protocol-)
- [Validation Rules](#validation-rules-)
- [Security Layer](#security-layer-)
- [How It Works Internally](#how-it-works-internally-)
- [Common Workflows](#common-workflows-)
- [Troubleshooting](#troubleshooting-)
- [Contributing](#contributing-)
- [Roadmap Ideas](#roadmap-ideas-)
- [License](#license-)

---

## Overview 🔭

Sportz is designed for scenarios where clients need both:
- reliable CRUD-like API operations (REST), and
- low-latency push notifications (WebSocket).

Typical flow:
1. Create matches with `POST /matches`.
2. Subscribe clients to a match via WebSocket.
3. Push commentary with `POST /matches/:id/commentary`.
4. Subscribers instantly receive commentary events.

---

## Architecture 🏗️

```mermaid
flowchart LR
  C1[Client App\nWeb/Mobile] -->|HTTP JSON| API[Express API]
  C1 -->|WebSocket /ws| WSS[ws Server]

  API --> V[Zod Validation]
  API --> SEC[Arcjet HTTP Guard]
  API --> DB[(PostgreSQL)]

  WSS --> WSEC[Arcjet WS Guard]
  WSS --> SUB[(In-memory\nmatch subscribers map)]

  API -->|broadcastMatchCreated| WSS
  API -->|broadcastCommentary| WSS
```

### Runtime View

```mermaid
sequenceDiagram
  participant U as API User
  participant E as Express
  participant Z as Zod
  participant D as Drizzle/Postgres
  participant W as WebSocket Clients

  U->>E: POST /matches
  E->>Z: validate payload
  Z-->>E: parsed data
  E->>D: insert match
  D-->>E: created row
  E->>W: match_created event
  E-->>U: 201 Created
```

---

## Project Structure 🗂️

```text
src/
  index.js                 # App bootstrap, route mounting, WS attach
  arcjet.js                # HTTP/WS security policies
  db/
    db.js                  # Drizzle + pg pool setup
    schema.js              # matches + commentary table schema
  routes/
    matches.js             # /matches endpoints
    commentary.js          # /matches/:id/commentary endpoints
  validation/
    matches.js             # Match request/query schema
    commentary.js          # Commentary request/query schema
  utils/
    match-status.js        # scheduled/live/finished status logic
  ws/
    server.js              # WebSocket protocol + subscriptions

drizzle/
  *.sql                    # Generated migration SQL
  meta/                    # Drizzle migration metadata
```

---

## Tech Stack 🧰

- Node.js (ESM)
- Express 5
- ws (WebSocket server)
- PostgreSQL + Drizzle ORM
- Zod (schema validation)
- Arcjet (bot/rate-limit/shield protections)

---

## Quick Start 🚀

### 1. Install dependencies

```bash
npm install
```

### 2. Create `.env`

```env
DATABASE_URL=postgres://user:password@localhost:5432/sportz
ARCJET_KEY=your_arcjet_key_here
ARCJET_MODE=DRY_RUN
PORT=8000
HOST=0.0.0.0
```

If `ARCJET_KEY` is omitted, security protections are disabled and the app still runs.

### 3. Generate and apply migrations

```bash
npm run db:generate
npm run db:migrate
```

### 4. Run the server

```bash
npm run dev
```

Server endpoints:
- HTTP: `http://localhost:8000`
- WebSocket: `ws://localhost:8000/ws`

---

## Environment Variables 🔐

| Variable | Required | Default | Description |
|---|---|---|---|
| `DATABASE_URL` | Yes | - | PostgreSQL connection string |
| `PORT` | No | `8000` | HTTP server port |
| `HOST` | No | `0.0.0.0` | Host bind address |
| `ARCJET_KEY` | No | - | Enables Arcjet protection when present |
| `ARCJET_MODE` | No | `LIVE` | Use `DRY_RUN` to observe without blocking |

---

## Database & Migrations 🗄️

The app uses Drizzle schema definitions in `src/db/schema.js` and generated SQL migrations in `drizzle/`.

### Main tables

- `matches`
- `commentary` (FK to `matches.id`)

### ER diagram

```mermaid
erDiagram
  matches ||--o{ commentary : has

  matches {
    int id PK
    text sport
    text home_team
    text away_team
    enum status
    timestamp start_time
    timestamp end_time
    int home_score
    int away_score
    timestamp created_at
  }

  commentary {
    int id PK
    int match_id FK
    int minute
    int sequence
    text period
    text event_type
    text actor
    text team
    text message
    json metadata
    text[] tags
    timestamp created_at
  }
```

---

## REST API 🌐

Base URL: `http://localhost:8000`

### Health

#### `GET /`

Response:

```json
{ "message": "Sportz server is running." }
```

### Matches

#### `GET /matches?limit=50`

Returns latest matches in descending `createdAt` order.

#### `POST /matches`

Request body example:

```json
{
  "sport": "football",
  "homeTeam": "JSM United",
  "awayTeam": "Rovers FC",
  "startTime": "2026-03-25T14:00:00.000Z",
  "endTime": "2026-03-25T16:00:00.000Z",
  "homeScore": 0,
  "awayScore": 0
}
```

Notes:
- `status` is computed automatically (`scheduled`, `live`, `finished`) using start/end time.
- After creation, all WebSocket clients receive a `match_created` event.

### Commentary

#### `GET /matches/:id/commentary?limit=100`

Returns commentary rows for one match in descending `createdAt` order.

#### `POST /matches/:id/commentary`

Request body example:

```json
{
  "minute": 42,
  "sequence": 120,
  "period": "2nd half",
  "eventType": "goal",
  "actor": "Alex Morgan",
  "team": "JSM United",
  "message": "GOAL! Powerful finish from outside the box.",
  "metadata": { "assist": "Sam Kerr" },
  "tags": ["goal", "shot"]
}
```

After creation, subscribers of that match get a real-time `commentary` event.

---

## WebSocket Protocol 📡

Connect to:

```text
ws://localhost:8000/ws
```

### Client -> Server messages

Subscribe to one match:

```json
{ "type": "subscribe", "matchId": 1 }
```

Unsubscribe:

```json
{ "type": "unsubscribe", "matchId": 1 }
```

### Server -> Client messages

Welcome:

```json
{ "type": "welcome" }
```

Subscribe ack:

```json
{ "type": "subscribed", "matchId": 1 }
```

New match broadcast (all clients):

```json
{ "type": "match_created", "data": { "id": 1, "...": "..." } }
```

Commentary broadcast (only subscribers of match):

```json
{ "type": "commentary", "data": { "id": 7, "matchId": 1, "...": "..." } }
```

### Subscription lifecycle diagram

```mermaid
sequenceDiagram
  participant C as Client
  participant S as WS Server
  participant M as Subscribers Map

  C->>S: connect /ws
  S-->>C: {type: "welcome"}
  C->>S: {type: "subscribe", matchId: 1}
  S->>M: add client to Set(matchId=1)
  S-->>C: {type: "subscribed", matchId: 1}

  Note over S: On commentary POST for match 1
  S-->>C: {type: "commentary", data: ...}

  C->>S: close socket
  S->>M: cleanup all subscriptions
```

---

## Validation Rules ✅

Validation is handled with Zod before DB operations.

### Match rules
- `sport`, `homeTeam`, `awayTeam`: non-empty strings
- `startTime`, `endTime`: ISO-like datetime strings containing `T`
- `endTime` must be after `startTime`
- Optional scores are non-negative integers
- `limit` query max is `100`

### Commentary rules
- `minute`, `sequence`: non-negative integers
- `period`, `eventType`, `actor`, `team`, `message`: non-empty strings
- `metadata`: object map
- `tags`: string array
- Route param `id` must be a positive integer

---

## Security Layer 🛡️

Arcjet is used for both HTTP and WS requests when `ARCJET_KEY` is set.

Enabled protections:
- `shield`
- `detectBot` (allows search engine and preview bots)
- `slidingWindow` rate limiting

Behavior:
- Denied HTTP requests return `403`
- Rate-limited HTTP requests return `429`
- Denied WebSocket upgrades return `403` or `429`

---

## How It Works Internally ⚙️

1. `src/index.js` boots Express and wraps it in an HTTP server.
2. Route handlers validate input with Zod.
3. Drizzle ORM reads/writes PostgreSQL.
4. Match creation triggers `broadcastMatchCreated` to all WS clients.
5. Commentary creation triggers `broadcastCommentary` to subscribed clients only.
6. WS server maintains in-memory subscription sets keyed by `matchId`.
7. Heartbeats (`ping`/`pong`) terminate stale sockets.

---

## Common Workflows 👨‍💻

### Create a match

```bash
curl -X POST http://localhost:8000/matches \
  -H "Content-Type: application/json" \
  -d '{
    "sport":"football",
    "homeTeam":"JSM United",
    "awayTeam":"Rovers FC",
    "startTime":"2026-03-25T14:00:00.000Z",
    "endTime":"2026-03-25T16:00:00.000Z",
    "homeScore":0,
    "awayScore":0
  }'
```

### Add commentary for match `1`

```bash
curl -X POST http://localhost:8000/matches/1/commentary \
  -H "Content-Type: application/json" \
  -d '{
    "minute":42,
    "sequence":120,
    "period":"2nd half",
    "eventType":"goal",
    "actor":"Alex Morgan",
    "team":"JSM United",
    "message":"GOAL! Powerful finish from outside the box.",
    "metadata":{"assist":"Sam Kerr"},
    "tags":["goal","shot"]
  }'
```

---

## Troubleshooting 🧪

### `DATABASE_URL is not defined`
- Ensure `.env` exists and includes `DATABASE_URL`
- Verify the DB is reachable from your machine

### WebSocket cannot connect
- Confirm server is running on correct `HOST`/`PORT`
- Ensure you connect to `/ws`
- Check Arcjet config if upgrade is denied (`403`/`429`)

### `Invalid payload` responses
- Compare your JSON body with schema examples above
- Ensure numeric fields are numbers (or numeric strings)

---

## Contributing 🤝

Contributions are welcome.

### Development setup

1. Fork and clone the repo
2. Install dependencies with `npm install`
3. Configure `.env`
4. Run migrations with `npm run db:migrate`
5. Start dev server with `npm run dev`

### Branch & commit style

- Branch naming: `feat/...`, `fix/...`, `docs/...`
- Keep commits focused and descriptive
- Include API examples when changing route behavior

### Pull request checklist

- [ ] Code compiles and server starts
- [ ] Endpoint behavior validated manually
- [ ] Migration files included when schema changes
- [ ] README/docs updated for API or protocol changes
- [ ] Backward compatibility considered

---

## Roadmap Ideas 🧠

- Add auth and role-based permissions
- Add update/delete commentary endpoints
- Add pagination with cursors
- Add OpenAPI/Swagger docs
- Add automated tests (unit + integration)
- Add Docker + Compose for one-command local setup

---

## License 📄

Currently configured as `ISC` in `package.json`.

---

Built with ❤️ for real-time sports experiences.
