# Implementation Plan: Educational BGP/OSPF Web Training Platform

**Branch**: `001-educational-web-platform` | **Date**: 2025-11-18 | **Spec**: [spec.md](spec.md)
**Input**: Feature specification from `/specs/001-educational-web-platform/spec.md`

## Summary

Build an educational web platform for BGP/OSPF training with visual network designer, interactive per-router CLI terminals, lesson builder, time machine replay, and anycast scenario library. Frontend uses Yew (Rust WASM) deployed to GitHub Pages, backend is Axum + PostgreSQL deployed to Fly.io/Railway, with WebSocket for real-time simulation updates.

**Technical Approach**: Extend existing bgpsim-web (Yew 0.20) with new components (CliTerminal via xterm.js, LessonBuilder, TimelineControl, SidecarPanel). Create new bgpemu2-server crate (Axum) with REST API + WebSocket. Use hybrid checkpoint strategy (IndexedDB + in-memory) for time machine feature. All 73 functional requirements addressed across 7 user stories (P1: Visual Designer & CLI Access, P2: Lesson Builder & Time Machine & Anycast, P3: Scenario Library & Sidecar).

## Technical Context

**Language/Version**: Rust 1.70+ (MSRV: stable-2 per Constitution XIII)

**Primary Dependencies**:

- Frontend: Yew 0.21+, Yewdux, gloo-net (WebSocket), xterm.js (JS interop), indexed_db_futures
- Backend: Axum 0.7, tokio-tungstenite (WebSocket), sqlx 0.7 (PostgreSQL), jsonwebtoken (JWT), argon2 (password hashing)
- Core: bgpsim 0.20.0 (existing library)

**Storage**: PostgreSQL 15+ (backend), IndexedDB (client-side time machine checkpoints)

**Testing**: cargo test + criterion benchmarks (Constitution X: <2s convergence for 100 routers, <100ms CLI response)

**Target Platform**:

- Frontend: WebAssembly (Chrome 90+, Firefox 88+, Safari 14+)
- Backend: Linux x86_64 (Fly.io/Railway)
- Deployment: GitHub Pages (static frontend) + Fly.io (dynamic backend)

**Project Type**: Web application (frontend + backend split)

**Performance Goals**:

- MVP: 100 routers, 10k prefixes, <2s BGP convergence (Constitution X)
- Target: 1k routers, 100k prefixes, <5s convergence
- CLI response: <100ms (Constitution Additional Constraints)
- WebSocket latency: <50ms p99
- Concurrent users: 100 with <5% degradation (FR-064)

**Constraints**:

- GitHub Pages: Static files only (no server-side execution)
- WASM size: <10MB compressed for fast load
- Browser memory: <500MB for time machine (1-hour simulation)
- Real-time updates: 10 updates/second during active simulation
- Offline support: Read-only cached topologies via IndexedDB

**Scale/Scope**:

- Users: 500 active students within 6 months (SC-016)
- Lessons: 20+ built-in scenarios (FR-050)
- Topologies: Up to 100 routers per MVP, 1k routers target
- Concurrent sessions: 100 users (FR-064)

## Constitution Check

*GATE: Must pass before Phase 0 research. Re-check after Phase 1 design.*

### Pre-Phase 0 Check ✅ PASS

- ✅ **Principle I (TDD)**: All 73 FRs testable via acceptance scenarios, validation engine has contract tests
- ✅ **Principle II (Library-First)**: Validation engine, time machine, CLI parser extracted as standalone libs
- ✅ **Principle III (CLI-First)**: Validation CLI tool planned for instructor testing without UI
- ✅ **Principle VII (Anycast NON-NEGOTIABLE)**: User Story 5 implements anycast CDN scenario (FR-043-049)
- ✅ **Principle X (Performance Targets)**: 100-router MVP <2s convergence, <100ms CLI (research.md addresses optimizations)
- ✅ **Principle XI (Educational UX CRITICAL)**: User Story 2 implements per-router CLI (xterm.js integration)
- ✅ **Principle XIII (Rust Discipline)**: rustfmt, clippy::pedantic in CI; no unsafe except WASM interop (justified)

### Post-Phase 1 Check ✅ PASS

- ✅ **Data Model**: 12 entities with PostgreSQL schemas, validation rules, state transitions documented
- ✅ **API Contracts**: OpenAPI 3.0 spec with 15 endpoints, WebSocket protocol with 12 message types
- ✅ **Testability**: All validation rules expressed as JSON criteria, automatable via validation engine
- ✅ **Performance**: Checkpoint strategy (<500ms seek), differential state updates (research.md Section 4)
- ✅ **Accessibility**: xterm.js screen reader support, WCAG 2.1 AA color palette (FR-065-069)

### Gates

No blocking violations. Constitution compliance verified for:

1. **Test-First**: Spec has 7 independent test scenarios per user story
2. **Modularity**: 3 standalone libraries planned (validation, time machine, CLI parser)
3. **Anycast**: First-class feature with 3 POPs, catchment visualization, failover
4. **Performance**: Benchmarks for MVP (100 routers) and target (1k routers) defined
5. **Educational UX**: Interactive CLI per router (distinct from orchestration CLI)

## Project Structure

### Documentation (this feature)

```text
specs/001-educational-web-platform/
├── plan.md              # This file ✅ Complete
├── research.md          # Phase 0 output ✅ Complete
├── data-model.md        # Phase 1 output ✅ Complete
├── quickstart.md        # Phase 1 output ✅ Complete
├── contracts/           # Phase 1 output ✅ Complete
│   ├── api-openapi.yaml         # REST API specification
│   └── websocket-protocol.md    # WebSocket message protocol
├── checklists/
│   └── requirements.md  # Quality validation ✅ Complete
└── tasks.md             # Phase 2 output (created by /speckit.tasks command - NOT YET)
```

### Source Code (repository root)

**Structure Decision**: Web application architecture (frontend + backend) selected based on:

- GitHub Pages constraint (static hosting only)
- Need for dynamic backend (auth, database, WebSocket server)
- Existing bgpsim-web crate provides foundation for frontend

```text
bgpsim/                       # Existing library (core simulation)
├── src/
│   ├── bgp/                  # BGP state machine
│   ├── ospf/                 # OSPF computation
│   ├── router/               # Router abstraction
│   └── types/                # Shared types
└── tests/

bgpsim-web/                   # Frontend (Yew WASM) - EXTEND
├── src/
│   ├── main.rs               # Entry point (existing)
│   ├── components/
│   │   ├── canvas.rs         # Topology visualization (existing)
│   │   ├── cli_terminal.rs   # NEW: xterm.js wrapper
│   │   ├── lesson_builder.rs # NEW: Instructor lesson creator
│   │   ├── timeline_control.rs # NEW: Time machine UI
│   │   ├── sidecar_panel.rs  # NEW: Dual-pane interface
│   │   └── scenario_library.rs # NEW: Template browser
│   ├── services/
│   │   ├── api_client.rs     # REST API calls (existing, extend)
│   │   ├── ws_client.rs      # NEW: WebSocket handler
│   │   └── time_machine.rs   # NEW: Checkpoint manager
│   ├── state/
│   │   └── app_state.rs      # Yewdux state (existing, extend)
│   └── models/               # NEW: Entity models (from data-model.md)
├── js/
│   └── terminal.js           # NEW: xterm.js bootstrap
├── package.json              # NEW: xterm.js dependency
├── index.html
└── Trunk.toml

bgpemu2-server/               # NEW: Backend (Axum)
├── src/
│   ├── main.rs               # Server entry point
│   ├── models/               # Entity structs (from data-model.md)
│   │   ├── user.rs
│   │   ├── lesson.rs
│   │   ├── topology.rs
│   │   └── ...
│   ├── api/
│   │   ├── auth.rs           # /auth endpoints
│   │   ├── lessons.rs        # /lessons endpoints
│   │   ├── topologies.rs     # /topologies endpoints
│   │   ├── scenarios.rs      # /scenarios endpoints
│   │   └── progress.rs       # /progress endpoints
│   ├── services/
│   │   ├── validation.rs     # Task validation engine
│   │   └── jwt.rs            # JWT auth logic
│   ├── ws/
│   │   ├── handler.rs        # WebSocket upgrade
│   │   ├── messages.rs       # Protocol message types
│   │   └── session.rs        # Per-connection state
│   └── db/
│       └── migrations/       # sqlx migrations
└── tests/
    ├── integration/
    └── contract/

bgpemu2-core/                 # NEW: Shared library
├── src/
│   ├── validation/           # Validation engine (library-first)
│   │   ├── engine.rs
│   │   ├── rules.rs
│   │   └── cli.rs            # CLI tool
│   ├── time_machine/         # Checkpoint manager (library-first)
│   │   ├── manager.rs
│   │   ├── checkpoint.rs
│   │   └── compression.rs
│   └── cli_parser/           # Router CLI parser (library-first)
│       ├── parser.rs
│       └── commands.rs
└── tests/

.github/
└── workflows/
    ├── deploy-frontend.yml   # NEW: Deploy to GitHub Pages
    └── deploy-backend.yml    # NEW: Deploy to Fly.io

migrations/                   # NEW: Database migrations
├── 20251118_001_initial_schema.sql
├── 20251118_002_seed_scenarios.sql
└── ...
```

## Complexity Tracking

> **Fill ONLY if Constitution Check has violations that must be justified**

No violations requiring justification. All complexity aligns with Constitution principles:

- **3 Crates**: bgpsim (existing), bgpsim-web (extend), bgpemu2-server (new), bgpemu2-core (new libs)
  - Justified: Frontend/backend separation required by deployment constraints (GitHub Pages static-only)
  - Library-first principle satisfied by bgpemu2-core extraction
- **xterm.js JS Interop**: Requires `unsafe` wasm-bindgen bindings
  - Justified: No mature Rust-native browser terminal; xterm.js industry standard (VS Code, Hyper)
  - Safety: wasm-bindgen generates safe bindings; interop tested via contract tests
