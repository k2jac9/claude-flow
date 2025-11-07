# TOML vs JSON for Rust Agentic Framework
## Why TOML is Superior for Rust Applications

---

## 📊 Quick Comparison

```toml
# TOML - Clean, readable, Rust-native
[agent]
id = "researcher-001"
type = "researcher"
capabilities = ["search", "analyze", "summarize"]

[agent.metadata]
created_at = 2025-11-07T10:30:00Z
priority = "high"

[memory]
ttl = 3600
cache_size = 1000

# Comments are supported!
[coordination]
topology = "mesh"
max_agents = 6
consensus_threshold = 0.66
```

```json
{
  "agent": {
    "id": "researcher-001",
    "type": "researcher",
    "capabilities": ["search", "analyze", "summarize"],
    "metadata": {
      "created_at": "2025-11-07T10:30:00Z",
      "priority": "high"
    }
  },
  "memory": {
    "ttl": 3600,
    "cache_size": 1000
  },
  "coordination": {
    "topology": "mesh",
    "max_agents": 6,
    "consensus_threshold": 0.66
  }
}
```

**Winner: TOML** (easier to read and edit)

---

## 🦀 Rust Integration

### Using TOML with Serde

```toml
# Cargo.toml
[dependencies]
serde = { version = "1.0", features = ["derive"] }
toml = "0.8"
```

### Configuration Files

```rust
use serde::{Deserialize, Serialize};
use std::fs;

#[derive(Debug, Serialize, Deserialize)]
struct AgentConfig {
    agent: AgentInfo,
    memory: MemoryConfig,
    coordination: CoordinationConfig,
}

#[derive(Debug, Serialize, Deserialize)]
struct AgentInfo {
    id: String,
    #[serde(rename = "type")]
    agent_type: String,
    capabilities: Vec<String>,
    metadata: Metadata,
}

#[derive(Debug, Serialize, Deserialize)]
struct Metadata {
    created_at: String,
    priority: String,
}

#[derive(Debug, Serialize, Deserialize)]
struct MemoryConfig {
    ttl: u64,
    cache_size: usize,
}

#[derive(Debug, Serialize, Deserialize)]
struct CoordinationConfig {
    topology: String,
    max_agents: usize,
    consensus_threshold: f64,
}

fn main() -> Result<(), Box<dyn std::error::Error>> {
    // Read TOML file
    let config_str = fs::read_to_string("config.toml")?;
    let config: AgentConfig = toml::from_str(&config_str)?;

    println!("Agent ID: {}", config.agent.id);
    println!("Topology: {}", config.coordination.topology);

    // Write TOML file
    let config_toml = toml::to_string_pretty(&config)?;
    fs::write("output.toml", config_toml)?;

    Ok(())
}
```

---

## 🎯 Recommended Usage Pattern

### Use TOML for:
- ✅ **Configuration files** (agent config, system settings)
- ✅ **Skill definitions** (replacing YAML)
- ✅ **Project metadata**
- ✅ **User-editable settings**

### Use JSON for:
- ✅ **API responses** (web standard)
- ✅ **Database storage** (Turso supports JSON)
- ✅ **Runtime data interchange**
- ✅ **Network messages**

### Use MessagePack/Bincode for:
- ✅ **High-performance serialization**
- ✅ **Inter-agent communication**
- ✅ **Binary protocols**

---

## 🔧 Updated Architecture

### Skills Definition (TOML instead of YAML)

```toml
# .claude/skills/code-review-swarm.toml

[skill]
name = "code-review-swarm"
version = "1.0.0"
description = "Distributed code review with parallel analysis"

triggers = [
  "review this pull request",
  "analyze code quality",
  "security audit"
]

[[agents]]
type = "security-analyst"
count = 2
capabilities = ["static-analysis", "vulnerability-detection", "dependency-scanning"]

[[agents]]
type = "performance-reviewer"
count = 1
capabilities = ["benchmarking", "profiling", "optimization-suggestions"]

[[agents]]
type = "code-quality-reviewer"
count = 3
capabilities = ["style-checking", "complexity-analysis", "test-coverage"]

[coordination]
topology = "mesh"
consensus = "majority"
timeout = "300s"

[memory]
persistence = "turso"
vector_search = true
ttl = "24h"

[execution]
parallel = true
max_concurrent = 6

[execution.retry_policy]
max_attempts = 3
backoff = "exponential"

[[outputs]]
type = "report"
format = "markdown"

[[outputs]]
type = "metrics"
format = "json"
```

### Parsing in Rust

```rust
use serde::{Deserialize, Serialize};
use std::fs;

#[derive(Debug, Serialize, Deserialize)]
struct SkillDefinition {
    skill: SkillInfo,
    triggers: Vec<String>,
    agents: Vec<AgentSpec>,
    coordination: CoordinationConfig,
    memory: MemoryConfig,
    execution: ExecutionConfig,
    outputs: Vec<OutputSpec>,
}

#[derive(Debug, Serialize, Deserialize)]
struct SkillInfo {
    name: String,
    version: String,
    description: String,
}

#[derive(Debug, Serialize, Deserialize)]
struct AgentSpec {
    #[serde(rename = "type")]
    agent_type: String,
    count: usize,
    capabilities: Vec<String>,
}

#[derive(Debug, Serialize, Deserialize)]
struct CoordinationConfig {
    topology: String,
    consensus: String,
    timeout: String,
}

#[derive(Debug, Serialize, Deserialize)]
struct MemoryConfig {
    persistence: String,
    vector_search: bool,
    ttl: String,
}

#[derive(Debug, Serialize, Deserialize)]
struct ExecutionConfig {
    parallel: bool,
    max_concurrent: usize,
    retry_policy: RetryPolicy,
}

#[derive(Debug, Serialize, Deserialize)]
struct RetryPolicy {
    max_attempts: usize,
    backoff: String,
}

#[derive(Debug, Serialize, Deserialize)]
struct OutputSpec {
    #[serde(rename = "type")]
    output_type: String,
    format: String,
}

pub struct SkillLoader;

impl SkillLoader {
    pub fn load(path: &str) -> Result<SkillDefinition, Box<dyn std::error::Error>> {
        let content = fs::read_to_string(path)?;
        let skill: SkillDefinition = toml::from_str(&content)?;
        Ok(skill)
    }
}

// Usage
fn main() -> Result<(), Box<dyn std::error::Error>> {
    let skill = SkillLoader::load(".claude/skills/code-review-swarm.toml")?;

    println!("Skill: {}", skill.skill.name);
    println!("Agents: {} types", skill.agents.len());
    println!("Topology: {}", skill.coordination.topology);

    Ok(())
}
```

---

## 💾 Turso with TOML-like Storage

### Store Configuration as TOML in Turso

```rust
use libsql::{Builder, Connection};

pub struct TursoConfigStore {
    conn: Connection,
}

impl TursoConfigStore {
    pub async fn store_config(
        &self,
        key: &str,
        config: &impl Serialize,
    ) -> Result<()> {
        // Serialize to TOML string
        let toml_str = toml::to_string_pretty(config)?;

        // Store in Turso
        self.conn.execute(
            "INSERT INTO configs (key, toml_data) VALUES (?1, ?2)
             ON CONFLICT(key) DO UPDATE SET toml_data = excluded.toml_data",
            libsql::params![key, toml_str],
        ).await?;

        Ok(())
    }

    pub async fn load_config<T: DeserializeOwned>(
        &self,
        key: &str,
    ) -> Result<Option<T>> {
        let mut stmt = self.conn.prepare(
            "SELECT toml_data FROM configs WHERE key = ?1"
        ).await?;

        let mut rows = stmt.query(libsql::params![key]).await?;

        if let Some(row) = rows.next().await? {
            let toml_str: String = row.get(0)?;
            let config: T = toml::from_str(&toml_str)?;
            Ok(Some(config))
        } else {
            Ok(None)
        }
    }
}

// Usage
async fn example() -> Result<()> {
    let store = TursoConfigStore { /* ... */ };

    // Store
    let config = AgentConfig { /* ... */ };
    store.store_config("agent-001", &config).await?;

    // Load
    let loaded: Option<AgentConfig> = store.load_config("agent-001").await?;

    Ok(())
}
```

---

## 🎨 Best Practices

### 1. Configuration Files → TOML

```toml
# config/agents.toml
[defaults]
timeout = 30
max_retries = 3

[agents.researcher]
capabilities = ["search", "analyze"]
priority = "high"

[agents.coder]
capabilities = ["implement", "refactor"]
priority = "medium"
```

### 2. Runtime Data → JSON (in Turso)

```rust
// Store runtime data as JSON in database
let task = Task {
    id: uuid::Uuid::new_v4(),
    data: serde_json::json!({
        "description": "Implement feature X",
        "priority": 5,
    }),
};
```

### 3. Inter-Agent Messages → MessagePack

```rust
use rmp_serde;

// Serialize to MessagePack (binary, fast)
let message = AgentMessage { /* ... */ };
let bytes = rmp_serde::to_vec(&message)?;

// Send over network
send_to_agent(&bytes).await?;

// Deserialize
let received: AgentMessage = rmp_serde::from_slice(&bytes)?;
```

---

## 📦 Updated Dependencies

```toml
[dependencies]
# TOML parsing
toml = "0.8"

# JSON for APIs and Turso
serde_json = "1.0"

# High-performance binary serialization
rmp-serde = "1.1"  # MessagePack
bincode = "1.3"     # Alternative

# Serialization framework
serde = { version = "1.0", features = ["derive"] }
```

---

## 🚀 Complete Example

### Agent Configuration (TOML)

```toml
# config/swarm.toml

[system]
name = "code-review-swarm"
version = "1.0.0"

[database]
provider = "turso"
url = "libsql://mydb.turso.io"
mode = "hybrid"

[agents.security]
type = "security-analyst"
count = 2
priority = "high"

[agents.security.capabilities]
static_analysis = true
vulnerability_scan = true
dependency_check = true

[agents.performance]
type = "performance-reviewer"
count = 1
priority = "medium"

[coordination]
topology = "mesh"
consensus_threshold = 0.66
timeout_seconds = 300

[memory]
cache_size = 1000
ttl_seconds = 3600
vector_dimension = 768
```

### Loading and Using

```rust
use serde::{Deserialize, Serialize};
use std::fs;

#[derive(Debug, Serialize, Deserialize)]
struct SwarmConfig {
    system: SystemInfo,
    database: DatabaseConfig,
    agents: AgentConfigs,
    coordination: CoordinationConfig,
    memory: MemoryConfig,
}

#[derive(Debug, Serialize, Deserialize)]
struct SystemInfo {
    name: String,
    version: String,
}

#[derive(Debug, Serialize, Deserialize)]
struct DatabaseConfig {
    provider: String,
    url: String,
    mode: String,
}

#[derive(Debug, Serialize, Deserialize)]
struct AgentConfigs {
    security: AgentTypeConfig,
    performance: AgentTypeConfig,
}

#[derive(Debug, Serialize, Deserialize)]
struct AgentTypeConfig {
    #[serde(rename = "type")]
    agent_type: String,
    count: usize,
    priority: String,
    capabilities: std::collections::HashMap<String, bool>,
}

#[derive(Debug, Serialize, Deserialize)]
struct CoordinationConfig {
    topology: String,
    consensus_threshold: f64,
    timeout_seconds: u64,
}

#[derive(Debug, Serialize, Deserialize)]
struct MemoryConfig {
    cache_size: usize,
    ttl_seconds: u64,
    vector_dimension: usize,
}

#[tokio::main]
async fn main() -> Result<(), Box<dyn std::error::Error>> {
    // Load configuration
    let config_str = fs::read_to_string("config/swarm.toml")?;
    let config: SwarmConfig = toml::from_str(&config_str)?;

    println!("System: {} v{}", config.system.name, config.system.version);
    println!("Database: {} ({})", config.database.provider, config.database.mode);
    println!("Security agents: {}", config.agents.security.count);
    println!("Topology: {}", config.coordination.topology);

    // Use configuration
    let swarm = initialize_swarm(config).await?;

    Ok(())
}

async fn initialize_swarm(config: SwarmConfig) -> Result<Swarm, Box<dyn std::error::Error>> {
    // Implementation using the config
    todo!()
}
```

---

## 📊 Format Decision Matrix

```
┌──────────────────┬──────┬──────┬─────────────┬─────────┐
│ Use Case         │ TOML │ JSON │ MessagePack │ Bincode │
├──────────────────┼──────┼──────┼─────────────┼─────────┤
│ Config files     │  ✅  │  ⚠️  │     ❌      │   ❌    │
│ Skill defs       │  ✅  │  ⚠️  │     ❌      │   ❌    │
│ API responses    │  ❌  │  ✅  │     ⚠️      │   ❌    │
│ Database         │  ⚠️  │  ✅  │     ✅      │   ⚠️    │
│ Inter-agent      │  ❌  │  ⚠️  │     ✅      │   ✅    │
│ Network          │  ❌  │  ✅  │     ✅      │   ⚠️    │
│ Human-readable   │  ✅  │  ⚠️  │     ❌      │   ❌    │
│ Performance      │  ⚠️  │  ⚠️  │     ✅      │   ✅    │
│ Size             │  ⚠️  │  ⚠️  │     ✅      │   ✅    │
└──────────────────┴──────┴──────┴─────────────┴─────────┘

Legend:
✅ Excellent choice
⚠️ Works but not ideal
❌ Avoid
```

---

## ✅ Final Recommendation

### Use TOML for:
1. **Agent configuration files**
2. **Skill definitions**
3. **System settings**
4. **Any human-edited config**

### Use JSON for:
1. **REST API responses**
2. **Database JSON columns** (Turso supports this)
3. **Web client communication**

### Use MessagePack/Bincode for:
1. **High-speed inter-agent messages**
2. **Binary protocols**
3. **Performance-critical serialization**

---

## 🎯 Updated Stack

```
Configuration Layer: TOML
├── Skills: .toml files
├── Agent configs: .toml files
└── System settings: .toml files

Runtime Layer: JSON + MessagePack
├── API: JSON (REST)
├── Database: JSON columns + TOML strings
├── Inter-agent: MessagePack (binary)
└── Web: JSON (standard)

Storage Layer: Turso
├── Structured data: SQL tables
├── Configuration: TOML strings
├── Runtime data: JSON columns
└── Binary data: BLOB (MessagePack)
```

---

## 🚀 Migration from JSON/YAML

### Simple Conversion

```bash
# Install converter
cargo install json2toml

# Convert JSON to TOML
json2toml < config.json > config.toml

# Or YAML to TOML
# (manual, or write a script)
```

### Programmatic Conversion

```rust
use serde_json::Value as JsonValue;

fn json_to_toml(json_str: &str) -> Result<String> {
    let json: JsonValue = serde_json::from_str(json_str)?;
    let toml = toml::to_string_pretty(&json)?;
    Ok(toml)
}

// Usage
let json = r#"{"name": "test", "value": 42}"#;
let toml = json_to_toml(json)?;
println!("{}", toml);
// Output:
// name = "test"
// value = 42
```

---

**Conclusion: TOML + Turso is the perfect combination for a Rust agentic framework!** 🦀

TOML gives you:
- ✅ Rust-native configuration
- ✅ Human-readable files
- ✅ Type-safe parsing
- ✅ Comments support

Combined with Turso for runtime data, you get the best of both worlds! 🎉
