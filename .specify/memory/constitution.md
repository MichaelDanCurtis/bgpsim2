<!--
Sync Impact Report
- Version change: 1.2.0 → 1.3.0
- Modified principles:
  - V. Observability, Versioning, Simplicity (reorganized into sub-bullets)
  - X. Performance & Scale Targets (added concrete metrics and PERFORMANCE.md requirement)
  - XI. Educational UX & Realism (clarified click-to-open router CLI vs. orchestration CLI)
- Added principles:
  - XIII. Rust Discipline (formatting, linting, unsafe code, MSRV, dependency auditing)
- Added sections: None
- Removed sections: None
- Templates requiring updates:
  - ⚠ .specify/templates/plan-template.md (needs review for Rust discipline)
  - ⚠ .specify/templates/spec-template.md (needs review for Rust discipline)
  - ⚠ .specify/templates/tasks-template.md (needs review for performance tracking)
- Follow-up TODOs:
  - Create PERFORMANCE.md baseline document
  - Set up cargo-audit and cargo-deny in CI
  - Configure clippy::pedantic in CI pipeline
  - Document MSRV in root Cargo.toml
  - Create scenario template for educational labs
  - Define CLI command grammar and vendor modes
  - Spec anycast implementation phases
-->

# BGPemu2 Constitution

## Core Principles

### I. Test-First and Contracts (NON-NEGOTIABLE)

TDD is mandatory. Write contract and acceptance tests before implementation; run
Red-Green-Refactor cycles for every change. Public interfaces MUST be specified
via testable contracts (CLI or library) and changes require test updates first.

### II. Library-First Modularity

Each feature starts as a standalone, self-contained library: independently
testable, documented, with a clear purpose. Avoid organization-only libraries;
prefer reuse and small, composable units.

### III. CLI-First, Text I/O

All libraries expose functionality via a CLI. Protocol: stdin/args → stdout;
errors → stderr. Support JSON and human-readable outputs where applicable to
enable automation and easy debugging.

### IV. Integration + Contract Testing

Integration tests are required for: new library contracts, contract changes,
cross-library communication, and shared schemas. Contract tests MUST remain
backward compatible unless a major version is planned with migration notes.

### V. Observability, Versioning, Simplicity

- **Observability:** Use structured logging with trace IDs; ensure
  deterministic, reproducible runs (seedable randomness, fixed inputs).
- **Versioning:** Follow SemVer strictly; breaking changes require a
  migration plan and MAJOR version bump.
- **Simplicity:** Start simple (YAGNI); remove accidental complexity;
  justify essential complexity in design docs.

### VI. RFC Compliance & Fidelity

Protocol behavior MUST align with relevant RFCs and drafts under test.
Where deliberate deviations are needed (for research), they MUST be explicit,
toggleable, and covered by tests and documentation.

### VII. Extensible Plugin Architecture

Core provides scheduler, topology, eventing, tracing, and I/O. BGP engines
(e.g., FRR, BIRD, GoBGP) and data-plane simulators integrate via stable plugin
interfaces with contract tests per adapter.

**Anycast Support (NON-NEGOTIABLE):** BGP anycast MUST be a first-class feature:
multiple routers advertise identical prefixes from different locations with
proper AS_PATH handling, traffic steering, and failover simulation. Educational
scenarios include CDN distribution, DNS root servers, and DDoS mitigation.

### VIII. Safety & Isolation by Default

Emulations MUST run in isolated environments (network namespaces, containers or
VMs). No external advertisements or route leaks. Resource limits must have safe
defaults and be configurable.

### IX. Reproducible Scenarios & Seeds

All scenarios are defined declaratively (DSL/config) and MUST be reproducible
using a single seed and versioned assets. Artifacts include topology, policies,
timing, and randomization controls.

### X. Performance & Scale Targets

Maintain a `PERFORMANCE.md` file documenting:

- **Minimum viable scale (MVP):** 100 routers, 10k prefixes, <2s convergence
- **Target scale:** 1k routers, 100k prefixes, <5s convergence
- **Regression policy:** >10% slowdown in benchmarks blocks merge
- **CI enforcement:** Run criterion benchmarks on every PR to main; compare
  against baseline

Benchmarks MUST cover: network generation, BGP convergence, OSPF computation,
forwarding state extraction, and CLI response time.

### XI. Educational UX & Realism

The emulator MUST provide a student-friendly experience that mirrors real
network operations: interactive CLI per router (distinct from the orchestration
CLI in Principle III—students click a router to open its shell) supporting
vendor-neutral commands (show ip bgp, show ip route, configure terminal);
optional vendor modes (Cisco IOS, FRR, Juniper-style) for authenticity;
real-time event visualization and step-through debugging; scenario library with
tutorials and learning objectives; mistake recovery that allows students to
break things safely and learn from errors.

### XII. Hybrid Simulation Modes

Support multiple operational modes to balance education and research needs:
event-driven mode (fast, deterministic, no wall-clock time) for upstream
compatibility; real-time mode that mirrors actual convergence timing for
SLA/performance teaching; step-through mode that pauses between events to
visualize state changes for pedagogy; replay mode to record and replay
scenarios with annotations.

### XIII. Rust Discipline

- **Formatting & Linting:** All code MUST pass `cargo fmt --check` and
  `cargo clippy -- -D warnings` in CI. Use clippy::pedantic for new modules;
  allow-list specific rules only with inline justification comments.
- **Unsafe Code:** Unsafe blocks require: (1) Safety comment explaining
  invariants, (2) PR review justification, (3) Test coverage of safety
  conditions. Prefer safe abstractions; unsafe for performance needs proof
  via benchmarks.
- **MSRV Policy:** Minimum Supported Rust Version is stable minus 2 releases.
  Test MSRV in CI. Document in Cargo.toml `rust-version` field.
- **Dependency Auditing:** Run `cargo-audit` and `cargo-deny` in CI. No
  dependencies with known vulnerabilities or incompatible licenses. Pin
  transitive deps in `Cargo.lock` for reproducibility.

## Real-World Fidelity

- **CLI Commands:** Support standard show/debug commands students encounter in
  production: `show ip bgp`, `show ip bgp summary`, `show ip bgp neighbors`,
  `show ip route`, `show ip ospf neighbor`, `show interfaces`, `configure
  terminal` mode with BGP/OSPF/static route configuration.
- **Timing Model:** Optional real-time mode (vs. event-driven) for
  latency/convergence teaching.
- **Packet Capture:** Export to PCAP with realistic BGP messages for Wireshark
  analysis.
- **Multi-Vendor:** Core commands vendor-neutral; optional Cisco/Juniper/FRR
  syntax modes.
- **State Persistence:** Save/restore network state; student checkpoints for
  long scenarios.

## Additional Constraints

- Determinism: Builds and runs MUST be reproducible across machines.
- Security: No secrets in source; use env or secret stores; enable
  dependency scanning in CI.
- Licensing: All dependencies MUST have compatible licenses and be tracked.
- Formatting/Linting: Enforce consistent formatting and linters in CI.
- Data: Default to immutable inputs/outputs; avoid hidden state.
- Interactive Performance: CLI response MUST feel instant (<100ms) for typical
  show commands on networks up to 100 routers.
- Crash Recovery: Student mistakes (invalid BGP config, routing loops) MUST
  NOT crash the simulator; provide helpful error messages.
- Accessibility: Support screen readers; ensure color-blind friendly
  visualizations.

## Project Direction & Scope

- **Primary Mission:** Educational BGP/OSPF emulator for university courses,
  certification prep (CCNP/CCIE), and network engineering training. Students
  learn by interacting with realistic router CLIs, not by writing code.
- Upstream Fork: Project originates as a fork of `bgpsim` and will maintain a
  periodic sync with upstream while contributing back improvements when
  applicable and license-compatible.
- Scope Expansion:
  - **Interactive CLI (CRITICAL):** Per-router terminal with vendor-neutral
    and vendor-specific command modes.
  - **Anycast BGP (CRITICAL):** First-class support for anycast scenarios
    (CDN, DNS root servers, DDoS mitigation).
  - Beyond control-plane simulation, add adapters for different BGP engines
    and optional data-plane emulation. Provide a scenario DSL, trace
    capture/export (pcap/json), and policy testing harnesses.
- Use-Cases (Priority Order):
  - **Education (PRIMARY):** Classroom labs, self-paced learning,
    certification prep, hands-on training.
  - Research: RFC conformance testing, protocol experimentation.
  - Professional: Pre-deployment testing, failure drills, performance
    benchmarking.

## Development Workflow

- Branches: `[###-feature-name]` convention; small, focused PRs.
- Reviews: Every PR MUST include Constitution Check notes and updated tests.
- CI Gates: Lint, unit, contract, and integration tests MUST pass.
- Releases: Tag with SemVer; changelog includes migration notes for breaking
  changes.
- Documentation: Update specs/plan/tasks alongside code changes.
- Upstream Sync: Track upstream changes; run contract and regression suites
  before merging upstream updates.
- Benchmarks: Maintain performance baselines; regressions block release.
- Scenario Testing: All educational scenarios MUST have learning objectives
  documented, expected student actions and outcomes, automated verification
  scripts, and instructor solution guides.

## Governance

- Authority: This constitution governs engineering practices for BGPemu2 and
  supersedes conflicting conventions.
- Amendments: Propose a PR referencing rationale and impact; include version
  bump type and migration considerations; require at least one maintainer
  approval.
- Versioning Policy: Use SemVer for this document. MAJOR for incompatible
  principle changes/removals, MINOR for additions/expansions, PATCH for
  clarifications.
- Compliance: PR reviewers MUST verify compliance; exceptions require written
  justification in the PR and an entry in the plan’s Complexity Tracking.

- Upstream Relationship: Document fork origin, license headers, and sync
  procedures. Prefer upstream-first fixes when feasible. Maintain a CHANGELOG
  section for upstream syncs with notes on divergence.

**Version**: 1.3.0 | **Ratified**: 2025-11-15 | **Last Amended**: 2025-11-18
