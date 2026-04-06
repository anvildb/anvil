# Anvil DB

**A Neo4j-inspired graph database written in Rust** — combining a property graph, native document store (NoSQL), and row-level security into a single binary.

Anvil eliminates the need to run separate databases for graph queries and document lookups. Store your data once, query it with Cypher, GraphQL, or REST, and let sync rules keep everything consistent.

## Install

### Linux / macOS

```bash
curl -fsSL https://anvildb.com/install.sh | sh
```

### Windows

```powershell
irm https://anvildb.com/install.ps1 | iex
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

### Install Paths

| | Linux / macOS | Windows |
|---|---|---|
| Binary | `/usr/local/bin/anvil` | `%LOCALAPPDATA%\Anvil\anvil.exe` |
| Config | `/etc/anvil/anvil.toml` | `%LOCALAPPDATA%\Anvil\anvil.toml` |
| Data | `/var/lib/anvil/` | `%LOCALAPPDATA%\Anvil\data\` |
| Logs | `/var/log/anvil/anvil.log` | `%LOCALAPPDATA%\Anvil\log\anvil.log` |

For local development builds (`cargo run`), paths default to `./data`, `./anvil.toml`, and stdout logging.

## Quick Start

```bash
# Start the server (daemonizes, logs to data/anvil.log)
anvil start

# Or run in foreground
anvil start --foreground

# Check status
anvil status

# Stop the server
anvil stop

# Import sample data
anvil import cypher ./movies.cypher

# Open the browser UI
open http://localhost:5175

# Default login: admin / anvil
```

The server runs on port **7474** (HTTP API) and **7687** (Bolt protocol). The browser UI runs on port **5175**.

## Features

### Cypher Query Language

Full Cypher implementation with multi-hop pattern matching, quantified paths, aggregation, and path finding.

```cypher
-- Create nodes and relationships
CREATE (a:Person {name: "Alice", age: 30})-[:KNOWS]->(b:Person {name: "Bob", age: 25})

-- Pattern matching with WHERE, ORDER BY, LIMIT
MATCH (n:Person)-[r:KNOWS]->(m:Person)
WHERE n.age > 25
RETURN n.name, m.name
ORDER BY n.name
LIMIT 10

-- Multi-hop traversals with quantified paths
MATCH (a:Person {name: 'Alice'})-[:KNOWS*1..5]->(f)
RETURN DISTINCT f.name, min(length(p)) AS depth

-- ALL SHORTEST paths
MATCH p = ALL SHORTEST (a:Person {name: 'Alice'})--+(b:Person {name: 'Bob'})
RETURN length(p) AS depth

-- Comma-separated patterns (join on shared variables)
MATCH (a:Person)-[:ACTED_IN]->(m:Movie), (b:Person)-[:DIRECTED]->(m)
RETURN a.name AS actor, b.name AS director, m.title

-- Negated relationship types
MATCH (a:Person)-[:!ACTED_IN]->(m:Movie)
RETURN a.name, m.title

-- MERGE with ON CREATE SET
MERGE (m:Movie {title: 'The Matrix'})
ON CREATE SET m.released = 1999, m.tagline = 'Welcome to the Real World'

-- Aggregation
MATCH (p:Person)-[:ACTED_IN]->(m:Movie)
RETURN p.name, collect(m.title) AS movies, count(m) AS movieCount
ORDER BY movieCount DESC
```

### Data Import

Import Neo4j-compatible Cypher scripts via CLI, REST API, or browser UI.

```bash
# CLI import
anvil import cypher ./movies.cypher

# REST API import
curl -X POST http://localhost:7474/db/import/cypher \
  -H "Content-Type: text/plain" \
  -H "Authorization: Bearer $TOKEN" \
  --data-binary @movies.cypher
# Response: {"total":2, "success":2, "nodesCreated":171, "relationshipsCreated":253, "errors":[]}
```

Supports multi-statement scripts (semicolon-separated), multi-CREATE without semicolons (Neo4j `initialDatabase.cypher` format), MERGE with ON CREATE SET, and variable binding across CREATE clauses.

### Native Document Store (NoSQL)

Built-in document collections with O(1) key lookups, composite keys, TTL, secondary indexes, batch operations, and filter expressions — no separate database needed.

Collections are organized into schemas: `public` (default) for application data, and `auth` for system collections.

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

Automatically keep graph nodes and document collections in sync.

```cypher
SYNC LABEL Person TO COLLECTION persons KEY name INCLUDE name, email, age
```

Now:
- Creating a `:Person` node auto-creates a document in `persons`
- Inserting a document in `persons` auto-creates a `:Person` node
- Updates propagate bidirectionally
- The `INCLUDE` clause controls which fields sync

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

### Authentication (JWT + JWKS)

Graph-native authentication with users and roles stored in schema-namespaced collections (`auth.*`) and synced to graph nodes.

- **RS256 JWT tokens** — asymmetric signing with persistent RSA key pair, 1-hour access tokens, 7-day refresh tokens
- **JWKS endpoint** — `GET /.well-known/jwks.json` for external token verification
- **Argon2id password hashing** — OWASP-recommended
- **Per-request session middleware** — identity extracted from Bearer token on every request
- **Graph sync** — `:User` and `:Role` nodes with `[:HAS_ROLE]` relationships
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

# Refresh an expired access token
curl -X POST http://localhost:7474/auth/refresh \
  -d '{"refresh_token": "abc..."}'
```

### Row-Level Security (RLS)

PostgreSQL-inspired fine-grained access control at the data level.

```cypher
-- Enable RLS on a label
ENABLE ROW LEVEL SECURITY ON :Project

-- Users can only see their own data
CREATE POLICY own_data ON :Project FOR SELECT TO reader
  USING (n.owner = current_user())

-- Multi-tenant isolation
CREATE POLICY tenant ON :Project FOR ALL TO authenticated
  USING (n.tenant_id = session('tenant_id'))
```

### Stored Functions & Triggers

```cypher
-- Create a stored function
CREATE FUNCTION greet(name STRING) RETURNS STRING
  RETURN 'Hello, ' + name + '!'

-- Create a trigger
CREATE TRIGGER audit_insert AFTER INSERT ON :Person
  SET NEW.created_at = timestamp()
```

### Edge Functions

JavaScript/TypeScript functions executed via HTTP endpoints with Deno or Node.js runtime.

```cypher
CREATE EDGE FUNCTION hello RUNTIME 'deno' ENTRYPOINT 'functions/hello.ts'
```

```bash
curl http://localhost:7474/edge/hello?name=Alice
# {"message": "Hello, Alice!"}
```

### Browser UI

React Router 7 web application with:

- **Login Screen** — JWT authentication with forced password change on first login
- **Cypher Editor** — syntax highlighting, Cmd+Enter execution, table/JSON/graph result views
- **GraphQL Playground** — query editor with introspection
- **Graph Visualization** — force-directed layout (+ hierarchical, circular, grid), zoom/pan, lasso select, minimap, PNG/SVG export, **focus mode** (click a node to orbit its direct connections in an evenly-spaced circle with highlighted edges/labels, non-neighbors pushed to periphery)
- **Data Import** — paste or upload Cypher scripts, displays nodes/relationships created
- **Document Store Manager** — collection CRUD, document browsing, JSON editor, sync rules
- **RLS Policy Manager** — create/drop policies, enable/disable RLS, policy simulator
- **Stored Functions & Triggers** — create, list, and manage
- **Schema Browser** — labels, relationship types, property keys, indexes, constraints
- **Schema Dropdown** — switch between `public` and `auth` schemas
- **Monitoring Dashboard** — real-time stats
- **Admin Panel** — user management, role management

### Additional Features

- **ACID Transactions** — MVCC with snapshot isolation, record-level locking, deadlock detection, ARIES-style WAL recovery
- **Storage Engine** — page-based with LRU cache, WAL with CRC32 checksums, B+ tree indexes (unique, composite, full-text, spatial), LZ4/Zstd compression
- **Graph Algorithms** — PageRank, Dijkstra/A*, BFS/DFS, Louvain community detection, connected components, betweenness/closeness/degree centrality, MST, triangle counting, node similarity
- **Clustering** — leader-follower WAL streaming replication, Raft consensus, read replicas, sharding
- **Plugin System** — native Rust + Lua/Python/WASM/JS/Starlark scripting runtimes
- **Built-in `apoc` Library** — 55 functions: bitwise, geospatial, collections, text similarity, conversions, math, date/time, graph refactoring, import/export
- **Client Drivers** — Rust, TypeScript, Python, Go with `anvil://` URI scheme
- **Observability** — event log, query analytics, configurable alerts

## Configuration

Config file is searched in order: `./anvil.toml` (local), `/etc/anvil/anvil.toml` (system), `%LOCALAPPDATA%\Anvil\anvil.toml` (Windows). Precedence: CLI flags > env vars > config file > defaults.

```toml
# anvil.toml
[server]
http_port = 7474
bolt_port = 7687
bind_address = "0.0.0.0"

[storage]
data_dir = "/var/lib/anvil"       # or "./data" for development
document_storage = "unified"       # or "split"

[logging]
level = "info"
log_file = "/var/log/anvil/anvil.log"  # empty = stdout

[auth]
enabled = true
default_password = "anvil"
access_token_ttl_secs = 3600      # 1 hour
refresh_token_ttl_secs = 604800   # 7 days
# RSA key pair auto-generated and persisted to data_dir/jwt_key.pem
```

Or use environment variables:

```bash
ANVIL_HTTP_PORT=8080 ANVIL_DATA_DIR=/var/lib/anvil anvil start
```

## API Endpoints

### Core
| Method | Endpoint | Description |
|--------|----------|-------------|
| GET | `/` | Server info |
| GET | `/health` | Health check |
| POST | `/db/query` | Execute Cypher query |
| POST | `/db/import/cypher` | Import Cypher script (returns nodes/rels created) |
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

### Edge Functions
| Method | Endpoint | Description |
|--------|----------|-------------|
| ANY | `/edge/{name}` | Invoke edge function |

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
    +--- Edge Functions (Deno/Node.js subprocess runtime)
    +--- Stored Functions & Triggers
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
