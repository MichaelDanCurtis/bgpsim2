# Data Model & Entity Schemas

**Feature**: Educational BGP/OSPF Web Training Platform  
**Created**: 2025-11-18  
**Status**: Phase 1 - Data Model Complete

## Overview

This document defines all entities, their attributes, relationships, validation rules, and state transitions for the educational web platform. Schemas are designed for:
- **PostgreSQL** backend storage (JSONB for flexible config fields)
- **Rust serde** serialization for API contracts
- **IndexedDB** client-side caching for offline support

---

## Entity Relationship Diagram

```
┌─────────────┐
│    User     │
└──────┬──────┘
       │ 1
       │
       │ *
┌──────┴────────┐         ┌──────────────┐
│   Progress    │────────▶│    Lesson    │
└───────────────┘ *     1 └──────┬───────┘
                                 │ 1
       ┌─────────────────────────┤
       │                         │
       │ *                       │ *
┌──────┴──────┐          ┌───────┴──────┐
│    Task     │          │     Quiz     │
└─────────────┘          └──────┬───────┘
                                │ 1
                                │
                                │ *
                         ┌──────┴──────┐
                         │  Question   │
                         └─────────────┘

┌─────────────┐
│   Scenario  │
└──────┬──────┘
       │ 1
       │
       │ 1
┌──────┴──────┐          ┌──────────────┐
│  Topology   │───────┬─▶│   Router     │
└─────────────┘ 1     │  └──────────────┘
                      │
                      │  ┌──────────────┐
                      └─▶│     Link     │
                         └──────────────┘

┌─────────────┐
│  Bookmark   │──────────▶ SimulationState
└─────────────┘ *       1 └──────────────┘
```

---

## Core Entities

### 1. User

**Purpose**: Represents a student, instructor, or admin with authentication and workspace isolation.

**Schema** (Rust):
```rust
#[derive(Debug, Clone, Serialize, Deserialize, sqlx::FromRow)]
pub struct User {
    pub user_id: Uuid,
    pub email: String,
    pub password_hash: String,  // Argon2 hashed
    pub name: String,
    pub role: UserRole,
    pub workspace_id: Uuid,
    pub created_at: DateTime<Utc>,
    pub last_login: Option<DateTime<Utc>>,
}

#[derive(Debug, Clone, Copy, Serialize, Deserialize, sqlx::Type)]
#[sqlx(type_name = "user_role", rename_all = "lowercase")]
pub enum UserRole {
    Student,
    Instructor,
    Admin,
}
```

**PostgreSQL Table**:
```sql
CREATE TYPE user_role AS ENUM ('student', 'instructor', 'admin');

CREATE TABLE users (
    user_id UUID PRIMARY KEY DEFAULT gen_random_uuid(),
    email VARCHAR(255) UNIQUE NOT NULL,
    password_hash VARCHAR(255) NOT NULL,
    name VARCHAR(255) NOT NULL,
    role user_role NOT NULL DEFAULT 'student',
    workspace_id UUID NOT NULL,
    created_at TIMESTAMPTZ NOT NULL DEFAULT NOW(),
    last_login TIMESTAMPTZ,
    CONSTRAINT email_format CHECK (email ~* '^[A-Za-z0-9._%+-]+@[A-Za-z0-9.-]+\.[A-Z|a-z]{2,}$')
);

CREATE INDEX idx_users_email ON users(email);
CREATE INDEX idx_users_workspace ON users(workspace_id);
```

**Validation Rules**:
- Email MUST match RFC 5322 format
- Password MUST be ≥12 characters, hashed with Argon2
- Name MUST be 1-255 characters
- `workspace_id` auto-generated on user creation

**State Transitions**: None (stateless entity)

---

### 2. Lesson

**Purpose**: Structured learning content with tasks, quizzes, and topology foundation.

**Schema** (Rust):
```rust
#[derive(Debug, Clone, Serialize, Deserialize, sqlx::FromRow)]
pub struct Lesson {
    pub lesson_id: Uuid,
    pub title: String,
    pub description: String,
    pub objectives: Vec<String>,  // Stored as JSONB
    pub difficulty: Difficulty,
    pub estimated_time_minutes: i32,
    pub topology_id: Uuid,
    pub task_ids: Vec<Uuid>,      // Ordered list
    pub quiz_ids: Vec<Uuid>,
    pub created_by: Uuid,          // Instructor user_id
    pub version: i32,
    pub published_status: PublishedStatus,
    pub created_at: DateTime<Utc>,
    pub updated_at: DateTime<Utc>,
}

#[derive(Debug, Clone, Copy, Serialize, Deserialize, sqlx::Type)]
#[sqlx(type_name = "difficulty", rename_all = "lowercase")]
pub enum Difficulty {
    Beginner,
    Intermediate,
    Advanced,
}

#[derive(Debug, Clone, Copy, Serialize, Deserialize, sqlx::Type)]
#[sqlx(type_name = "published_status", rename_all = "lowercase")]
pub enum PublishedStatus {
    Draft,
    Published,
    Archived,
}
```

**PostgreSQL Table**:
```sql
CREATE TYPE difficulty AS ENUM ('beginner', 'intermediate', 'advanced');
CREATE TYPE published_status AS ENUM ('draft', 'published', 'archived');

CREATE TABLE lessons (
    lesson_id UUID PRIMARY KEY DEFAULT gen_random_uuid(),
    title VARCHAR(255) NOT NULL,
    description TEXT NOT NULL,
    objectives JSONB NOT NULL,  -- Array of strings
    difficulty difficulty NOT NULL,
    estimated_time_minutes INTEGER NOT NULL,
    topology_id UUID NOT NULL REFERENCES topologies(topology_id) ON DELETE RESTRICT,
    task_ids JSONB NOT NULL,    -- Ordered array of UUIDs
    quiz_ids JSONB NOT NULL,    -- Array of UUIDs
    created_by UUID NOT NULL REFERENCES users(user_id),
    version INTEGER NOT NULL DEFAULT 1,
    published_status published_status NOT NULL DEFAULT 'draft',
    created_at TIMESTAMPTZ NOT NULL DEFAULT NOW(),
    updated_at TIMESTAMPTZ NOT NULL DEFAULT NOW(),
    CONSTRAINT estimated_time_positive CHECK (estimated_time_minutes > 0)
);

CREATE INDEX idx_lessons_created_by ON lessons(created_by);
CREATE INDEX idx_lessons_status ON lessons(published_status);
CREATE INDEX idx_lessons_difficulty ON lessons(difficulty);
```

**Validation Rules**:
- Title MUST be 1-255 characters
- Description MUST be non-empty
- Objectives MUST contain 1-10 items
- `estimated_time_minutes` MUST be 1-300 (5 hours max)
- Task order MUST be maintained in `task_ids` array

**State Transitions**:
```
Draft ─────┐
    ▲      │ publish
    │      ▼
    │  Published ────┐
    │      ▲         │ archive
    │      │         ▼
    └──────┴───── Archived
      (unarchive)
```

---

### 3. Task

**Purpose**: Individual step within a lesson with validation criteria and progressive hints.

**Schema** (Rust):
```rust
#[derive(Debug, Clone, Serialize, Deserialize, sqlx::FromRow)]
pub struct Task {
    pub task_id: Uuid,
    pub lesson_id: Uuid,
    pub order: i32,
    pub description: String,
    pub validation_criteria: ValidationCriteria,
    pub hints: Vec<Hint>,
    pub created_at: DateTime<Utc>,
}

#[derive(Debug, Clone, Serialize, Deserialize, sqlx::Type)]
#[sqlx(type_name = "jsonb")]
pub struct ValidationCriteria {
    pub rules: Vec<ValidationRule>,
}

#[derive(Debug, Clone, Serialize, Deserialize)]
#[serde(tag = "type")]
pub enum ValidationRule {
    BgpSessionState { 
        router_id: String, 
        neighbor_ip: String, 
        expected_state: BgpState 
    },
    RouteExists { 
        router_id: String, 
        prefix: String, 
        next_hop: Option<String> 
    },
    OspfNeighborCount { 
        router_id: String, 
        expected_count: u32 
    },
    ConfigContains { 
        router_id: String, 
        pattern: String 
    },
}

#[derive(Debug, Clone, Serialize, Deserialize)]
pub struct Hint {
    pub level: HintLevel,
    pub text: String,
}

#[derive(Debug, Clone, Copy, Serialize, Deserialize, PartialEq)]
pub enum HintLevel {
    Nudge,      // Level 1: General direction
    Detailed,   // Level 2: Specific steps
    Solution,   // Level 3: Complete answer
}

#[derive(Debug, Clone, Copy, Serialize, Deserialize, sqlx::Type)]
#[sqlx(type_name = "bgp_state", rename_all = "lowercase")]
pub enum BgpState {
    Idle,
    Connect,
    Active,
    OpenSent,
    OpenConfirm,
    Established,
}
```

**PostgreSQL Table**:
```sql
CREATE TYPE bgp_state AS ENUM ('idle', 'connect', 'active', 'opensent', 'openconfirm', 'established');

CREATE TABLE tasks (
    task_id UUID PRIMARY KEY DEFAULT gen_random_uuid(),
    lesson_id UUID NOT NULL REFERENCES lessons(lesson_id) ON DELETE CASCADE,
    "order" INTEGER NOT NULL,
    description TEXT NOT NULL,
    validation_criteria JSONB NOT NULL,
    hints JSONB NOT NULL,  -- Array of {level, text}
    created_at TIMESTAMPTZ NOT NULL DEFAULT NOW(),
    CONSTRAINT task_order_positive CHECK ("order" > 0),
    UNIQUE(lesson_id, "order")
);

CREATE INDEX idx_tasks_lesson ON tasks(lesson_id, "order");
```

**Validation Rules**:
- Description MUST be non-empty
- `validation_criteria` MUST contain 1-10 rules
- Hints MUST have exactly 3 levels (Nudge, Detailed, Solution) in order
- `order` MUST be positive and unique within lesson

**State Transitions**: Tasks are stateless; completion tracked in Progress entity

---

### 4. Topology

**Purpose**: Network graph representation with routers, links, and configurations.

**Schema** (Rust):
```rust
#[derive(Debug, Clone, Serialize, Deserialize, sqlx::FromRow)]
pub struct Topology {
    pub topology_id: Uuid,
    pub name: String,
    pub router_ids: Vec<Uuid>,
    pub link_ids: Vec<Uuid>,
    pub external_peer_ids: Vec<Uuid>,
    pub config_snapshot: serde_json::Value,  // Full bgpsim config
    pub created_at: DateTime<Utc>,
    pub owner_user_id: Uuid,
}

impl Topology {
    pub fn to_yaml(&self) -> Result<String, serde_yaml::Error> {
        serde_yaml::to_string(&self.config_snapshot)
    }
    
    pub fn from_yaml(yaml: &str) -> Result<Self, serde_yaml::Error> {
        // Parse and construct topology
    }
}
```

**PostgreSQL Table**:
```sql
CREATE TABLE topologies (
    topology_id UUID PRIMARY KEY DEFAULT gen_random_uuid(),
    name VARCHAR(255) NOT NULL,
    router_ids JSONB NOT NULL,        -- Array of UUIDs
    link_ids JSONB NOT NULL,          -- Array of UUIDs
    external_peer_ids JSONB NOT NULL, -- Array of UUIDs
    config_snapshot JSONB NOT NULL,   -- Full network config
    created_at TIMESTAMPTZ NOT NULL DEFAULT NOW(),
    owner_user_id UUID NOT NULL REFERENCES users(user_id),
    CONSTRAINT topology_name_length CHECK (char_length(name) BETWEEN 1 AND 255)
);

CREATE INDEX idx_topologies_owner ON topologies(owner_user_id);
```

**Validation Rules**:
- Name MUST be 1-255 characters
- MUST contain ≥1 router
- Links MUST reference existing routers
- `config_snapshot` MUST be valid bgpsim YAML

**State Transitions**: None (immutable once created; modifications create new version)

---

### 5. Router

**Purpose**: Network device node with BGP/OSPF configuration and runtime state.

**Schema** (Rust):
```rust
#[derive(Debug, Clone, Serialize, Deserialize, sqlx::FromRow)]
pub struct Router {
    pub router_id: Uuid,
    pub topology_id: Uuid,
    pub name: String,            // Display name (e.g., "R1")
    pub position: Position,      // Canvas coordinates
    pub as_number: u32,
    pub bgp_config: BgpConfig,
    pub ospf_config: Option<OspfConfig>,
    pub running_config: String,  // Full CLI config text
    pub state: RouterState,
}

#[derive(Debug, Clone, Serialize, Deserialize)]
pub struct Position {
    pub x: f64,
    pub y: f64,
}

#[derive(Debug, Clone, Serialize, Deserialize)]
pub struct BgpConfig {
    pub router_id: String,       // IP format
    pub neighbors: Vec<BgpNeighbor>,
    pub networks: Vec<String>,   // Prefixes to advertise
    pub route_maps: Vec<RouteMap>,
}

#[derive(Debug, Clone, Serialize, Deserialize)]
pub struct BgpNeighbor {
    pub ip: String,
    pub remote_as: u32,
    pub description: Option<String>,
}

#[derive(Debug, Clone, Serialize, Deserialize)]
pub struct OspfConfig {
    pub router_id: String,
    pub areas: Vec<OspfArea>,
}

#[derive(Debug, Clone, Serialize, Deserialize)]
pub struct OspfArea {
    pub area_id: u32,
    pub networks: Vec<String>,   // IP/mask
}

#[derive(Debug, Clone, Serialize, Deserialize)]
pub struct RouteMap {
    pub name: String,
    pub entries: Vec<RouteMapEntry>,
}

#[derive(Debug, Clone, Serialize, Deserialize)]
pub struct RouteMapEntry {
    pub sequence: u32,
    pub action: RouteMapAction,
    pub conditions: Vec<String>,
    pub set_actions: Vec<String>,
}

#[derive(Debug, Clone, Copy, Serialize, Deserialize)]
pub enum RouteMapAction {
    Permit,
    Deny,
}

#[derive(Debug, Clone, Copy, Serialize, Deserialize, sqlx::Type)]
#[sqlx(type_name = "router_state", rename_all = "lowercase")]
pub enum RouterState {
    Running,
    Stopped,
    Failed,
}
```

**PostgreSQL Table**:
```sql
CREATE TYPE router_state AS ENUM ('running', 'stopped', 'failed');

CREATE TABLE routers (
    router_id UUID PRIMARY KEY DEFAULT gen_random_uuid(),
    topology_id UUID NOT NULL REFERENCES topologies(topology_id) ON DELETE CASCADE,
    name VARCHAR(50) NOT NULL,
    position JSONB NOT NULL,          -- {x, y}
    as_number INTEGER NOT NULL,
    bgp_config JSONB NOT NULL,
    ospf_config JSONB,                -- Optional
    running_config TEXT NOT NULL,
    state router_state NOT NULL DEFAULT 'stopped',
    CONSTRAINT as_number_range CHECK (as_number BETWEEN 1 AND 4294967295),
    UNIQUE(topology_id, name)
);

CREATE INDEX idx_routers_topology ON routers(topology_id);
```

**Validation Rules**:
- Name MUST match `^[A-Za-z0-9_-]+$` (alphanumeric + underscore/dash)
- AS number MUST be 1-4294967295
- BGP router_id MUST be valid IPv4 address
- Position x, y MUST be ≥0

**State Transitions**:
```
Stopped ──────┐
    ▲         │ start
    │         ▼
    │     Running ─────┐
    │         │        │ error
    │         │ stop   ▼
    └─────────┴───── Failed
         (reset)
```

---

### 6. Link

**Purpose**: Connection between routers with cost, latency, and operational state.

**Schema** (Rust):
```rust
#[derive(Debug, Clone, Serialize, Deserialize, sqlx::FromRow)]
pub struct Link {
    pub link_id: Uuid,
    pub topology_id: Uuid,
    pub source_router_id: Uuid,
    pub dest_router_id: Uuid,
    pub weight: i32,             // OSPF cost
    pub latency_ms: Option<f64>, // Optional for real-time mode
    pub state: LinkState,
}

#[derive(Debug, Clone, Copy, Serialize, Deserialize, sqlx::Type)]
#[sqlx(type_name = "link_state", rename_all = "lowercase")]
pub enum LinkState {
    Up,
    Down,
    Degraded,  // Future: packet loss simulation
}
```

**PostgreSQL Table**:
```sql
CREATE TYPE link_state AS ENUM ('up', 'down', 'degraded');

CREATE TABLE links (
    link_id UUID PRIMARY KEY DEFAULT gen_random_uuid(),
    topology_id UUID NOT NULL REFERENCES topologies(topology_id) ON DELETE CASCADE,
    source_router_id UUID NOT NULL REFERENCES routers(router_id),
    dest_router_id UUID NOT NULL REFERENCES routers(router_id),
    weight INTEGER NOT NULL DEFAULT 1,
    latency_ms DOUBLE PRECISION,
    state link_state NOT NULL DEFAULT 'up',
    CONSTRAINT weight_positive CHECK (weight > 0),
    CONSTRAINT latency_positive CHECK (latency_ms IS NULL OR latency_ms >= 0),
    CONSTRAINT no_self_loop CHECK (source_router_id != dest_router_id),
    UNIQUE(topology_id, source_router_id, dest_router_id)
);

CREATE INDEX idx_links_topology ON links(topology_id);
CREATE INDEX idx_links_source ON links(source_router_id);
CREATE INDEX idx_links_dest ON links(dest_router_id);
```

**Validation Rules**:
- Weight MUST be >0
- Latency MUST be ≥0 if specified
- Source and destination MUST be different routers
- Bidirectional links require two Link entries (A→B and B→A)

**State Transitions**:
```
Up ←──────┐
 │         │ restore
 │ fail    │
 ▼         │
Down ──────┘
```

---

### 7. SimulationState

**Purpose**: Point-in-time snapshot of simulation with routing tables, BGP FSM states, and events.

**Schema** (Rust):
```rust
#[derive(Debug, Clone, Serialize, Deserialize)]
pub struct SimulationState {
    pub state_id: Uuid,
    pub topology_id: Uuid,
    pub timestamp: f64,          // Simulation time in seconds
    pub router_states: HashMap<Uuid, RouterRuntimeState>,
    pub event_log: Vec<SimulationEvent>,
    pub bookmark_label: Option<String>,
}

#[derive(Debug, Clone, Serialize, Deserialize)]
pub struct RouterRuntimeState {
    pub routing_table: Vec<Route>,
    pub bgp_neighbors: Vec<BgpNeighborState>,
    pub ospf_neighbors: Vec<OspfNeighborState>,
}

#[derive(Debug, Clone, Serialize, Deserialize)]
pub struct Route {
    pub prefix: String,
    pub next_hop: String,
    pub protocol: RouteProtocol,
    pub metric: u32,
    pub admin_distance: u32,
}

#[derive(Debug, Clone, Copy, Serialize, Deserialize)]
pub enum RouteProtocol {
    Connected,
    Static,
    Bgp,
    Ospf,
}

#[derive(Debug, Clone, Serialize, Deserialize)]
pub struct BgpNeighborState {
    pub neighbor_ip: String,
    pub remote_as: u32,
    pub state: BgpState,
    pub uptime_seconds: u32,
    pub prefixes_received: u32,
    pub prefixes_sent: u32,
}

#[derive(Debug, Clone, Serialize, Deserialize)]
pub struct OspfNeighborState {
    pub neighbor_id: String,
    pub state: OspfState,
    pub priority: u8,
}

#[derive(Debug, Clone, Copy, Serialize, Deserialize)]
pub enum OspfState {
    Down,
    Init,
    TwoWay,
    ExStart,
    Exchange,
    Loading,
    Full,
}

#[derive(Debug, Clone, Serialize, Deserialize)]
pub struct SimulationEvent {
    pub timestamp: f64,
    pub event_type: EventType,
    pub router_id: Option<Uuid>,
    pub details: String,
}

#[derive(Debug, Clone, Serialize, Deserialize)]
pub enum EventType {
    BgpSessionUp,
    BgpSessionDown,
    BgpUpdateSent,
    BgpUpdateReceived,
    OspfNeighborUp,
    OspfNeighborDown,
    LinkUp,
    LinkDown,
    ConfigChange,
}
```

**Storage Strategy**:
- **In-Memory**: Recent 5 checkpoints (last 50 seconds at 10s intervals)
- **IndexedDB**: Older checkpoints compressed with miniz_oxide
- **PostgreSQL**: Not stored in DB (too large, client-side only)

**Validation Rules**:
- Timestamp MUST be monotonically increasing
- Router states MUST match topology router_ids

---

### 8. Scenario

**Purpose**: Reusable topology template with metadata and popularity tracking.

**Schema** (Rust):
```rust
#[derive(Debug, Clone, Serialize, Deserialize, sqlx::FromRow)]
pub struct Scenario {
    pub scenario_id: Uuid,
    pub name: String,
    pub description: String,
    pub difficulty: Difficulty,
    pub protocol_tags: Vec<ProtocolTag>,
    pub topology_id: Uuid,
    pub is_public: bool,
    pub popularity_score: i32,   // View count + clone count
    pub created_by: Uuid,
    pub created_at: DateTime<Utc>,
}

#[derive(Debug, Clone, Copy, Serialize, Deserialize, sqlx::Type)]
#[sqlx(type_name = "protocol_tag", rename_all = "lowercase")]
pub enum ProtocolTag {
    Bgp,
    Ospf,
    Anycast,
    Mpls,
    VpnL3,
}
```

**PostgreSQL Table**:
```sql
CREATE TYPE protocol_tag AS ENUM ('bgp', 'ospf', 'anycast', 'mpls', 'vpnl3');

CREATE TABLE scenarios (
    scenario_id UUID PRIMARY KEY DEFAULT gen_random_uuid(),
    name VARCHAR(255) NOT NULL,
    description TEXT NOT NULL,
    difficulty difficulty NOT NULL,
    protocol_tags protocol_tag[] NOT NULL,  -- Array type
    topology_id UUID NOT NULL REFERENCES topologies(topology_id),
    is_public BOOLEAN NOT NULL DEFAULT false,
    popularity_score INTEGER NOT NULL DEFAULT 0,
    created_by UUID NOT NULL REFERENCES users(user_id),
    created_at TIMESTAMPTZ NOT NULL DEFAULT NOW(),
    CONSTRAINT popularity_non_negative CHECK (popularity_score >= 0)
);

CREATE INDEX idx_scenarios_public ON scenarios(is_public) WHERE is_public = true;
CREATE INDEX idx_scenarios_difficulty ON scenarios(difficulty);
CREATE INDEX idx_scenarios_protocol ON scenarios USING GIN(protocol_tags);
```

**Validation Rules**:
- Name MUST be 1-255 characters
- MUST have 1-5 protocol tags
- Popularity score MUST be ≥0

---

### 9. Quiz

**Purpose**: Assessment questions embedded in lessons with auto-grading.

**Schema** (Rust):
```rust
#[derive(Debug, Clone, Serialize, Deserialize, sqlx::FromRow)]
pub struct Quiz {
    pub quiz_id: Uuid,
    pub lesson_id: Uuid,
    pub question_ids: Vec<Uuid>,
    pub auto_grade: bool,
    pub passing_score: f32,      // Percentage (0.0-1.0)
}
```

**PostgreSQL Table**:
```sql
CREATE TABLE quizzes (
    quiz_id UUID PRIMARY KEY DEFAULT gen_random_uuid(),
    lesson_id UUID NOT NULL REFERENCES lessons(lesson_id) ON DELETE CASCADE,
    question_ids JSONB NOT NULL,
    auto_grade BOOLEAN NOT NULL DEFAULT true,
    passing_score REAL NOT NULL DEFAULT 0.7,
    CONSTRAINT passing_score_range CHECK (passing_score BETWEEN 0.0 AND 1.0)
);

CREATE INDEX idx_quizzes_lesson ON quizzes(lesson_id);
```

---

### 10. Question

**Purpose**: Individual quiz question with answer validation.

**Schema** (Rust):
```rust
#[derive(Debug, Clone, Serialize, Deserialize, sqlx::FromRow)]
pub struct Question {
    pub question_id: Uuid,
    pub quiz_id: Uuid,
    pub text: String,
    pub question_type: QuestionType,
    pub options: Option<Vec<String>>,  // For multiple choice
    pub correct_answer: String,
    pub explanation: String,
}

#[derive(Debug, Clone, Copy, Serialize, Deserialize, sqlx::Type)]
#[sqlx(type_name = "question_type", rename_all = "snake_case")]
pub enum QuestionType {
    MultipleChoice,
    FillInBlank,
}
```

**PostgreSQL Table**:
```sql
CREATE TYPE question_type AS ENUM ('multiple_choice', 'fill_in_blank');

CREATE TABLE questions (
    question_id UUID PRIMARY KEY DEFAULT gen_random_uuid(),
    quiz_id UUID NOT NULL REFERENCES quizzes(quiz_id) ON DELETE CASCADE,
    text TEXT NOT NULL,
    question_type question_type NOT NULL,
    options JSONB,  -- Required if multiple_choice
    correct_answer TEXT NOT NULL,
    explanation TEXT NOT NULL,
    CONSTRAINT options_required_for_mc CHECK (
        (question_type != 'multiple_choice') OR (options IS NOT NULL)
    )
);

CREATE INDEX idx_questions_quiz ON questions(quiz_id);
```

---

### 11. Progress

**Purpose**: Tracks student lesson completion, task status, hints used, and quiz scores.

**Schema** (Rust):
```rust
#[derive(Debug, Clone, Serialize, Deserialize, sqlx::FromRow)]
pub struct Progress {
    pub progress_id: Uuid,
    pub user_id: Uuid,
    pub lesson_id: Uuid,
    pub current_task_id: Option<Uuid>,
    pub completed_tasks: Vec<Uuid>,
    pub hints_used: HashMap<Uuid, u32>,  // task_id -> hint count
    pub quiz_scores: HashMap<Uuid, f32>, // quiz_id -> score (0.0-1.0)
    pub started_at: DateTime<Utc>,
    pub completion_timestamp: Option<DateTime<Utc>>,
}
```

**PostgreSQL Table**:
```sql
CREATE TABLE progress (
    progress_id UUID PRIMARY KEY DEFAULT gen_random_uuid(),
    user_id UUID NOT NULL REFERENCES users(user_id) ON DELETE CASCADE,
    lesson_id UUID NOT NULL REFERENCES lessons(lesson_id) ON DELETE CASCADE,
    current_task_id UUID REFERENCES tasks(task_id),
    completed_tasks JSONB NOT NULL DEFAULT '[]',
    hints_used JSONB NOT NULL DEFAULT '{}',
    quiz_scores JSONB NOT NULL DEFAULT '{}',
    started_at TIMESTAMPTZ NOT NULL DEFAULT NOW(),
    completion_timestamp TIMESTAMPTZ,
    UNIQUE(user_id, lesson_id)
);

CREATE INDEX idx_progress_user ON progress(user_id);
CREATE INDEX idx_progress_lesson ON progress(lesson_id);
```

**Validation Rules**:
- `current_task_id` MUST be from lesson's task list
- `completed_tasks` MUST be subset of lesson's tasks
- Quiz scores MUST be 0.0-1.0

**State Transitions**:
```
Not Started ────┐
                │ start_lesson
                ▼
            In Progress ────┐
                │           │ complete_final_task
                │           ▼
                └──────▶ Completed
```

---

### 12. Bookmark

**Purpose**: User-saved simulation states with custom labels for time machine.

**Schema** (Rust):
```rust
#[derive(Debug, Clone, Serialize, Deserialize, sqlx::FromRow)]
pub struct Bookmark {
    pub bookmark_id: Uuid,
    pub user_id: Uuid,
    pub topology_id: Uuid,
    pub simulation_state: SimulationState,  // Serialized state
    pub label: String,
    pub created_at: DateTime<Utc>,
}
```

**PostgreSQL Table**:
```sql
CREATE TABLE bookmarks (
    bookmark_id UUID PRIMARY KEY DEFAULT gen_random_uuid(),
    user_id UUID NOT NULL REFERENCES users(user_id) ON DELETE CASCADE,
    topology_id UUID NOT NULL REFERENCES topologies(topology_id) ON DELETE CASCADE,
    simulation_state JSONB NOT NULL,
    label VARCHAR(255) NOT NULL,
    created_at TIMESTAMPTZ NOT NULL DEFAULT NOW()
);

CREATE INDEX idx_bookmarks_user ON bookmarks(user_id);
CREATE INDEX idx_bookmarks_topology ON bookmarks(topology_id);
```

**Storage Strategy**:
- Store compressed JSON in JSONB field
- Limit 50 bookmarks per user per topology

---

## Shared Types & Validation

### Router Name Convention
- Format: `^[A-Za-z][A-Za-z0-9_-]*$` (letter start, alphanumeric + underscore/dash)
- Examples: `R1`, `Border-Router-1`, `Core_Switch_A`

### IP Address Validation
- IPv4: `^\d{1,3}\.\d{1,3}\.\d{1,3}\.\d{1,3}$` with each octet 0-255
- IPv6: Standard colon-separated hexadecimal
- Use `ipnet` crate for parsing and validation

### Prefix Validation
- CIDR notation: `IP/mask` (e.g., `192.0.2.0/24`)
- Mask MUST be 0-32 for IPv4, 0-128 for IPv6

### AS Number Validation
- Range: 1-4294967295 (32-bit)
- Reserved ranges avoided in templates (0, 23456, 64496-64511, 65535)

---

## Migration Strategy

### Version 1.0 Schema (Initial)
```sql
-- Run all CREATE TABLE statements in order:
-- 1. users
-- 2. topologies
-- 3. routers, links (depends on topologies)
-- 4. lessons (depends on topologies, users)
-- 5. tasks, quizzes (depends on lessons)
-- 6. questions (depends on quizzes)
-- 7. scenarios (depends on topologies, users)
-- 8. progress (depends on users, lessons, tasks)
-- 9. bookmarks (depends on users, topologies)
```

### Future Migrations
- Use `sqlx migrate` for version-controlled schema changes
- Track in `migrations/` directory
- Test migrations on dev database before production

---

## Performance Considerations

1. **Indexes**: All foreign keys indexed for join performance
2. **JSONB Columns**: Use GIN indexes for array containment queries
3. **Partitioning**: Consider partitioning `progress` table by `user_id` if >1M rows
4. **Archival**: Move completed lessons >1 year old to archive table

---

## Constitution Compliance Notes

- ✅ **Principle I (TDD)**: All entities have validation rules testable via unit tests
- ✅ **Principle II (Library-First)**: Validation logic extracted to `bgpemu2-core` crate
- ✅ **Principle V (Versioning)**: Lessons versioned; migration path for schema changes
- ✅ **Principle XIII (Rust Discipline)**: All derives use `serde`, `sqlx`; no unsafe code

---

**Status**: ✅ Data model complete. Ready for API contract generation (Phase 1).
