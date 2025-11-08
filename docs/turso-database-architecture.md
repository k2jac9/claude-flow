# Turso Database Architecture for Rust Agentic Framework
## Edge-Distributed, Rust-Native Database for AI Agents

> **Why Turso?** Built on libSQL (SQLite fork), written in Rust, edge-distributed, and perfect for distributed agent systems.

---

## 🎯 Why Turso is Perfect for Agent Systems

### Key Advantages

| Feature | Benefit for Agent Framework | Impact |
|---------|----------------------------|--------|
| **Rust Native** | Zero-overhead FFI, native integration | 10x faster than Node.js bindings |
| **Edge Distribution** | Agents can sync across geo-locations | Global agent coordination |
| **libSQL Foundation** | SQLite compatibility + extensions | Easy migration, proven reliability |
| **Embedded Mode** | Single-agent scenarios, development | No infrastructure needed |
| **Multi-tenancy** | Separate databases per swarm | Isolation & security |
| **Real-time Sync** | Agent state synchronization | Distributed consensus |
| **Vector Support** | Semantic search (via extensions) | Built-in AI capabilities |

---

## 🏗️ Architecture Comparison

### Turso vs Other Options

```
┌────────────────────────────────────────────────────────┐
│               Database Options Comparison               │
├────────────┬──────────┬──────────┬──────────┬──────────┤
│  Feature   │  Turso   │ SurrealDB│  SQLite  │ Postgres │
├────────────┼──────────┼──────────┼──────────┼──────────┤
│ Rust Native│    ✅    │    ✅    │    ⚠️    │    ❌    │
│ Distributed│    ✅    │    ✅    │    ❌    │    ⚠️    │
│ Embedded   │    ✅    │    ⚠️    │    ✅    │    ❌    │
│ Edge Sync  │    ✅    │    ❌    │    ❌    │    ❌    │
│ SQL Compat │    ✅    │    ⚠️    │    ✅    │    ✅    │
│ Maturity   │    ⚠️    │    ⚠️    │    ✅    │    ✅    │
│ Learning   │   Easy   │  Medium  │   Easy   │   Easy   │
│ Cost       │   Free*  │   Free   │   Free   │   Free   │
└────────────┴──────────┴──────────┴──────────┴──────────┘

* Free tier: 500 databases, 1GB storage, 1B row reads/month
```

### Recommended Hybrid Approach

```
Primary: Turso (Edge + Embedded)
├── Embedded Mode: Local development, single agents
├── Edge Mode: Distributed swarms, production
└── Vector Extension: Semantic search

Complementary: Custom HNSW (Rust)
└── High-performance vector search for specific use cases
```

---

## 🔧 Turso Integration Architecture

### 1. Connection Management

```rust
use libsql::{Builder, Connection, Database};
use std::sync::Arc;
use tokio::sync::RwLock;

#[derive(Clone)]
pub struct TursoManager {
    // Embedded database for local/development
    local_db: Arc<Database>,

    // Remote database for distributed mode
    remote_db: Option<Arc<Database>>,

    // Connection pool
    connections: Arc<RwLock<Vec<Connection>>>,

    config: TursoConfig,
}

#[derive(Debug, Clone)]
pub struct TursoConfig {
    pub mode: DatabaseMode,
    pub local_path: String,
    pub remote_url: Option<String>,
    pub auth_token: Option<String>,
    pub sync_interval: std::time::Duration,
    pub max_connections: usize,
}

#[derive(Debug, Clone)]
pub enum DatabaseMode {
    Embedded,           // Local only (development)
    Remote,             // Cloud only (production)
    Hybrid,             // Local with remote sync (optimal)
}

impl TursoManager {
    pub async fn new(config: TursoConfig) -> Result<Self> {
        let local_db = Builder::new_local(&config.local_path)
            .build()
            .await?;

        let remote_db = if let (Some(url), Some(token)) =
            (&config.remote_url, &config.auth_token) {
            Some(Arc::new(
                Builder::new_remote(url.clone(), token.clone())
                    .build()
                    .await?
            ))
        } else {
            None
        };

        // Initialize connection pool
        let mut connections = Vec::new();
        for _ in 0..config.max_connections {
            let conn = local_db.connect()?;
            connections.push(conn);
        }

        Ok(Self {
            local_db: Arc::new(local_db),
            remote_db,
            connections: Arc::new(RwLock::new(connections)),
            config,
        })
    }

    pub async fn get_connection(&self) -> Result<Connection> {
        let mut conns = self.connections.write().await;

        if let Some(conn) = conns.pop() {
            Ok(conn)
        } else {
            // Create new connection if pool is empty
            Ok(self.local_db.connect()?)
        }
    }

    pub async fn return_connection(&self, conn: Connection) {
        let mut conns = self.connections.write().await;
        if conns.len() < self.config.max_connections {
            conns.push(conn);
        }
    }

    pub async fn sync(&self) -> Result<()> {
        if let Some(remote) = &self.remote_db {
            // Turso handles sync automatically in hybrid mode
            // This is for manual trigger if needed
            self.local_db.sync().await?;
        }
        Ok(())
    }
}
```

### 2. Agent Memory Schema

```rust
use libsql::params;

pub struct AgentMemoryStore {
    manager: Arc<TursoManager>,
}

impl AgentMemoryStore {
    pub async fn initialize_schema(&self) -> Result<()> {
        let conn = self.manager.get_connection().await?;

        // Agents table
        conn.execute(
            r#"
            CREATE TABLE IF NOT EXISTS agents (
                id TEXT PRIMARY KEY,
                agent_type TEXT NOT NULL,
                capabilities TEXT NOT NULL, -- JSON array
                status TEXT NOT NULL,
                metadata TEXT,
                created_at INTEGER NOT NULL,
                updated_at INTEGER NOT NULL
            )
            "#,
            [],
        ).await?;

        // Tasks table
        conn.execute(
            r#"
            CREATE TABLE IF NOT EXISTS tasks (
                id TEXT PRIMARY KEY,
                description TEXT NOT NULL,
                priority INTEGER NOT NULL,
                status TEXT NOT NULL,
                assigned_agents TEXT, -- JSON array of agent IDs
                dependencies TEXT,    -- JSON array of task IDs
                result TEXT,
                created_at INTEGER NOT NULL,
                completed_at INTEGER
            )
            "#,
            [],
        ).await?;

        // Memory table (agent state)
        conn.execute(
            r#"
            CREATE TABLE IF NOT EXISTS memory (
                id TEXT PRIMARY KEY,
                agent_id TEXT NOT NULL,
                key TEXT NOT NULL,
                value TEXT NOT NULL,
                metadata TEXT,
                embedding BLOB,       -- For vector search
                created_at INTEGER NOT NULL,
                ttl INTEGER,
                access_count INTEGER DEFAULT 0,
                UNIQUE(agent_id, key)
            )
            "#,
            [],
        ).await?;

        // Swarms table
        conn.execute(
            r#"
            CREATE TABLE IF NOT EXISTS swarms (
                id TEXT PRIMARY KEY,
                name TEXT NOT NULL,
                topology TEXT NOT NULL,
                config TEXT NOT NULL, -- JSON
                status TEXT NOT NULL,
                created_at INTEGER NOT NULL,
                updated_at INTEGER NOT NULL
            )
            "#,
            [],
        ).await?;

        // Messages table (inter-agent communication)
        conn.execute(
            r#"
            CREATE TABLE IF NOT EXISTS messages (
                id TEXT PRIMARY KEY,
                swarm_id TEXT NOT NULL,
                from_agent TEXT NOT NULL,
                to_agent TEXT,        -- NULL for broadcast
                message_type TEXT NOT NULL,
                content TEXT NOT NULL,
                created_at INTEGER NOT NULL,
                FOREIGN KEY (swarm_id) REFERENCES swarms(id)
            )
            "#,
            [],
        ).await?;

        // Consensus table (voting records)
        conn.execute(
            r#"
            CREATE TABLE IF NOT EXISTS consensus (
                id TEXT PRIMARY KEY,
                swarm_id TEXT NOT NULL,
                proposal_id TEXT NOT NULL,
                agent_id TEXT NOT NULL,
                vote TEXT NOT NULL,   -- approve/reject/abstain
                reasoning TEXT,
                created_at INTEGER NOT NULL,
                FOREIGN KEY (swarm_id) REFERENCES swarms(id)
            )
            "#,
            [],
        ).await?;

        // Create indexes
        conn.execute("CREATE INDEX IF NOT EXISTS idx_memory_agent ON memory(agent_id)", []).await?;
        conn.execute("CREATE INDEX IF NOT EXISTS idx_tasks_status ON tasks(status)", []).await?;
        conn.execute("CREATE INDEX IF NOT EXISTS idx_messages_swarm ON messages(swarm_id)", []).await?;

        self.manager.return_connection(conn).await;
        Ok(())
    }

    pub async fn store_memory(&self, entry: MemoryEntry) -> Result<()> {
        let conn = self.manager.get_connection().await?;

        conn.execute(
            r#"
            INSERT INTO memory (id, agent_id, key, value, metadata, embedding, created_at, ttl)
            VALUES (?1, ?2, ?3, ?4, ?5, ?6, ?7, ?8)
            ON CONFLICT(agent_id, key) DO UPDATE SET
                value = excluded.value,
                metadata = excluded.metadata,
                embedding = excluded.embedding,
                access_count = access_count + 1
            "#,
            params![
                entry.id,
                entry.agent_id,
                entry.key,
                serde_json::to_string(&entry.value)?,
                serde_json::to_string(&entry.metadata)?,
                entry.embedding,
                entry.created_at,
                entry.ttl,
            ],
        ).await?;

        self.manager.return_connection(conn).await;
        Ok(())
    }

    pub async fn retrieve_memory(&self, agent_id: &str, key: &str) -> Result<Option<MemoryEntry>> {
        let conn = self.manager.get_connection().await?;

        let mut stmt = conn.prepare(
            "SELECT id, agent_id, key, value, metadata, embedding, created_at, ttl, access_count
             FROM memory WHERE agent_id = ?1 AND key = ?2"
        ).await?;

        let mut rows = stmt.query(params![agent_id, key]).await?;

        let entry = if let Some(row) = rows.next().await? {
            Some(MemoryEntry {
                id: row.get(0)?,
                agent_id: row.get(1)?,
                key: row.get(2)?,
                value: serde_json::from_str(&row.get::<String>(3)?)?,
                metadata: serde_json::from_str(&row.get::<String>(4)?)?,
                embedding: row.get(5)?,
                created_at: row.get(6)?,
                ttl: row.get(7)?,
            })
        } else {
            None
        };

        self.manager.return_connection(conn).await;
        Ok(entry)
    }

    pub async fn query_memory(
        &self,
        agent_id: &str,
        pattern: &str
    ) -> Result<Vec<MemoryEntry>> {
        let conn = self.manager.get_connection().await?;

        let mut stmt = conn.prepare(
            "SELECT id, agent_id, key, value, metadata, embedding, created_at, ttl
             FROM memory WHERE agent_id = ?1 AND key LIKE ?2
             ORDER BY created_at DESC"
        ).await?;

        let mut rows = stmt.query(params![agent_id, format!("%{}%", pattern)]).await?;
        let mut entries = Vec::new();

        while let Some(row) = rows.next().await? {
            entries.push(MemoryEntry {
                id: row.get(0)?,
                agent_id: row.get(1)?,
                key: row.get(2)?,
                value: serde_json::from_str(&row.get::<String>(3)?)?,
                metadata: serde_json::from_str(&row.get::<String>(4)?)?,
                embedding: row.get(5)?,
                created_at: row.get(6)?,
                ttl: row.get(7)?,
            });
        }

        self.manager.return_connection(conn).await;
        Ok(entries)
    }
}
```

### 3. Edge Replication for Distributed Agents

```rust
use libsql::replication::Replicator;

pub struct DistributedAgentCoordinator {
    turso: Arc<TursoManager>,
    replicas: Vec<ReplicaInfo>,
}

#[derive(Debug, Clone)]
pub struct ReplicaInfo {
    pub location: String,      // us-east, eu-west, ap-south
    pub url: String,
    pub latency_ms: u64,
}

impl DistributedAgentCoordinator {
    pub async fn new(config: TursoConfig) -> Result<Self> {
        let turso = Arc::new(TursoManager::new(config).await?);

        // Discover replicas (Turso provides this automatically)
        let replicas = vec![
            ReplicaInfo {
                location: "us-east".to_string(),
                url: "libsql://your-db-us-east.turso.io".to_string(),
                latency_ms: 50,
            },
            ReplicaInfo {
                location: "eu-west".to_string(),
                url: "libsql://your-db-eu-west.turso.io".to_string(),
                latency_ms: 120,
            },
        ];

        Ok(Self { turso, replicas })
    }

    pub async fn spawn_agent_near_user(
        &self,
        user_location: &str,
        agent_type: AgentType,
    ) -> Result<Agent> {
        // Find nearest replica
        let nearest = self.find_nearest_replica(user_location)?;

        // Create agent with local database connection
        let agent = Agent {
            id: uuid::Uuid::new_v4(),
            agent_type,
            database_url: nearest.url.clone(),
            location: nearest.location.clone(),
        };

        // Register agent in distributed database
        self.register_agent(&agent).await?;

        Ok(agent)
    }

    fn find_nearest_replica(&self, location: &str) -> Result<&ReplicaInfo> {
        // Simple geo-routing logic
        self.replicas.iter()
            .min_by_key(|r| r.latency_ms)
            .ok_or_else(|| anyhow::anyhow!("No replicas available"))
    }

    pub async fn sync_agent_state(&self, agent_id: &str) -> Result<()> {
        // Turso handles this automatically with eventual consistency
        // This method is for explicit sync triggers
        self.turso.sync().await?;
        Ok(())
    }
}
```

### 4. Vector Search with Turso Extensions

```rust
use libsql::params;

pub struct TursoVectorStore {
    manager: Arc<TursoManager>,
}

impl TursoVectorStore {
    pub async fn initialize(&self) -> Result<()> {
        let conn = self.manager.get_connection().await?;

        // Enable vector extension (if available)
        conn.execute(
            r#"
            CREATE VIRTUAL TABLE IF NOT EXISTS vector_index
            USING vec0(
                embedding float[768]
            )
            "#,
            [],
        ).await?;

        // Create mapping table
        conn.execute(
            r#"
            CREATE TABLE IF NOT EXISTS vector_mappings (
                id TEXT PRIMARY KEY,
                vector_id INTEGER,
                content TEXT NOT NULL,
                metadata TEXT,
                FOREIGN KEY (vector_id) REFERENCES vector_index(rowid)
            )
            "#,
            [],
        ).await?;

        self.manager.return_connection(conn).await;
        Ok(())
    }

    pub async fn insert_with_embedding(
        &self,
        id: &str,
        content: &str,
        embedding: &[f32],
        metadata: Option<serde_json::Value>,
    ) -> Result<()> {
        let conn = self.manager.get_connection().await?;

        // Insert vector
        conn.execute(
            "INSERT INTO vector_index (embedding) VALUES (?1)",
            params![embedding],
        ).await?;

        let vector_id = conn.last_insert_rowid();

        // Insert mapping
        conn.execute(
            "INSERT INTO vector_mappings (id, vector_id, content, metadata) VALUES (?1, ?2, ?3, ?4)",
            params![
                id,
                vector_id,
                content,
                serde_json::to_string(&metadata)?,
            ],
        ).await?;

        self.manager.return_connection(conn).await;
        Ok(())
    }

    pub async fn semantic_search(
        &self,
        query_embedding: &[f32],
        limit: usize,
    ) -> Result<Vec<SearchResult>> {
        let conn = self.manager.get_connection().await?;

        let mut stmt = conn.prepare(
            r#"
            SELECT
                vm.id,
                vm.content,
                vm.metadata,
                vec_distance_cosine(vi.embedding, ?1) as distance
            FROM vector_index vi
            JOIN vector_mappings vm ON vi.rowid = vm.vector_id
            ORDER BY distance ASC
            LIMIT ?2
            "#
        ).await?;

        let mut rows = stmt.query(params![query_embedding, limit]).await?;
        let mut results = Vec::new();

        while let Some(row) = rows.next().await? {
            results.push(SearchResult {
                id: row.get(0)?,
                content: row.get(1)?,
                metadata: serde_json::from_str(&row.get::<String>(2)?)?,
                distance: row.get(3)?,
            });
        }

        self.manager.return_connection(conn).await;
        Ok(results)
    }
}
```

---

## 🌍 Deployment Architectures

### 1. Single Developer (Embedded Mode)

```rust
// Development configuration
let config = TursoConfig {
    mode: DatabaseMode::Embedded,
    local_path: "./dev.db".to_string(),
    remote_url: None,
    auth_token: None,
    sync_interval: Duration::from_secs(0),
    max_connections: 5,
};

let manager = TursoManager::new(config).await?;
```

### 2. Small Team (Hybrid Mode)

```rust
// Hybrid: local development + shared remote
let config = TursoConfig {
    mode: DatabaseMode::Hybrid,
    local_path: "./local.db".to_string(),
    remote_url: Some("libsql://team-db.turso.io".to_string()),
    auth_token: Some(std::env::var("TURSO_AUTH_TOKEN")?),
    sync_interval: Duration::from_secs(5),
    max_connections: 10,
};

let manager = TursoManager::new(config).await?;
```

### 3. Production (Distributed Edge)

```rust
// Production: multi-region edge deployment
let config = TursoConfig {
    mode: DatabaseMode::Remote,
    local_path: "./cache.db".to_string(), // Local cache
    remote_url: Some("libsql://prod-db.turso.io".to_string()),
    auth_token: Some(std::env::var("TURSO_AUTH_TOKEN")?),
    sync_interval: Duration::from_secs(1),
    max_connections: 50,
};

let manager = TursoManager::new(config).await?;

// Deploy agents to multiple regions
let coordinator = DistributedAgentCoordinator::new(config).await?;
coordinator.spawn_agent_near_user("us-east", AgentType::Researcher).await?;
coordinator.spawn_agent_near_user("eu-west", AgentType::Coder).await?;
```

---

## 📊 Performance Characteristics

### Latency Comparison

```
Operation              | Local SQLite | Turso Embedded | Turso Edge
-----------------------|--------------|----------------|------------
Single Read            | 0.1ms        | 0.1ms         | 5-50ms*
Single Write           | 0.5ms        | 0.5ms         | 5-50ms*
Batch Read (100)       | 5ms          | 5ms           | 20-100ms*
Batch Write (100)      | 50ms         | 50ms          | 50-200ms*
Vector Search (k=10)   | 10ms         | 10ms          | 15-60ms*

* Depends on user location relative to nearest edge
```

### Scalability

```
Metric                 | Turso Capability
-----------------------|----------------------------------
Max Databases          | 500 (free), unlimited (paid)
Max Connections        | 1000+ per database
Max Database Size      | Unlimited (paid)
Replication Regions    | 30+ edge locations
Sync Latency           | <100ms global average
```

---

## 💰 Cost Analysis

### Turso Pricing

```yaml
Free Tier (Starter):
  - Databases: 500
  - Storage: 1 GB per database
  - Rows Read: 1 billion/month
  - Rows Written: 25 million/month
  - Locations: 3 primary
  - Cost: $0

Scaler ($29/month):
  - Databases: Unlimited
  - Storage: 100 GB included
  - Rows Read: 1 billion included
  - Rows Written: 100 million included
  - Locations: All edge locations
  - Cost: $29/month + overages

Enterprise:
  - Custom pricing
  - Dedicated support
  - SLA guarantees
```

### Cost Comparison

```
Scenario: 10 swarms, 100 agents, 1M operations/day

Turso:
  - Free tier: $0 (well within limits)
  - Scaler tier: $29/month

Traditional Stack (PostgreSQL + Redis + Vector DB):
  - PostgreSQL: $20/month (managed)
  - Redis: $15/month (managed)
  - Vector DB: $30/month (Pinecone starter)
  - Total: $65/month

Savings: 55% cost reduction with Turso
```

---

## 🔐 Security Features

### Built-in Security

```rust
pub struct SecureTursoManager {
    manager: Arc<TursoManager>,
    encryption_key: Secret<String>,
}

impl SecureTursoManager {
    pub async fn store_sensitive(
        &self,
        agent_id: &str,
        key: &str,
        value: &str,
    ) -> Result<()> {
        // Encrypt before storing
        let encrypted = self.encrypt(value)?;

        let entry = MemoryEntry {
            id: uuid::Uuid::new_v4().to_string(),
            agent_id: agent_id.to_string(),
            key: key.to_string(),
            value: serde_json::json!({ "encrypted": encrypted }),
            metadata: serde_json::json!({ "encrypted": true }),
            embedding: None,
            created_at: chrono::Utc::now().timestamp(),
            ttl: None,
        };

        // Store encrypted data
        self.store_memory(entry).await?;

        Ok(())
    }

    pub async fn retrieve_sensitive(
        &self,
        agent_id: &str,
        key: &str,
    ) -> Result<Option<String>> {
        let entry = self.retrieve_memory(agent_id, key).await?;

        if let Some(entry) = entry {
            let encrypted = entry.value.get("encrypted")
                .and_then(|v| v.as_str())
                .ok_or_else(|| anyhow::anyhow!("Invalid encrypted data"))?;

            let decrypted = self.decrypt(encrypted)?;
            Ok(Some(decrypted))
        } else {
            Ok(None)
        }
    }

    fn encrypt(&self, plaintext: &str) -> Result<String> {
        // Use encryption key
        todo!("Implement encryption")
    }

    fn decrypt(&self, ciphertext: &str) -> Result<String> {
        // Use encryption key
        todo!("Implement decryption")
    }
}
```

---

## 🚀 Migration Path

### From SQLite

```rust
pub async fn migrate_from_sqlite(
    sqlite_path: &str,
    turso_config: TursoConfig,
) -> Result<()> {
    // Open SQLite
    let sqlite = sqlx::SqlitePool::connect(sqlite_path).await?;

    // Initialize Turso
    let turso = TursoManager::new(turso_config).await?;
    let conn = turso.get_connection().await?;

    // Copy tables
    let tables = sqlx::query("SELECT name FROM sqlite_master WHERE type='table'")
        .fetch_all(&sqlite)
        .await?;

    for table in tables {
        let table_name: String = table.get("name");

        // Get schema
        let schema = sqlx::query(&format!("SELECT sql FROM sqlite_master WHERE name = '{}'", table_name))
            .fetch_one(&sqlite)
            .await?;

        let create_sql: String = schema.get("sql");

        // Create in Turso
        conn.execute(&create_sql, []).await?;

        // Copy data
        let rows = sqlx::query(&format!("SELECT * FROM {}", table_name))
            .fetch_all(&sqlite)
            .await?;

        // Insert into Turso
        for row in rows {
            // Convert and insert
            // (implementation depends on table structure)
        }
    }

    turso.return_connection(conn).await;
    Ok(())
}
```

---

## 📚 Cargo Dependencies

```toml
[dependencies]
# Turso/libSQL
libsql = "0.4"

# Async runtime
tokio = { version = "1.35", features = ["full"] }

# Serialization
serde = { version = "1.0", features = ["derive"] }
serde_json = "1.0"

# Error handling
anyhow = "1.0"
thiserror = "1.0"

# Utilities
uuid = { version = "1.6", features = ["v4"] }
chrono = { version = "0.4", features = ["serde"] }

# Security
secrecy = "0.8"
```

---

## 🎯 Recommended Architecture

```
┌─────────────────────────────────────────────────┐
│         Rust Agentic Framework Stack            │
├─────────────────────────────────────────────────┤
│  Primary Database: Turso (libSQL)               │
│  ├── Embedded: Development & single agents      │
│  ├── Hybrid: Team collaboration                 │
│  └── Edge: Production distributed swarms        │
│                                                  │
│  Vector Search: Turso vec0 + Custom HNSW        │
│  ├── Turso vec0: Integrated semantic search     │
│  └── Custom HNSW: High-performance use cases    │
│                                                  │
│  Caching: In-memory LRU + Turso local cache     │
│                                                  │
│  Replication: Automatic edge sync (Turso)       │
└─────────────────────────────────────────────────┘
```

---

## ✅ Decision: Use Turso as Primary Database

**Reasons:**
1. ✅ **Rust-native** - Perfect integration
2. ✅ **Edge distribution** - Built-in global sync
3. ✅ **Embedded + Distributed** - Flexibility for all scenarios
4. ✅ **SQLite compatible** - Easy migration, familiar SQL
5. ✅ **Cost-effective** - Generous free tier
6. ✅ **Vector support** - vec0 extension for AI workloads
7. ✅ **Active development** - Modern, well-maintained
8. ✅ **Simple API** - Clean Rust SDK

**Trade-offs:**
- ⚠️ Newer technology (less mature than PostgreSQL)
- ⚠️ Vendor lock-in (but can export to SQLite)
- ⚠️ Vector search not as advanced as specialized DBs

**Verdict:** Turso is the **optimal choice** for a Rust-based agentic framework! 🎯

---

## 📖 Next Steps

1. **Prototype** Turso integration with basic agent storage
2. **Benchmark** performance vs SQLite/SurrealDB
3. **Test** edge replication with distributed agents
4. **Implement** vector search with vec0 extension
5. **Build** migration tools from claude-flow's SQLite

Ready to build with Turso? 🚀
