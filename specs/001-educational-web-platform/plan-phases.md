## Phased Implementation

### Phase 0: Foundation (Weeks 1-2)

**Goal**: Set up development environment, create new crates, database schema

**Deliverables**:

1. **bgpemu2-server crate**
   - Cargo.toml with Axum, sqlx, tokio-tungstenite dependencies
   - main.rs with basic HTTP server
   - Database connection pool
   - `.env` configuration loading

2. **Database Schema**
   - Run all migrations from data-model.md
   - Seed scenarios table with 5 built-in templates
   - Create test fixtures for integration tests

3. **bgpemu2-core crate**
   - Validation engine skeleton (rules parser, engine trait)
   - Time machine manager (checkpoint struct, compression)
   - CLI parser foundation (command enum, parser trait)

4. **CI/CD Pipeline**
   - GitHub Actions for rustfmt + clippy
   - cargo test on every PR
   - criterion benchmarks baseline

**Acceptance Criteria**:

- ✅ `cargo build --all` succeeds
- ✅ Database migrations run without errors
- ✅ `cargo test --all` passes (even if minimal tests)
- ✅ CI pipeline runs on PR

---

### Phase 1: Core Backend API (Weeks 3-5)

**Goal**: Implement REST API endpoints, authentication, basic topology CRUD

**Deliverables**:

1. **Authentication** (FR-057-060)
   - POST /auth/register (email/password with Argon2)
   - POST /auth/login (JWT token generation)
   - GET /auth/me (current user profile)
   - Middleware for JWT validation

2. **Topology Management** (FR-001-007)
   - POST /topologies (create topology from JSON/YAML)
   - GET /topologies (list user topologies)
   - GET /topologies/:id (fetch single topology)
   - PUT /topologies/:id (update topology)
   - DELETE /topologies/:id
   - GET /topologies/:id/export (YAML export)

3. **Scenarios** (FR-050-056)
   - GET /scenarios (list public templates with filters)
   - POST /scenarios/:id/clone (clone to user workspace)
   - Seed 5 scenarios: hub-spoke, full mesh, route reflector, OSPF multi-area, anycast CDN

4. **Contract Tests**
   - Integration tests for all endpoints
   - Validate OpenAPI spec compliance
   - Test JWT auth flows

**Acceptance Criteria**:

- ✅ All 15 REST endpoints functional
- ✅ Integration tests achieve >80% coverage
- ✅ API responses match OpenAPI schema
- ✅ JWT authentication works end-to-end

---

### Phase 2: WebSocket & Real-Time Updates (Weeks 6-7)

**Goal**: Implement WebSocket protocol for simulation state streaming

**Deliverables**:

1. **WebSocket Server** (websocket-protocol.md)
   - GET /ws endpoint with token auth
   - Subscribe/Unsubscribe message handling
   - SimulationUpdate differential state push
   - EventNotification for discrete events
   - Ping/Pong heartbeat (30s interval)

2. **Simulation State Manager**
   - In-memory simulation runner (bgpsim integration)
   - Broadcast channel for multi-client updates
   - Per-topology session management

3. **Performance Optimization**
   - Differential state updates (only changes sent)
   - Message batching (max 10 updates/second)
   - Connection pooling (100 concurrent)

**Acceptance Criteria**:

- ✅ WebSocket connects with JWT query param
- ✅ Client receives SimulationUpdate within 50ms of state change (p99)
- ✅ 100 concurrent connections supported without degradation
- ✅ Heartbeat timeout closes stale connections

---

### Phase 3: Frontend Components (Weeks 8-11)

**Goal**: Build Yew components for visual designer, CLI terminal, sidecar panel

**Deliverables**:

1. **CliTerminal Component** (FR-010-014, User Story 2)
   - xterm.js integration via wasm-bindgen
   - Command history (up/down arrows)
   - Tab completion for common commands
   - Send commands to backend via WebSocket
   - Render CLI output with ANSI colors

2. **SidecarPanel Component** (FR-008-016, User Story 7)
   - Tabs: BGP Neighbors, OSPF Adjacencies, Routes Summary, Quick Configs
   - Real-time state updates from WebSocket
   - Quick config preview modal before applying
   - Sync with CLI (changes in either reflect immediately)

3. **Canvas Enhancements** (FR-001-007, User Story 1)
   - Extend existing canvas.rs
   - Add drag-drop for router placement
   - Link creation via click-drag
   - Real-time BGP UPDATE animation (optional toggle)
   - Link state coloring (green=up, gray=down, orange=degraded)

4. **API Client Service**
   - Extend api_client.rs with all 15 REST endpoints
   - Error handling with user-friendly messages
   - Loading states for async operations

**Acceptance Criteria**:

- ✅ Double-click router opens CLI terminal
- ✅ `show ip bgp` returns results in <100ms
- ✅ Sidecar updates within 100ms of simulation state change
- ✅ Canvas supports 100-router topology without lag

---

### Phase 4: Lesson Builder & Validation (Weeks 12-14)

**Goal**: Instructor tools for creating lessons with tasks and validation

**Deliverables**:

1. **Lesson Builder Component** (FR-017-025, User Story 3)
   - Create lesson form (title, objectives, difficulty, estimated time)
   - Upload topology or select from templates
   - Add ordered tasks with descriptions
   - Define validation criteria (JSON rules editor)
   - Add 3-level hints per task (Nudge, Detailed, Solution)
   - Embed quiz questions (multiple choice, fill-in-blank)
   - Publish lesson (draft → published state transition)

2. **Validation Engine** (bgpemu2-core/validation)
   - Implement ValidationRule enum:
     - BgpSessionState: Check BGP FSM state
     - RouteExists: Verify prefix in routing table
     - OspfNeighborCount: Count OSPF adjacencies
     - ConfigContains: Pattern match in running-config
   - Engine.validate() runs all rules against topology state
   - Return ValidationResult with pass/fail + diagnostics

3. **Backend Validation Endpoint** (FR-023-024)
   - POST /tasks/:id/validate
   - Accept topology state JSON
   - Run validation engine
   - Return diagnostics for failures

4. **Student Lesson View** (FR-026-034, User Story 3)
   - List lessons with filters (difficulty, protocol, status)
   - Load lesson with initial topology
   - Display Task 1 instructions, lock subsequent tasks
   - "Need a hint?" button with progressive reveal
   - "Validate Task" button triggers validation
   - Unlock next task on success

**Acceptance Criteria**:

- ✅ Instructor creates 5-task lesson in <30 minutes (SC-007)
- ✅ Student completes beginner lesson with 90% first-attempt rate (SC-003)
- ✅ Validation engine runs in <500ms for 100-router topology
- ✅ Hint usage tracked in progress table

---

### Phase 5: Time Machine & Anycast (Weeks 15-17)

**Goal**: Time travel simulation controls and anycast scenario

**Deliverables**:

1. **TimelineControl Component** (FR-035-042, User Story 4)
   - Play/Pause/Step Forward/Step Backward buttons
   - Timeline scrubber with key event markers
   - Speed selector (1x, 2x, 5x, 10x)
   - Bookmark creation with custom labels
   - Compare Mode: side-by-side state views with diffs
   - Export replay as JSON trace

2. **Time Machine Manager** (bgpemu2-core/time_machine)
   - Hybrid checkpoint strategy:
     - Full checkpoint every 10 seconds (miniz_oxide compressed)
     - In-memory event log between checkpoints
     - IndexedDB storage for old checkpoints
   - Seek algorithm: find nearest checkpoint, replay events forward
   - Memory limit: keep 5 recent checkpoints in RAM, older in IndexedDB

3. **Anycast Scenario** (FR-043-049, User Story 5)
   - Create "Anycast CDN Distribution" template:
     - 3 POPs (New York, London, Tokyo) advertising 192.0.2.0/24
     - 10 client locations with AS_PATH-based best path selection
   - Catchment visualization: color-coded map showing which clients use which POP
   - Failover simulation: right-click POP → "Simulate Failure"
   - Catchment re-mapping animation (real-time)
   - Embedded quiz: AS_PATH manipulation, failover timing, catchment analysis

**Acceptance Criteria**:

- ✅ Seek to arbitrary timestamp completes in <500ms (FR-036)
- ✅ Anycast scenario shows 3 POPs advertising same prefix (FR-044)
- ✅ Catchment visualization updates in real-time on failure (FR-046)
- ✅ Memory usage <500MB for 1-hour simulation

---

### Phase 6: Polish & Performance (Weeks 18-20)

**Goal**: Accessibility, performance tuning, deployment automation

**Deliverables**:

1. **Accessibility** (FR-065-069)
   - Keyboard navigation for all interactive elements
   - ARIA labels for screen readers
   - Color-blind safe palette (IBM Color Blind Safe)
   - Light/dark theme toggle
   - WCAG 2.1 AA audit (automated + manual)

2. **Performance Optimization**
   - WASM optimization: `opt-level = 3`, `lto = "thin"`
   - Lazy loading for large topologies (viewport culling)
   - Web Workers for BGP convergence (wasm-bindgen-rayon)
   - Benchmark: 100-router convergence <2s, CLI <100ms

3. **Deployment Automation**
   - GitHub Actions: Build WASM + deploy to GitHub Pages
   - Fly.io: Deploy backend + managed PostgreSQL
   - CORS configuration for production domains
   - Health check endpoints (/health, /ready)

4. **Documentation**
   - Update quickstart.md with final deployment URLs
   - API documentation (openapi-generator-cli)
   - Instructor guide: creating lessons
   - Student guide: using platform

**Acceptance Criteria**:

- ✅ WCAG 2.1 AA audit shows zero critical violations (SC-011)
- ✅ 100-router simulation runs in <2s (Constitution X)
- ✅ GitHub Pages deployment succeeds in <5 minutes
- ✅ 100 concurrent users supported (FR-064)

---

## Milestones

| Milestone | Target Date | Deliverables | Success Criteria |
|-----------|-------------|--------------|------------------|
| **M0: Foundation** | Week 2 | Server skeleton, DB schema, CI | All tests pass, migrations run |
| **M1: Backend API** | Week 5 | REST endpoints, auth, topology CRUD | OpenAPI compliance, >80% test coverage |
| **M2: WebSocket** | Week 7 | Real-time updates, simulation streaming | <50ms latency, 100 concurrent users |
| **M3: Frontend Core** | Week 11 | CLI terminal, sidecar, canvas | <100ms CLI response, real-time sync |
| **M4: Lessons** | Week 14 | Lesson builder, validation engine, student view | Instructor creates lesson in <30min |
| **M5: Time Machine** | Week 17 | Timeline controls, anycast scenario | <500ms seek, catchment visualization |
| **M6: Production** | Week 20 | Deployment, polish, documentation | GitHub Pages live, 100 users supported |

---

## Risks & Mitigation

### Technical Risks

1. **WASM Performance (High Impact, Medium Probability)**
   - **Risk**: 100-router BGP convergence exceeds 2s target in browser
   - **Mitigation**: 
     - Profile early with criterion benchmarks (Week 3)
     - Use Web Workers for parallel simulation (research.md Section 6)
     - Implement incremental updates (only compute changed routes)
   - **Fallback**: Run heavy simulations server-side, stream results to client

2. **WebSocket Scale (Medium Impact, Low Probability)**
   - **Risk**: 100 concurrent connections degrade performance
   - **Mitigation**:
     - Load test with k6 or Artillery (Week 7)
     - Optimize broadcast channel (avoid per-message cloning)
     - Use connection pooling, rate limiting (100 msg/sec)
   - **Fallback**: Deploy multiple backend instances with load balancer

3. **xterm.js Integration (Medium Impact, Medium Probability)**
   - **Risk**: wasm-bindgen JS interop causes memory leaks or crashes
   - **Mitigation**:
     - Contract tests for all JS interop calls (Week 8)
     - Properly drop JS handles in Rust Drop impl
     - Test with Valgrind/LeakSanitizer equivalent
   - **Fallback**: Use simpler HTML-based terminal (lose ANSI colors, features)

4. **Time Machine Memory (High Impact, Medium Probability)**
   - **Risk**: Long simulations (>1 hour) exceed 500MB browser memory limit
   - **Mitigation**:
     - Checkpoint compression with miniz_oxide (research.md Section 4)
     - Offload old checkpoints to IndexedDB
     - Implement checkpoint garbage collection (keep last 100)
   - **Fallback**: Limit simulation duration to 30 minutes for MVP

### Non-Technical Risks

5. **GitHub Pages Limitation (Low Impact, High Probability)**
   - **Risk**: Static hosting insufficient for dynamic features
   - **Mitigation**:
     - Already mitigated by hybrid architecture (research.md Section 3)
     - Backend API on separate domain (Fly.io)
   - **Fallback**: Migrate frontend to Vercel/Netlify if needed (still free tier)

6. **Instructor Adoption (High Impact, Medium Probability)**
   - **Risk**: Lesson builder too complex, instructors don't create content
   - **Mitigation**:
     - User testing with 5 instructors during Phase 4 (Week 12)
     - Iterate based on feedback (target <30min lesson creation)
     - Provide lesson templates with pre-filled validation rules
   - **Fallback**: Create 20+ built-in lessons to seed platform

7. **Student Engagement (Medium Impact, Low Probability)**
   - **Risk**: 75% lesson completion rate not achieved
   - **Mitigation**:
     - Progressive hints reduce frustration (SC-003: 90% first-attempt)
     - Gamification: badges for completed lessons
     - Track analytics (hint usage, time-on-task) to identify friction
   - **Fallback**: Add "Ask Instructor" chat feature (deferred to v2)

---

## Dependencies

### External Dependencies

- **bgpsim library**: Requires stable API for topology queries, state snapshots
  - **Risk**: Upstream breaking changes
  - **Mitigation**: Pin to bgpsim 0.20.0, test before upgrading

- **Yew framework**: Upgrading 0.20 → 0.21+ has breaking changes
  - **Risk**: Hook API changes require refactor
  - **Mitigation**: Create compatibility layer for existing components

- **PostgreSQL**: Requires 15+ for advanced JSONB features
  - **Risk**: Fly.io may default to older version
  - **Mitigation**: Specify `postgres:15-alpine` in fly.toml

### Internal Dependencies

- **Phase 3 blocks Phase 4**: Lesson builder needs working CLI terminal
- **Phase 2 blocks Phase 3**: Frontend needs WebSocket before real-time updates
- **Phase 4 blocks Phase 6**: Performance testing needs complete feature set

**Critical Path**: Phase 0 → Phase 2 → Phase 3 → Phase 4 → Phase 6

---

## Open Questions (For Discussion)

1. **MSRV Policy**: Constitution XIII says "stable-2", but upstream bgpsim may require newer. How to handle?
   - **Proposed**: Test MSRV in CI, document exceptions in Cargo.toml comments

2. **CLI Command Grammar**: Vendor-neutral vs Cisco IOS syntax. Which to implement first?
   - **Proposed**: Vendor-neutral for MVP (User Story 2), Cisco mode in v2

3. **Lesson Sharing**: Should students share custom scenarios publicly?
   - **Proposed**: Defer to v2 (privacy concerns, moderation needed)

4. **Data Retention**: How long to keep student progress data?
   - **Proposed**: 2 years, then archive/anonymize (GDPR compliance)

---

## Success Metrics (Post-Launch)

*To be measured after 3-month pilot (align with SC-001 through SC-016)*

- **Adoption**: 500 active student users (SC-016)
- **Engagement**: 75% lesson completion rate for beginner lessons (SC-008)
- **Performance**: <2s convergence for 100-router topologies (Constitution X)
- **Usability**: 95% "easy to use" rating in post-lesson survey (SC-010)
- **Instructor Efficiency**: 70% reduction in lab setup time vs GNS3 (SC-013)
- **Learning Outcomes**: 40% increase in BGP concept retention (SC-014)

---

## Appendix A: Technology Decisions Summary

**From research.md**:

- **Frontend**: Yew 0.21+ (existing codebase continuity)
- **Backend**: Axum (ergonomic, first-class WebSocket)
- **Database**: PostgreSQL (JSONB for flexible schemas)
- **WebSocket**: gloo-net (client) + tokio-tungstenite (server)
- **Terminal**: xterm.js (industry standard, accessibility)
- **Time Machine**: Hybrid checkpoints (IndexedDB + in-memory)
- **Deployment**: GitHub Pages (frontend) + Fly.io (backend)

**From Constitution**:

- **TDD**: Red-Green-Refactor for all features (Principle I)
- **Library-First**: Validation, time machine, CLI parser as standalone libs (Principle II)
- **Performance**: 100 routers MVP, <2s convergence, <100ms CLI (Principle X)
- **Anycast**: First-class feature with CDN, DNS, DDoS scenarios (Principle VII NON-NEGOTIABLE)

---

## Appendix B: Related Documents

- **Feature Spec**: [spec.md](spec.md) - 73 functional requirements, 7 user stories, 16 success criteria
- **Research**: [research.md](research.md) - Technology decisions, architecture, performance optimizations
- **Data Model**: [data-model.md](data-model.md) - 12 entities, PostgreSQL schemas, validation rules
- **API Contracts**: [contracts/api-openapi.yaml](contracts/api-openapi.yaml) - OpenAPI 3.0 spec
- **WebSocket Protocol**: [contracts/websocket-protocol.md](contracts/websocket-protocol.md) - Message types, examples
- **Quickstart**: [quickstart.md](quickstart.md) - Local dev setup, first scenario walkthrough
- **Constitution**: [.specify/memory/constitution.md](../../.specify/memory/constitution.md) - Project governance

---

**Plan Status**: ✅ **COMPLETE** - Ready for task breakdown via `/speckit.tasks` command.

**Next Steps**:

1. User approval of plan structure and phasing
2. Run `/speckit.tasks` to generate granular task list
3. Begin Phase 0 implementation (Week 1)
