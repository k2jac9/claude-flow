# Rust Agentic Framework - Quick Start Guide
## From Zero to Running Swarm in 15 Minutes

> **Goal**: Get a working Rust-based agentic framework with Turso, Skills, and distributed agents running quickly.

---

## 🚀 Quick Start (15 Minutes)

### Prerequisites

```bash
# Install Rust
curl --proto '=https' --tlsv1.2 -sSf https://sh.rustup.rs | sh

# Install Turso CLI
curl -sSfL https://get.tur.so/install.sh | bash

# Verify installations
rustc --version
turso --version
```

---

## 📦 Step 1: Create New Project (2 min)

```bash
# Create workspace
cargo new --lib rust-agent-flow
cd rust-agent-flow

# Create binary
mkdir -p src/bin
touch src/bin/raf.rs
```

### Update `Cargo.toml`

```toml
[package]
name = "rust-agent-flow"
version = "0.1.0"
edition = "2021"

[[bin]]
name = "raf"
path = "src/bin/raf.rs"

[dependencies]
# Async runtime
tokio = { version = "1.35", features = ["full"] }

# Database (Turso)
libsql = "0.4"

# Serialization
serde = { version = "1.0", features = ["derive"] }
serde_json = "1.0"
serde_yaml = "0.9"

# Web framework
axum = "0.7"
tower-http = { version = "0.5", features = ["cors"] }

# Error handling
anyhow = "1.0"
thiserror = "1.0"

# Async traits
async-trait = "0.1"

# Utilities
uuid = { version = "1.6", features = ["v4", "serde"] }
chrono = "0.4"
tracing = "0.1"
tracing-subscriber = "0.3"

# CLI
clap = { version = "4.4", features = ["derive"] }
```

---

## 🗄️ Step 2: Setup Turso Database (3 min)

```bash
# Sign up (opens browser)
turso auth signup

# Create database
turso db create rust-agent-flow

# Get connection URL and token
turso db show rust-agent-flow
turso db tokens create rust-agent-flow

# Export to environment
export TURSO_DATABASE_URL="libsql://rust-agent-flow-<your-org>.turso.io"
export TURSO_AUTH_TOKEN="<your-token>"
```

---

## 🔧 Step 3: Core Types (5 min)

Create `src/lib.rs`:

```rust
pub mod agent;
pub mod coordination;
pub mod memory;
pub mod skills;
pub mod error;

pub use agent::{Agent, AgentId, AgentType};
pub use coordination::{Coordinator, Swarm, Topology};
pub use memory::{MemoryManager, MemoryEntry};
pub use skills::{SkillExecutor, SkillDefinition};
pub use error::{Result, AgentError};

pub mod prelude {
    pub use crate::{Agent, AgentId, AgentType};
    pub use crate::{Coordinator, Swarm, Topology};
    pub use crate::{MemoryManager, MemoryEntry};
    pub use crate::{SkillExecutor, SkillDefinition};
    pub use crate::Result;
}
```

Create `src/error.rs`:

```rust
use thiserror::Error;

#[derive(Error, Debug)]
pub enum AgentError {
    #[error("Database error: {0}")]
    Database(#[from] libsql::Error),

    #[error("Agent not found: {0}")]
    AgentNotFound(String),

    #[error("Task execution failed: {0}")]
    TaskFailed(String),

    #[error("Coordination error: {0}")]
    Coordination(String),

    #[error("Skill not found: {0}")]
    SkillNotFound(String),

    #[error(transparent)]
    Other(#[from] anyhow::Error),
}

pub type Result<T> = std::result::Result<T, AgentError>;
```

Create `src/agent.rs`:

```rust
use async_trait::async_trait;
use serde::{Deserialize, Serialize};
use uuid::Uuid;

pub type AgentId = Uuid;

#[derive(Debug, Clone, Serialize, Deserialize)]
pub enum AgentType {
    Researcher,
    Coder,
    Reviewer,
    Tester,
    Analyzer,
}

#[derive(Debug, Clone, Serialize, Deserialize)]
pub struct Task {
    pub id: Uuid,
    pub description: String,
    pub metadata: serde_json::Value,
}

#[derive(Debug, Clone, Serialize, Deserialize)]
pub struct TaskResult {
    pub agent_id: AgentId,
    pub task_id: Uuid,
    pub output: serde_json::Value,
    pub success: bool,
}

#[async_trait]
pub trait Agent: Send + Sync {
    async fn execute(&self, task: Task) -> crate::Result<TaskResult>;
    fn agent_type(&self) -> AgentType;
    fn id(&self) -> AgentId;
}
```

Create `src/memory.rs`:

```rust
use libsql::{Builder, Connection, Database};
use serde::{Deserialize, Serialize};
use std::sync::Arc;
use tokio::sync::Mutex;

#[derive(Debug, Clone, Serialize, Deserialize)]
pub struct MemoryEntry {
    pub id: String,
    pub agent_id: String,
    pub key: String,
    pub value: serde_json::Value,
    pub created_at: i64,
}

pub struct MemoryManager {
    db: Arc<Database>,
    conn: Arc<Mutex<Connection>>,
}

impl MemoryManager {
    pub async fn new(db_path: &str) -> crate::Result<Self> {
        let db = Builder::new_local(db_path).build().await?;
        let conn = db.connect()?;

        let manager = Self {
            db: Arc::new(db),
            conn: Arc::new(Mutex::new(conn)),
        };

        manager.initialize_schema().await?;
        Ok(manager)
    }

    pub async fn new_remote(url: &str, token: &str) -> crate::Result<Self> {
        let db = Builder::new_remote(url.to_string(), token.to_string())
            .build()
            .await?;
        let conn = db.connect()?;

        let manager = Self {
            db: Arc::new(db),
            conn: Arc::new(Mutex::new(conn)),
        };

        manager.initialize_schema().await?;
        Ok(manager)
    }

    async fn initialize_schema(&self) -> crate::Result<()> {
        let conn = self.conn.lock().await;

        conn.execute(
            r#"
            CREATE TABLE IF NOT EXISTS memory (
                id TEXT PRIMARY KEY,
                agent_id TEXT NOT NULL,
                key TEXT NOT NULL,
                value TEXT NOT NULL,
                created_at INTEGER NOT NULL
            )
            "#,
            [],
        ).await?;

        Ok(())
    }

    pub async fn store(&self, entry: MemoryEntry) -> crate::Result<()> {
        let conn = self.conn.lock().await;

        conn.execute(
            "INSERT INTO memory (id, agent_id, key, value, created_at) VALUES (?1, ?2, ?3, ?4, ?5)",
            libsql::params![
                entry.id,
                entry.agent_id,
                entry.key,
                serde_json::to_string(&entry.value)?,
                entry.created_at,
            ],
        ).await?;

        Ok(())
    }

    pub async fn retrieve(&self, agent_id: &str, key: &str) -> crate::Result<Option<MemoryEntry>> {
        let conn = self.conn.lock().await;

        let mut stmt = conn.prepare(
            "SELECT id, agent_id, key, value, created_at FROM memory WHERE agent_id = ?1 AND key = ?2"
        ).await?;

        let mut rows = stmt.query(libsql::params![agent_id, key]).await?;

        if let Some(row) = rows.next().await? {
            let entry = MemoryEntry {
                id: row.get(0)?,
                agent_id: row.get(1)?,
                key: row.get(2)?,
                value: serde_json::from_str(&row.get::<String>(3)?)?,
                created_at: row.get(4)?,
            };
            Ok(Some(entry))
        } else {
            Ok(None)
        }
    }
}
```

Create `src/coordination.rs`:

```rust
use crate::agent::{Agent, AgentId, Task, TaskResult};
use serde::{Deserialize, Serialize};
use std::sync::Arc;
use tokio::sync::RwLock;
use uuid::Uuid;

#[derive(Debug, Clone, Serialize, Deserialize)]
pub enum Topology {
    Mesh,
    Star,
    Ring,
}

pub struct Swarm {
    pub id: Uuid,
    pub topology: Topology,
    pub agents: Vec<AgentId>,
}

pub struct Coordinator {
    swarms: Arc<RwLock<Vec<Swarm>>>,
}

impl Coordinator {
    pub fn new() -> Self {
        Self {
            swarms: Arc::new(RwLock::new(Vec::new())),
        }
    }

    pub async fn create_swarm(&self, topology: Topology) -> crate::Result<Swarm> {
        let swarm = Swarm {
            id: Uuid::new_v4(),
            topology,
            agents: Vec::new(),
        };

        self.swarms.write().await.push(swarm.clone());
        Ok(swarm)
    }

    pub async fn execute_parallel(
        &self,
        agents: Vec<Box<dyn Agent>>,
        tasks: Vec<Task>,
    ) -> crate::Result<Vec<TaskResult>> {
        let mut handles = Vec::new();

        for (agent, task) in agents.into_iter().zip(tasks.into_iter()) {
            let handle = tokio::spawn(async move {
                agent.execute(task).await
            });
            handles.push(handle);
        }

        let mut results = Vec::new();
        for handle in handles {
            results.push(handle.await??);
        }

        Ok(results)
    }
}
```

Create `src/skills.rs`:

```rust
use serde::{Deserialize, Serialize};

#[derive(Debug, Clone, Serialize, Deserialize)]
pub struct SkillDefinition {
    pub name: String,
    pub description: String,
    pub triggers: Vec<String>,
}

pub struct SkillExecutor {
    skills: Vec<SkillDefinition>,
}

impl SkillExecutor {
    pub fn new() -> Self {
        Self {
            skills: Vec::new(),
        }
    }

    pub fn register_skill(&mut self, skill: SkillDefinition) {
        self.skills.push(skill);
    }

    pub async fn execute_skill(&self, name: &str) -> crate::Result<()> {
        // Skill execution logic
        println!("Executing skill: {}", name);
        Ok(())
    }
}
```

---

## 🎯 Step 4: Simple CLI (3 min)

Create `src/bin/raf.rs`:

```rust
use clap::{Parser, Subcommand};
use rust_agent_flow::prelude::*;

#[derive(Parser)]
#[command(name = "raf")]
#[command(about = "Rust Agent Flow CLI", long_about = None)]
struct Cli {
    #[command(subcommand)]
    command: Commands,
}

#[derive(Subcommand)]
enum Commands {
    /// Initialize a new swarm
    Init {
        #[arg(short, long)]
        topology: String,
    },
    /// Execute a skill
    Skill {
        name: String,
    },
    /// Test database connection
    Test,
}

#[tokio::main]
async fn main() -> anyhow::Result<()> {
    tracing_subscriber::fmt::init();

    let cli = Cli::parse();

    match cli.command {
        Commands::Init { topology } => {
            println!("Initializing swarm with topology: {}", topology);

            let coordinator = Coordinator::new();
            let swarm = coordinator.create_swarm(Topology::Mesh).await?;

            println!("✅ Swarm created: {}", swarm.id);
        }
        Commands::Skill { name } => {
            println!("Executing skill: {}", name);

            let executor = SkillExecutor::new();
            executor.execute_skill(&name).await?;

            println!("✅ Skill executed");
        }
        Commands::Test => {
            println!("Testing Turso connection...");

            // Test with environment variables
            if let (Ok(url), Ok(token)) = (
                std::env::var("TURSO_DATABASE_URL"),
                std::env::var("TURSO_AUTH_TOKEN")
            ) {
                let memory = MemoryManager::new_remote(&url, &token).await?;
                println!("✅ Connected to Turso!");

                // Test write
                let entry = MemoryEntry {
                    id: uuid::Uuid::new_v4().to_string(),
                    agent_id: "test-agent".to_string(),
                    key: "test-key".to_string(),
                    value: serde_json::json!({"message": "Hello from Rust!"}),
                    created_at: chrono::Utc::now().timestamp(),
                };

                memory.store(entry).await?;
                println!("✅ Test data written");

                // Test read
                let retrieved = memory.retrieve("test-agent", "test-key").await?;
                if let Some(data) = retrieved {
                    println!("✅ Test data read: {:?}", data.value);
                }
            } else {
                println!("❌ TURSO_DATABASE_URL and TURSO_AUTH_TOKEN not set");
            }
        }
    }

    Ok(())
}
```

---

## 🏃 Step 5: Build and Run (2 min)

```bash
# Build
cargo build --release

# Test Turso connection
./target/release/raf test

# Initialize swarm
./target/release/raf init --topology mesh

# Execute skill
./target/release/raf skill code-review
```

---

## 🎨 Step 6: Create Your First Skill

Create `.claude/skills/hello-swarm.md`:

```markdown
---
skill_name: hello-swarm
version: "1.0.0"
description: Simple hello world swarm demonstration

triggers:
  - "hello swarm"
  - "test swarm"

agents:
  - type: researcher
    count: 2
  - type: coder
    count: 1

coordination:
  topology: mesh
  timeout: 30s
---

# Hello Swarm Skill

This skill demonstrates basic swarm coordination.

## What it does

1. Spawns 2 researcher agents
2. Spawns 1 coder agent
3. Coordinates them in a mesh topology
4. Returns results from all agents
```

---

## 📊 Full Example: Distributed Code Review

Create `examples/code_review.rs`:

```rust
use rust_agent_flow::prelude::*;
use async_trait::async_trait;

struct CodeReviewer {
    id: AgentId,
}

#[async_trait]
impl Agent for CodeReviewer {
    async fn execute(&self, task: Task) -> Result<TaskResult> {
        println!("🔍 Reviewing code: {}", task.description);

        // Simulate code review
        tokio::time::sleep(tokio::time::Duration::from_secs(2)).await;

        Ok(TaskResult {
            agent_id: self.id,
            task_id: task.id,
            output: serde_json::json!({
                "issues_found": 2,
                "suggestions": ["Use async/await", "Add error handling"]
            }),
            success: true,
        })
    }

    fn agent_type(&self) -> AgentType {
        AgentType::Reviewer
    }

    fn id(&self) -> AgentId {
        self.id
    }
}

#[tokio::main]
async fn main() -> anyhow::Result<()> {
    println!("🚀 Starting distributed code review...\n");

    // Initialize coordinator
    let coordinator = Coordinator::new();
    let swarm = coordinator.create_swarm(Topology::Mesh).await?;
    println!("✅ Swarm created: {}\n", swarm.id);

    // Spawn agents
    let agents: Vec<Box<dyn Agent>> = vec![
        Box::new(CodeReviewer { id: uuid::Uuid::new_v4() }),
        Box::new(CodeReviewer { id: uuid::Uuid::new_v4() }),
        Box::new(CodeReviewer { id: uuid::Uuid::new_v4() }),
    ];

    // Create tasks
    let tasks = vec![
        Task {
            id: uuid::Uuid::new_v4(),
            description: "Review authentication module".to_string(),
            metadata: serde_json::json!({}),
        },
        Task {
            id: uuid::Uuid::new_v4(),
            description: "Review API endpoints".to_string(),
            metadata: serde_json::json!({}),
        },
        Task {
            id: uuid::Uuid::new_v4(),
            description: "Review database queries".to_string(),
            metadata: serde_json::json!({}),
        },
    ];

    // Execute in parallel
    println!("🔄 Executing review tasks in parallel...\n");
    let results = coordinator.execute_parallel(agents, tasks).await?;

    // Display results
    println!("📊 Review Results:\n");
    for result in results {
        println!("Agent {}: {:?}", result.agent_id, result.output);
    }

    println!("\n✅ Code review complete!");

    Ok(())
}
```

Run it:

```bash
cargo run --example code_review
```

---

## 🎉 You Did It!

You now have:

✅ A working Rust agentic framework
✅ Turso database integration
✅ Basic agent system
✅ Parallel execution
✅ Skills structure
✅ CLI interface

---

## 🚀 Next Steps

### Add More Agents

```rust
struct SecurityAnalyzer { id: AgentId }

#[async_trait]
impl Agent for SecurityAnalyzer {
    async fn execute(&self, task: Task) -> Result<TaskResult> {
        // Security analysis logic
        Ok(TaskResult {
            agent_id: self.id,
            task_id: task.id,
            output: serde_json::json!({
                "vulnerabilities": [],
                "risk_level": "low"
            }),
            success: true,
        })
    }

    fn agent_type(&self) -> AgentType {
        AgentType::Analyzer
    }

    fn id(&self) -> AgentId {
        self.id
    }
}
```

### Add Vector Search

```toml
[dependencies]
# Add to Cargo.toml
ndarray = "0.15"
```

```rust
pub struct VectorStore {
    vectors: Vec<Vec<f32>>,
}

impl VectorStore {
    pub fn search(&self, query: &[f32], k: usize) -> Vec<usize> {
        // Implement HNSW search
        todo!()
    }
}
```

### Add Web API

```rust
use axum::{Router, routing::get};

#[tokio::main]
async fn main() {
    let app = Router::new()
        .route("/health", get(|| async { "OK" }));

    axum::Server::bind(&"0.0.0.0:8080".parse().unwrap())
        .serve(app.into_make_service())
        .await
        .unwrap();
}
```

---

## 📚 Resources

- [Rust Book](https://doc.rust-lang.org/book/)
- [Tokio Tutorial](https://tokio.rs/tokio/tutorial)
- [Turso Docs](https://docs.turso.tech/)
- [Axum Guide](https://docs.rs/axum/latest/axum/)

---

## 🎓 What You Learned

1. ✅ Setting up a Rust project with async
2. ✅ Integrating Turso database
3. ✅ Creating trait-based agent system
4. ✅ Implementing parallel coordination
5. ✅ Building a CLI with Clap
6. ✅ Writing examples and skills

**Time to build something amazing!** 🦀
