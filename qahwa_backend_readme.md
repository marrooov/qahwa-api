# ☕ Qahwa (El Qahwa) — Backend API & Real-Time Multiplayer Engine

**Qahwa** is a social gaming and real-time communication backend built for the Qahwa Flutter mobile application. Inspired by Plato, Qahwa brings friends together in a virtual café atmosphere to chat, track live presence, compete in server-authoritative multiplayer games (**Chess** and **Dominoes**), and progress through global levels and competitive Elo ratings.

---

## 🏗️ Architecture & Design Philosophy

Qahwa is architected as a **Modular Monolith** using **ASP.NET Core 8**, designed for strict data consistency, low-latency real-time gameplay, zero Firebase/Nakama lock-in, and cost-efficient cloud hosting.

### Key Architectural Highlights
* **Modular Monolith Boundaries:** Cleanly separated domain modules (`Identity`, `Profiles`, `Social`, `Messaging`, `Presence`, `Notifications`, `Rooms`, `Chess`, `Domino`, `Progression`) communicating via strongly typed contracts and in-process domain events.
* **Server-Authoritative Game Engines:** All Chess legal move validation (FEN/SAN/UCI), check/checkmate/draw detection, clock management, Domino shuffling, tile placement, and blocked-game resolution occur strictly on the server.
* **Hidden-Information Security:** Domino game states are never broadcast raw; each player receives a tailored projection containing only their own hand, open board ends, and the opponent's remaining tile count.
* **Single Multiplexed SignalR Hub (`/hubs/qahwa`):** Handles real-time chat, delivery/read receipts, multi-device presence, room lobbies, matchmaking notifications, and live game moves over a single authenticated WebSocket connection.
* **Horizontal Scale Ready:** Uses in-memory state abstractions (`IPresenceTracker`, `IMatchmakingQueue`) on single-node deployments ($0–$5/mo) that seamlessly switch to a **Redis** backplane when scaling out to multiple instances.

---

## 🛠️ Technology Stack

| Layer | Technology | Purpose |
| :--- | :--- | :--- |
| **Runtime & Framework** | **ASP.NET Core 8 (.NET 8 LTS)** | High-throughput REST API & background workers |
| **Real-Time Engine** | **ASP.NET Core SignalR** | Authenticated WebSockets, groups, presence & game state sync |
| **Primary Database** | **PostgreSQL 16** | Strict ACID relational data + `JSONB` for game states |
| **Data Access** | **EF Core 8 + Dapper** | Schema migrations, ACID transactions & fast read queries |
| **Object Storage & CDN** | **Cloudflare R2** | Zero-egress profile avatar storage & global CDN delivery |
| **Image Processing** | **SixLabors.ImageSharp** | Server-side validation, EXIF stripping & WebP resizing |
| **Authentication & Security** | **JWT + SHA-256 Refresh Tokens + Argon2id** | Stateless access tokens, token rotation & reuse detection |
| **Client Integration** | **Flutter (Dart)** | Compatible with `dio`, `signalr_netcore`, `flutter_secure_storage` |

---

## 📁 Project Structure

```text
qahwa-backend/
├── docs/                           # Architecture, ERD, REST & SignalR specs
├── src/
│   ├── Qahwa.Api/                  # Entry point, Controllers, SignalR Hub, Middleware
│   ├── Qahwa.SharedKernel/         # Base entities, Result<T>, Domain Events, Guards
│   └── Modules/
│       ├── Identity/               # Registration, Login, Argon2id, JWT, Token Rotation
│       ├── Profiles/               # Player profiles, ImageSharp processing, R2 storage
│       ├── Social/                 # Friendships, Friend Requests, Blocking policies
│       ├── Messaging/              # 1:1 Conversations, Idempotent messages, Read receipts
│       ├── Presence/               # Multi-device connection tracking (Online/InGame)
│       ├── Notifications/          # Persistent activity feed & real-time alerts
│       ├── Rooms/                  # Unified lobbies, Invites, Ready checks, Matchmaking
│       ├── Chess/                  # Authoritative Chess engine, timers, draw offers
│       ├── Domino/                 # Authoritative Double-Six Domino engine & projections
│       └── Progression/            # Atomic match settlement, XP ledger, Levels & Elo
└── tests/
    ├── Qahwa.UnitTests/            # Chess/Domino rules, Elo & XP calculation tests
    └── Qahwa.IntegrationTests/     # End-to-end API & SignalR hub tests (Testcontainers)
```

---

## ✨ Core Features

### 🔐 Identity & Security
* Registration, email OTP verification, login, logout, password change & reset.
* **Argon2id** password hashing and signed short-lived **JWT** access tokens.
* **Refresh Token Rotation** with family reuse detection (automatically revokes compromised device sessions).
* Multi-session management and per-endpoint rate limiting.

### 👤 Profiles & Avatars
* Default avatar fallback upon registration.
* Secure upload pipeline: MIME/magic-byte validation $\rightarrow$ EXIF stripping $\rightarrow$ $256\times256$ WebP compression $\rightarrow$ Cloudflare R2 upload $\rightarrow$ old avatar cleanup.
* URL versioning (`avatar_version`) for instant cache invalidation across Flutter clients.

### 🤝 Social & Real-Time Messaging
* Case-insensitive username search (`pg_trgm`) and unique ID lookup.
* Symmetric friendships, directed friend requests, and strict user blocking (automatically severs friendships, pending requests, chat access, and game invites).
* Real-time 1:1 messaging with Unicode emoji support, client-generated `idempotency_key` deduplication, delivery acknowledgments, read receipts, unread counters, and cursor pagination.

### ♟️ Multiplayer Games & Progression
* **Unified Room & Matchmaking System:** Private invite-code rooms, public lobbies, and rating-aware matchmaking queue with expanding Elo search windows.
* **Online Chess:** Server-validated moves, FEN/SAN history, check/checkmate/stalemate/draw detection, resignation, draw offers, clocks, and 30-second disconnect grace periods.
* **Online Dominoes (Double-Six):** Server-side shuffling, private hand projection, valid move detection, pass/blocked-game resolution, and extensible rule sets (`IDominoRuleSet`).
* **Progression & Ratings:** Separate **Global Player Level** (driven by an immutable `xp_transactions` ledger) and game-specific **Competitive Elo Ratings** for Chess and Dominoes, settled atomically inside a single database transaction upon match completion.

---

## 🚀 Getting Started (Local Development)

### Prerequisites
* [.NET 8 SDK](https://dotnet.microsoft.com/download/dotnet/8.0)
* [PostgreSQL 16+](https://www.postgresql.org/) (or Docker Desktop)
* Cloudflare R2 bucket credentials (or MinIO for local S3-compatible testing)

### 1. Clone the Repository
```bash
git clone https://github.com/your-username/qahwa-backend.git
cd qahwa-backend
```

### 2. Configure Environment Variables
Copy the example configuration file and populate your local secrets:
```bash
cp src/Qahwa.Api/appsettings.Development.example.json src/Qahwa.Api/appsettings.Development.json
```

Required settings in `appsettings.Development.json`:
```json
{
  "ConnectionStrings": {
    "DefaultConnection": "Host=localhost;Port=5432;Database=qahwa_dev;Username=postgres;Password=your_password"
  },
  "Jwt": {
    "Issuer": "https://api.qahwa.app",
    "Audience": "qahwa-flutter-client",
    "SecretKey": "REPLACE_WITH_A_SECURE_64_CHAR_RANDOM_SECRET_KEY",
    "AccessTokenMinutes": 15,
    "RefreshTokenDays": 30
  },
  "Storage": {
    "Provider": "CloudflareR2",
    "AccountId": "your_r2_account_id",
    "AccessKey": "your_r2_access_key",
    "SecretKey": "your_r2_secret_key",
    "BucketName": "qahwa-avatars",
    "PublicCdnBaseUrl": "https://cdn.qahwa.app"
  }
}
```

### 3. Apply Database Migrations
```bash
dotnet ef database update \
  --project src/Qahwa.Api \
  --startup-project src/Qahwa.Api
```

### 4. Run the API & SignalR Server
```bash
dotnet run --project src/Qahwa.Api
```
* **Swagger UI (Development):** `http://localhost:5000/swagger`
* **SignalR Hub Endpoint:** `ws://localhost:5000/hubs/qahwa`

---

## 🗺️ Development Roadmap

* [x] **Phase 1: Requirements & Architecture** — Tech stack evaluation, Modular Monolith design, game rules & hosting plan.
* [x] **Phase 2: Database Design** — Complete PostgreSQL 16 schema, ERD, indexes, constraints & ACID transaction boundaries.
* [ ] **Phase 3: API & Real-Time Contracts** — REST endpoint specs, JSON payloads & SignalR event definitions.
* [ ] **Phase 4: Incremental Implementation** — Identity, Profiles/R2, Social, Chat, Presence, Rooms, Chess, Dominoes, Matchmaking & Progression.
* [ ] **Phase 5: Testing & Deployment** — Unit/Integration tests, load verification & Docker deployment.

---

## 🔒 Security Policy
* Clients are **never** trusted for user identity, match outcomes, move legality, XP gains, or rating adjustments.
* All SQL queries are parameterized via EF Core / Dapper.
* Immutable ledger tables (`match_results`, `xp_transactions`) are protected against modification at both the application and PostgreSQL trigger levels.