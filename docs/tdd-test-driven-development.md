# Test-Driven Development for Rust Legion Flow
## Building Reliable AI Agents with TDD Excellence

> **TDD**: Write the test first, watch it fail, make it pass, refactor. Repeat. Build confidence, not bugs.

---

## 🎯 TDD Philosophy for Legion Framework

### Why TDD is Critical for Agent Systems

```
Traditional Development:
Code → Test → Debug → Fix → Repeat
Issues: Late bug discovery, fragile systems, fear of changes

Test-Driven Development:
Test → Code → Refactor → Repeat
Benefits: Early bug prevention, confident refactoring, living documentation
```

### TDD + Legion + Shape Up

```
Shape Up Cycle (6 weeks):
├── Week 0: Write acceptance tests (what success looks like)
├── Week 1-2: TDD implementation (red-green-refactor)
├── Week 3-4: Integration tests (legions working together)
├── Week 5: Performance tests (benchmarks)
├── Week 6: Ship with confidence (100% tested)
└── Cooldown: Refactor, improve test suite
```

---

## 🦀 Rust Testing Fundamentals

### Test Types Hierarchy

```
┌────────────────────────────────────────┐
│         Test Pyramid for Legion        │
├────────────────────────────────────────┤
│              E2E Tests                 │ ← Few (expensive, slow)
│         ╱                  ╲           │
│        ╱  Integration Tests ╲          │ ← Some (moderate cost)
│       ╱                      ╲         │
│      ╱      Unit Tests        ╲        │ ← Many (cheap, fast)
│     ╱                          ╲       │
│    ╱   Property-Based Tests    ╲      │ ← Foundation (automated)
│   ╱____________________________╲     │
└────────────────────────────────────────┘
```

### Rust Test Structure

```rust
// src/legion/commander.rs

#[cfg(test)]
mod tests {
    use super::*;

    #[test]
    fn test_deploy_legion_creates_valid_legion() {
        // Arrange
        let commander = LegionCommander::new();
        let config = LegionConfig {
            formation: LegionFormation::Phalanx,
            strength: 6,
        };

        // Act
        let legion = commander.deploy_legion(config).unwrap();

        // Assert
        assert_eq!(legion.formation, LegionFormation::Phalanx);
        assert_eq!(legion.legionnaires.len(), 0); // Not recruited yet
        assert!(legion.id != Uuid::nil());
    }

    #[test]
    fn test_recruit_legionnaire_increases_strength() {
        // Arrange
        let mut commander = LegionCommander::new();
        let legion = commander.deploy_legion(default_config()).unwrap();

        // Act
        commander.recruit_legionnaire(
            &legion.id,
            LegionRole::Legionnaire,
            "researcher"
        ).unwrap();

        // Assert
        let status = commander.legion_status(&legion.id).unwrap();
        assert_eq!(status.strength, 1);
    }

    #[test]
    #[should_panic(expected = "Legion not found")]
    fn test_recruit_to_nonexistent_legion_panics() {
        let commander = LegionCommander::new();
        let fake_id = Uuid::new_v4();

        commander.recruit_legionnaire(
            &fake_id,
            LegionRole::Legionnaire,
            "researcher"
        ).unwrap();
    }
}
```

---

## 🔴🟢🔵 Red-Green-Refactor Cycle

### The TDD Workflow

```rust
// STEP 1: RED - Write failing test
#[test]
fn test_legion_execute_mission_returns_results() {
    let commander = LegionCommander::new();
    let legion = commander.deploy_legion(default_config()).unwrap();

    // This will fail - method doesn't exist yet
    let results = commander.execute_mission(&legion.id).await.unwrap();

    assert!(!results.is_empty());
}

// STEP 2: GREEN - Minimal implementation to pass
impl LegionCommander {
    pub async fn execute_mission(&self, legion_id: &LegionId) -> Result<Vec<TaskResult>> {
        // Simplest thing that makes test pass
        Ok(vec![TaskResult::default()])
    }
}

// STEP 3: REFACTOR - Improve implementation
impl LegionCommander {
    pub async fn execute_mission(&self, legion_id: &LegionId) -> Result<Vec<TaskResult>> {
        let legion = self.get_legion(legion_id)?;
        let mut results = Vec::new();

        for legionnaire in &legion.legionnaires {
            let result = legionnaire.execute_task().await?;
            results.push(result);
        }

        Ok(results)
    }
}
```

---

## 🎯 TDD for Legion Components

### 1. Legion Commander TDD

```rust
// tests/legion_commander_test.rs

use rust_legion_flow::prelude::*;
use tokio;

#[tokio::test]
async fn test_deploy_legion_with_phalanx_formation() {
    // Red: Write test first
    let commander = LegionCommander::new();

    let legion = commander.deploy_legion(LegionConfig {
        formation: LegionFormation::Phalanx,
        strength: 6,
        command_structure: CommandStructure::Hierarchical,
    }).await.unwrap();

    assert_eq!(legion.formation, LegionFormation::Phalanx);
    assert_eq!(legion.max_strength, 6);
    assert!(legion.commander.is_some());
}

#[tokio::test]
async fn test_legion_readiness_increases_with_recruits() {
    let commander = LegionCommander::new();
    let legion = commander.deploy_legion(default_config()).await.unwrap();

    // Initial readiness should be 0
    let status1 = commander.legion_status(&legion.id).await.unwrap();
    assert_eq!(status1.readiness, 0.0);

    // Recruit legionnaires
    for i in 0..6 {
        commander.recruit_legionnaire(
            &legion.id,
            LegionRole::Legionnaire,
            &format!("agent-{}", i)
        ).await.unwrap();
    }

    // Readiness should be 100% when at full strength
    let status2 = commander.legion_status(&legion.id).await.unwrap();
    assert_eq!(status2.readiness, 1.0);
}

#[tokio::test]
async fn test_legion_formation_affects_coordination() {
    let commander = LegionCommander::new();

    // Phalanx: high collaboration
    let phalanx = commander.deploy_legion(LegionConfig {
        formation: LegionFormation::Phalanx,
        ..default()
    }).await.unwrap();

    // Skirmish: independent
    let skirmish = commander.deploy_legion(LegionConfig {
        formation: LegionFormation::Skirmish,
        ..default()
    }).await.unwrap();

    assert!(phalanx.coordination_factor() > skirmish.coordination_factor());
}
```

### 2. Shape Up Cycle TDD

```rust
// tests/shape_up_cycle_test.rs

#[test]
fn test_cycle_manager_starts_in_progress() {
    let manager = CycleManager::new();

    assert_eq!(manager.current_cycle.status, CycleStatus::InProgress);
    assert_eq!(manager.current_cycle.number, 1);
}

#[test]
fn test_cycle_transitions_to_cooldown_after_6_weeks() {
    let mut manager = CycleManager::new();

    // Simulate 6 weeks passing
    manager.current_cycle.start_date = Utc::now() - Duration::weeks(6);

    manager.advance_cycle().await.unwrap();

    assert_eq!(manager.current_cycle.status, CycleStatus::Cooldown);
}

#[test]
fn test_betting_table_evaluates_pitches() {
    let betting_table = BettingTable::new();

    let pitch = Pitch {
        name: "test-feature".to_string(),
        problem: "Users need X".to_string(),
        appetite: Appetite::Small,
        solution: "Build Y".to_string(),
        risks: vec![],
        rabbit_holes: vec!["Don't do Z".to_string()],
        no_gos: vec![],
        agents_required: vec!["coder".to_string()],
    };

    let score = betting_table.score_pitch(&pitch).await;

    assert!(score > 0.5); // Should pass basic criteria
}

#[test]
fn test_hill_chart_tracks_progress() {
    let mut chart = HillChart {
        project_name: "test".to_string(),
        scopes: vec![
            Scope {
                name: "auth".to_string(),
                position: 0.0,
                confidence: Confidence::Low,
                ..default()
            }
        ],
    };

    // Update progress
    chart.scopes[0].update_position(25.0, Confidence::Medium);

    assert_eq!(chart.scopes[0].position, 25.0);
    assert_eq!(chart.scopes[0].status(), ScopeStatus::Uphill);

    // Continue to top of hill
    chart.scopes[0].update_position(25.0, Confidence::High);
    assert_eq!(chart.scopes[0].status(), ScopeStatus::TopOfHill);

    // Downhill execution
    chart.scopes[0].update_position(25.0, Confidence::High);
    assert_eq!(chart.scopes[0].status(), ScopeStatus::Downhill);
}
```

### 3. TOON Parser TDD

```rust
// tests/toon_parser_test.rs

#[test]
fn test_parse_simple_toon() {
    let toon = r#"
name: test
value: 42
enabled: true
    "#;

    let result: ToonValue = toon::from_str(toon).unwrap();

    assert_eq!(result.get("name").unwrap().as_str(), Some("test"));
    assert_eq!(result.get("value").unwrap().as_u64(), Some(42));
    assert_eq!(result.get("enabled").unwrap().as_bool(), Some(true));
}

#[test]
fn test_parse_compact_arrays() {
    let toon = "capabilities: [search analyze summarize]";

    let result: ToonValue = toon::from_str(toon).unwrap();
    let caps = result.get("capabilities").unwrap().as_array().unwrap();

    assert_eq!(caps.len(), 3);
    assert_eq!(caps[0].as_str(), Some("search"));
}

#[test]
fn test_parse_inline_objects() {
    let toon = "metadata: {status: active, priority: high}";

    let result: ToonValue = toon::from_str(toon).unwrap();
    let meta = result.get("metadata").unwrap().as_object().unwrap();

    assert_eq!(meta.get("status").unwrap().as_str(), Some("active"));
    assert_eq!(meta.get("priority").unwrap().as_str(), Some("high"));
}

#[test]
fn test_toon_roundtrip() {
    let original = LegionConfig {
        formation: LegionFormation::Phalanx,
        strength: 6,
        ..default()
    };

    let toon_str = toon::to_string(&original).unwrap();
    let parsed: LegionConfig = toon::from_str(&toon_str).unwrap();

    assert_eq!(original, parsed);
}
```

---

## 🔬 Property-Based Testing

### Using Proptest for Legion

```toml
[dev-dependencies]
proptest = "1.4"
```

```rust
use proptest::prelude::*;

proptest! {
    #[test]
    fn test_legion_strength_never_exceeds_max(
        strength in 1usize..100,
        recruits in 0usize..200
    ) {
        let commander = LegionCommander::new();
        let mut legion = commander.deploy_legion(LegionConfig {
            strength,
            ..default()
        }).unwrap();

        // Try to recruit more than max
        for i in 0..recruits {
            let _ = commander.recruit_legionnaire(
                &legion.id,
                LegionRole::Legionnaire,
                &format!("agent-{}", i)
            );
        }

        let status = commander.legion_status(&legion.id).unwrap();
        prop_assert!(status.strength <= strength);
    }

    #[test]
    fn test_legion_id_always_unique(
        count in 1usize..1000
    ) {
        let commander = LegionCommander::new();
        let mut ids = std::collections::HashSet::new();

        for _ in 0..count {
            let legion = commander.deploy_legion(default_config()).unwrap();
            prop_assert!(ids.insert(legion.id));
        }
    }

    #[test]
    fn test_hill_position_bounded(
        delta in -100.0f32..200.0f32
    ) {
        let mut scope = Scope {
            name: "test".to_string(),
            position: 50.0,
            confidence: Confidence::Medium,
            ..default()
        };

        scope.update_position(delta, Confidence::High);

        // Position must stay in [0, 100]
        prop_assert!(scope.position >= 0.0);
        prop_assert!(scope.position <= 100.0);
    }
}
```

---

## 🎪 Integration Testing

### Testing Legion Coordination

```rust
// tests/integration/legion_coordination_test.rs

#[tokio::test]
async fn test_phalanx_formation_coordinates_agents() {
    // Setup
    let commander = LegionCommander::new();
    let turso = TursoManager::new_local("test.db").await.unwrap();

    let legion = commander.deploy_legion(LegionConfig {
        formation: LegionFormation::Phalanx,
        strength: 3,
        ..default()
    }).await.unwrap();

    // Recruit agents
    for i in 0..3 {
        commander.recruit_legionnaire(
            &legion.id,
            LegionRole::Legionnaire,
            &format!("agent-{}", i)
        ).await.unwrap();
    }

    // Execute coordinated task
    let task = Task {
        id: Uuid::new_v4(),
        description: "Review code".to_string(),
        metadata: json!({}),
    };

    let results = commander.execute_coordinated_task(
        &legion.id,
        task
    ).await.unwrap();

    // Verify coordination
    assert_eq!(results.len(), 3); // All agents participated
    assert!(results.iter().all(|r| r.success)); // All succeeded

    // Check memory for coordination traces
    let coordination_log = turso.load::<Vec<CoordinationEvent>>(
        &format!("legion:{}", legion.id)
    ).await.unwrap();

    assert!(coordination_log.is_some());
}

#[tokio::test]
async fn test_shape_up_cycle_completes_project() {
    // Full Shape Up cycle integration test
    let mut manager = CycleManager::new();

    // Betting: Select pitch
    let pitch = create_test_pitch();
    manager.submit_pitch(pitch).await.unwrap();

    // Start cycle
    manager.run_betting_table().await.unwrap();
    manager.start_new_cycle().await.unwrap();

    assert_eq!(manager.active_projects.len(), 1);

    // Execute work (simulate 6 weeks)
    let project = &mut manager.active_projects[0];
    for scope in &mut project.hill_chart.scopes {
        scope.update_position(100.0, Confidence::High);
    }

    // Ship
    manager.advance_cycle().await.unwrap();
    assert_eq!(manager.current_cycle.status, CycleStatus::Cooldown);

    // Verify shipped
    let shipped = manager.shipped_projects();
    assert_eq!(shipped.len(), 1);
}
```

---

## 🏗️ Test Architecture

### Test Organization

```
rust-legion-flow/
├── src/
│   └── lib.rs
├── tests/
│   ├── unit/              # Unit tests (fast)
│   │   ├── legion_test.rs
│   │   ├── formation_test.rs
│   │   └── toon_test.rs
│   ├── integration/       # Integration tests (moderate)
│   │   ├── legion_coordination_test.rs
│   │   ├── shape_up_cycle_test.rs
│   │   └── turso_integration_test.rs
│   ├── e2e/              # End-to-end tests (slow)
│   │   ├── full_cycle_test.rs
│   │   └── multi_legion_test.rs
│   └── common/           # Test utilities
│       ├── mod.rs
│       ├── fixtures.rs
│       └── helpers.rs
└── benches/              # Performance tests
    └── legion_bench.rs
```

---

## 📊 Test Coverage Requirements

### Coverage Targets

```toml
# Cargo.toml
[dev-dependencies]
tarpaulin = "0.27"

# Run coverage
# cargo tarpaulin --out Html
```

```
Coverage Requirements:
├── Critical paths: 100% (Legion deployment, task execution)
├── Core logic: 95% (Formation selection, coordination)
├── Utilities: 90% (Helpers, parsers)
├── UI/CLI: 80% (User-facing code)
└── Overall: 90%+
```

### Coverage Configuration

```toml
# tarpaulin.toml
[report]
out = ["Html", "Lcov"]

[run]
timeout = 120
follow-exec = true
post-test-delay = 1

[coverage]
exclude-files = [
    "tests/*",
    "benches/*",
    "examples/*"
]
```

---

## 🚀 CI/CD Integration

### GitHub Actions Workflow

```yaml
# .github/workflows/test.yml
name: Test Suite

on: [push, pull_request]

jobs:
  test:
    runs-on: ubuntu-latest
    steps:
      - uses: actions/checkout@v3

      - name: Setup Rust
        uses: actions-rs/toolchain@v1
        with:
          toolchain: stable
          override: true

      - name: Run Unit Tests
        run: cargo test --lib

      - name: Run Integration Tests
        run: cargo test --test '*'

      - name: Run Doctests
        run: cargo test --doc

      - name: Check Code Coverage
        run: |
          cargo install tarpaulin
          cargo tarpaulin --out Xml
          bash <(curl -s https://codecov.io/bash)

  property-test:
    runs-on: ubuntu-latest
    steps:
      - uses: actions/checkout@v3

      - name: Property-Based Tests
        run: cargo test --features proptest

  benchmark:
    runs-on: ubuntu-latest
    steps:
      - uses: actions/checkout@v3

      - name: Run Benchmarks
        run: cargo bench --no-fail-fast
```

---

## 🎯 TDD Best Practices for Legion

### 1. Test Naming Convention

```rust
// Pattern: test_<component>_<action>_<expected_result>

#[test]
fn test_legion_deploy_creates_valid_legion() { }

#[test]
fn test_commander_recruit_increases_strength() { }

#[test]
fn test_formation_phalanx_enables_collaboration() { }

#[test]
fn test_hill_chart_update_changes_position() { }
```

### 2. Arrange-Act-Assert (AAA)

```rust
#[test]
fn test_example() {
    // Arrange: Setup test data
    let commander = LegionCommander::new();
    let config = LegionConfig::default();

    // Act: Execute the behavior
    let legion = commander.deploy_legion(config).unwrap();

    // Assert: Verify the outcome
    assert!(legion.id != Uuid::nil());
}
```

### 3. Test Data Builders

```rust
// tests/common/builders.rs

pub struct LegionConfigBuilder {
    formation: LegionFormation,
    strength: usize,
    command_structure: CommandStructure,
}

impl LegionConfigBuilder {
    pub fn new() -> Self {
        Self {
            formation: LegionFormation::Phalanx,
            strength: 6,
            command_structure: CommandStructure::Hierarchical,
        }
    }

    pub fn formation(mut self, formation: LegionFormation) -> Self {
        self.formation = formation;
        self
    }

    pub fn strength(mut self, strength: usize) -> Self {
        self.strength = strength;
        self
    }

    pub fn build(self) -> LegionConfig {
        LegionConfig {
            formation: self.formation,
            strength: self.strength,
            command_structure: self.command_structure,
        }
    }
}

// Usage in tests
#[test]
fn test_with_builder() {
    let config = LegionConfigBuilder::new()
        .formation(LegionFormation::Vanguard)
        .strength(5)
        .build();

    assert_eq!(config.formation, LegionFormation::Vanguard);
}
```

### 4. Mocks and Stubs

```rust
use mockall::*;

#[automock]
trait LegionRepository {
    async fn save(&self, legion: &Legion) -> Result<()>;
    async fn find(&self, id: &LegionId) -> Result<Option<Legion>>;
}

#[tokio::test]
async fn test_commander_saves_legion() {
    let mut mock_repo = MockLegionRepository::new();

    mock_repo
        .expect_save()
        .times(1)
        .returning(|_| Ok(()));

    let commander = LegionCommander::with_repo(mock_repo);
    let legion = commander.deploy_legion(default_config()).await.unwrap();

    // Mock verifies save was called
}
```

---

## 📈 Mutation Testing

### Cargo Mutants

```toml
[dev-dependencies]
cargo-mutants = "23.0"
```

```bash
# Run mutation testing
cargo mutants

# Expected output:
# ✅ All mutants killed by tests
# Coverage: 98.5%
```

---

## 🎨 Test-Driven TOON Configuration

### Test Configuration Files

```toon
# tests/fixtures/test-legion.toon
legion:
  name: test-legion
  formation: phalanx
  strength: 3

  legionnaires:
    - role: scout, type: researcher, id: scout-1
    - role: legionnaire, type: coder, id: coder-1
    - role: legionnaire, type: coder, id: coder-2

  mission:
    objective: test-task
    appetite: small
    deadline: 1h
```

```rust
#[test]
fn test_load_legion_from_toon() {
    let toon_str = std::fs::read_to_string("tests/fixtures/test-legion.toon").unwrap();
    let config: LegionConfig = toon::from_str(&toon_str).unwrap();

    assert_eq!(config.legion.name, "test-legion");
    assert_eq!(config.legion.formation, LegionFormation::Phalanx);
    assert_eq!(config.legion.legionnaires.len(), 3);
}
```

---

## ✅ TDD Checklist for Every Feature

```markdown
Before implementing any feature:

1. [ ] Write acceptance test (what success looks like)
2. [ ] Write unit tests (red phase)
3. [ ] Implement minimal code (green phase)
4. [ ] Refactor (blue phase)
5. [ ] Add property-based tests (edge cases)
6. [ ] Write integration tests (component interaction)
7. [ ] Update documentation
8. [ ] Check coverage (>90%)
9. [ ] Run mutation tests
10. [ ] Commit with passing tests
```

---

## 🚀 TDD in Shape Up Cycles

### Week-by-Week TDD Integration

```toon
# Shape Up Cycle with TDD

week_0_shaping:
  activities:
    - Write acceptance criteria as tests
    - Define success metrics
    - Create test fixtures

week_1_2_uphill:
  activities:
    - TDD for core logic (red-green-refactor)
    - Unit test coverage: 95%+
    - Property-based tests for edge cases
  hill_position: 0 → 50

week_3_4_downhill:
  activities:
    - Integration tests
    - Component interaction tests
    - Performance benchmarks
  hill_position: 50 → 80

week_5_shipping:
  activities:
    - E2E tests
    - Mutation testing
    - Coverage verification
  hill_position: 80 → 100

week_6_completion:
  activities:
    - Final test suite run
    - Coverage report
    - Ship with confidence

cooldown:
  activities:
    - Refactor tests
    - Improve test utilities
    - Add missing test cases
```

---

## 📚 Recommended Testing Crates

```toml
[dev-dependencies]
# Core testing
tokio-test = "0.4"       # Async testing utilities
pretty_assertions = "1.4" # Better assertion output

# Property-based testing
proptest = "1.4"         # Property testing
quickcheck = "1.0"       # Alternative property testing

# Mocking
mockall = "0.12"         # Mock generation
wiremock = "0.5"         # HTTP mocking

# Coverage
tarpaulin = "0.27"       # Code coverage

# Mutation testing
cargo-mutants = "23.0"   # Mutation testing

# Benchmarking
criterion = "0.5"        # Benchmarking framework

# Fuzzing
cargo-fuzz = "0.11"      # Fuzz testing
```

---

## 🎯 Summary

```
┌────────────────────────────────────────────┐
│   TDD-Driven Rust Legion Flow               │
├────────────────────────────────────────────┤
│ Philosophy:    Test First, Code Second     │
│ Coverage:      90%+ (95% core, 100% critical)
│ Methodology:   Red-Green-Refactor          │
│ Property Tests: Automated edge cases       │
│ Integration:   Shape Up cycles             │
│ CI/CD:         Automated testing pipeline  │
│ Mutation:      High mutation score         │
└────────────────────────────────────────────┘

Result:
✅ Confident deployments
✅ Fearless refactoring
✅ Living documentation
✅ Early bug detection
✅ Production reliability
```

---

**TDD + Legion + Shape Up = Reliable, Predictable, Professional Software Delivery** 🧪

_Red, Green, Refactor. Repeat. Ship with confidence._ ✅
