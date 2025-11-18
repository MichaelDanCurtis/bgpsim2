# Quickstart Guide

**Feature**: Educational BGP/OSPF Web Training Platform  
**Last Updated**: 2025-11-18  
**Target Audience**: Developers setting up local environment

## Prerequisites

### Required Tools

1. **Rust Toolchain** (1.70+)
   ```powershell
   # Install rustup (Windows)
   winget install Rustlang.Rustup
   
   # Verify installation
   rustc --version  # Should be 1.70 or newer
   cargo --version
   ```

2. **Trunk** (WASM Build Tool)
   ```powershell
   cargo install trunk
   trunk --version  # Should be 0.18+
   ```

3. **wasm-bindgen-cli**
   ```powershell
   cargo install wasm-bindgen-cli
   wasm-bindgen --version
   ```

4. **PostgreSQL** (15+)
   ```powershell
   # Windows: Use official installer from postgresql.org
   # Or via Chocolatey
   choco install postgresql15
   
   # Create database
   psql -U postgres
   CREATE DATABASE bgpemu2_dev;
   \q
   ```

5. **Node.js** (18+ for xterm.js)
   ```powershell
   winget install OpenJS.NodeJS.LTS
   node --version  # Should be 18+
   npm --version
   ```

6. **sqlx-cli** (Database Migrations)
   ```powershell
   cargo install sqlx-cli --features postgres
   sqlx --version
   ```

### Optional Tools

- **Tailwind CSS CLI** (if modifying styles)
  ```powershell
  npm install -g tailwindcss
  ```

---

## Project Setup

### 1. Clone Repository

```powershell
git clone https://github.com/yourusername/bgpemu2.git
cd bgpemu2
```

### 2. Install Frontend Dependencies

```powershell
cd bgpsim-web
npm install  # Installs xterm.js and related packages
cd ..
```

### 3. Configure Environment Variables

Create `.env` file in project root:

```env
# Database
DATABASE_URL=postgres://postgres:password@localhost:5432/bgpemu2_dev

# Server
SERVER_HOST=127.0.0.1
SERVER_PORT=3000

# JWT Secret (generate with: openssl rand -base64 32)
JWT_SECRET=your-secret-key-here

# CORS (for local dev)
CORS_ORIGIN=http://localhost:8080

# WebSocket
WS_PING_INTERVAL_SECS=30
WS_IDLE_TIMEOUT_SECS=1800

# Logging
RUST_LOG=info,bgpemu2_server=debug
```

### 4. Run Database Migrations

```powershell
sqlx migrate run
# Applies all migrations from migrations/ directory
```

---

## Running Locally

### Terminal 1: Backend Server

```powershell
cd bgpemu2-server
cargo run --release

# Should see:
# [INFO] Server listening on http://127.0.0.1:3000
# [INFO] WebSocket endpoint: ws://127.0.0.1:3000/v1/ws
```

### Terminal 2: Frontend Dev Server

```powershell
cd bgpsim-web
trunk serve --open

# Should see:
# [INFO] 📦 starting build
# [INFO] ✅ success
# [INFO] 📡 serving static content at http://127.0.0.1:8080
```

Browser opens automatically at `http://127.0.0.1:8080`

---

## First Scenario: Creating Your First Topology

### Step 1: Register Account

1. Navigate to `http://localhost:8080`
2. Click **"Register"**
3. Fill form:
   - Email: `student@example.com`
   - Password: `SecurePassword123!`
   - Name: `Test Student`
   - Role: `Student`
4. Click **"Create Account"**

You're automatically logged in after registration.

### Step 2: Create Simple BGP Topology

1. Click **"New Topology"** button
2. **Add Routers**:
   - Drag **Router** icon onto canvas (creates R1)
   - Drag again (creates R2)
3. **Connect Routers**:
   - Click R1, drag to R2
   - Link appears between routers
4. **Configure BGP**:
   - Click R1 → **Sidecar opens**
   - Go to **"BGP Config"** tab
   - Fill form:
     ```
     AS Number: 65001
     Router ID: 10.0.0.1
     Neighbor IP: 10.0.0.2
     Neighbor AS: 65002
     Advertise Networks: 192.0.2.0/24
     ```
   - Click **"Apply"**
   
   - Click R2 → **Sidecar opens**
   - Fill form:
     ```
     AS Number: 65002
     Router ID: 10.0.0.2
     Neighbor IP: 10.0.0.1
     Neighbor AS: 65001
     Advertise Networks: 198.51.100.0/24
     ```
   - Click **"Apply"**

### Step 3: Start Simulation

1. Click **"Start Simulation"** button (play icon)
2. Watch BGP session establish:
   - Links turn **green** (BGP Established)
   - Route counters update in sidecar
3. **Verify Routes**:
   - Double-click R1 → **CLI terminal opens**
   - Type: `show ip bgp`
   - Should see:
     ```
     BGP table version is 2, local router ID is 10.0.0.1
     Status codes: * valid, > best
        Network          Next Hop            Metric LocPrf Weight Path
     *> 192.0.2.0/24     0.0.0.0                  0         32768 i
     *> 198.51.100.0/24  10.0.0.2                 0             0 65002 i
     ```

### Step 4: Test Configuration Changes

In R1 CLI terminal:

```
R1> configure terminal
R1(config)# router bgp 65001
R1(config-router)# network 203.0.113.0 mask 255.255.255.0
R1(config-router)# exit
R1(config)# exit
R1> show ip bgp
```

New network `203.0.113.0/24` appears in BGP table!

### Step 5: Use Time Machine

1. Click **"Pause"** button
2. Drag **Timeline scrubber** backward
3. Watch simulation rewind to earlier state
4. Click **"Bookmark"** → Name: "Post-Config State"
5. Click **"Play"** to resume

### Step 6: Save Topology

1. Click **"Save Topology"**
2. Name: `My First BGP Lab`
3. Click **"Save"**

Topology persists to database and appears in **"My Topologies"** list.

---

## Common Development Tasks

### Rebuild Frontend After Rust Changes

```powershell
cd bgpsim-web
trunk clean
trunk build --release
```

### Run Database Migrations

```powershell
# Create new migration
sqlx migrate add <migration_name>

# Apply migrations
sqlx migrate run

# Revert last migration
sqlx migrate revert
```

### Generate API Documentation

```powershell
# Install openapi-generator-cli
npm install -g @openapitools/openapi-generator-cli

# Generate Rust client from OpenAPI spec
cd specs/001-educational-web-platform/contracts
openapi-generator-cli generate \
  -i api-openapi.yaml \
  -g rust \
  -o ../../../bgpemu2-client
```

### Run Backend Tests

```powershell
cd bgpemu2-server
cargo test --all-features
```

### Run Frontend Tests

```powershell
cd bgpsim-web
cargo test --target wasm32-unknown-unknown
```

### Lint and Format

```powershell
# Format all code
cargo fmt --all

# Run clippy
cargo clippy --all-targets --all-features -- -D warnings

# Fix clippy issues automatically
cargo clippy --fix --all-targets --all-features
```

---

## Troubleshooting

### Problem: `cargo build` fails with "linker error"

**Solution**: Install build tools
```powershell
# Windows
winget install Microsoft.VisualStudio.2022.BuildTools

# Or install Visual Studio with "Desktop development with C++"
```

### Problem: `trunk serve` fails with "wasm-opt not found"

**Solution**: Install binaryen
```powershell
# Windows
choco install binaryen

# Or download from https://github.com/WebAssembly/binaryen/releases
```

### Problem: WebSocket connection fails

**Solution**: Check CORS configuration
```env
# .env file
CORS_ORIGIN=http://localhost:8080  # Match Trunk dev server URL
```

### Problem: Database connection refused

**Solution**: Verify PostgreSQL is running
```powershell
# Windows Service Manager
services.msc  # Find "postgresql-x64-15" service, Start it

# Or via CLI
pg_ctl -D "C:\Program Files\PostgreSQL\15\data" start
```

### Problem: Frontend build is slow

**Solution**: Use debug builds for faster iteration
```powershell
trunk serve  # Uses debug build (fast compile, larger WASM)

# For production build:
trunk build --release  # Slower compile, optimized WASM
```

---

## GitHub Pages Deployment

### Setup GitHub Actions Workflow

1. **Enable GitHub Pages**:
   - Go to repo **Settings → Pages**
   - Source: **GitHub Actions**

2. **Create Workflow File** `.github/workflows/deploy.yml`:

```yaml
name: Deploy to GitHub Pages

on:
  push:
    branches: [main]
  workflow_dispatch:

permissions:
  contents: read
  pages: write
  id-token: write

jobs:
  build-deploy:
    runs-on: ubuntu-latest
    steps:
      - uses: actions/checkout@v3
      
      - name: Setup Rust
        uses: actions-rust-lang/setup-rust-toolchain@v1
        with:
          toolchain: stable
      
      - name: Install Trunk
        uses: jetli/trunk-action@v0.4.0
      
      - name: Install wasm-bindgen-cli
        run: cargo install wasm-bindgen-cli
      
      - name: Build WASM
        run: |
          cd bgpsim-web
          trunk build --release --public-url /bgpemu2/
      
      - name: Setup Pages
        uses: actions/configure-pages@v3
      
      - name: Upload artifact
        uses: actions/upload-pages-artifact@v2
        with:
          path: './bgpsim-web/dist'
      
      - name: Deploy to GitHub Pages
        id: deployment
        uses: actions/deploy-pages@v2
```

3. **Push to Main Branch**:
```powershell
git add .github/workflows/deploy.yml
git commit -m "Add GitHub Pages deployment workflow"
git push origin main
```

4. **Access Deployed Site**:
   - Wait for Actions to complete (~5 minutes first time)
   - Visit `https://yourusername.github.io/bgpemu2/`

### Update API Endpoint for Production

**bgpsim-web/src/config.rs**:
```rust
pub fn api_base_url() -> &'static str {
    if cfg!(debug_assertions) {
        "http://localhost:3000/v1"
    } else {
        "https://api.bgpemu2.fly.dev/v1"  // Production backend
    }
}

pub fn ws_url() -> String {
    if cfg!(debug_assertions) {
        "ws://localhost:3000/v1/ws".to_string()
    } else {
        "wss://api.bgpemu2.fly.dev/v1/ws".to_string()
    }
}
```

---

## Backend Deployment (Fly.io)

### 1. Install Fly CLI

```powershell
# Windows PowerShell
iwr https://fly.io/install.ps1 -useb | iex

# Verify
fly version
```

### 2. Login to Fly.io

```powershell
fly auth login
# Opens browser for authentication
```

### 3. Create Fly.io App

```powershell
cd bgpemu2-server
fly launch --no-deploy

# Answer prompts:
# App name: bgpemu2-api
# Region: Choose closest to users
# PostgreSQL: Yes (creates managed database)
```

### 4. Configure Secrets

```powershell
fly secrets set \
  JWT_SECRET=$(openssl rand -base64 32) \
  DATABASE_URL="postgres://user:pass@host/bgpemu2"
```

### 5. Deploy

```powershell
fly deploy

# Monitor logs
fly logs
```

### 6. Access API

Production API available at: `https://bgpemu2-api.fly.dev/v1`

---

## Next Steps

### For Students
- Complete built-in tutorials in **"Lessons"** tab
- Clone scenarios from **"Template Library"**
- Experiment with anycast scenarios

### For Instructors
- Create lesson via **"Lesson Builder"**
- Add validation tasks with hints
- Publish lesson for students

### For Developers
- Read [data-model.md](data-model.md) for entity schemas
- Review [api-openapi.yaml](contracts/api-openapi.yaml) for REST API
- Check [websocket-protocol.md](contracts/websocket-protocol.md) for real-time updates
- See [research.md](research.md) for architecture decisions

---

## Additional Resources

- **BGPsim Library Docs**: https://github.com/nsg-ethz/bgpsim
- **Yew Framework**: https://yew.rs/docs/getting-started/introduction
- **Axum Web Framework**: https://docs.rs/axum/latest/axum/
- **Trunk Build Tool**: https://trunkrs.dev/
- **Constitution**: `.specify/memory/constitution.md` (project governance)

---

## Support

- **Issues**: https://github.com/yourusername/bgpemu2/issues
- **Discussions**: https://github.com/yourusername/bgpemu2/discussions
- **Discord**: [Join BGPemu2 Community](#)

---

**Status**: ✅ Quickstart guide complete. Local dev setup functional.
