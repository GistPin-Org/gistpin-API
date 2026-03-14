# Gist API

The backend API and on-chain indexer for Gist. This service is the bridge between your clients (web + mobile) and both the Stellar/Soroban blockchain and the Postgres database.

---

## What This Repo Does

- **Indexes** on-chain events from the `GistRegistry` Soroban contract
- **Stores** enriched gist data in Postgres + PostGIS for fast geospatial queries
- **Exposes** a REST API consumed by `gist-web` and `gist-mobile`
- **Bridges** to IPFS/Arweave for full gist content storage (the chain only holds a hash)

---

## Tech Stack

| Layer | Choice |
|---|---|
| Language | TypeScript |
| Runtime | Node.js >= 20 |
| Framework | NestJS |
| Database | PostgreSQL 15 + PostGIS extension |
| ORM / Query | TypeORM (with PostGIS support) |
| Blockchain | Stellar Horizon + Soroban RPC |
| Storage bridge | IPFS via Pinata (or self-hosted node) |
| Testing | Jest (built into NestJS) |

---

## Project Layout

```
gist-backend/
├── src/
│   ├── main.ts                    # App bootstrap
│   ├── app.module.ts              # Root module
│   ├── gists/                     # Gist feature module
│   │   ├── gists.module.ts
│   │   ├── gists.controller.ts    # Route handlers
│   │   ├── gists.service.ts       # Business logic
│   │   ├── gists.repository.ts    # DB queries (PostGIS)
│   │   ├── dto/
│   │   │   ├── create-gist.dto.ts
│   │   │   └── query-gists.dto.ts
│   │   └── entities/
│   │       └── gist.entity.ts
│   ├── indexer/                   # Soroban event watcher
│   │   ├── indexer.module.ts
│   │   └── indexer.service.ts
│   ├── soroban/                   # Soroban RPC client wrapper
│   │   ├── soroban.module.ts
│   │   └── soroban.service.ts
│   ├── ipfs/                      # IPFS pinning service wrapper
│   │   ├── ipfs.module.ts
│   │   └── ipfs.service.ts
│   ├── geo/                       # Geospatial helpers (cell encoding, etc.)
│   │   └── geo.service.ts
│   └── config/
│       └── configuration.ts       # Typed config via @nestjs/config
├── migrations/                    # TypeORM migration files
├── test/                          # e2e tests
├── .env.example
├── docker-compose.yml             # Optional — only needed if using Docker for Postgres
├── package.json
└── README.md
```

---

## Prerequisites

- **Node.js** >= 20 — [nodejs.org](https://nodejs.org)
- **PostgreSQL 15** with the **PostGIS extension** — see database setup options below
- **npm** (comes with Node.js — no extra install needed)

> **Why PostGIS?** The core feature of Gist is querying gists by distance — *"show me everything within 500m of this coordinate."* Plain Postgres can't do that efficiently. PostGIS adds a spatial index that makes this instant, even at scale. It's not a separate database — just one extension enabled inside your existing Postgres.

---

## Local Development

### 1. Clone and install

```bash
git clone https://github.com/gist-app/gist-backend.git
cd gist-backend
npm install
```

### 2. Set up Postgres + PostGIS

Pick whichever path suits you. Both end with a running Postgres that has PostGIS available.

---

#### Option A — Docker (quickest, nothing to install manually)

Requires [Docker Desktop](https://www.docker.com/products/docker-desktop/) or Docker Engine.

```bash
docker-compose up -d
```

This starts a `postgis/postgis:15-3.3` container on port `5432` with the credentials from `docker-compose.yml` (`user: gist`, `password: gist`, `db: gist`). PostGIS is pre-installed in this image — nothing else to do.

---

#### Option B — Manual Postgres install (no Docker required)

**macOS (Homebrew):**

```bash
brew install postgresql@15 postgis
brew services start postgresql@15
```

**Ubuntu / Debian:**

```bash
sudo apt update
sudo apt install -y postgresql-15 postgresql-15-postgis-3
sudo systemctl start postgresql
```

**Windows:** Download the installer from [postgresql.org](https://www.postgresql.org/download/windows/) and tick the PostGIS option in Stack Builder after install.

Once Postgres is running, create the database and user:

```bash
psql -U postgres
```

```sql
CREATE USER gist WITH PASSWORD 'gist';
CREATE DATABASE gist OWNER gist;
-- Connect to the new database and enable PostGIS
\c gist
CREATE EXTENSION postgis;
\q
```

---

### 3. Environment variables

```bash
cp .env.example .env
```

Open `.env` and fill in the values. Minimum required for local dev:

```env
# App
PORT=3000
NODE_ENV=development

# Database — match whatever you set in step 2
DATABASE_HOST=localhost
DATABASE_PORT=5432
DATABASE_USER=gist
DATABASE_PASSWORD=gist
DATABASE_NAME=gist

# Soroban / Stellar
SOROBAN_RPC_URL=https://soroban-testnet.stellar.org
STELLAR_NETWORK_PASSPHRASE=Test SDF Network ; September 2015
CONTRACT_ID_GIST_REGISTRY=<your_deployed_contract_id>

# IPFS (Pinata)
PINATA_API_KEY=
PINATA_SECRET_KEY=

# Optional: backend signing keypair (for submitting txs to Soroban)
STELLAR_SECRET_KEY=
```

### 4. Run database migrations

```bash
npm run migration:run
```

### 5. Start the dev server

```bash
npm run start:dev
```

The API will be available at: `http://localhost:3000`

---

## API Overview (MVP)

### Health

```
GET /health
```

Returns `{ status: "ok" }`.

---

### Read Gists

```
GET /gists?lat=5.6037&lon=-0.1870&radius=500&limit=20&cursor=
```

| Query Param | Type | Description |
|---|---|---|
| `lat` | `number` | Latitude (required) |
| `lon` | `number` | Longitude (required) |
| `radius` | `number` | Radius in metres (default: `500`, max: `5000`) |
| `limit` | `number` | Max results (default: `20`, max: `100`) |
| `cursor` | `string` | Pagination cursor from previous response |

**Response:**

```json
{
  "data": [
    {
      "gistId": "42",
      "lat": 5.6042,
      "lon": -0.1865,
      "text": "Great street food here tonight",
      "authorAddress": null,
      "contentCid": "bafybeihash...",
      "createdAt": "2025-06-01T10:22:00Z",
      "txHash": "abc123..."
    }
  ],
  "nextCursor": "eyJpZCI6NDJ9",
  "total": 1
}
```

---

### Create a Gist

```
POST /gists
Content-Type: application/json
```

**Body:**

```json
{
  "lat": 5.6037,
  "lon": -0.1870,
  "text": "Anyone know a good spot nearby?",
  "authorAddress": "GXXXXXXXXXXXXXXXXXXXXXXXXXXXXXXXXXXXXXXXXXXXXXXXXXXXXXXX"
}
```

`authorAddress` is optional — anonymous posting is fully supported.

**What happens internally:**

1. Validate + sanitise input
2. Pin content to IPFS → receive CID
3. Derive `locationCell` from `(lat, lon)` using S2 / geohash
4. Submit `post_gist(author, locationCell, contentHash)` transaction to Soroban
5. Persist the record in Postgres
6. Return the created gist

**Response:** `201 Created` with the full gist object.

---

## Database Model (MVP)

Table: `gists`

| Column | Type | Notes |
|---|---|---|
| `id` | `uuid` PK | Internal primary key |
| `gist_id` | `bigint` UNIQUE | On-chain ID from contract |
| `location_cell` | `bigint` | Coarse geo cell (S2/geohash) |
| `location` | `geography(Point, 4326)` | PostGIS point for geo queries |
| `lat` | `float8` | Stored for convenience |
| `lon` | `float8` | Stored for convenience |
| `content_cid` | `text` | IPFS/Arweave CID |
| `text` | `text` | Full gist text (cached from IPFS) |
| `author_address` | `text` | Nullable — anonymous posts allowed |
| `tx_hash` | `text` | Stellar transaction hash |
| `created_at` | `timestamptz` | |

Key indexes: `GIST index on location` (PostGIS), `location_cell`, `created_at DESC`.

---

## Indexer

`src/indexer/indexer.service.ts` runs as a background NestJS worker. On startup it:

1. Reads the last processed ledger sequence from DB
2. Polls Soroban RPC for new `GistRegistry` contract events
3. Hydrates and upserts each new gist into Postgres

This keeps the DB in sync with on-chain state, and ensures any gist posted directly on-chain (not via the API) still appears in query results.

---

## Working with the Contracts

If you're also developing `gist-contracts`, run a local Stellar network:

```bash
# In gist-contracts repo
stellar network start local

# Deploy the contract, then copy the contract ID into your .env:
# CONTRACT_ID_GIST_REGISTRY=<output_id>
```

The Soroban service (`src/soroban/soroban.service.ts`) wraps `@stellar/stellar-sdk` and handles transaction building, signing, and submission for you.

---

## Useful Scripts

| Command | Description |
|---|---|
| `npm run start:dev` | Start with hot-reload |
| `npm run build` | Compile TypeScript |
| `npm run start:prod` | Run compiled output |
| `npm test` | Unit tests |
| `npm run test:e2e` | End-to-end tests |
| `npm run migration:generate -- -n MigrationName` | Generate a new migration |
| `npm run migration:run` | Apply pending migrations |
| `npm run migration:revert` | Revert last migration |

---

## Contribution Guidelines

- Keep business logic in `services/`, keep controllers thin (just validate + delegate).
- Any breaking API changes must be opened as an issue and discussed before implementation.
- All new behaviour should come with a unit or e2e test.
- For PostGIS queries, add explain-analyse output to your PR description so we can review query plans.

For global contribution rules: [gist-meta/CONTRIBUTING.md](https://github.com/gist-app/gist-meta/blob/main/CONTRIBUTING.md)
