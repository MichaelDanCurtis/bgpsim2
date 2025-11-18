# Research & Technology Decisions

**Feature**: Educational BGP/OSPF Web Training Platform  
**Created**: 2025-11-18  
**Status**: Phase 0 - Research Complete

## Executive Summary

This document resolves all technical unknowns identified during planning for the educational web platform. Key decisions: **Yew 0.21+** for frontend (already in use), **GitHub Pages + GitHub Actions** for static hosting, **Backend API as separate service** (not GitHub Pages compatible for dynamic features), **Yewdux** for state management, **IndexedDB** for client-side persistence.

---

## Research Tasks

### 1. Frontend Framework Choice

**Unknown**: Existing bgpsim-web framework and upgrade path for educational features.

**Finding**: 
- **Current Stack**: Yew 0.20.0 (Rust WASM framework) with Yewdux 0.9 for state management
- **Build Tool**: Trunk (Rust WASM bundler) configured in `Trunk.toml`
- **UI Components**: yew-lucide (icon library), custom Canvas component for topology visualization
- **Architecture**: Component-based with functional components, hooks, and context API

**Decision**: **Continue with Yew framework, upgrade to Yew 0.21+**

**Rationale**:
1. **Existing Investment**: bgpsim-web already uses Yew with sophisticated topology canvas, state management, and WASM integration
2. **Rust Alignment**: Keeps frontend in Rust per user preference, maintains type safety across full stack
3. **Component Reusability**: Existing `Canvas`, `Sidebar`, `Header` components provide foundation for visual designer
4. **WASM Performance**: Yew's WASM runtime meets <100ms CLI response requirement (Constitution X)
5. **GitHub Pages Compatible**: Trunk generates static assets deployable to GitHub Pages

**Alternatives Considered**:
- **Leptos**: More modern Rust framework with fine-grained reactivity, but would require full rewrite
- **Dioxus**: React-like Rust framework, but less mature ecosystem than Yew
- **TypeScript/React**: Better ecosystem for web features, but loses Rust type safety and requires dual-language maintenance

**Migration Path**:
- Upgrade Yew 0.20 → 0.21+ (breaking changes in hook APIs)
- Extend existing Canvas component for lesson overlays, CLI modals, sidecar panels
- Add new components: `CliTerminal`, `LessonBuilder`, `TimelineControl`, `SidecarPanel`

---

### 2. WebSocket Library for Real-Time Simulation Updates

**Unknown**: Best Rust WebSocket library for browser WASM + server communication.

**Finding**:
- **Client-Side Options**:
  - `gloo-net` (already in dependencies): High-level WASM networking with WebSocket support
  - `ws_stream_wasm`: Lower-level WebSocket bindings for web-sys
  - `web-sys` WebSocket API: Direct browser API access

- **Server-Side Options**:
  - `tokio-tungstenite`: Async WebSocket with Tokio runtime (most popular)
  - `axum` with WebSocket layer: Integrated with Axum web framework
  - `actix-web` WebSocket: Integrated with Actix framework

**Decision**: **gloo-net (client) + tokio-tungstenite with Axum (server)**

**Rationale**:
1. **gloo-net**: Already in dependencies, high-level API abstracts browser WebSocket complexity
2. **tokio-tungstenite**: Industry standard, excellent async support, integrates with Axum
3. **Axum**: Modern, ergonomic, Tower-based web framework with first-class WebSocket support
4. **Type Safety**: Shared Rust types between client/server via serde (Topology, SimulationState, etc.)

**Alternatives Considered**:
- **actix-web**: Mature but more boilerplate than Axum
- **warp**: Functional but less intuitive API than Axum

**WebSocket Message Protocol**:
```rust
// Shared types between client and server
#[derive(Serialize, Deserialize)]
enum WsMessage {
    SimulationUpdate { state: SimulationState, timestamp: f64 },
    EventNotification { event: BgpEvent, router_id: RouterId },
    ValidationResult { task_id: TaskId, passed: bool, diagnostics: String },
    Error { code: u16, message: String },
}
```

---

### 3. GitHub Pages Deployment Strategy

**Unknown**: How to deploy Rust WASM app with backend API to GitHub Pages.

**Finding**:
- **GitHub Pages Constraints**:
  - Static file hosting only (HTML, CSS, JS, WASM)
  - No server-side execution (no WebSocket server, no database, no API)
  - HTTPS by default at `username.github.io/repo-name`
  - Custom domain support with CNAME

- **Architecture Options**:
  1. **Full Static (GitHub Pages Only)**: Client-only WASM app with no persistence, no multi-user
  2. **Hybrid (GitHub Pages + External Backend)**: Static frontend on GitHub Pages, backend API on separate host (Fly.io, Railway, Cloudflare Workers)
  3. **Monorepo with Actions Deploy**: Build WASM frontend, deploy to `gh-pages` branch; deploy backend elsewhere

**Decision**: **Hybrid Architecture - Static Frontend on GitHub Pages + Backend API on Fly.io/Railway**

**Rationale**:
1. **GitHub Pages for Frontend**: Perfect for static WASM assets, free hosting, automatic HTTPS
2. **Separate Backend Service**: Required for user auth, lesson persistence, WebSocket server, database
3. **Development Simplicity**: Frontend and backend can develop independently with local mock server
4. **Cost-Effective**: GitHub Pages free + Fly.io free tier (3 shared-cpu VMs) or Railway free tier
5. **Production Ready**: Can migrate backend to paid tier as usage scales (Constitution X: 100 concurrent users)

**Deployment Workflow**:
1. **Frontend (GitHub Pages)**:
   - Trunk builds WASM to `dist/` directory
   - GitHub Actions workflow deploys `dist/` to `gh-pages` branch
   - Access at `https://username.github.io/bgpemu2/`

2. **Backend (Fly.io/Railway)**:
   - Axum server with WebSocket support
   - PostgreSQL database (Fly.io Postgres or Railway DB)
   - Deployed via CLI or GitHub Actions
   - CORS configured to allow GitHub Pages origin

**Alternatives Considered**:
- **Cloudflare Workers**: Serverless edge compute, but WebSocket support limited (Durable Objects complexity)
- **Netlify/Vercel**: Better serverless functions, but Rust support requires custom Docker containers
- **Full GitHub Pages Static**: Would lose user accounts, lesson persistence, real-time collaboration (acceptable for MVP demo but not production)

**GitHub Actions Workflow** (`.github/workflows/deploy.yml`):
```yaml
name: Deploy to GitHub Pages
on:
  push:
    branches: [main]
jobs:
  build-deploy:
    runs-on: ubuntu-latest
    steps:
      - uses: actions/checkout@v3
      - uses: jetli/trunk-action@v0.4.0
      - uses: jetli/wasm-bindgen-action@v0.2.0
      - name: Build WASM
        run: |
          cd bgpsim-web
          trunk build --release --public-url /bgpemu2/
      - name: Deploy to GitHub Pages
        uses: peaceiris/actions-gh-pages@v3
        with:
          github_token: ${{ secrets.GITHUB_TOKEN }}
          publish_dir: ./bgpsim-web/dist
```

---

### 4. State Management for Time Machine Feature

**Unknown**: How to efficiently store and restore simulation states for rewind/fast-forward.

**Finding**:
- **State Size Estimation**:
  - 100-router topology: ~50KB routing tables + 20KB BGP FSM states = 70KB per snapshot
  - 10-minute scenario at 1 snapshot/second = 600 snapshots × 70KB = 42MB
  - Browser memory limit: ~2GB typical, but should target <500MB for time machine

- **Storage Strategies**:
  1. **Full Snapshots**: Store complete state at each timestamp (simple, memory-heavy)
  2. **Event Sourcing**: Store initial state + event log, replay to reconstruct (memory-efficient, slow for random access)
  3. **Hybrid Checkpoints**: Full snapshots every N seconds + event log between (balanced)
  4. **Differential Compression**: Store diffs between snapshots (complex, good compression)

**Decision**: **Hybrid Checkpoint Strategy with IndexedDB Offload**

**Rationale**:
1. **Checkpoints Every 10 Seconds**: Full state snapshot stored to IndexedDB
2. **In-Memory Event Log**: Keep events between checkpoints in memory
3. **Random Access**: Jump to nearest checkpoint, replay forward from events
4. **Memory Management**: Keep last 5 checkpoints in memory, older in IndexedDB
5. **Compression**: Use `miniz_oxide` (already in deps) for checkpoint compression

**Implementation**:
```rust
struct TimelineManager {
    checkpoints: Vec<Checkpoint>,      // In-memory recent checkpoints
    event_log: Vec<TimedEvent>,        // Events between last checkpoint and now
    checkpoint_interval: Duration,      // 10 seconds default
    db: IndexedDbStore,                // Persistent storage
}

struct Checkpoint {
    timestamp: f64,
    state: CompressedState,            // miniz_oxide compressed
    bookmark_label: Option<String>,
}

impl TimelineManager {
    fn seek_to(&mut self, target_time: f64) -> SimulationState {
        let checkpoint = self.find_nearest_checkpoint(target_time);
        let events = self.event_log.iter()
            .filter(|e| e.timestamp > checkpoint.timestamp && e.timestamp <= target_time);
        replay_events(checkpoint.state.decompress(), events)
    }
}
```

**Alternatives Considered**:
- **Pure Event Sourcing**: Too slow for random access (must replay from beginning)
- **Full Snapshots Only**: Exceeds memory budget for long simulations
- **Server-Side Storage**: Adds latency, requires backend API for time machine (defer to v2)

**Performance Targets**:
- Checkpoint creation: <50ms (background task, non-blocking)
- Seek to arbitrary timestamp: <500ms (within FR-036 requirement)
- Memory footprint: <500MB for 1-hour simulation

---

### 5. CLI Terminal Emulation

**Unknown**: Best approach for per-router interactive CLI in browser.

**Finding**:
- **Terminal Emulator Libraries**:
  - **xterm.js**: Industry standard, used by VS Code, full VT100 emulation (TypeScript)
  - **Rust Terminal Emulators**: 
    - `crossterm`: CLI for native Rust apps (not WASM-compatible)
    - `ratatui`: TUI framework (not browser-compatible)
    - No mature Rust-native browser terminal libraries

- **Architecture Options**:
  1. **xterm.js + JS Interop**: Use xterm.js via wasm-bindgen JS interop
  2. **Custom Canvas-Based Terminal**: Render text in Canvas/WebGL (high effort)
  3. **HTML Text Input + Output Div**: Simple but limited (no VT100, no colors)

**Decision**: **xterm.js with wasm-bindgen JS Interop**

**Rationale**:
1. **Production Quality**: xterm.js is battle-tested (VS Code, Hyper, many cloud IDEs)
2. **Full Terminal Features**: ANSI colors, cursor control, history, copy-paste
3. **Low Integration Cost**: wasm-bindgen makes JS interop straightforward
4. **Performance**: Canvas-based rendering meets <100ms response requirement
5. **Accessibility**: xterm.js has screen reader support built-in (Constitution requirement)

**Implementation**:
```rust
use wasm_bindgen::prelude::*;

#[wasm_bindgen(module = "/js/terminal.js")]
extern "C" {
    type Terminal;
    #[wasm_bindgen(constructor)]
    fn new() -> Terminal;
    #[wasm_bindgen(method)]
    fn write(this: &Terminal, data: &str);
    #[wasm_bindgen(method)]
    fn on_data(this: &Terminal, callback: &Closure<dyn FnMut(String)>);
}

struct CliTerminal {
    xterm: Terminal,
    router_id: RouterId,
    command_buffer: String,
    history: Vec<String>,
}
```

**JS Interop File** (`bgpsim-web/js/terminal.js`):
```javascript
import { Terminal } from 'xterm';
export { Terminal };
```

**Alternatives Considered**:
- **Custom Canvas Terminal**: Reinventing wheel, high maintenance burden
- **HTML-Only Terminal**: Insufficient features for realistic CLI experience

---

### 6. Best Practices for Rust WASM Performance

**Unknown**: How to meet <100ms CLI response and <2s convergence targets in WASM.

**Finding**:
- **WASM Performance Characteristics**:
  - CPU performance: ~80% of native Rust (JIT compilation)
  - Memory access: Direct linear memory, fast
  - Garbage collection: None (Rust owns memory)
  - Main thread blocking: WASM runs on main thread, can block UI

- **Optimization Strategies**:
  1. **Web Workers**: Offload heavy computation (BGP convergence) to worker threads
  2. **Incremental Updates**: Don't recalculate full routing tables, only diffs
  3. **Request Animation Frame**: Batch UI updates to 60fps
  4. **WASM Optimization Flags**: `opt-level = "z"` for size, `lto = true` for performance

**Decision**: **Multi-Pronged Optimization Strategy**

**Rationale**: No single technique sufficient; combine multiple approaches.

**Optimizations**:

1. **Web Workers for Simulation** (via `wasm-bindgen-rayon`):
   ```rust
   use wasm_bindgen_rayon::init_thread_pool;
   
   // Run BGP convergence in worker pool
   let future = spawn(async move {
       network.simulate_until_converged()
   });
   ```

2. **Incremental State Diffing**:
   ```rust
   struct RoutingTableDiff {
       added: Vec<Route>,
       removed: Vec<Route>,
       modified: Vec<(Route, Route)>,  // (old, new)
   }
   
   // Only send diffs over WebSocket
   fn compute_diff(old: &RoutingTable, new: &RoutingTable) -> RoutingTableDiff
   ```

3. **Lazy Loading for Large Topologies**:
   - Render only visible routers on canvas (viewport culling)
   - Load routing tables on-demand when router clicked

4. **Cargo.toml Optimizations**:
   ```toml
   [profile.release]
   opt-level = 3           # Max speed (trade size for perf)
   lto = "thin"            # Link-time optimization
   codegen-units = 1       # Better optimization, slower build
   
   [profile.release.package."*"]
   opt-level = "z"         # Optimize dependencies for size
   ```

5. **Benchmark Targets** (via Criterion):
   - CLI command parsing: <1ms
   - Routing table lookup: <5ms
   - Canvas render (100 routers): <16ms (60fps)
   - BGP convergence (100 routers): <2000ms

**Alternatives Considered**:
- **Full Backend Simulation**: Loses offline capability, adds latency
- **Native Desktop App**: Better performance but loses web accessibility

---

## Technology Stack Summary

| Component | Technology | Rationale |
|-----------|-----------|-----------|
| **Frontend Framework** | Yew 0.21+ | Existing codebase, Rust type safety, WASM performance |
| **State Management** | Yewdux + IndexedDB | Already in use, persistent storage for time machine |
| **Build Tool** | Trunk | Existing, purpose-built for Rust WASM projects |
| **UI Components** | Custom Yew + yew-lucide | Existing canvas, icon library |
| **Terminal Emulator** | xterm.js (JS interop) | Industry standard, accessibility support |
| **WebSocket Client** | gloo-net | Already in deps, high-level WASM networking |
| **Backend Framework** | Axum + tokio | Modern, ergonomic, first-class WebSocket support |
| **WebSocket Server** | tokio-tungstenite | Standard async WebSocket, integrates with Axum |
| **Database** | PostgreSQL | Mature, JSONB support for flexible schemas |
| **Frontend Hosting** | GitHub Pages | Free, static WASM hosting, auto-deploy |
| **Backend Hosting** | Fly.io / Railway | Free tier, Rust support, managed Postgres |
| **CI/CD** | GitHub Actions | Native GitHub integration, Trunk support |
| **Compression** | miniz_oxide | Already in deps, checkpoint compression |
| **Time Machine Storage** | IndexedDB | Browser persistent storage, quota management |

---

## Architecture Diagram

```
┌─────────────────────────────────────────────────────────────────┐
│                     GitHub Pages (Static)                       │
│  ┌────────────────────────────────────────────────────────┐    │
│  │  Yew Frontend (WASM)                                   │    │
│  │  ├─ Canvas (Topology Visualization)                    │    │
│  │  ├─ CliTerminal (xterm.js)                             │    │
│  │  ├─ TimelineControl (Time Machine)                     │    │
│  │  ├─ SidecarPanel (Dual-Pane Interface)                 │    │
│  │  ├─ LessonBuilder (Instructor Tools)                   │    │
│  │  └─ bgpsim WASM (Core Simulation)                      │    │
│  └────────────────────────────────────────────────────────┘    │
│                           │                                     │
│                           │ WebSocket (gloo-net)                │
└───────────────────────────┼─────────────────────────────────────┘
                            │
                            ▼
┌─────────────────────────────────────────────────────────────────┐
│                  Backend API (Fly.io/Railway)                   │
│  ┌────────────────────────────────────────────────────────┐    │
│  │  Axum Server (Rust)                                    │    │
│  │  ├─ REST API (Auth, Lessons, Scenarios)               │    │
│  │  ├─ WebSocket Handler (tokio-tungstenite)             │    │
│  │  ├─ Validation Engine (Task Verification)             │    │
│  │  └─ Session Manager (User State)                      │    │
│  └────────────────────────────────────────────────────────┘    │
│                           │                                     │
│                           ▼                                     │
│  ┌────────────────────────────────────────────────────────┐    │
│  │  PostgreSQL Database                                   │    │
│  │  ├─ users (auth, roles)                                │    │
│  │  ├─ lessons (content, tasks)                           │    │
│  │  ├─ topologies (networks, configs)                     │    │
│  │  ├─ progress (student tracking)                        │    │
│  │  └─ scenarios (templates)                              │    │
│  └────────────────────────────────────────────────────────┘    │
└─────────────────────────────────────────────────────────────────┘
```

---

## Dependencies & Integration Points

### New Rust Dependencies

**Frontend (bgpsim-web/Cargo.toml)**:
```toml
[dependencies]
# Existing: yew, yewdux, gloo-net, web-sys, wasm-bindgen, serde_json

# New additions:
indexed_db_futures = "0.4"      # IndexedDB access
wasm-bindgen-rayon = "1.2"      # Web Workers for parallel simulation
chrono = { version = "0.4", features = ["wasmbind"] }  # Timestamps
uuid = { version = "1.0", features = ["wasm-bindgen", "v4"] }  # IDs
```

**Backend (new crate: bgpemu2-server/Cargo.toml)**:
```toml
[dependencies]
axum = "0.7"                    # Web framework
tokio = { version = "1", features = ["full"] }
tokio-tungstenite = "0.21"      # WebSocket server
tower-http = { version = "0.5", features = ["cors"] }  # CORS middleware
sqlx = { version = "0.7", features = ["postgres", "runtime-tokio", "uuid"] }
serde = { version = "1", features = ["derive"] }
serde_json = "1.0"
uuid = { version = "1.0", features = ["v4"] }
jsonwebtoken = "9.2"            # JWT auth
argon2 = "0.5"                  # Password hashing
tracing = "0.1"                 # Structured logging
tracing-subscriber = "0.3"
```

### JS Dependencies (package.json for xterm.js)

```json
{
  "dependencies": {
    "xterm": "^5.3.0",
    "xterm-addon-fit": "^0.8.0",
    "xterm-addon-web-links": "^0.9.0"
  }
}
```

---

## Open Questions & Future Research

1. **Real-Time Collaboration (v2)**: Evaluate CRDT libraries (Yrs, Automerge) for multi-student topology editing
2. **Mobile Support (v2)**: Research responsive canvas interactions, touch-optimized CLI
3. **LMS Integration (v2)**: SCORM 1.2 vs 2004, LTI 1.3 standards for Canvas/Moodle
4. **AI Assistance (Future)**: Feasibility of local LLM (WASM) for CLI help vs cloud API
5. **Video Recording (Future)**: MediaRecorder API for session replay with instructor voiceover

---

## Constitution Compliance Notes

- ✅ **Principle I (TDD)**: Research identifies testable components (CLI <1ms, convergence <2s)
- ✅ **Principle II (Library-First)**: Validation engine, time machine as standalone libs
- ✅ **Principle X (Performance)**: Optimization strategies target 100-router MVP, <100ms CLI
- ✅ **Principle XIII (Rust Discipline)**: All new code uses rustfmt, clippy::pedantic, no unsafe except WASM interop (justified)
- ✅ **Accessibility**: xterm.js screen reader support, WCAG 2.1 AA color palette

---

**Status**: ✅ All technical unknowns resolved. Ready for Phase 1 (Data Model & Contracts).
