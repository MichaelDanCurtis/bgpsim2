# Feature Specification: Educational BGP/OSPF Web Training Platform

**Feature Branch**: `001-educational-web-platform`  
**Created**: 2025-11-18  
**Status**: Draft  
**Input**: User description: "Educational BGP/OSPF web training platform with visual network designer, dual-pane router interface (sidecar + CLI), lesson builder, guided labs with progressive hints, anycast scenario library, and time machine replay for interactive learning"

## User Scenarios & Testing *(mandatory)*

### User Story 1 - Visual Network Design & Basic Simulation (Priority: P1)

A networking student accesses the web platform, drags routers and links onto a canvas to create a simple BGP topology (2 border routers, 2 external peers), configures basic BGP peering via a sidecar panel, starts the simulation, and observes route propagation in real-time.

**Why this priority**: Core MVP - enables basic topology creation and simulation without which no other features can function. Delivers immediate value by allowing students to visualize network behavior.

**Independent Test**: Can be fully tested by creating a 4-router topology, configuring BGP sessions via sidecar, starting simulation, and verifying routes appear in routing tables.

**Acceptance Scenarios**:

1. **Given** empty canvas, **When** student drags 4 router icons onto canvas and connects them with links, **Then** topology is created with proper graph structure
2. **Given** topology with 2 routers, **When** student opens sidecar panel and clicks "Add BGP Neighbor", **Then** BGP session configuration form appears with validation
3. **Given** configured topology, **When** student clicks "Start Simulation", **Then** BGP state machines initialize and routes propagate within 2 seconds
4. **Given** running simulation, **When** student hovers over a link, **Then** real-time traffic flow animation shows active BGP updates
5. **Given** converged topology, **When** student clicks a router, **Then** sidecar shows current BGP neighbor status, route count, and OSPF adjacencies

---

### User Story 2 - Interactive Per-Router CLI Access (Priority: P1)

A student working on a lab double-clicks a router icon to open an emulated CLI terminal, types `show ip bgp` to view the BGP routing table, enters configuration mode with `configure terminal`, adds a new BGP neighbor, and sees the change reflected immediately in both the CLI and the visual topology.

**Why this priority**: Critical for educational mission per Constitution Principle XI - students must interact with realistic router CLIs to learn production skills. Without CLI, this is just a visualization tool.

**Independent Test**: Can be tested by opening any router's CLI, executing standard show commands, making config changes, and verifying state synchronization between CLI and visual interface.

**Acceptance Scenarios**:

1. **Given** running simulation, **When** student double-clicks router R1, **Then** terminal window opens with vendor-neutral CLI prompt `R1>`
2. **Given** open CLI terminal, **When** student types `show ip bgp` and presses Enter, **Then** formatted BGP routing table displays within 100ms
3. **Given** CLI prompt, **When** student types `configure terminal`, **Then** prompt changes to `R1(config)#` and enters config mode
4. **Given** config mode, **When** student enters `router bgp 65001` then `neighbor 10.0.0.2 remote-as 65002`, **Then** BGP session configuration is applied and neighbor appears in topology sidecar
5. **Given** CLI session, **When** student presses Tab key mid-command, **Then** command auto-completes with available options
6. **Given** CLI error, **When** student enters invalid syntax, **Then** helpful error message displays with correction hint (e.g., "Unknown command. Did you mean 'show ip route'?")

---

### User Story 3 - Lesson Builder & Guided Lab Workflow (Priority: P2)

An instructor creates a new lesson titled "BGP Route Reflection Basics", defines learning objectives, uploads a pre-built topology, adds 5 progressive tasks with validation checkpoints, embeds hints for common mistakes, and publishes the lesson. A student then works through the lesson step-by-step, receiving guidance and automatic validation.

**Why this priority**: Transforms platform from sandbox to structured learning environment. Enables self-paced learning and instructor-led courses, addressing primary educational mission.

**Independent Test**: Can be tested by creating a complete lesson with multiple tasks, having a test student complete it, and verifying all checkpoints, hints, and validations work correctly.

**Acceptance Scenarios**:

1. **Given** instructor dashboard, **When** instructor clicks "Create New Lesson", **Then** lesson builder form opens with fields for title, objectives, difficulty, estimated time
2. **Given** lesson builder, **When** instructor uploads topology YAML file, **Then** topology preview renders showing all routers and links
3. **Given** lesson with topology, **When** instructor adds Task 1 "Configure IBGP Full Mesh" with validation rule "All IBGP sessions must be Established", **Then** task is saved with auto-verification criteria
4. **Given** published lesson, **When** student opens lesson and starts Task 1, **Then** task instructions display with locked subsequent tasks (Tasks 2-5 grayed out)
5. **Given** student working on task, **When** student clicks "Need a hint?" button, **Then** first hint displays (nudge-level), with option to reveal detailed hint
6. **Given** completed task, **When** student clicks "Validate", **Then** system checks BGP session states and shows green checkmark if criteria met, red X with diagnostic message if failed
7. **Given** validated Task 1, **When** system confirms success, **Then** Task 2 unlocks and student can proceed

---

### User Story 4 - Time Machine Simulation Control (Priority: P2)

During a complex BGP convergence scenario, a student pauses the simulation mid-convergence, uses a timeline scrubber to rewind 30 seconds, places a bookmark at "Pre-Failure State", steps forward through events one-by-one to observe BGP UPDATE messages, then fast-forwards to convergence and compares before/after routing tables side-by-side.

**Why this priority**: Enables deep understanding of protocol behavior by allowing students to examine state changes at their own pace. Critical for debugging skills and "aha!" moments when learning routing protocols.

**Independent Test**: Can be tested by running a failure scenario (link down), using all time controls (pause/play/step/rewind), verifying simulation state accurately reflects selected timestamp.

**Acceptance Scenarios**:

1. **Given** running simulation, **When** student clicks "Pause" button, **Then** simulation freezes and timeline scrubber appears with current timestamp
2. **Given** paused simulation, **When** student drags timeline scrubber backward 30 seconds, **Then** topology rewinds to that state (routing tables, BGP session states restore)
3. **Given** paused simulation, **When** student clicks "Step Forward" button, **Then** single next event executes (e.g., one BGP UPDATE processed) with visual highlight
4. **Given** timeline position, **When** student clicks "Bookmark" icon, **Then** bookmark named "Pre-Failure State" is saved and appears on timeline
5. **Given** bookmarked states, **When** student selects bookmark dropdown, **Then** can jump to any saved state instantly
6. **Given** two different timestamps, **When** student enables "Compare Mode", **Then** split-screen shows routing tables from both timestamps with diffs highlighted
7. **Given** completed scenario, **When** student clicks "Export Replay", **Then** JSON trace file downloads with all events and annotations

---

### User Story 5 - Anycast CDN Scenario Lab (Priority: P2)

A student opens the pre-built "Anycast CDN Distribution" scenario from the scenario library, which contains 3 edge POPs each advertising 192.0.2.0/24 via BGP, observes traffic steering based on AS_PATH length, triggers a POP failure, watches failover behavior, and answers embedded quiz questions about anycast catchment areas.

**Why this priority**: Directly addresses Constitution Principle VII requirement for first-class anycast support. Provides real-world applicable skills (CDN, DNS root servers, DDoS mitigation).

**Independent Test**: Can be tested by loading scenario, verifying all 3 POPs advertise same prefix correctly, simulating failure, confirming traffic reroutes, and completing quiz with auto-grading.

**Acceptance Scenarios**:

1. **Given** scenario library page, **When** student selects "Anycast CDN Distribution" scenario, **Then** pre-configured topology loads with 3 anycast POPs + 10 client locations
2. **Given** loaded anycast scenario, **When** simulation starts, **Then** all 3 POPs show identical prefix 192.0.2.0/24 in BGP table with different AS_PATHs
3. **Given** running anycast simulation, **When** student clicks "Show Catchment Areas", **Then** visual overlay shows which clients use which POP based on best AS_PATH
4. **Given** catchment visualization, **When** student disables POP-2 (link down), **Then** affected clients' catchment re-maps to POP-1 and POP-3 with animated transitions
5. **Given** anycast scenario, **When** student opens embedded quiz, **Then** questions appear about AS_PATH manipulation, failover timing, and traffic engineering
6. **Given** completed quiz, **When** student submits answers, **Then** auto-grading provides instant feedback with explanations for wrong answers

---

### User Story 6 - Scenario Template Library & Import (Priority: P3)

A student browses the built-in scenario library containing 20+ templates (hub-spoke VPN, MPLS L3VPN, BGP route reflection, OSPF multi-area), filters by difficulty level "Intermediate", previews the "Route Reflector Design" template, clones it to their workspace, modifies AS numbers, and saves as a custom scenario for later use.

**Why this priority**: Accelerates learning by providing starting points; reduces friction for beginners. Lower priority than core creation tools but enhances productivity.

**Independent Test**: Can be tested by browsing library, filtering results, cloning a template, modifying it, saving, and re-loading custom scenario.

**Acceptance Scenarios**:

1. **Given** main dashboard, **When** student clicks "Scenario Library", **Then** grid view shows template cards with thumbnails, difficulty badges, and popularity metrics
2. **Given** scenario library, **When** student sets filter to "Difficulty: Intermediate" and "Protocol: BGP", **Then** results narrow to matching templates only
3. **Given** template card, **When** student clicks "Preview", **Then** modal shows topology diagram, learning objectives, and estimated completion time
4. **Given** selected template, **When** student clicks "Clone to Workspace", **Then** editable copy appears in student's personal scenarios list
5. **Given** cloned scenario, **When** student modifies router R1 AS number from 65001 to 65100, **Then** change persists and affects BGP session validation
6. **Given** modified scenario, **When** student clicks "Save As" and names it "My Custom iBGP Lab", **Then** scenario saves and appears in "My Scenarios" section

---

### User Story 7 - Dual-Pane Sidecar Monitoring & Quick Config (Priority: P3)

While working on a lab, a student clicks router R2 to open the sidecar panel, views a real-time table showing all BGP neighbors with state (Established/Idle/Active), clicks a "Quick Config" button to apply a pre-defined route-map template, sees the config snippet preview before applying, and confirms the change is reflected in the router's running config accessible via CLI.

**Why this priority**: Enhances UX by providing high-level monitoring alongside detailed CLI access. Reduces cognitive load for common tasks while maintaining CLI access for deep work.

**Independent Test**: Can be tested by clicking any router, interacting with sidecar features (neighbor status, quick configs), and verifying sync with CLI state.

**Acceptance Scenarios**:

1. **Given** running simulation with router R2 selected, **When** sidecar panel opens, **Then** displays tabs: "BGP Neighbors", "OSPF Adjacencies", "Routes Summary", "Quick Configs"
2. **Given** "BGP Neighbors" tab, **When** tab is active, **Then** table shows neighbor IP, remote AS, state, uptime, prefixes received/sent with color-coded states (green=Established, red=Idle)
3. **Given** "Quick Configs" tab, **When** student clicks "Apply Route-Map: Prefer AS 65002", **Then** modal shows config snippet preview with diff highlighting
4. **Given** config preview modal, **When** student clicks "Apply", **Then** config is applied and sidecar updates to show new route-map active
5. **Given** applied quick config, **When** student opens CLI and types `show running-config`, **Then** route-map configuration appears in output matching preview
6. **Given** sidecar "Routes Summary", **When** displayed, **Then** shows prefix counts by protocol (BGP: 150, OSPF: 45, Static: 5) with expandable sections

---

### Edge Cases

- **What happens when student pauses simulation during mid-BGP-UPDATE processing?**  
  System completes current atomic event (UPDATE fully processed) before pausing, preventing inconsistent state.

- **How does CLI handle simultaneous config changes from sidecar and terminal?**  
  Last-write-wins with visual notification in both interfaces showing conflict timestamp and source.

- **What if student creates topology exceeding performance limits (>100 routers on MVP)?**  
  System displays warning modal: "This topology exceeds recommended router count (100). Performance may degrade. Continue?" with option to proceed or simplify.

- **How does time machine handle very long simulations (>1 hour)?**  
  Timeline uses logarithmic scale for long durations; events are sampled at key state transitions (FSM changes, route updates) to limit memory.

- **What happens when student enters invalid BGP configuration in CLI?**  
  System validates config against RFC requirements, shows error with specific clause violated, and suggests correction without crashing simulator (per Constitution crash recovery requirement).

- **How are anycast catchment areas calculated when AS_PATH lengths are equal?**  
  System uses BGP tiebreaker rules (router-ID) and visualizes with tooltip explaining tiebreaker reason.

- **What if student tries to run two lessons simultaneously?**  
  System allows multiple browser tabs but isolates simulation state per tab; optionally warns about potential confusion.

- **How does system handle browser refresh mid-lesson?**  
  Auto-saves checkpoint every 60 seconds; on reload, offers "Resume from checkpoint" or "Restart lesson" options.

## Requirements *(mandatory)*

### Functional Requirements

#### Visual Network Designer

- **FR-001**: System MUST provide drag-and-drop canvas for placing router, switch, and external peer icons
- **FR-002**: System MUST allow link creation by clicking source router and dragging to destination router
- **FR-003**: System MUST validate topology in real-time (e.g., OSPF area mismatches, disconnected subgraphs) with visual warnings
- **FR-004**: System MUST support topology import/export in YAML and JSON formats compatible with bgpsim library schemas
- **FR-005**: System MUST render topology with automatic layout (force-directed graph) and manual node repositioning
- **FR-006**: System MUST display link state visually (up/down via color: green=up, gray=down, orange=degraded)
- **FR-007**: System MUST show real-time BGP UPDATE propagation as animated packets flowing along links (optional toggle)

#### Dual-Pane Router Interface

- **FR-008**: System MUST provide sidecar panel with tabs: BGP Neighbors, OSPF Adjacencies, Routes Summary, Quick Configs
- **FR-009**: Sidecar MUST update in real-time (<100ms latency) reflecting current simulation state
- **FR-010**: System MUST allow opening per-router CLI terminal in modal or separate window via double-click or context menu
- **FR-011**: CLI terminal MUST support vendor-neutral command set: `show ip bgp`, `show ip bgp summary`, `show ip bgp neighbors`, `show ip route`, `show ip ospf neighbor`, `show interfaces`
- **FR-012**: CLI terminal MUST support configuration mode: `configure terminal`, `router bgp [AS]`, `neighbor [IP] remote-as [AS]`, `network [prefix] mask [netmask]`
- **FR-013**: CLI MUST provide tab completion for commands and context-sensitive help via `?` character
- **FR-014**: CLI MUST maintain command history (up/down arrows) per router session
- **FR-015**: Quick Configs in sidecar MUST show preview modal before applying configuration changes
- **FR-016**: System MUST synchronize state between sidecar and CLI (changes in either immediately reflect in both)

#### Lesson Builder

- **FR-017**: System MUST provide instructor interface for creating lessons with: title, description, learning objectives, difficulty level, estimated time
- **FR-018**: Lesson builder MUST allow uploading or selecting topology from library as lesson foundation
- **FR-019**: System MUST support adding ordered tasks with: task description, validation criteria (e.g., "All BGP sessions Established"), pass/fail logic
- **FR-020**: System MUST allow adding progressive hints per task: Level 1 (nudge), Level 2 (detailed), Level 3 (solution)
- **FR-021**: System MUST support embedding quiz questions (multiple choice, fill-in-blank) with auto-grading logic
- **FR-022**: System MUST lock subsequent tasks until prior tasks pass validation
- **FR-023**: System MUST provide task validation engine that checks topology state against defined criteria (e.g., BGP FSM states, route existence, OSPF neighbor counts)
- **FR-024**: System MUST display validation results with pass/fail status and diagnostic messages for failures
- **FR-025**: System MUST save lesson as versioned artifact (lesson_id, version, creation_timestamp) for reproducibility

#### Guided Labs & Student Workflow

- **FR-026**: System MUST display lesson list for students with filters: difficulty, protocol (BGP/OSPF), topic, completion status
- **FR-027**: System MUST load lesson with initial topology state and display Task 1 instructions
- **FR-028**: System MUST show task progress indicator (e.g., "Task 3 of 7 Complete")
- **FR-029**: System MUST provide "Need a hint?" button that progressively reveals hints (Level 1 → Level 2 → Level 3)
- **FR-030**: System MUST track hint usage per student per task for analytics
- **FR-031**: System MUST provide "Validate Task" button that runs validation criteria and shows results
- **FR-032**: System MUST unlock next task only after current task passes validation
- **FR-033**: System MUST allow students to retry failed tasks unlimited times
- **FR-034**: System MUST display quiz questions at designated points with immediate feedback on submission

#### Time Machine & Simulation Control

- **FR-035**: System MUST provide simulation controls: Play, Pause, Step Forward (single event), Step Backward, Speed (1x, 2x, 5x, 10x)
- **FR-036**: System MUST display timeline scrubber showing simulation time and key events (BGP state changes, link failures)
- **FR-037**: System MUST allow scrubbing timeline backward and forward with state restoration
- **FR-038**: System MUST support bookmarking simulation states with user-defined labels
- **FR-039**: System MUST provide "Compare Mode" showing side-by-side views of topology state at two different timestamps
- **FR-040**: System MUST highlight differences in Compare Mode (routing table diffs, neighbor state changes)
- **FR-041**: System MUST export simulation replay as JSON trace with: events, timestamps, state snapshots, user annotations
- **FR-042**: System MUST support importing and replaying exported traces with annotation overlays

#### Anycast Scenario Library

- **FR-043**: System MUST include pre-built anycast scenarios: CDN Distribution (3 POPs), DNS Root Servers (13 clusters), DDoS Mitigation (anycast scrubbing)
- **FR-044**: Anycast scenarios MUST configure multiple routers to advertise identical prefix with different AS_PATHs
- **FR-045**: System MUST visualize anycast catchment areas (which clients use which POP) based on BGP best path selection
- **FR-046**: System MUST update catchment visualization in real-time when anycast instance fails or AS_PATH changes
- **FR-047**: Anycast scenarios MUST include embedded learning content explaining: anycast mechanics, AS_PATH manipulation, failover behavior, catchment analysis
- **FR-048**: System MUST allow triggering anycast POP failures via UI (right-click POP → "Simulate Failure")
- **FR-049**: System MUST measure and display anycast failover metrics: convergence time, affected clients, new catchment distribution

#### Scenario Template Library

- **FR-050**: System MUST provide built-in library of 20+ topology templates: hub-spoke, full mesh, route reflector, OSPF multi-area, BGP anycast
- **FR-051**: System MUST display templates with: thumbnail preview, difficulty badge, protocol tags, popularity score
- **FR-052**: System MUST support filtering templates by: difficulty, protocol, topology type, keyword search
- **FR-053**: System MUST allow previewing template with: topology diagram, learning objectives, estimated time
- **FR-054**: System MUST support cloning templates to student workspace as editable copies
- **FR-055**: System MUST allow students to save custom scenarios to personal library with: name, description, tags
- **FR-056**: System MUST support exporting custom scenarios as YAML files for sharing

#### Authentication & User Management

- **FR-057**: System MUST support user roles: Student, Instructor, Admin
- **FR-058**: System MUST authenticate users via email/password (SSO/OAuth deferred to v2 for institutional integration)
- **FR-059**: System MUST maintain separate workspaces per user with isolated scenarios and lesson progress
- **FR-060**: System MUST track lesson completion status per student (not started, in progress, completed, score)

#### Performance & Scalability

- **FR-061**: System MUST render topology and update UI within 100ms for networks up to 100 routers (per Constitution X MVP)
- **FR-062**: System MUST warn users when topology exceeds 100 routers and may experience degradation
- **FR-063**: System MUST run BGP convergence simulation within 2 seconds for 100-router topologies (per Constitution X MVP)
- **FR-064**: System MUST support 100 concurrent users with <5% performance degradation (university course-scale deployment)

#### Accessibility & UX

- **FR-065**: System MUST support keyboard navigation for all interactive elements (topology, CLI, sidecar)
- **FR-066**: System MUST use color-blind safe palette (IBM Color Blind Safe or equivalent)
- **FR-067**: System MUST never rely on color alone to convey information (use symbols + color)
- **FR-068**: System MUST provide ARIA labels for screen reader compatibility
- **FR-069**: System MUST support light and dark theme modes

#### Data Persistence & Recovery

- **FR-070**: System MUST auto-save student progress every 60 seconds during active lesson
- **FR-071**: System MUST offer "Resume from checkpoint" option on browser refresh or crash
- **FR-072**: System MUST persist custom scenarios, lesson progress, and bookmarks to backend storage
- **FR-073**: System MUST export scenarios as self-contained YAML files with all topology and config data

### Key Entities

- **User**: Represents a student, instructor, or admin. Attributes: user_id, role (student/instructor/admin), email, name, created_at, workspace_id
- **Lesson**: Structured learning content. Attributes: lesson_id, title, description, objectives (list), difficulty (beginner/intermediate/advanced), estimated_time_minutes, topology_id, tasks (list of Task), quizzes (list of Quiz), created_by (instructor user_id), version, published_status
- **Task**: Individual step within a lesson. Attributes: task_id, lesson_id, order (1,2,3...), description, validation_criteria (JSON rules), hints (list: Level 1/2/3), locked (boolean), completion_status (per student)
- **Topology**: Network graph representation. Attributes: topology_id, name, routers (list of Router), links (list of Link), external_peers (list), config_snapshot (JSON), created_at, owner_user_id
- **Router**: Network device node. Attributes: router_id, name (e.g., "R1"), position_x, position_y, as_number, bgp_config (JSON), ospf_config (JSON), running_config (text), state (running/stopped)
- **Link**: Connection between routers. Attributes: link_id, source_router_id, dest_router_id, weight, state (up/down), latency_ms
- **SimulationState**: Point-in-time snapshot of simulation. Attributes: state_id, topology_id, timestamp, router_states (JSON: routing tables, BGP FSM states), event_log (list of events), bookmark_label (optional)
- **Scenario**: Reusable topology template. Attributes: scenario_id, name, description, difficulty, protocol_tags (BGP/OSPF/anycast), topology_id, is_public (boolean), popularity_score
- **Quiz**: Assessment questions. Attributes: quiz_id, lesson_id, questions (list of Question), auto_grade (boolean)
- **Question**: Single quiz item. Attributes: question_id, text, type (multiple_choice/fill_in_blank), options (list), correct_answer, explanation
- **Progress**: Student lesson tracking. Attributes: progress_id, user_id, lesson_id, current_task_id, completed_tasks (list), hints_used (count per task), quiz_scores (list), completion_timestamp
- **Bookmark**: Saved simulation state. Attributes: bookmark_id, user_id, topology_id, simulation_state_id, label, created_at

## Success Criteria *(mandatory)*

### Measurable Outcomes

- **SC-001**: Students can create a 4-router BGP topology, configure peering, and start simulation within 5 minutes (measured via user testing with 10+ students)
- **SC-002**: CLI commands (`show ip bgp`, `show ip route`) return results within 100ms for topologies up to 100 routers (benchmark per Constitution X)
- **SC-003**: 90% of students successfully complete first guided lab (BGP basics) on first attempt with ≤2 hints used (tracked via analytics)
- **SC-004**: Time machine rewind/fast-forward operations restore correct simulation state with <500ms latency (verified via automated integration tests)
- **SC-005**: Anycast failover scenarios demonstrate catchment rerouting within 2 seconds of POP failure (measured in scenario auto-tests)
- **SC-006**: System supports 100 concurrent users with <5% performance degradation (load tested with simulated users)
- **SC-007**: Instructor can create and publish a complete 5-task lesson in under 30 minutes (user study with 5 instructors)
- **SC-008**: Student lesson completion rate (finish all tasks) reaches 75% for beginner lessons (tracked over 3-month pilot)
- **SC-009**: CLI tab completion and command help reduce syntax errors by 60% compared to free-form entry (A/B test measurement)
- **SC-010**: 95% of students report platform is "easy to use" or "very easy to use" in post-lesson survey (Likert scale survey)
- **SC-011**: Accessibility audit (WCAG 2.1 AA) shows zero critical violations for screen reader and keyboard navigation (automated + manual audit)
- **SC-012**: Topology export/import maintains full fidelity (all config, state preserved) in 100% of round-trip tests (automated verification suite)

### Business/Educational Metrics

- **SC-013**: Reduce instructor time spent on lab setup by 70% compared to GNS3/EVE-NG manual setup (instructor time tracking study)
- **SC-014**: Increase student BGP concept retention by 40% compared to lecture-only format (pre/post knowledge assessment)
- **SC-015**: Enable self-paced learning with 80% of students completing labs without instructor intervention (support ticket tracking)
- **SC-016**: Platform adoption reaches 500 active student users within 6 months of launch (analytics tracking)

## Assumptions

- **Assumption 1**: Students have modern web browsers (Chrome 90+, Firefox 88+, Safari 14+) with WebAssembly support for running bgpsim WebAssembly module
- **Assumption 2**: Typical lesson duration is 30-90 minutes based on CCNP lab standards
- **Assumption 3**: Instructors are familiar with BGP/OSPF concepts and can write validation criteria in structured format (JSON rules)
- **Assumption 4**: Initial deployment targets university classroom/lab environment with dedicated server infrastructure
- **Assumption 5**: Students have basic networking knowledge (OSI model, IP addressing) before using platform
- **Assumption 6**: Vendor-neutral CLI commands are sufficient for 80% of learning objectives; vendor-specific modes (Cisco IOS) are enhancement
- **Assumption 7**: Real-time collaboration (multiple students on same topology) is deferred to v2; v1 supports single-user topology instances
- **Assumption 8**: Authentication via email/password is sufficient for MVP; institutional SSO is future enhancement
- **Assumption 9**: Mobile/tablet support is view-only for MVP; full editing requires desktop/laptop
- **Assumption 10**: Backend storage is persistent (SQL/NoSQL database); no offline-only mode required

## Out of Scope (Explicitly Excluded for v1)

- **Collaborative multi-student topologies** (students editing same topology simultaneously) - deferred to v2
- **Data-plane simulation** (actual packet forwarding, bandwidth, latency modeling) - control-plane only per Constitution focus
- **MPLS, SR, or advanced protocols** beyond BGP and OSPF - future expansion
- **Instructor analytics dashboard** with detailed student performance heatmaps - deferred to v2
- **Mobile app** (native iOS/Android) - web-only for v1
- **LMS integration** (SCORM, Canvas, Moodle) - deferred to v2
- **Video tutorials** embedded in lessons - instructor can link external videos
- **AI-powered troubleshooting assistant** - deferred pending AI/ML feasibility study
- **Federated login** (OAuth, SAML) - email/password only for v1
- **Internationalization (i18n)** - English-only interface for v1
- **GNS3/EVE-NG import** - YAML/JSON only for v1

## Dependencies

- **bgpsim library** (Rust core): Must have stable API for topology creation, BGP/OSPF simulation, state queries
- **bgpsim-web** (existing WebAssembly frontend): May serve as foundation for visual designer and CLI components
- **WebSocket server**: Required for real-time simulation state updates to frontend
- **Backend API**: RESTful or GraphQL API for user management, lesson CRUD, scenario storage
- **Database**: PostgreSQL or similar for user data, lessons, progress tracking, scenarios
- **Authentication service**: JWT-based auth or session management
- **WebAssembly toolchain**: Rust compiled to WASM for running bgpsim in browser
- **Frontend framework**: React, Vue, or similar for interactive UI (NEEDS CLARIFICATION: existing bgpsim-web uses specific framework?)

## Notes

- This specification defines features for v1 MVP of educational web platform
- Constitution Principle XIII (Rust Discipline) applies to all Rust/backend code
- Constitution Principle X (Performance Targets) defines 100-router MVP scale
- Anycast support is NON-NEGOTIABLE per Constitution Principle VII
- Interactive CLI per-router is CRITICAL per Constitution Principle XI
- All timing/performance requirements reference Constitution benchmarks
- Lesson validation engine must be library-first (Principle II) for reusability in CLI tools
- Time machine feature requires careful memory management to avoid browser OOM on long simulations
