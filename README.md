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

## Roadmap

**761 of 985 tasks completed (77%)**

### Phase 0: Project Scaffolding — Complete (8/8)

Rust workspace, React Router 7 browser app, CI, tooling.

### Phase 1: Core Graph Data Model — Complete (33/33)

- [x] **Primitive Types** — NodeId, RelationshipId, LabelId, PropertyValue enum
- [x] **Graph Entities** — Node, Relationship, Path structs with builder patterns
- [x] **Schema & Constraints** — Label/type registries, unique/existence/key constraints
- [x] **Graph API** — Full CRUD trait with label operations, traversals, statistics

### Phase 2: Storage Engine — Complete (30/30)

- [x] **Page-Based Store** — Fixed-size pages, LRU cache, dirty tracking, mmap option
- [x] **Record Stores** — Node, relationship, property, string, array, label stores
- [x] **ID Allocation** — Sequential with free-list recycling, thread-safe
- [x] **Write-Ahead Log** — Segmented WAL, fsync on commit, crash recovery, checkpointing
- [x] **File Layout** — graph.db, properties.db, strings.db, WAL directory, lock file

### Phase 3: Indexing — Complete (19/19)

- [x] **B+ Tree** — On-disk, page-backed, range scans, prefix scans, bulk loading, concurrent access
- [x] **Index Types** — Single-property, composite, unique, full-text (BM25), spatial (R-tree), index-only scans
- [x] **Index Management** — CREATE/DROP INDEX, automatic maintenance, statistics

### Phase 4: Transaction Engine — Complete (17/17)

- [x] **Transaction Manager** — Begin/commit/rollback, savepoints, read-only mode
- [x] **Concurrency Control** — MVCC version chains, 3 isolation levels, deadlock detection via wait-for graph
- [x] **Recovery** — ARIES-style redo/undo, WAL replay, transaction table reconstruction
- [x] **Garbage Collection** — Version chain pruning, oldest-active tracking

### Phase 5: Cypher Query Language — Complete (78/78)

- [x] **Lexer** — Hand-written with 40+ token types, string escapes, Unicode, nested comments
- [x] **Parser** — Recursive descent, full Cypher grammar, MATCH/CREATE/MERGE/DELETE/SET/REMOVE/WITH/UNWIND/FOREACH/CALL/UNION
- [x] **Semantic Analysis** — Variable scoping, type inference, undefined variable detection
- [x] **Built-in Functions** — 30+ functions: aggregation, scalar, string, math, list, path
- [x] **Query Planner** — Cost-based optimizer, statistics-driven, index selection, join ordering
- [x] **Query Executor** — Volcano model, 25+ operators, hash/merge joins, parallel aggregation

### Phase 6: GraphQL API — Complete (36/36)

- [x] **Schema Generation** — Auto-generated from graph structure, Relay-spec connections
- [x] **Queries** — Filtering, ordering, pagination, nested traversals
- [x] **Mutations** — Create/update/delete nodes and relationships
- [x] **Subscriptions** — Real-time via graphql-transport-ws
- [x] **Cypher Passthrough** — Execute raw Cypher via GraphQL
- [x] **Integration** — Introspection, error handling, batch DataLoader

### Phase 7: Server — Complete (39/39)

- [x] **HTTP Server** — Axum/Tokio, CORS, REST + GraphQL + WebSocket endpoints
- [x] **Authentication** — JWT RS256, JWKS, Argon2id, per-request session middleware
- [x] **Configuration** — TOML config, env vars, CLI flags, system path detection
- [x] **Multi-Database** — Database routing, schema isolation
- [x] **Connection Management** — Pool limits, idle timeout, graceful shutdown

### Phase 8: CLI — Complete (20/20)

- [x] **Server Management** — start/stop/status, daemon mode, PID file, log file
- [x] **Interactive Shell** — Cypher REPL, history, formatting, parameter binding
- [x] **Data Operations** — Import Cypher scripts, config init, config display

### Phase 9: Browser UI — Complete (65/65)

- [x] **Query Editor** — Cypher highlighting, Cmd+Enter, table/JSON/graph/plan views, settings-aware defaults
- [x] **Graph Visualization** — Force-directed (+ hierarchical, circular, grid), focus mode with neighbor orbit, lasso select, minimap, PNG/SVG export
- [x] **Schema Browser** — Labels, types, property keys, indexes, constraints, schema dropdown
- [x] **Document Manager** — Collection CRUD, document browsing, sync rules, schema-filtered
- [x] **Admin Panel** — User/role management, event log, alerts (admin-only)
- [x] **Monitoring** — Active queries, store sizes, memory, throughput, slow query log
- [x] **Settings** — Theme, editor preferences, graph defaults, connection profiles
- [x] **Data Import** — Cypher script paste/upload with node/relationship counts
- [x] **Security** — Role-based UI (admin-only nav items, schema dropdown), JWT role extraction

### Phase 10: Graph Algorithms — Complete (16/16)

PageRank, Dijkstra/A*, BFS/DFS, Louvain, label propagation, connected components, betweenness/closeness/degree centrality, MST, triangle counting, node similarity.

### Phase 11: Performance & Scalability — Complete (19/19)

- [x] **Query Performance** — Plan caching, result caching, parallel operators
- [x] **Storage Performance** — Page cache tuning, compression (LZ4/Zstd), tiered storage
- [x] **Concurrency** — Lock-free readers, partitioned indexes, work stealing
- [x] **Benchmarking** — Criterion benchmarks, LDBC SNB workload, regression tracking

### Phase 12: Operations & Reliability — Complete (13/13)

- [x] **Backup & Restore** — Online snapshots, incremental backup, point-in-time recovery
- [x] **Observability** — Prometheus metrics, structured logging, OpenTelemetry tracing
- [x] **Data Integrity** — CRC32 checksums, consistency checks, repair tools

### Phase 13: Clustering & Replication — Complete (7/7)

Raft consensus, WAL streaming replication, leader-follower architecture, read replicas, hash/label-based sharding, rolling upgrades, gossip protocol.

### Phase 14: Ecosystem — Complete (22/22)

- [x] **Client Drivers** — Rust, TypeScript, Python, Go with `anvil://` URI scheme
- [x] **Migration Tooling** — Neo4j Cypher dump import, CSV, GraphML, JSON-LD
- [x] **Documentation** — mdBook site, Cypher reference, API docs, architecture guide, tutorials
- [x] **Testing** — 2,230+ tests, integration tests, property-based tests, fuzz testing

### Phase 15: Plugin System — Complete (98/98)

- [x] **Plugin Framework** — Native Rust + Lua/Python/WASM/JS/TS/Starlark, sandboxed, hot-reload, namespace isolation
- [x] **APOC Library** — 55 built-in functions: bitwise, geospatial (Haversine), collections, text similarity (Levenshtein, Jaro-Winkler), conversions, math, date/time, graph refactoring, import/export

### Phase 16: Native Document Store — Complete (57/57)

- [x] **Document Model** — Schema-namespaced collections, composite keys, TTL, secondary indexes (GSI/LSI)
- [x] **Query API** — Filter expressions, batch operations, paginated scan, REST endpoints
- [x] **Graph-Document Sync** — Bidirectional sync rules, conflict resolution, INCLUDE clause
- [x] **Storage Modes** — Unified (single file), split (separate files), shared/separate WAL
- [x] **Transactions** — Document operations participate in graph transactions

### Phase 17: Row-Level Security — Complete (48/48)

- [x] **Policy DDL** — CREATE/DROP POLICY, ENABLE/DISABLE RLS, FORCE RLS
- [x] **Policy Predicates** — current_user(), current_roles(), session(), permissive/restrictive modes
- [x] **Sync Pairs** — Policies on labels automatically protect paired document collections
- [x] **Schema Protection** — Label reservation by schema, cross-schema mutation blocking
- [x] **Admin Tools** — Policy simulator, audit logging, policy browser in UI

### Phase 18: Database Functions — Complete (33/33)

- [x] **Function DDL** — CREATE/DROP/SHOW FUNCTION, typed parameters with defaults
- [x] **Execution** — CALL...YIELD, inline in expressions, body caching
- [x] **Schema Functions** — Namespace-qualified functions, collection/graph access
- [x] **Browser UI** — Create/edit/delete functions, test panel, call log

### Phase 19: Triggers — Complete (39/39)

- [x] **Trigger DDL** — CREATE/DROP/ENABLE/DISABLE TRIGGER, priority ordering
- [x] **Event Types** — BEFORE/AFTER on INSERT/UPDATE/DELETE, label and collection targets
- [x] **Trigger Body** — OLD/NEW variables, RAISE, SET, conditional logic
- [x] **Execution** — Cascade detection (max 16 levels), sync rule interaction
- [x] **Browser UI** — Create/edit triggers, activity log, dependency analysis

### Phase 20: Observability — Complete (46/46)

- [x] **Event Log** — Ring buffer with configurable retention, structured event types
- [x] **Query Analytics** — p95/p99 latency, slow query detection, top-N queries
- [x] **Dependency Analysis** — Trigger cascades, function calls, sync rule interactions
- [x] **Alerts** — Configurable rules, error rate thresholds, built-in defaults
- [x] **Browser UI** — Event log explorer, query stats, alerts panel

### Phase 21: Data Import — In Progress (7/57)

- [x] **Cypher Script Import** — CLI, REST API, browser UI, Neo4j-compatible format
- [ ] **CSV Import** — Header mapping, type inference, batch loading
- [ ] **JSON Import** — Nested document to graph mapping
- [ ] **GraphML Import** — Standard graph interchange format
- [ ] **Export** — Cypher dump, CSV, JSON, GraphML
- [ ] **Example Datasets** — Movies, social network, recommendations
- [ ] **Documentation** — Import/export guides

### Phase 22: Edge Functions — In Progress (6/64)

- [x] **Function Runtime** — Deno/Node.js subprocess execution, HTTP triggers (partial)
- [ ] **Deployment & Management** — Versioning, rollback, resource limits
- [ ] **Database Access** — Graph/document queries from function context
- [ ] **Environment Variables & Secrets** — Encrypted secret storage
- [ ] **Local Development** — Hot reload, debug logging, test harness
- [ ] **Browser UI** — Function editor, logs, deployment status
- [ ] **Documentation** — Guides and API reference

### Phase 23: Hooks — Planned (0/44)

- [ ] **Hook Definition** — CREATE/DROP/ENABLE/DISABLE HOOK DDL
- [ ] **System Events** — ON SERVER START/SHUTDOWN, ON CONNECT/DISCONNECT, ON SCHEMA CHANGE, ON BACKUP
- [ ] **Scheduled Hooks** — Cron expressions, interval-based, distributed lock for single execution
- [ ] **Webhook Integration** — HTTP POST to external URLs, retry policy, signature verification
- [ ] **Hook Body** — Cypher execution, variable access, error handling
- [ ] **Browser UI** — Hook management, execution history
- [ ] **Documentation** — Overview, reference, tutorials

### Phase 24: Bolt Protocol — Planned (0/76)

- [ ] **PackStream Serialization** — Binary encode/decode for all Cypher types (nodes, rels, paths, scalars)
- [ ] **Chunked Transport** — u16 length-prefixed chunks over TCP, async with Tokio
- [ ] **Bolt Handshake** — Magic preamble, version negotiation (Bolt v4.4/v5.0)
- [ ] **Authentication** — HELLO/LOGON messages, credential validation against auth.users
- [ ] **Query Execution** — RUN/PULL/DISCARD messages, streaming records
- [ ] **Transactions** — BEGIN/COMMIT/ROLLBACK, auto-commit mode, state machine
- [ ] **Error Handling** — Neo4j-compatible error codes, FAILURE/RESET messages
- [ ] **Server Integration** — TCP listener alongside HTTP, shared AppState, graceful shutdown
- [ ] **Routing** — ROUTE message for standalone mode, multi-database via routing context
- [ ] **Driver Compatibility** — Test with official Neo4j drivers (Python, JS, Java, Go)
- [ ] **Performance** — Bolt vs HTTP benchmarks, zero-copy encoding, backpressure
- [ ] **TLS Support** — Optional bolt+s://, certificate configuration
- [ ] **Documentation** — Protocol overview, driver connection guides, compatibility matrix

## License

Copyright (c) 2026 Devforge Pty Ltd. All rights reserved.

## Links

- **Website**: [anvildb.com](https://anvildb.com)
- **Install**: `curl -fsSL https://anvildb.com/install.sh | sh`
- **Documentation**: [anvildb.com/docs](https://anvildb.com/docs)
