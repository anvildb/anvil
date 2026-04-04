# Anvil Graph Database

**A Neo4j-inspired graph database written in Rust** — combining a property graph, native document store (NoSQL), and row-level security into a single binary.

Anvil eliminates the need to run separate databases for graph queries and document lookups. Store your data once, query it with Cypher, GraphQL, or REST, and let sync rules keep everything consistent.

## Install

```bash
curl -fsSL https://anvildb.com/install.sh | sh
```

Or download a binary from [Releases](https://github.com/devforgeinc/anvil/releases).

### Supported Platforms

| Platform | Architecture | Binary |
|----------|-------------|--------|
| Linux | x86_64 | `anvil-vX.Y.Z-x86_64-unknown-linux-gnu.tar.gz` |
| Linux | aarch64 (ARM64) | `anvil-vX.Y.Z-aarch64-unknown-linux-gnu.tar.gz` |
| macOS | x86_64 (Intel) | `anvil-vX.Y.Z-x86_64-apple-darwin.tar.gz` |
| macOS | aarch64 (Apple Silicon) | `anvil-vX.Y.Z-aarch64-apple-darwin.tar.gz` |
| Windows | x86_64 | `anvil-vX.Y.Z-x86_64-pc-windows-msvc.zip` |

## Quick Start

```bash
# Start the server (daemonizes by default)
anvil start

# Or run in foreground
anvil start --foreground

# Check status
anvil status

# Stop the server
anvil stop

# Open the browser UI
open http://localhost:5175

# Default login: admin / anvil
```

The server runs on port **7474** (HTTP API) and **7687** (Bolt protocol). The browser UI runs on port **5175**. The server forks to background by default and writes its PID to `data/anvil.pid`.

## Features

### Property Graph (like Neo4j)

Full Cypher query language with nodes, relationships, labels, and properties.

```cypher
-- Create nodes and relationships
CREATE (a:Person {name: "Alice", age: 30})-[:KNOWS]->(b:Person {name: "Bob", age: 25})

-- Query with pattern matching
MATCH (n:Person)-[r:KNOWS]->(m:Person) WHERE n.age > 25 RETURN n, r, m

-- Filter by properties
MATCH (n:Person {name: "Alice"}) RETURN n.name, n.age
```

### Native Document Store (NoSQL)

Built-in document collections with O(1) key lookups, composite keys, TTL, secondary indexes, batch operations, and filter expressions — no separate database needed.

Collections are organized into schemas: `public` (default, no prefix needed) for application data, and `auth` for system collections. The browser UI provides a schema dropdown to switch between views.

```bash
# Create a collection
curl -X POST http://localhost:7474/docs/orders \
  -H "Authorization: Bearer $TOKEN" \
  -d '{"composite_keys": false}'

# Insert a document
curl -X PUT http://localhost:7474/docs/orders/order-1 \
  -H "Authorization: Bearer $TOKEN" \
  -d '{"body": {"item": "Widget", "qty": 10, "status": "pending"}}'

# Query with filters
curl -X POST http://localhost:7474/docs/orders/query \
  -H "Authorization: Bearer $TOKEN" \
  -d '{"filter": {"op": "eq", "field": "status", "value": "pending"}}'
```

Or via Cypher:

```cypher
CREATE DOCUMENT IN orders order1 {item: "Widget", qty: 10, status: "pending"}
MATCH DOCUMENT d IN orders WHERE d.status = "pending" RETURN d.item, d.qty
```

### Graph-Document Sync

Automatically keep graph nodes and document collections in sync. Define a sync rule and existing data is backfilled immediately.

```cypher
SYNC LABEL Person TO COLLECTION persons KEY name INCLUDE name, email, age
```

Now:
- Creating a `:Person` node auto-creates a document in `persons`
- Inserting a document in `persons` auto-creates a `:Person` node
- Updates propagate bidirectionally
- The `INCLUDE` clause controls which fields sync (keeps documents lean)

### GraphQL API

Auto-generated GraphQL schema from your graph structure with introspection and query support.

```graphql
{
  users(where: { username: "admin" }) {
    id
    username
    email
  }
}
```

```graphql
{
  __schema {
    types { name kind }
  }
}
```

### Authentication (JWT + JWKS)

Graph-native authentication with users and roles stored in schema-namespaced collections (`auth.*`) and synced to graph nodes.

- **RS256 JWT tokens** — asymmetric signing with persistent RSA key pair (`data/jwt_key.pem`), 1-hour access tokens, 7-day refresh tokens
- **JWKS endpoint** — `GET /.well-known/jwks.json` for external token verification
- **Argon2id password hashing** — OWASP-recommended
- **Per-request session middleware** — identity extracted from Bearer token on every request
- **Schema-namespaced collections** — `auth.users`, `auth.roles`, `auth.user_roles`, `auth.refresh_tokens`
- **Graph sync** — `:User` and `:Role` nodes with `[:HAS_ROLE]` relationships, synced from auth collections
- **Forced password change** — default admin user must change password on first login

```bash
# Login
curl -X POST http://localhost:7474/auth/login \
  -d '{"username": "admin", "password": "anvil"}'

# Response: { "idToken": "eyJ...", "refreshToken": "abc...", "accessToken": "eyJ..." }

# Use the token
curl http://localhost:7474/db/query \
  -H "Authorization: Bearer eyJ..." \
  -d '{"query": "MATCH (n) RETURN n"}'

# Register a new user
curl -X POST http://localhost:7474/auth/register \
  -H "Authorization: Bearer eyJ..." \
  -d '{"username": "alice", "password": "secret", "email": "alice@example.com"}'

# Refresh an expired access token
curl -X POST http://localhost:7474/auth/refresh \
  -d '{"refresh_token": "abc..."}'
```

### Row-Level Security (RLS)

PostgreSQL-inspired fine-grained access control at the data level. Policies on graph labels automatically protect paired document collections via sync pairs.

```cypher
-- Enable RLS on a label
ENABLE ROW LEVEL SECURITY ON :Project

-- Users can only see their own data
CREATE POLICY own_data ON :Project FOR SELECT TO reader
  USING (n.owner = current_user())

-- Multi-tenant isolation
CREATE POLICY tenant ON :Project FOR ALL TO authenticated
  USING (n.tenant_id = session('tenant_id'))

-- Simulate what a user would see
SIMULATE POLICY AS alice WITH ROLE reader ON :Project
```

RLS sync pairs: when a sync rule links a label to a collection (e.g. `SYNC LABEL Person TO COLLECTION persons`), a single policy on `:Person` automatically applies to the `persons` collection too.

### Browser UI

React Router 7 web application with:

- **Login Screen** — JWT authentication with forced password change on first login
- **Cypher Editor** — syntax highlighting, Cmd+Enter execution, table/JSON/graph result views
- **GraphQL Playground** — query editor with introspection, data/JSON views
- **Graph Visualization** — force-directed layout, node/edge inspection and editing, double-click to expand neighbors, per-label caption properties, right-click context menus
- **Document Store Manager** — collection CRUD, document browsing with pagination, JSON editor, sync rule management
- **RLS Policy Manager** — create/drop policies, enable/disable RLS, policy simulator
- **Schema Browser** — labels, relationship types, property keys, indexes, constraints
- **Schema Dropdown** — switch between `public` and `auth` schemas (like Supabase)
- **Monitoring Dashboard** — real-time stats from server
- **Admin Panel** — user management, role management, database info

### Additional Features

- **ACID Transactions** — MVCC with snapshot isolation, record-level locking, deadlock detection, ARIES-style WAL recovery
- **Storage Engine** — page-based with LRU cache, WAL with CRC32 checksums, B+ tree indexes (unique, composite, full-text, spatial), LZ4/Zstd compression
- **Graph Algorithms** — PageRank, Dijkstra/A*, BFS/DFS, Louvain community detection, connected components, betweenness/closeness/degree centrality, MST, triangle counting, node similarity
- **Clustering** — leader-follower WAL streaming replication, Raft consensus, read replicas, sharding
- **Plugin System** — native Rust + Lua/Python/WASM/JS/Starlark scripting runtimes
- **Built-in `apoc` Library** — 55 functions: bitwise, geospatial, collections, text similarity, conversions, math, date/time, graph refactoring, import/export
- **Client Drivers** — Rust, TypeScript, Python, Go with `anvil://` URI scheme

## Configuration

```toml
# anvil.toml
[server]
http_port = 7474
bolt_port = 7687
bind_address = "0.0.0.0"

[storage]
data_dir = "./data"
document_storage = "unified"   # or "split"

[auth]
enabled = true
default_password = "anvil"
access_token_ttl_secs = 3600   # 1 hour
refresh_token_ttl_secs = 604800 # 7 days
# RSA key pair auto-generated and persisted to data/jwt_key.pem
```

Or use environment variables:

```bash
ANVIL_HTTP_PORT=8080 ANVIL_DATA_DIR=/var/data anvil start
```

## API Endpoints

### Core
| Method | Endpoint | Description |
|--------|----------|-------------|
| GET | `/` | Server info |
| GET | `/health` | Health check |
| POST | `/db/query` | Execute Cypher query |
| GET | `/db/{name}/schema` | Get database schema |
| GET | `/db/{name}/graph` | Get full graph data |
| POST | `/graphql` | GraphQL endpoint |

### Authentication
| Method | Endpoint | Description |
|--------|----------|-------------|
| POST | `/auth/login` | Login, returns RS256 JWT tokens |
| POST | `/auth/refresh` | Refresh access token |
| POST | `/auth/register` | Register new user (requires auth) |
| POST | `/auth/change-password` | Change password |
| GET | `/.well-known/jwks.json` | JWKS public keys for token verification |

### Document Store
| Method | Endpoint | Description |
|--------|----------|-------------|
| GET | `/docs` | List collections |
| POST | `/docs/{collection}` | Create collection |
| DELETE | `/docs/{collection}` | Drop collection |
| GET | `/docs/{collection}/{id}` | Get document |
| PUT | `/docs/{collection}/{id}` | Upsert document |
| DELETE | `/docs/{collection}/{id}` | Delete document |
| POST | `/docs/{collection}/query` | Query with filters |
| POST | `/docs/{collection}/batch` | Batch operations |
| GET | `/docs/{collection}/scan` | Paginated scan |

### Admin
| Method | Endpoint | Description |
|--------|----------|-------------|
| GET | `/admin/stats` | Server statistics |
| GET | `/admin/users` | List users |
| GET | `/admin/roles` | List roles |

## Architecture

```
Browser UI (React Router 7 + TypeScript)
    |
    v  HTTP / WebSocket / GraphQL
Server (Axum + Tokio)
    |
    +--- Cypher Engine (Lexer -> Parser -> Planner -> Executor)
    +--- GraphQL (Auto-schema + Query Execution)
    +--- Document Store (Collections, Indexes, TTL, Sync)
    +--- Auth (JWT RS256, JWKS, Argon2id, Per-request Sessions)
    +--- RLS Engine (Policies, Predicates, Cascading Visibility)
    +--- Algorithms (PageRank, Shortest Paths, Community Detection)
    +--- Plugin SDK (Native + Lua/Python/WASM/JS/Starlark)
    |
    v
Storage (Pages, WAL, B+ Tree, MVCC, Compression)
    |
    v
Core (Graph Trait, Nodes, Relationships, Properties, Labels)
```

## Tech Stack

- **Server**: Rust (edition 2024), Axum, Tokio
- **Browser**: React Router 7, TypeScript, Tailwind CSS v4, D3.js
- **Auth**: jsonwebtoken (RS256), Argon2id
- **Storage**: Custom page-based engine with WAL
- **Minimum Rust Version**: 1.85+

## License

Copyright (c) 2026 Devforge Pty Ltd. All rights reserved.

## Links

- **Website**: [anvildb.com](https://anvildb.com)
- **Install**: `curl -fsSL https://anvildb.com/install.sh | sh`
- **Documentation**: [anvildb.com/docs](https://anvildb.com/docs)
