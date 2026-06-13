# TheGist API

On-chain indexer and query layer for the TheGist protocol.

This is **not** a traditional REST backend. All write operations go directly to Soroban — this service only reads from the chain and serves the results.

---

## What This Service Does

TheGist-API is a **Soroban event indexer** built with Node.js and the Stellar SDK. It listens for `GistPosted` events emitted by the `GistRegistry` contract and writes them to a local PostgreSQL + PostGIS database for fast geospatial querying.

It then exposes a lightweight **read-only GraphQL API** so that clients can query nearby gists by coordinates and radius.

```
Soroban RPC ──► Indexer (polls events) ──► PostGIS (stores + indexes)
                                                    │
                                                    ▼
                                          GraphQL API (read-only)
                                                    │
                                                    ▼
                                        Mobile / Web Clients
```

There is no write path through this service. Posting a gist means signing and submitting a Stellar transaction directly to Soroban from the client.

---

## Stack

| Layer | Choice |
|-------|--------|
| Runtime | Node.js >= 20 |
| Blockchain | Stellar SDK + Soroban RPC |
| Database | PostgreSQL 15 + PostGIS extension |
| GraphQL | Apollo Server |
| Language | TypeScript |

---

## How the Indexer Works

1. On startup, reads the last processed ledger sequence from the database
2. Polls Soroban RPC for new `GistRegistry` contract events
3. Parses each `GistPosted` event — extracts IPFS CID, geohash, author address, and timestamp
4. Decodes the geohash to a lat/lon point and stores it in PostGIS with a spatial index
5. Any gist posted directly on-chain (not via a client app) is automatically included

The indexer runs as a background service alongside the GraphQL server. Both start with `npm start`.

---

## GraphQL API

### Queries

#### `nearbyGists(lat, lon, radiusMeters)`

Returns all gists within `radiusMeters` of the given coordinate, ordered by distance.

```graphql
query {
  nearbyGists(lat: 51.5074, lon: -0.1278, radiusMeters: 500) {
    gistId
    ipfsCid
    geohash
    author
    timestamp
    expiry
    distanceMeters
  }
}
```

#### `gistById(id)`

Returns a single gist by its on-chain ID.

```graphql
query {
  gistById(id: "42") {
    gistId
    ipfsCid
    geohash
    author
    timestamp
  }
}
```

#### `gistsByAuthor(address)`

Returns all gists posted by a given Stellar address.

```graphql
query {
  gistsByAuthor(address: "GXXXXXXXXXXXXXXXXXXXXXXXXXXXXXXXXXXXXXXXXXXXXXXXXXXXXXXX") {
    gistId
    ipfsCid
    geohash
    timestamp
  }
}
```

---

## Local Setup

### 1. Clone and install

```bash
git clone https://github.com/TheGist-Org/TheGist-API.git
cd TheGist-API
npm install
```

### 2. Set up PostgreSQL + PostGIS

```bash
# macOS
brew install postgresql@15 postgis
brew services start postgresql@15

# Ubuntu / Debian
sudo apt install -y postgresql-15 postgresql-15-postgis-3
sudo systemctl start postgresql
```

Create the database:

```sql
CREATE USER thegist WITH PASSWORD 'thegist';
CREATE DATABASE thegist OWNER thegist;
\c thegist
CREATE EXTENSION postgis;
```

### 3. Environment variables

```bash
cp .env.example .env
```

```env
# Soroban
SOROBAN_RPC_URL=https://soroban-testnet.stellar.org
CONTRACT_ID=<your_deployed_gist_registry_contract_id>

# Database
DATABASE_URL=postgresql://thegist:thegist@localhost:5432/thegist

# IPFS
IPFS_GATEWAY=https://ipfs.io/ipfs/

# Server
PORT=4000
```

### 4. Run migrations

```bash
npm run migrate
```

### 5. Start the service

```bash
npm start
```

The GraphQL playground is available at `http://localhost:4000/graphql`.

---

## Project Layout

```
TheGist-API/
├── src/
│   ├── indexer/          # Soroban event polling and parsing
│   ├── graphql/          # Apollo Server schema and resolvers
│   ├── db/               # PostGIS queries and migrations
│   ├── stellar/          # Stellar SDK wrapper
│   └── index.ts          # Entry point
├── migrations/
├── .env.example
├── package.json
└── README.md
```

---

## Contributing

- This service is read-only by design. Do not add write endpoints — all writes go through Soroban directly.
- PostGIS query changes should include an `EXPLAIN ANALYZE` output in the PR description.
- New GraphQL fields must be documented in the schema with descriptions.

For global contribution rules, see [CONTRIBUTING.md](https://github.com/TheGist-Org/theGist-Meta/blob/main/CONTRIBUTING.md).

---

## License

MIT
