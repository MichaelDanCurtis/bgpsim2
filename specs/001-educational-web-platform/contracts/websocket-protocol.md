# WebSocket Protocol Specification

**Feature**: Educational BGP/OSPF Web Training Platform  
**Created**: 2025-11-18  
**Version**: 1.0.0

## Overview

WebSocket protocol for real-time bidirectional communication between Yew WASM frontend and Axum backend server. Enables low-latency simulation state updates, event notifications, and task validation results.

**Endpoint**: `wss://api.bgpemu2.example.com/v1/ws?token=<JWT>`

---

## Connection Lifecycle

### 1. Handshake

**Client** (Yew WASM via gloo-net):
```rust
use gloo_net::websocket::{futures::WebSocket, Message};

let url = format!("wss://api.bgpemu2.example.com/v1/ws?token={}", jwt_token);
let ws = WebSocket::open(&url)?;
```

**Server** (Axum + tokio-tungstenite):
```rust
use axum::{
    extract::{Query, WebSocketUpgrade},
    response::Response,
};

async fn ws_handler(
    ws: WebSocketUpgrade,
    Query(params): Query<HashMap<String, String>>,
) -> Response {
    let token = params.get("token").unwrap();
    // Validate JWT token
    ws.on_upgrade(|socket| handle_socket(socket, user_id))
}
```

### 2. Subscription

After connection established, client sends subscription message:

**Client → Server**:
```json
{
  "type": "Subscribe",
  "topology_id": "550e8400-e29b-41d4-a716-446655440000"
}
```

**Server Response**:
```json
{
  "type": "SubscribeAck",
  "topology_id": "550e8400-e29b-41d4-a716-446655440000",
  "current_state": {
    "timestamp": 45.2,
    "router_states": { /* ... */ }
  }
}
```

### 3. Active Session

Server pushes updates when simulation state changes. Client can send control commands.

### 4. Teardown

**Client-Initiated**:
```json
{
  "type": "Unsubscribe",
  "topology_id": "550e8400-e29b-41d4-a716-446655440000"
}
```

**Server-Initiated** (idle timeout after 30 minutes):
```json
{
  "type": "Error",
  "code": 1000,
  "message": "Connection closed due to inactivity"
}
```

---

## Message Types

### Client → Server Messages

#### 1. Subscribe
Join simulation updates for a topology.

```rust
#[derive(Serialize, Deserialize)]
struct Subscribe {
    #[serde(rename = "type")]
    msg_type: String,  // "Subscribe"
    topology_id: Uuid,
}
```

**Example**:
```json
{
  "type": "Subscribe",
  "topology_id": "550e8400-e29b-41d4-a716-446655440000"
}
```

#### 2. Unsubscribe
Leave simulation updates.

```rust
#[derive(Serialize, Deserialize)]
struct Unsubscribe {
    #[serde(rename = "type")]
    msg_type: String,  // "Unsubscribe"
    topology_id: Uuid,
}
```

#### 3. SimulationControl
Control simulation playback (play, pause, step).

```rust
#[derive(Serialize, Deserialize)]
struct SimulationControl {
    #[serde(rename = "type")]
    msg_type: String,  // "SimulationControl"
    topology_id: Uuid,
    action: ControlAction,
}

#[derive(Serialize, Deserialize)]
enum ControlAction {
    Play,
    Pause,
    StepForward,
    StepBackward,
    Seek { timestamp: f64 },
    SetSpeed { multiplier: f32 },  // 1x, 2x, 5x, 10x
}
```

**Example**:
```json
{
  "type": "SimulationControl",
  "topology_id": "550e8400-e29b-41d4-a716-446655440000",
  "action": {
    "Seek": { "timestamp": 30.5 }
  }
}
```

#### 4. ConfigChange
Apply configuration change to router.

```rust
#[derive(Serialize, Deserialize)]
struct ConfigChange {
    #[serde(rename = "type")]
    msg_type: String,  // "ConfigChange"
    topology_id: Uuid,
    router_id: Uuid,
    config_commands: Vec<String>,  // CLI commands
}
```

**Example**:
```json
{
  "type": "ConfigChange",
  "topology_id": "550e8400-e29b-41d4-a716-446655440000",
  "router_id": "660e8400-e29b-41d4-a716-446655440001",
  "config_commands": [
    "configure terminal",
    "router bgp 65001",
    "neighbor 10.0.0.2 remote-as 65002",
    "exit"
  ]
}
```

#### 5. ValidateTask
Request task validation.

```rust
#[derive(Serialize, Deserialize)]
struct ValidateTask {
    #[serde(rename = "type")]
    msg_type: String,  // "ValidateTask"
    task_id: Uuid,
    topology_id: Uuid,
}
```

#### 6. Ping
Keep-alive heartbeat.

```rust
#[derive(Serialize, Deserialize)]
struct Ping {
    #[serde(rename = "type")]
    msg_type: String,  // "Ping"
    timestamp: i64,
}
```

---

### Server → Client Messages

#### 1. SubscribeAck
Confirm subscription with current state.

```rust
#[derive(Serialize, Deserialize)]
struct SubscribeAck {
    #[serde(rename = "type")]
    msg_type: String,  // "SubscribeAck"
    topology_id: Uuid,
    current_state: SimulationState,
}
```

#### 2. SimulationUpdate
Push simulation state changes.

```rust
#[derive(Serialize, Deserialize)]
struct SimulationUpdate {
    #[serde(rename = "type")]
    msg_type: String,  // "SimulationUpdate"
    topology_id: Uuid,
    timestamp: f64,
    state_diff: StateDiff,  // Only changed data
}

#[derive(Serialize, Deserialize)]
struct StateDiff {
    router_updates: HashMap<Uuid, RouterStateDiff>,
    link_updates: Vec<LinkStateChange>,
}

#[derive(Serialize, Deserialize)]
struct RouterStateDiff {
    routing_table_changes: RoutingTableDiff,
    bgp_neighbor_changes: Vec<BgpNeighborChange>,
}

#[derive(Serialize, Deserialize)]
struct RoutingTableDiff {
    added: Vec<Route>,
    removed: Vec<Route>,
    modified: Vec<(Route, Route)>,  // (old, new)
}
```

**Example**:
```json
{
  "type": "SimulationUpdate",
  "topology_id": "550e8400-e29b-41d4-a716-446655440000",
  "timestamp": 12.5,
  "state_diff": {
    "router_updates": {
      "660e8400-e29b-41d4-a716-446655440001": {
        "routing_table_changes": {
          "added": [
            {
              "prefix": "192.0.2.0/24",
              "next_hop": "10.0.0.2",
              "protocol": "Bgp",
              "metric": 0,
              "admin_distance": 20
            }
          ],
          "removed": [],
          "modified": []
        },
        "bgp_neighbor_changes": [
          {
            "neighbor_ip": "10.0.0.2",
            "old_state": "Active",
            "new_state": "Established"
          }
        ]
      }
    },
    "link_updates": []
  }
}
```

#### 3. EventNotification
Push discrete simulation events (BGP UPDATE, link down, etc.).

```rust
#[derive(Serialize, Deserialize)]
struct EventNotification {
    #[serde(rename = "type")]
    msg_type: String,  // "EventNotification"
    topology_id: Uuid,
    event: SimulationEvent,
}

#[derive(Serialize, Deserialize)]
struct SimulationEvent {
    timestamp: f64,
    event_type: EventType,
    router_id: Option<Uuid>,
    details: String,
}
```

**Example**:
```json
{
  "type": "EventNotification",
  "topology_id": "550e8400-e29b-41d4-a716-446655440000",
  "event": {
    "timestamp": 10.2,
    "event_type": "BgpSessionUp",
    "router_id": "660e8400-e29b-41d4-a716-446655440001",
    "details": "BGP session with 10.0.0.2 (AS 65002) entered Established state"
  }
}
```

#### 4. ValidationResult
Task validation outcome.

```rust
#[derive(Serialize, Deserialize)]
struct ValidationResult {
    #[serde(rename = "type")]
    msg_type: String,  // "ValidationResult"
    task_id: Uuid,
    passed: bool,
    diagnostics: String,
    failed_rules: Vec<FailedRule>,
}

#[derive(Serialize, Deserialize)]
struct FailedRule {
    rule_type: String,
    expected: String,
    actual: String,
}
```

**Example** (Failed):
```json
{
  "type": "ValidationResult",
  "task_id": "770e8400-e29b-41d4-a716-446655440002",
  "passed": false,
  "diagnostics": "BGP session R1->10.0.0.2 is in Active state, expected Established",
  "failed_rules": [
    {
      "rule_type": "BgpSessionState",
      "expected": "established",
      "actual": "active"
    }
  ]
}
```

**Example** (Passed):
```json
{
  "type": "ValidationResult",
  "task_id": "770e8400-e29b-41d4-a716-446655440002",
  "passed": true,
  "diagnostics": "All validation criteria met",
  "failed_rules": []
}
```

#### 5. ConfigApplied
Confirmation that config change was applied.

```rust
#[derive(Serialize, Deserialize)]
struct ConfigApplied {
    #[serde(rename = "type")]
    msg_type: String,  // "ConfigApplied"
    topology_id: Uuid,
    router_id: Uuid,
    success: bool,
    error_message: Option<String>,
}
```

**Example** (Success):
```json
{
  "type": "ConfigApplied",
  "topology_id": "550e8400-e29b-41d4-a716-446655440000",
  "router_id": "660e8400-e29b-41d4-a716-446655440001",
  "success": true,
  "error_message": null
}
```

**Example** (Error):
```json
{
  "type": "ConfigApplied",
  "topology_id": "550e8400-e29b-41d4-a716-446655440000",
  "router_id": "660e8400-e29b-41d4-a716-446655440001",
  "success": false,
  "error_message": "Invalid BGP AS number: 99999999999"
}
```

#### 6. Pong
Heartbeat response.

```rust
#[derive(Serialize, Deserialize)]
struct Pong {
    #[serde(rename = "type")]
    msg_type: String,  // "Pong"
    timestamp: i64,
}
```

#### 7. Error
Error notification.

```rust
#[derive(Serialize, Deserialize)]
struct Error {
    #[serde(rename = "type")]
    msg_type: String,  // "Error"
    code: u16,
    message: String,
}
```

**Error Codes**:
- `1000`: Normal closure
- `1001`: Going away (server shutdown)
- `1002`: Protocol error
- `1003`: Unsupported message type
- `4001`: Unauthorized (invalid token)
- `4004`: Topology not found
- `4009`: Rate limit exceeded

---

## Protocol Rules

### 1. Message Format
- All messages are JSON-encoded text frames
- Binary frames not supported in v1
- UTF-8 encoding required

### 2. Heartbeat
- Client MUST send Ping every 30 seconds
- Server responds with Pong
- Connection closed if no Ping received for 60 seconds

### 3. Rate Limiting
- Max 100 messages/second per connection
- Exceeding limit triggers Error message with code 4009

### 4. Subscription Limits
- Max 5 concurrent topology subscriptions per connection
- Attempting 6th subscription triggers Error

### 5. Message Ordering
- SimulationUpdate messages guaranteed in timestamp order
- EventNotification messages may arrive out-of-order (client must sort by timestamp)

### 6. Reconnection
- Client SHOULD implement exponential backoff (1s, 2s, 4s, 8s, 16s max)
- Server SHOULD maintain simulation state for 5 minutes after disconnect

---

## Implementation Examples

### Client (Yew WASM)

```rust
use gloo_net::websocket::{futures::WebSocket, Message};
use futures::{SinkExt, StreamExt};
use serde_json;

pub struct WsClient {
    socket: WebSocket,
    topology_id: Uuid,
}

impl WsClient {
    pub async fn connect(jwt_token: &str, topology_id: Uuid) -> Result<Self, JsValue> {
        let url = format!("wss://api.bgpemu2.example.com/v1/ws?token={}", jwt_token);
        let ws = WebSocket::open(&url)?;
        
        let mut client = Self {
            socket: ws,
            topology_id,
        };
        
        // Subscribe to topology
        client.subscribe().await?;
        
        Ok(client)
    }
    
    async fn subscribe(&mut self) -> Result<(), JsValue> {
        let msg = Subscribe {
            msg_type: "Subscribe".to_string(),
            topology_id: self.topology_id,
        };
        
        let json = serde_json::to_string(&msg).unwrap();
        self.socket.send(Message::Text(json)).await?;
        Ok(())
    }
    
    pub async fn recv(&mut self) -> Option<ServerMessage> {
        let msg = self.socket.next().await?;
        match msg {
            Ok(Message::Text(text)) => {
                serde_json::from_str(&text).ok()
            }
            _ => None,
        }
    }
    
    pub async fn send_control(&mut self, action: ControlAction) -> Result<(), JsValue> {
        let msg = SimulationControl {
            msg_type: "SimulationControl".to_string(),
            topology_id: self.topology_id,
            action,
        };
        
        let json = serde_json::to_string(&msg).unwrap();
        self.socket.send(Message::Text(json)).await?;
        Ok(())
    }
}
```

### Server (Axum)

```rust
use axum::{
    extract::{ws::WebSocket, WebSocketUpgrade, Query},
    response::Response,
};
use tokio::sync::broadcast;

async fn ws_handler(
    ws: WebSocketUpgrade,
    Query(params): Query<HashMap<String, String>>,
) -> Response {
    let token = params.get("token").expect("Missing token");
    let user_id = validate_jwt(token).expect("Invalid token");
    
    ws.on_upgrade(move |socket| handle_socket(socket, user_id))
}

async fn handle_socket(mut socket: WebSocket, user_id: Uuid) {
    let (mut sender, mut receiver) = socket.split();
    
    // Broadcast channel for simulation updates
    let (tx, mut rx) = broadcast::channel::<ServerMessage>(100);
    
    // Spawn sender task
    tokio::spawn(async move {
        while let Ok(msg) = rx.recv().await {
            let json = serde_json::to_string(&msg).unwrap();
            if sender.send(Message::Text(json)).await.is_err() {
                break;
            }
        }
    });
    
    // Receive loop
    while let Some(Ok(msg)) = receiver.next().await {
        if let Message::Text(text) = msg {
            let client_msg: ClientMessage = serde_json::from_str(&text).unwrap();
            handle_client_message(client_msg, &tx, user_id).await;
        }
    }
}

async fn handle_client_message(
    msg: ClientMessage,
    tx: &broadcast::Sender<ServerMessage>,
    user_id: Uuid,
) {
    match msg {
        ClientMessage::Subscribe { topology_id } => {
            // Load current state
            let state = load_simulation_state(topology_id).await;
            let ack = ServerMessage::SubscribeAck {
                topology_id,
                current_state: state,
            };
            tx.send(ack).ok();
        }
        ClientMessage::SimulationControl { action, .. } => {
            // Apply control action
            apply_simulation_control(action).await;
        }
        // ... handle other message types
    }
}
```

---

## Performance Targets

- **Message Latency**: <50ms server → client (99th percentile)
- **Update Frequency**: 10 updates/second during active simulation (Constitution X: <100ms CLI response)
- **Throughput**: 100 concurrent connections per server instance (Constitution FR-064: 100 users)
- **Payload Size**: SimulationUpdate <10KB per message (compressed diffs)

---

## Security Considerations

1. **Authentication**: JWT token required in WebSocket URL query param
2. **Authorization**: Server validates user owns/has access to requested topology
3. **Rate Limiting**: 100 msg/sec per connection to prevent DoS
4. **Input Validation**: All config commands validated before applying
5. **Resource Limits**: Max 5 topology subscriptions per connection

---

## Constitution Compliance

- ✅ **Principle I (TDD)**: WebSocket protocol testable via mock server/client
- ✅ **Principle X (Performance)**: <50ms latency meets <100ms CLI requirement
- ✅ **Principle XIII (Rust Discipline)**: serde for serialization, no unsafe code
- ✅ **FR-064**: 100 concurrent connections supported

---

**Status**: ✅ WebSocket protocol specification complete.
