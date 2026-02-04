​# Sensu-Go Architecture - Knowledge Transfer Documentation

## Table of Contents
1. [Agent Backend Communication](#1-agent-backend-communication)
2. [How Backend Starts/Initiates](#2-how-backend-startsintiates)
3. [How Agent Receives Connection](#3-how-agent-receives-connection)
4. [Protocols Used](#4-protocols-used)
5. [Check Scheduler](#5-check-scheduler)
6. [How Events Generate](#6-how-events-generate)
7. [How Events Are Received by Backend](#7-how-events-are-received-by-backend)
8. [Handler, Filter, and Transform Processes](#8-handler-filter-and-transform-processes)
9. [Pub-Sub Architecture - WizardBus](#9-pub-sub-architecture---wizardbus)
10. [Additional Code-Level Information](#10-additional-code-level-information)

---

## 1. Agent Backend Communication

### Overview
Agent connects to backend via **WebSocket protocol**. The transport layer provides bidirectional messaging between agent and backend.

### Key Files
- `transport/client.go` - WebSocket client implementation
- `agent/agent.go` - Agent main logic
- `backend/agentd/agentd.go` - Backend agent daemon
- `backend/agentd/session.go` - Agent session management

### Key Functions/Methods
| Function | Location | Purpose |
|----------|----------|---------|
| `Connect()` | transport/client.go | Establishes WebSocket connection |
| `webSocketHandler()` | backend/agentd/agentd.go | Backend handler for WebSocket upgrades |
| `NewSession()` | backend/agentd/session.go | Creates agent session on backend |
| `handleMessage()` | backend/agentd/session.go | Routes incoming messages |

### Data Flow
```
Agent → WebSocket → Backend Agentd → Session → Message Bus
```

### Technical Details
- Uses **gorilla/websocket** library
- Connection URL format: `ws://backend:8081` or `wss://backend:8081` (TLS)
- Supports TLS with client certificates
- Includes reconnection logic with exponential backoff
- Handshake timeout: 15 seconds
- Message types: `MessageTypeKeepalive`, `MessageTypeEvent`, `MessageTypeEntityConfig`

### Code Locations
- **transport/client.go**: Lines 1-75 - WebSocket client connection
- **agent/agent.go**: Lines 1-250 - Agent initialization and connection management
- **backend/agentd/agentd.go**: Lines 1-300 - Agentd daemon setup
- **backend/agentd/session.go**: Lines 1-150 - Session creation and management

---

## 2. How Backend Starts/Initiates

### Overview
Backend starts multiple daemons in sequence during initialization. Each daemon implements a common interface with `Start()`, `Stop()`, `Err()`, and `Name()` methods.

### Key Files
- `backend/backend.go` - Main backend initialization and startup
- `backend/etcd/id.go` - Backend ID registration with etcd

### Initialization Sequence

#### Phase 1: Initialize() Function
**Location**: `backend/backend.go` Lines 300-750

**Steps**:
1. Create Backend struct
2. Initialize context and cancellation
3. Create etcd client with gRPC
4. Create etcd store (data persistence layer)
5. Initialize JWT secret
6. Register backend with etcd (get unique ID via lease)
7. Initialize message bus (WizardBus)
8. Initialize asset manager
9. Create all daemon instances:
   - **BackendIDGetter** - Maintains backend registration
   - **WizardBus** - Message bus
   - **EventD** - Event processing
   - **SchedulerD** - Check scheduling
   - **Agentd** - Agent connections
   - **Pipelined** - Handler pipeline
   - **Keepalived** - Keepalive monitoring

#### Phase 2: Start() Method
**Location**: `backend/backend.go` Lines 714-750

**Steps**:
1. Loop through daemons in order
2. Call `Start()` on each daemon
3. Add started daemons to wait groups
4. Monitor error channels from all daemons

### Daemon Start Order
```
1. BackendIDGetter (registers with etcd)
   ↓
2. WizardBus (message bus)
   ↓
3. AssetManager (asset handling)
   ↓
4. EventD (event processing)
   ↓
5. SchedulerD (check scheduling)
   ↓
6. Agentd (agent connections on port 8081)
   ↓
7. Pipelined (handler pipeline)
   ↓
8. Keepalived (keepalive monitoring)
```

### Etcd Connection
**Location**: `backend/backend.go` Lines 200-250

- Uses **gRPC** for etcd communication
- Connection includes TLS support
- Backend gets unique ID via etcd lease mechanism
- Enables distributed backend coordination

### Key Functions
| Function | Purpose |
|----------|---------|
| `Initialize()` | Sets up all backend components |
| `Start()` | Starts all daemons sequentially |
| `NewBackendIDGetter()` | Registers backend with etcd |
| `New()` | Creates daemon instances |
| `newClient()` | Creates etcd client |

---

## 3. How Agent Receives Connection

### Overview
The agent initiates the connection to the backend. The backend's **Agentd daemon** listens for incoming WebSocket connections and manages agent sessions.

### Backend Side: Agentd

#### Startup
**Location**: `backend/agentd/agentd.go` Lines 222-240

```go
func (a *Agentd) Start() error {
    // HTTP server starts on port 8081 (default)
    // Waits for ready state before accepting connections
}
```

#### HTTP to WebSocket Upgrade
**Location**: `backend/agentd/agentd.go` Lines 271-290

**Process**:
1. Agent sends HTTP request to `ws://backend:8081/`
2. Agentd's `webSocketHandler` receives request
3. Authentication middleware validates credentials (JWT)
4. Authorization middleware checks permissions
5. HTTP connection upgraded to WebSocket via `upgrader.Upgrade()`
6. New `Session` object created for the agent

#### Session Creation
**Location**: `backend/agentd/session.go` Lines 150-250

**Session Components**:
- **Transport**: WebSocket connection wrapper
- **MessageHandler**: Routes incoming messages by type
- **Subscriptions**: Map of topics the agent is subscribed to
- **EntityConfig**: Agent's entity configuration
- **Bus**: Reference to WizardBus for publishing

### Agent Side: Connection Establishment

**Location**: `agent/agent.go` Lines 250-450

#### Connection Process
```go
func (a *Agent) Run(ctx context.Context) error {
    // 1. Select backend from list
    // 2. Create WebSocket connection
    // 3. Send initial keepalive
    // 4. Subscribe to check topics
    // 5. Start message handlers
}
```

### Data Flow
```
Agent HTTP Request
    ↓
Agentd HTTP Server (port 8081)
    ↓
Authentication Middleware (JWT validation)
    ↓
Authorization Middleware (permission check)
    ↓
Upgrade HTTP → WebSocket
    ↓
Create Session Object
    ↓
Session subscribes to entity config updates
    ↓
Agent subscribed to check topics on MessageBus
    ↓
Bidirectional communication established
```

### Key Files
- `backend/agentd/agentd.go` - HTTP server and WebSocket upgrade
- `backend/agentd/session.go` - Session management
- `backend/agentd/middlewares.go` - Authentication/Authorization
- `agent/agent.go` - Agent connection logic

### Authentication Flow
**Location**: `backend/agentd/middlewares.go` Lines 1-100

1. Agent sends JWT token in request header or uses mTLS
2. Middleware extracts and validates token
3. Token contains: username, groups, namespace
4. Context populated with authentication claims
5. Connection allowed or rejected

---

## 4. Protocols Used

### WebSocket (Primary Agent-Backend Protocol)

**Purpose**: Bidirectional communication between agents and backend

**Files**: 
- `transport/client.go` - Client implementation
- `backend/agentd/agentd.go` - Server implementation

**Details**:
- Library: `gorilla/websocket`
- Default port: **8081**
- Protocols: `ws://` (plain) or `wss://` (TLS)
- Supports client certificate authentication (mTLS)
- Message types defined in transport package

**Message Types**:
| Type | Value | Purpose |
|------|-------|---------|
| MessageTypeKeepalive | 0 | Agent keepalive messages |
| MessageTypeEvent | 1 | Check result events |
| MessageTypeEntityConfig | 2 | Entity configuration updates |

### gRPC

**Purpose**: Communication between backend and etcd cluster

**Location**: `backend/backend.go` Lines 208-220

**Details**:
- Library: `google.golang.org/grpc`
- Used exclusively for etcd communication
- Not exposed to agents or external clients
- Metrics tracked:
  - `etcd_network_client_grpc_received_bytes_total`
  - `etcd_network_client_grpc_sent_bytes_total`

### HTTP/HTTPS

**Purpose**: REST API, metrics, WebSocket handshake

**Components**:
1. **REST API (apid)** - Port 8080
2. **Agent API** - Port 3031 (local agent API)
3. **Prometheus Metrics** - Embedded in HTTP servers
4. **WebSocket Handshake** - Initial HTTP upgrade request

**Files**:
- `backend/apid/*` - REST API implementation
- `agent/api.go` - Agent local API

### Protobuf / JSON

**Purpose**: Message serialization formats

**Location**: `agent/agent.go` Lines 50-75

**Content-Type Headers**:
- `application/json` - JSON serialization
- `application/octet-stream` - Protocol Buffers (binary)

**Details**:
- Both agent and backend support both formats
- Format negotiated during connection handshake
- Protobuf is more efficient (smaller payload)
- JSON is more human-readable for debugging

**Serialization Functions**:
```go
// JSON
MarshalJSON(msg proto.Message) ([]byte, error)
UnmarshalJSON(b []byte, msg proto.Message) error

// Protobuf
proto.Marshal(pb proto.Message) ([]byte, error)
proto.Unmarshal(buf []byte, pb proto.Message) error
```

---

## 5. Check Scheduler - Backend Build Request to Agent Process

### Overview
The **SchedulerD daemon** watches check configurations in etcd and schedules check executions based on their interval or cron configuration. It builds check requests and publishes them to the message bus for agents to consume.

### Key Components

#### 1. SchedulerD Daemon
**Location**: `backend/schedulerd/schedulerd.go`

**Responsibilities**:
- Watch for check configuration changes in etcd
- Create appropriate scheduler type for each check
- Manage scheduler lifecycle
- Track metrics for active schedulers

**Initialization** (Lines 1-100):
```go
func New(ctx context.Context, c Config, opts ...Option) (*Schedulerd, error) {
    s := &Schedulerd{
        store:       c.Store,
        queueGetter: c.QueueGetter,
        bus:         c.Bus,
        ringPool:    c.RingPool,
    }
    s.checkWatcher = NewCheckWatcher(...)
    s.adhocRequestExecutor = NewAdhocRequestExecutor(...)
    return s, nil
}
```

#### 2. Check Watcher
**Location**: `backend/schedulerd/check_watcher.go`

**Purpose**: Monitors etcd for check configuration changes

**Process**:
1. Watches etcd key prefix: `/sensu.io/checks/`
2. Receives events: PUT (create/update), DELETE
3. For each event, creates/updates/removes scheduler
4. Dispatches to appropriate scheduler type

#### 3. Scheduler Types

**Location**: `backend/schedulerd/` various files

| Scheduler Type | File | Trigger Mechanism | Use Case |
|----------------|------|-------------------|----------|
| **Interval Scheduler** | interval_scheduler.go | Time-based interval | Regular checks (every 60s) |
| **Cron Scheduler** | cron_scheduler.go | Cron expression | Complex schedules |
| **Round-Robin Interval** | roundrobin_interval.go | Interval + Ring | Distributed interval checks |
| **Round-Robin Cron** | roundrobin_cron.go | Cron + Ring | Distributed cron checks |

#### 4. Check Executor
**Location**: `backend/schedulerd/executor.go`

**Key Functions**:

##### buildRequest()
**Purpose**: Creates CheckRequest from CheckConfig

```go
func (c *CheckExecutor) buildRequest(check *corev2.CheckConfig) (*corev2.CheckRequest, error) {
    request := &corev2.CheckRequest{
        Config: check,
        Issued: time.Now().Unix(),
    }
    // Handle secrets provider if configured
    // Add assets information
    return request, nil
}
```

##### execute()
**Purpose**: Publishes check request to subscription topics

```go
func (c *CheckExecutor) execute(check *corev2.CheckConfig) error {
    request, err := c.buildRequest(check)
    
    // Publish to each subscription
    for _, sub := range check.Subscriptions {
        topic := messaging.SubscriptionTopic(check.Namespace, sub)
        c.bus.Publish(topic, request)
    }
    return err
}
```

### Check Request Flow

```
1. Check Config Created in etcd
   /sensu.io/checks/default/check-cpu
   
2. CheckWatcher receives PUT event
   
3. Determine Scheduler Type
   - Interval? → IntervalScheduler
   - Cron? → CronScheduler
   - Round-robin? → RoundRobinScheduler
   
4. Scheduler triggers at scheduled time
   
5. Executor.buildRequest()
   - Creates CheckRequest object
   - Adds issued timestamp
   - Resolves secrets if needed
   
6. Executor.execute()
   - Publishes to each subscription topic
   - Topic format: "sensu:check:{namespace}:{subscription}"
   
7. MessageBus distributes to subscribed agents
   
8. Agent receives CheckRequest
   
9. Agent executes check command
```

### Proxy Check Requests

**Location**: `backend/schedulerd/proxy_check.go`

**Purpose**: Execute checks on behalf of other entities (monitoring external services)

**Process**:
1. Check config contains `ProxyRequests` field
2. Scheduler queries for matching entities
3. For each matched entity:
   - Substitute entity tokens in check command
   - Create separate CheckRequest
   - Publish to subscription topic

**Token Substitution Example**:
```
Command: "check_http -H {{ .name }}"
Entity name: "webserver1"
Result: "check_http -H webserver1"
```

### Subscription Topics

**Format**: `sensu:check:{namespace}:{subscription}`

**Examples**:
- `sensu:check:default:linux` - All Linux agents in default namespace
- `sensu:check:prod:webservers` - All webserver agents in prod namespace
- `sensu:check:default:entity:agent1` - Specific agent (entity subscription)

### Key Files Summary

| File | Lines | Purpose |
|------|-------|---------|
| schedulerd.go | 1-150 | Main scheduler daemon |
| check_watcher.go | All | Watches check configs in etcd |
| executor.go | 1-150 | Builds and publishes requests |
| interval_scheduler.go | All | Interval-based scheduling |
| cron_scheduler.go | All | Cron-based scheduling |
| roundrobin_interval.go | 1-200 | Round-robin interval scheduling |
| roundrobin_cron.go | All | Round-robin cron scheduling |
| proxy_check.go | 1-150 | Proxy check handling |

### Metrics

**Prometheus Metrics** tracked by schedulerd:
- `sensu_go_interval_schedulers{namespace}` - Active interval schedulers
- `sensu_go_cron_schedulers{namespace}` - Active cron schedulers
- `sensu_go_round_robin_interval_schedulers{namespace}` - Active RR interval
- `sensu_go_round_robin_cron_schedulers{namespace}` - Active RR cron

---

## 6. How Event Generates

### Overview
Events are generated when agents execute checks. An event contains the check result, entity information, optional metrics, and metadata.

### Agent-Side Event Generation

#### 1. Check Execution
**Location**: `agent/check_handler.go`

**Process Flow**:
```
1. Agent receives CheckRequest from MessageBus
   ↓
2. handleCheck() processes the request
   ↓
3. executeCheck() runs the check command
   ↓
4. Command execution captures:
   - Exit status (0=OK, 1=Warning, 2=Critical, 3=Unknown)
   - Standard output (STDOUT)
   - Standard error (STDERR)
   - Execution duration
   ↓
5. Parse output for metrics (if present)
   ↓
6. createEvent() builds Event object
   ↓
7. Event marshaled to JSON/Protobuf
   ↓
8. Sent to backend via WebSocket
```

#### 2. Check Execution Details
**Location**: `agent/check_handler.go` Lines 1-400

**Key Steps**:

##### Prepare Check Environment
```go
- Set environment variables
- Prepare stdin (if configured)
- Set working directory
- Prepare timeout enforcement
```

##### Execute Command
```go
func (a *Agent) executeCheck(ctx context.Context, check *corev2.CheckConfig) {
    // Create command executor
    cmd := a.executor.Execute(ctx, execution)
    
    // Run with timeout
    output, status, err := cmd.Run()
    
    // Capture results
}
```

##### Parse Metrics
Supported metric formats:
- **Nagios Performance Data**: `label=value[UOM];[warn];[crit];[min];[max]`
- **Graphite**: `metric.path value timestamp`
- **InfluxDB Line Protocol**: `measurement,tag=value field=value timestamp`
- **OpenTSDB**: `put metric timestamp value tag=value`

#### 3. Event Structure
**Location**: `agent/event.go` Lines 1-150

**Event Object**:
```go
type Event struct {
    // Entity information (the agent)
    Entity *Entity
    
    // Check information and results
    Check *Check
    
    // Metrics (optional)
    Metrics *Metrics
    
    // Timestamp
    Timestamp int64
    
    // Sequence number
    Sequence int64
    
    // Metadata
    Metadata *Metadata
}
```

**Entity Fields**:
- Name (agent hostname)
- Namespace
- Subscriptions (list)
- EntityClass (agent/proxy)
- System info (OS, platform, arch)
- Network interfaces

**Check Fields**:
- Name
- Command
- Status (0-3)
- Output (STDOUT)
- Executed timestamp
- Duration
- Interval
- Timeout
- Handlers (list)
- Subscriptions

**Metrics Fields**:
- Points (array of metric points)
  - Name
  - Value
  - Timestamp
  - Tags

#### 4. Event Creation
**Location**: `agent/event.go` Lines 50-100

```go
func createEvent(check *corev2.CheckConfig, output string, status uint32) *corev2.Event {
    event := &corev2.Event{
        Timestamp: time.Now().Unix(),
        Entity:    a.getEntity(),
        Check: &corev2.Check{
            Name:     check.Name,
            Command:  check.Command,
            Status:   status,
            Output:   output,
            Executed: executedTime,
            Duration: duration,
            Interval: check.Interval,
        },
    }
    
    // Parse metrics if present
    if metrics := parseMetrics(output); metrics != nil {
        event.Metrics = metrics
    }
    
    return event
}
```

### Check Execution Components

#### 1. Check Handler
**Location**: `agent/check_handler.go` Lines 100-300

**Responsibilities**:
- Queue management
- Duplicate check prevention
- Hook execution (pre/post)
- Asset download and extraction
- Check execution
- Result processing

#### 2. Command Executor
**Location**: Uses `command.Executor` interface

**Features**:
- Timeout enforcement
- Environment variable injection
- Working directory management
- STDIN/STDOUT/STDERR capture
- Process cleanup

#### 3. Hook Execution
**Location**: `agent/hook.go`

**Hooks**:
- **Pre-hooks**: Run before check execution
- **Post-hooks**: Run after check execution
- Can modify check execution environment

### Event Types

#### 1. Check Event
- Contains check execution results
- Status: OK (0), Warning (1), Critical (2), Unknown (3)
- Always has Entity and Check fields

#### 2. Metrics-Only Event
- No check results, only metrics
- Used for passive metric collection
- Entity field present, Check field may be minimal

#### 3. Keepalive Event
- Special event type for agent health
- Generated every keepalive-interval
- Status based on keepalive timeouts

### Keepalive Generation
**Location**: `agent/agent.go` Lines 500-600

```go
func (a *Agent) sendKeepalive(ctx context.Context) {
    event := &corev2.Event{
        Entity: a.getEntity(),
        Check: &corev2.Check{
            Name:   "keepalive",
            Status: 0, // OK
        },
        Timestamp: time.Now().Unix(),
    }
    
    a.sendEvent(event)
}
```

**Keepalive Timing**:
- Interval: Configurable (default 20s)
- Warning timeout: Configurable
- Critical timeout: Configurable
- Backend marks entity unhealthy if keepalives stop

### Metrics Parsing

**Location**: `agent/check_handler.go` (metric parsing functions)

**Process**:
1. Scan check output line by line
2. Match against metric format patterns
3. Extract: name, value, timestamp, tags
4. Create MetricPoint objects
5. Add to Event.Metrics.Points array

**Example Graphite Metric**:
```
Input: "server.cpu.usage 85.5 1609459200"
Output: MetricPoint{
    Name: "server.cpu.usage",
    Value: 85.5,
    Timestamp: 1609459200,
}
```

### Key Files Summary

| File | Purpose |
|------|---------|
| agent/check_handler.go | Check execution and event creation |
| agent/event.go | Event structure and creation |
| agent/hook.go | Pre/post check hooks |
| command/executor.go | Command execution wrapper |
| types/event.go | Event type definitions (core/v2) |

---

## 7. How Events Are Received by Backend

### Overview
Events sent by agents are received by the backend's **EventD daemon**, which processes, validates, stores them in etcd, and publishes them to the pipeline for handler processing.

### Event Reception Flow

```
Agent → WebSocket → Session → MessageBus (TopicEventRaw)
    ↓
EventD Workers (multiple goroutines)
    ↓
Process Event
    ↓
Store in etcd
    ↓
MessageBus (TopicEvent)
    ↓
Pipelined (handlers)
```

### 1. Session Receives Event

**Location**: `backend/agentd/session.go` Lines 630-660

```go
func (s *Session) handleEvent(_ context.Context, payload []byte) error {
    // 1. Unmarshal event from JSON/Protobuf
    event := &corev2.Event{}
    if err := s.unmarshal(payload, event); err != nil {
        return err
    }
    
    // 2. Validate event structure
    if err := event.Validate(); err != nil {
        return err
    }
    
    // 3. Add entity subscription
    event.Entity.Subscriptions = corev2.AddEntitySubscription(
        event.Entity.Name, 
        event.Entity.Subscriptions,
    )
    
    // 4. Track metrics
    eventBytesSummary.WithLabelValues(eventType).Observe(float64(len(payload)))
    
    // 5. Publish to raw event topic
    return s.bus.Publish(messaging.TopicEventRaw, event)
}
```

**Topic**: `messaging.TopicEventRaw` = "sensu.io/events_raw"

### 2. EventD Daemon

**Location**: `backend/eventd/eventd.go`

#### Initialization (Lines 1-200)
```go
func New(ctx context.Context, c Config, opts ...Option) (*Eventd, error) {
    e := &Eventd{
        store:             c.Store,
        bus:               c.Bus,
        livenessFactory:   c.LivenessFactory,
        bufferSize:        c.BufferSize,
        workerCount:       c.WorkerCount,
        eventStore:        c.EventStore,
    }
    
    // Create worker pool
    e.workerPool = newWorkerPool(e.workerCount)
    
    return e, nil
}
```

#### Configuration
- **BufferSize**: Event queue size (default: 100)
- **WorkerCount**: Number of concurrent processors (default: 100)
- **StoreTimeout**: Etcd operation timeout (default: 60s)

#### Start (Lines 200-250)
```go
func (e *Eventd) Start() error {
    // Subscribe to raw events from agents
    sub, err := e.bus.Subscribe(
        messaging.TopicEventRaw,
        "eventd",
        e, // Receiver interface
    )
    
    // Start worker goroutines
    e.startWorkers()
    
    return nil
}
```

### 3. Event Processing Pipeline

**Location**: `backend/eventd/eventd.go` Lines 400-700

#### Main Processing Function
```go
func (e *Eventd) handleEvent(ctx context.Context, event *corev2.Event) error {
    // Step 1: Create or update entity
    if err := e.createOrUpdateEntity(ctx, event); err != nil {
        return err
    }
    
    // Step 2: Check if event is silenced
    silenced, err := e.isSilenced(ctx, event)
    if err != nil {
        return err
    }
    event.Check.Silenced = silenced
    
    // Step 3: Update liveness switches
    if err := e.updateLiveness(ctx, event); err != nil {
        return err
    }
    
    // Step 4: Store event in etcd
    if err := e.updateEvent(ctx, event); err != nil {
        return err
    }
    
    // Step 5: Count metrics
    e.countMetrics(event)
    
    // Step 6: Publish to pipeline
    return e.bus.Publish(messaging.TopicEvent, event)
}
```

#### Detailed Steps

##### Step 1: Entity Management
**Location**: `backend/eventd/entity.go`

```go
func (e *Eventd) createOrUpdateEntity(ctx context.Context, event *corev2.Event) error {
    entity := event.Entity
    
    // Check if entity exists
    existing, err := e.store.GetEntityByName(ctx, entity.Name)
    
    if err != nil {
        // Entity doesn't exist - create it
        if entity.EntityClass == corev2.EntityProxyClass {
            // Proxy entity - create with limited lifecycle
            return e.createProxyEntity(ctx, entity)
        }
        // Agent entity - create with full info
        return e.store.UpdateEntity(ctx, entity)
    }
    
    // Entity exists - update it
    return e.updateEntity(ctx, existing, entity)
}
```

##### Step 2: Silencing Check
**Location**: `backend/eventd/silenced.go`

```go
func (e *Eventd) isSilenced(ctx context.Context, event *corev2.Event) ([]string, error) {
    // Get silenced entries for:
    // - Entity subscription
    // - Check subscription
    // - Entity:Check combination
    
    silencedEntries := e.store.GetSilencedEntriesBySubscription(ctx, ...)
    
    // Check if any silenced entries match
    for _, entry := range silencedEntries {
        if entry.Matches(event) {
            silencedIDs = append(silencedIDs, entry.Name)
        }
    }
    
    return silencedIDs, nil
}
```

##### Step 3: Liveness Update
**Location**: `backend/liveness/liveness.go`

```go
// Update liveness switches for the entity
func (e *Eventd) updateLiveness(ctx context.Context, event *corev2.Event) error {
    switch := e.livenessFactory.New(event.Entity.Name, event.Entity.Namespace)
    
    // Mark entity as alive (extends etcd lease)
    if err := switch.Alive(ctx); err != nil {
        return err
    }
    
    return nil
}
```

**Liveness Mechanism**:
- Creates etcd key with lease
- Lease TTL matches keepalive timeout
- Alive() extends the lease
- If lease expires, entity marked as dead
- Separate switches for keepalives and events

##### Step 4: Store Event
**Location**: `backend/eventd/eventd.go` Lines 500-600

```go
func (e *Eventd) updateEvent(ctx context.Context, event *corev2.Event) error {
    ctx, cancel := context.WithTimeout(ctx, e.storeTimeout)
    defer cancel()
    
    // Store event in etcd
    // Key format: /sensu.io/events/{namespace}/{entity}/{check}
    _, err := e.eventStore.UpdateEvent(ctx, event)
    
    return err
}
```

**Event Storage**:
- Key: `/sensu.io/events/{namespace}/{entity}/{check}`
- Example: `/sensu.io/events/default/agent1/check-cpu`
- Previous event overwritten
- Event history available via watch mechanism

##### Step 5: Metrics Counting
```go
func (e *Eventd) countMetrics(event *corev2.Event) {
    if event.HasMetrics() {
        count := len(event.Metrics.Points)
        MetricPointsProcessed.Add(float64(count))
    }
}
```

##### Step 6: Publish to Pipeline
```go
// Publish processed event to pipeline topic
e.bus.Publish(messaging.TopicEvent, event)
```

**Topic**: `messaging.TopicEvent` = "sensu.io/events"

### Worker Pool Architecture

**Location**: `backend/eventd/eventd.go`

```
EventD Receiver Channel
    ↓
Buffer (channel, size: BufferSize)
    ↓
├─ Worker 1 (goroutine)
├─ Worker 2 (goroutine)
├─ Worker 3 (goroutine)
├─ ...
└─ Worker N (goroutine)
    ↓
Each worker: handleEvent()
```

**Benefits**:
- Concurrent event processing
- Prevents blocking on slow operations
- Configurable parallelism
- Rate limiting via buffer

### Error Handling

**Location**: Throughout eventd package

**Error Cases**:
1. **Validation errors**: Event rejected, error logged
2. **Store errors**: Retry with backoff, then error logged
3. **Liveness errors**: Logged but don't block processing
4. **Bus publish errors**: Logged, event processing continues

### Metrics and Observability

**Prometheus Metrics**:

| Metric | Type | Purpose |
|--------|------|---------|
| `sensu_go_events_processed{status,type}` | Counter | Total events processed |
| `sensu_go_event_metric_points_processed` | Counter | Total metric points |
| `sensu_go_event_handler_duration{status,type}` | Summary | Processing latency |
| `sensu_go_event_handlers_busy` | Gauge | Active workers |
| `sensu_go_eventd_create_proxy_entity_duration` | Summary | Proxy entity creation time |
| `sensu_go_eventd_update_event_duration` | Summary | Event storage time |
| `sensu_go_eventd_bus_publish_duration` | Summary | Bus publish time |

**Labels**:
- **status**: success/error
- **type**: check/metrics/unknown

### Key Files Summary

| File | Lines | Purpose |
|------|-------|---------|
| backend/eventd/eventd.go | 1-894 | Main event processing logic |
| backend/eventd/entity.go | 1-200 | Entity creation/update |
| backend/eventd/silenced.go | 1-150 | Silencing evaluation |
| backend/liveness/liveness.go | 1-400 | Liveness switch management |
| backend/agentd/session.go | 630-660 | Event reception from agent |

---

## 8. Handler, Filter, and Transform Processes

### Overview
The **Pipelined daemon** implements the Sensu event pipeline, processing events through a sequence of filters, mutators (transforms), and handlers. This allows flexible event processing, notification routing, and data transformation.

### Pipeline Architecture

```
MessageBus (TopicEvent)
    ↓
Pipelined Daemon
    ↓
Event → Filters → Mutators → Handlers → External Systems
```

### 1. Pipelined Daemon

**Location**: `backend/pipelined/pipelined.go`

#### Initialization (Lines 1-120)
```go
func New(c Config, options ...Option) (*Pipelined, error) {
    p := &Pipelined{
        bus:          c.Bus,
        eventChan:    make(chan interface{}, c.BufferSize),
        workerCount:  c.WorkerCount,
        store:        c.Store,
        storeTimeout: c.StoreTimeout,
    }
    
    // Register adapters for filters, mutators, handlers
    p.adapters = []pipeline.Adapter{
        &filter.IsIncidentAdapter{},
        &filter.HasMetricsAdapter{},
        &mutator.JSONAdapter{},
        &handler.LegacyAdapter{},
    }
    
    return p, nil
}
```

#### Start (Lines 130-145)
```go
func (p *Pipelined) Start() error {
    // Subscribe to processed events
    sub, err := p.bus.Subscribe(
        messaging.TopicEvent,
        "pipelined",
        p,
    )
    
    // Create worker pool
    p.createWorkers(p.workerCount, p.eventChan)
    
    return nil
}
```

#### Configuration
- **BufferSize**: Event queue size (default: 100)
- **WorkerCount**: Concurrent pipeline processors (default: 100)
- **StoreTimeout**: Store operation timeout (default: 60s)

### 2. Filters

#### Purpose
Filters determine whether an event should continue through the pipeline or be filtered out (stopped).

#### Filter Types

##### Built-in Filters

**Location**: `backend/pipeline/filter/`

| Filter | File | Purpose |
|--------|------|---------|
| **is_incident** | is_incident.go | Passes only events with non-OK status |
| **has_metrics** | has_metrics.go | Passes only events with metrics |
| **not_silenced** | not_silenced.go | Filters silenced events |
| **allow_list** | allow_list.go | Passes events matching allow list |

**Example: is_incident**
```go
// Location: backend/pipeline/filter/is_incident.go
func (i *IsIncidentAdapter) Filter(ctx context.Context, ref *corev2.ResourceReference, event *corev2.Event) (bool, error) {
    // Return false to filter out (stop), true to pass
    
    // Pass if check status is not OK (0)
    if event.Check.Status != 0 {
        return true, nil // Pass - is an incident
    }
    
    // Filter if transitioning from non-OK to OK (resolving)
    if event.Check.History != nil {
        // Check previous status
        if previousStatus != 0 {
            return true, nil // Pass - incident resolving
        }
    }
    
    return false, nil // Filter - not an incident
}
```

##### EventFilter (JavaScript)

**Purpose**: Custom filters using JavaScript expressions

**Example**:
```javascript
// Filter definition
{
    "name": "business_hours",
    "expressions": [
        "event.timestamp > 1609459200",
        "event.check.status === 2"
    ]
}
```

**Evaluation**:
- Uses Otto JavaScript VM
- Event object available in scope
- All expressions must return true to pass

#### Filter Execution

**Location**: `backend/pipeline/filter.go`

```go
func (a *AdapterV1) executeFilters(ctx context.Context, event *corev2.Event, filters []*corev2.ResourceReference) (bool, error) {
    for _, filterRef := range filters {
        // Find adapter that can handle this filter
        adapter := a.getFilterAdapter(filterRef)
        
        // Execute filter
        filtered, err := adapter.Filter(ctx, filterRef, event)
        if err != nil {
            return false, err
        }
        
        if filtered {
            // Event was filtered - stop processing
            return true, nil
        }
    }
    
    // No filters matched - continue pipeline
    return false, nil
}
```

**Filter Logic**: ALL filters must pass (AND logic)

### 3. Mutators (Transforms)

#### Purpose
Mutators transform event data before sending to handlers. They can modify, enrich, or reformat the event.

#### Mutator Types

**Location**: `backend/pipeline/mutator/`

| Mutator Type | File | Description |
|--------------|------|-------------|
| **JSON** | json.go | Pass-through (identity) - returns event as JSON |
| **Pipe** | pipe.go | External command that transforms event |
| **JavaScript** | javascript.go | JavaScript-based transformation |

##### JSON Mutator
```go
// Location: backend/pipeline/mutator/json.go
func (j *JSONAdapter) Mutate(ctx context.Context, ref *corev2.ResourceReference, event *corev2.Event, assets []asset.Asset) ([]byte, error) {
    // Simply marshal event to JSON
    eventData, err := json.Marshal(event)
    return eventData, err
}
```

##### Pipe Mutator
**Location**: `backend/pipeline/mutator/pipe.go`

**Process**:
1. Marshal event to JSON
2. Execute external command with event JSON on STDIN
3. Capture transformed data from STDOUT
4. Return transformed data

**Example**:
```go
func (p *PipeAdapter) run(ctx context.Context, mutator *corev2.Mutator, event *corev2.Event, assets []asset.Asset) ([]byte, error) {
    // Marshal event
    eventData, err := json.Marshal(event)
    
    // Create command execution
    execution := &command.Execution{
        Command: mutator.Command,
        Stdin:   string(eventData),
        Timeout: mutator.Timeout,
    }
    
    // Execute
    result := executor.Execute(ctx, execution)
    
    return []byte(result.Output), nil
}
```

##### JavaScript Mutator
**Location**: `backend/pipeline/mutator/javascript.go`

**Process**:
- Executes JavaScript code using Otto VM
- Event object available as `event`
- Script must return transformed event

**Example**:
```javascript
// Add custom field
event.custom_field = "value";
return event;
```

#### Mutator Execution

**Location**: `backend/pipeline/mutator.go`

```go
func (a *AdapterV1) executeMutator(ctx context.Context, event *corev2.Event, mutatorRef *corev2.ResourceReference) ([]byte, error) {
    // Find adapter for mutator type
    adapter := a.getMutatorAdapter(mutatorRef)
    
    // Load assets if needed
    assets := a.loadAssets(ctx, mutatorRef)
    
    // Execute mutator
    mutatedData, err := adapter.Mutate(ctx, mutatorRef, event, assets)
    
    return mutatedData, err
}
```

### 4. Handlers

#### Purpose
Handlers process events and send them to external systems (alerting, ticketing, metrics, etc.)

#### Handler Types

**Location**: `backend/pipeline/handler/`

| Handler Type | Description | Example Use Case |
|--------------|-------------|------------------|
| **Pipe** | Execute external command | Send to custom script |
| **TCP/UDP** | Send to socket | Metrics to Graphite |
| **HTTP** | POST to HTTP endpoint | Webhook, Slack, PagerDuty |
| **gRPC** | Call gRPC service | Custom gRPC integration |
| **Set** | Execute multiple handlers | Fan-out to multiple destinations |

##### Pipe Handler
**Location**: `backend/pipeline/handler/legacy.go`

```go
// Execute command with event data on STDIN
func executePipeHandler(ctx context.Context, handler *corev2.Handler, eventData []byte, assets []asset.Asset) error {
    execution := &command.Execution{
        Command: handler.Command,
        Stdin:   string(eventData),
        Timeout: handler.Timeout,
        Env:     buildEnv(handler),
    }
    
    result := executor.Execute(ctx, execution)
    
    if result.Status != 0 {
        return fmt.Errorf("handler failed with status %d", result.Status)
    }
    
    return nil
}
```

**Example**: Send to Slack
```bash
#!/bin/bash
# handler-slack.sh
# Receives event JSON on STDIN

EVENT=$(cat)
CHANNEL="#alerts"

curl -X POST "https://hooks.slack.com/services/..." \
  -d "{\"channel\": \"$CHANNEL\", \"text\": \"$EVENT\"}"
```

##### HTTP Handler
```go
// Send event data via HTTP POST
func executeHTTPHandler(ctx context.Context, handler *corev2.Handler, eventData []byte) error {
    req, err := http.NewRequest("POST", handler.URL, bytes.NewReader(eventData))
    req.Header.Set("Content-Type", "application/json")
    
    // Add custom headers
    for key, value := range handler.Headers {
        req.Header.Set(key, value)
    }
    
    client := &http.Client{Timeout: handler.Timeout}
    resp, err := client.Do(req)
    
    if resp.StatusCode >= 400 {
        return fmt.Errorf("handler returned status %d", resp.StatusCode)
    }
    
    return nil
}
```

##### Handler Set
**Purpose**: Execute multiple handlers in parallel or sequence

```go
func executeHandlerSet(ctx context.Context, set *corev2.Handler, eventData []byte) error {
    var wg sync.WaitGroup
    errors := make(chan error, len(set.Handlers))
    
    for _, handlerRef := range set.Handlers {
        wg.Add(1)
        go func(ref string) {
            defer wg.Done()
            if err := executeHandler(ctx, ref, eventData); err != nil {
                errors <- err
            }
        }(handlerRef)
    }
    
    wg.Wait()
    close(errors)
    
    // Check for errors
    for err := range errors {
        if err != nil {
            return err
        }
    }
    
    return nil
}
```

### Complete Pipeline Flow

#### Configuration Example
```yaml
---
# Check configuration
type: CheckConfig
api_version: core/v2
metadata:
  name: check-cpu
spec:
  command: check-cpu.sh
  interval: 60
  subscriptions:
    - linux
  handlers:
    - slack-critical
  pipelines:
    - type: Pipeline
      api_version: core/v2
      name: production-alerts
```

```yaml
---
# Pipeline configuration
type: Pipeline
api_version: core/v2
metadata:
  name: production-alerts
spec:
  workflows:
    - name: critical-alerts
      filters:
        - name: is_incident
          type: EventFilter
          api_version: core/v2
      mutator:
        name: add-context
        type: Mutator
        api_version: core/v2
      handler:
        name: pagerduty
        type: Handler
        api_version: core/v2
```

#### Execution Flow

```
1. Event received by Pipelined
   │
2. Load Pipeline configuration
   │
3. Execute Filters
   ├─ is_incident → PASS
   ├─ business_hours → PASS
   └─ not_silenced → PASS
   │
4. Execute Mutator (if configured)
   ├─ Load mutator: "add-context"
   ├─ Transform event data
   └─ Return mutated data
   │
5. Execute Handler
   ├─ Load handler: "pagerduty"
   ├─ Send mutated data
   └─ Log result
   │
6. Complete (success/error logged)
```

### Error Handling

**Location**: Throughout pipeline package

**Error Cases**:
1. **Filter error**: Stop pipeline, log error
2. **Mutator error**: Stop pipeline, log error
3. **Handler error**: Log error, continue (don't block other pipelines)

**Retry Logic**:
- Handlers may implement retry logic
- Exponential backoff for transient failures
- Maximum retry attempts configurable

### Metrics

**Prometheus Metrics**:

| Metric | Purpose |
|--------|---------|
| `sensu_go_pipelined_message_handler_duration{status,has_pipelines}` | Pipeline processing time |
| Handler-specific metrics | Track handler execution |

### Key Files Summary

| File | Purpose |
|------|---------|
| backend/pipelined/pipelined.go | Pipeline daemon |
| backend/pipeline/filter/is_incident.go | Incident filter |
| backend/pipeline/filter/has_metrics.go | Metrics filter |
| backend/pipeline/mutator/pipe.go | Pipe mutator |
| backend/pipeline/mutator/json.go | JSON mutator |
| backend/pipeline/handler/legacy.go | Legacy handler execution |

---

## 9. Pub-Sub Architecture - WizardBus.go

### Overview
**WizardBus** is Sensu-Go's internal message bus implementing a publish-subscribe pattern. It enables asynchronous, decoupled communication between backend daemons (agentd, eventd, schedulerd, pipelined, etc.).

### Core Concept

```
Publisher (e.g., Scheduler)
    ↓
WizardBus.Publish(topic, message)
    ↓
WizardTopic (goroutine per topic)
    ↓
Fan-out to all Subscribers
    ↓
Subscriber 1    Subscriber 2    Subscriber 3
```

### 1. WizardBus Structure

**Location**: `backend/messaging/wizard_bus.go`

#### Core Components

```go
type WizardBus struct {
    running  atomic.Value       // Bus running state
    topics   sync.Map           // Map of topic name -> *wizardTopic
    errchan  chan error         // Error channel
}

type wizardTopic struct {
    id       string                      // Topic name
    bindings map[string]Subscriber       // Consumer name -> Subscriber
    mu       sync.RWMutex                // Protects bindings
    done     chan struct{}               // Shutdown signal
}

type Subscriber interface {
    Receiver() chan<- interface{}  // Channel to receive messages
}
```

#### Initialization (Lines 62-80)

```go
func NewWizardBus(cfg WizardBusConfig, opts ...WizardOption) (*WizardBus, error) {
    bus := &WizardBus{
        errchan: make(chan error, 1),
    }
    
    // Apply functional options
    for _, opt := range opts {
        if err := opt(bus); err != nil {
            return nil, err
        }
    }
    
    return bus, nil
}
```

#### Start/Stop (Lines 81-100)

```go
func (b *WizardBus) Start() error {
    b.running.Store(true)
    return nil
}

func (b *WizardBus) Stop() error {
    b.running.Store(false)
    close(b.errchan)
    
    // Close all topics
    b.topics.Range(func(_, value interface{}) bool {
        value.(*wizardTopic).Close()
        return true
    })
    
    return nil
}
```

### 2. Topics

#### Topic Creation

**Location**: `backend/messaging/wizard_bus.go` Lines 110-120

```go
func (b *WizardBus) createTopic(topic string) *wizardTopic {
    wTopic := &wizardTopic{
        id:       topic,
        bindings: make(map[string]Subscriber),
        done:     make(chan struct{}),
    }
    return wTopic
}
```

Each topic gets its own goroutine for message distribution.

#### Topic Names and Format

**Standard Topics**:

| Topic | Constant | Purpose |
|-------|----------|---------|
| `sensu.io/keepalives` | TopicKeepalive | Agent keepalive messages |
| `sensu.io/events_raw` | TopicEventRaw | Raw events from agents |
| `sensu.io/events` | TopicEvent | Processed events for pipeline |

**Dynamic Topics** (Check Subscriptions):

Format: `sensu:check:{namespace}:{subscription}`

Examples:
- `sensu:check:default:linux`
- `sensu:check:production:webservers`
- `sensu:check:default:entity:agent1`

#### Topic Lifecycle

```
1. First Subscribe() call
   ↓
2. createTopic() if doesn't exist
   ↓
3. Start topic goroutine
   ↓
4. Topic receives messages via Send()
   ↓
5. Topic fans out to all subscribers
   ↓
6. Last Unsubscribe() call
   ↓
7. Topic goroutine continues until Stop()
```

### 3. Publishing Messages

**Location**: `backend/messaging/wizard_bus.go` Lines 160-182

```go
func (b *WizardBus) Publish(topic string, msg interface{}) error {
    // Check if bus is running
    if running := b.running.Load(); running == nil || !running.(bool) {
        return errors.New("bus no longer running")
    }
    
    // Record metrics
    genericTopic := findGenericTopic(topic)
    then := time.Now()
    defer func() {
        duration := time.Since(then)
        messagePublishedDurations.WithLabelValues(genericTopic).Observe(
            float64(duration) / float64(time.Millisecond),
        )
        messagePublishedCounter.WithLabelValues(genericTopic).Inc()
    }()
    
    // Get topic
    value, ok := b.topics.Load(topic)
    if !ok {
        // No subscribers - noop
        return nil
    }
    
    // Send to topic
    wTopic := value.(*wizardTopic)
    wTopic.Send(msg)
    
    return nil
}
```

**Important**: If no subscribers exist for a topic, Publish() is a no-op (doesn't create topic).

### 4. Subscribing to Topics

**Location**: `backend/messaging/wizard_bus.go` Lines 130-155

```go
func (b *WizardBus) Subscribe(topic string, consumer string, sub Subscriber) (Subscription, error) {
    // Check if bus is running
    if !b.running.Load().(bool) {
        return Subscription{}, errors.New("bus no longer running")
    }
    
    // Get or create topic
    var t *wizardTopic
    value, ok := b.topics.Load(topic)
    if !ok || value.(*wizardTopic).IsClosed() {
        t = b.createTopic(topic)
        b.topics.Store(topic, t)
    } else {
        t = value.(*wizardTopic)
    }
    
    // Subscribe to topic
    subscription, err := t.Subscribe(consumer, sub)
    return subscription, err
}
```

**Subscriber Interface**:

```go
type Subscriber interface {
    Receiver() chan<- interface{}
}
```

**Example Subscriber**:

```go
type MyComponent struct {
    eventChan chan interface{}
}

func (c *MyComponent) Receiver() chan<- interface{} {
    return c.eventChan
}

// Usage
bus.Subscribe("sensu.io/events", "my-component", myComponent)
```

### 5. Topic Message Distribution

**Location**: `backend/messaging/wizard_topic.go`

```go
type wizardTopic struct {
    id       string
    bindings map[string]Subscriber  // consumer name -> subscriber
    mu       sync.RWMutex
    done     chan struct{}
}

func (t *wizardTopic) Send(msg interface{}) {
    t.mu.RLock()
    defer t.mu.RUnlock()
    
    // Fan out to all subscribers
    for consumerName, subscriber := range t.bindings {
        select {
        case subscriber.Receiver() <- msg:
            // Message sent successfully
        case <-t.done:
            // Topic closing
            return
        default:
            // Subscriber channel full - log warning
            logger.WithField("consumer", consumerName).Warn("dropping message")
        }
    }
}
```

**Message Distribution**:
- Messages sent to ALL subscribers (fan-out)
- Non-blocking sends (uses select with default)
- If subscriber channel is full, message may be dropped

### 6. Subscription Management

**Location**: `backend/messaging/wizard_topic.go`

#### Subscribe
```go
func (t *wizardTopic) Subscribe(consumer string, sub Subscriber) (Subscription, error) {
    t.mu.Lock()
    defer t.mu.Unlock()
    
    // Add binding
    t.bindings[consumer] = sub
    
    // Create subscription object
    subscription := Subscription{
        topic:    t.id,
        consumer: consumer,
        cancel:   t.unsubscribe,
    }
    
    return subscription, nil
}
```

#### Unsubscribe
```go
func (t *wizardTopic) unsubscribe(consumer string) error {
    t.mu.Lock()
    defer t.mu.Unlock()
    
    // Remove binding
    delete(t.bindings, consumer)
    
    return nil
}
```

#### Subscription Object
```go
type Subscription struct {
    topic    string
    consumer string
    cancel   func(string) error
}

func (s *Subscription) Cancel() error {
    return s.cancel(s.consumer)
}
```

### 7. Message Flow Examples

#### Example 1: Check Request Flow

```go
// Scheduler publishes check request
check := &corev2.CheckRequest{...}
topic := "sensu:check:default:linux"
schedulerd.bus.Publish(topic, check)
    ↓
// WizardTopic fans out to all Linux agents
Session1.Receiver() <- check  // agent1
Session2.Receiver() <- check  // agent2
Session3.Receiver() <- check  // agent3
    ↓
// Each agent receives and processes
agent1.handleCheck(check)
agent2.handleCheck(check)
agent3.handleCheck(check)
```

#### Example 2: Event Processing Flow

```go
// Agent session publishes raw event
event := &corev2.Event{...}
session.bus.Publish(messaging.TopicEventRaw, event)
    ↓
// EventD receives
eventd.Receiver() <- event
    ↓
// EventD processes and re-publishes
processedEvent := eventd.processEvent(event)
eventd.bus.Publish(messaging.TopicEvent, processedEvent)
    ↓
// Pipelined receives
pipelined.Receiver() <- processedEvent
    ↓
// Pipelined executes handlers
pipelined.executeHandlers(processedEvent)
```

#### Example 3: Keepalive Flow

```go
// Agent sends keepalive
keepalive := &corev2.Event{
    Entity: agentEntity,
    Check:  &corev2.Check{Name: "keepalive"},
}
session.bus.Publish(messaging.TopicKeepalive, keepalive)
    ↓
// EventD receives keepalive
eventd.Receiver() <- keepalive
    ↓
// EventD updates liveness
eventd.updateLiveness(keepalive)
```

### 8. Thread Safety

**WizardBus Thread Safety**:
- Uses `sync.Map` for topics map (concurrent reads/writes)
- Uses `atomic.Value` for running state
- Each topic has its own `sync.RWMutex`

**WizardTopic Thread Safety**:
- RWMutex protects bindings map
- Read lock during message distribution
- Write lock during subscribe/unsubscribe

**Message Immutability**:
⚠️ **WARNING**: Messages should be treated as IMMUTABLE by consumers.
- Multiple goroutines receive same message reference
- Modifying messages creates data races
- Use defensive copying if modification needed

### 9. Metrics and Observability

**Prometheus Metrics**:

```go
// Message published counter
messagePublishedCounter = prometheus.NewCounterVec(
    prometheus.CounterOpts{
        Name: "sensu_go_bus_messages_published",
        Help: "Total messages published to wizard bus",
    },
    []string{"topic"},
)

// Message publish duration
messagePublishedDurations = prometheus.NewSummaryVec(
    prometheus.SummaryOpts{
        Name:       "sensu_go_bus_message_duration",
        Help:       "Message publish latency distributions",
        Objectives: map[float64]float64{0.5: 0.05, 0.9: 0.01, 0.99: 0.001},
    },
    []string{"topic"},
)
```

**Tracked Metrics**:
- Total messages published per topic
- Message publish latency per topic
- Active subscriptions (via topic counter)

### 10. Design Patterns

#### Pub-Sub Pattern
- Decouples publishers from subscribers
- Publishers don't know who receives messages
- Subscribers don't know who sends messages

#### Fan-Out Pattern
- Single message → Multiple consumers
- Each subscriber gets copy of message
- Parallel processing

#### Topic-Based Routing
- Messages routed by topic name
- Wildcard subscriptions not supported
- Subscribers choose which topics to listen to

### Key Files Summary

| File | Lines | Purpose |
|------|-------|---------|
| backend/messaging/wizard_bus.go | 1-182 | Main message bus implementation |
| backend/messaging/wizard_topic.go | 1-200 | Topic and subscription management |
| backend/messaging/message_bus.go | 1-50 | Topic constants and interfaces |

### Common Usage Patterns

#### Pattern 1: Component Initialization
```go
// Component subscribes during Start()
func (c *Component) Start() error {
    sub, err := c.bus.Subscribe(
        messaging.TopicEvent,
        "component-name",
        c, // implements Subscriber interface
    )
    c.subscription = sub
    return err
}

// Component unsubscribes during Stop()
func (c *Component) Stop() error {
    return c.subscription.Cancel()
}
```

#### Pattern 2: Message Reception
```go
// Component with receiver channel
type Component struct {
    eventChan chan interface{}
}

func (c *Component) Receiver() chan<- interface{} {
    return c.eventChan
}

// Message processing goroutine
func (c *Component) processMessages() {
    for msg := range c.eventChan {
        // Process message
        event := msg.(*corev2.Event)
        c.handleEvent(event)
    }
}
```

---

## 10. Additional Code-Level Information

### Code Organization

#### Package Structure
```
sensu-go/
├── agent/                  # Agent implementation
│   ├── agent.go           # Main agent logic
│   ├── check_handler.go   # Check execution
│   ├── event.go           # Event creation
│   └── api.go             # Agent API
│
├── backend/               # Backend implementation
│   ├── backend.go         # Backend initialization
│   ├── agentd/            # Agent connection handling
│   │   ├── agentd.go      # Agentd daemon
│   │   ├── session.go     # Agent sessions
│   │   └── middlewares.go # Auth/authz
│   │
│   ├── eventd/            # Event processing
│   │   ├── eventd.go      # Eventd daemon
│   │   ├── entity.go      # Entity management
│   │   └── silenced.go    # Silencing logic
│   │
│   ├── schedulerd/        # Check scheduling
│   │   ├── schedulerd.go  # Schedulerd daemon
│   │   ├── executor.go    # Check execution
│   │   └── check_watcher.go # Check config watching
│   │
│   ├── pipelined/         # Handler pipeline
│   │   └── pipelined.go   # Pipelined daemon
│   │
│   ├── messaging/         # Message bus
│   │   ├── wizard_bus.go  # Bus implementation
│   │   └── wizard_topic.go # Topic management
│   │
│   ├── pipeline/          # Pipeline components
│   │   ├── filter/        # Filters
│   │   ├── mutator/       # Mutators
│   │   └── handler/       # Handlers
│   │
│   ├── store/             # Storage layer
│   │   └── etcd/          # Etcd implementation
│   │
│   └── liveness/          # Liveness monitoring
│
├── transport/             # WebSocket transport
│   ├── client.go          # Client implementation
│   └── transport.go       # Transport interface
│
└── types/                 # Core data types
    ├── event.go           # Event structures
    ├── entity.go          # Entity structures
    └── check.go           # Check structures
```

### Common Go Patterns

#### 1. Daemon Interface
```go
// All daemons implement this interface
type Daemon interface {
    Start() error
    Stop() error
    Err() <-chan error
    Name() string
}
```

#### 2. Context Cancellation
```go
// Pattern for graceful shutdown
type Component struct {
    ctx    context.Context
    cancel context.CancelFunc
}

func NewComponent(ctx context.Context) *Component {
    c := &Component{}
    c.ctx, c.cancel = context.WithCancel(ctx)
    return c
}

func (c *Component) Start() error {
    go c.run()
    return nil
}

func (c *Component) run() {
    for {
        select {
        case <-c.ctx.Done():
            return
        case msg := <-c.msgChan:
            c.process(msg)
        }
    }
}

func (c *Component) Stop() error {
    c.cancel()
    return nil
}
```

#### 3. Worker Pool Pattern
```go
func (c *Component) createWorkers(count int, workChan <-chan interface{}) {
    for i := 0; i < count; i++ {
        c.wg.Add(1)
        go func() {
            defer c.wg.Done()
            for work := range workChan {
                c.processWork(work)
            }
        }()
    }
}
```

#### 4. Functional Options
```go
type Option func(*Component) error

func WithTimeout(timeout time.Duration) Option {
    return func(c *Component) error {
        c.timeout = timeout
        return nil
    }
}

func New(opts ...Option) (*Component, error) {
    c := &Component{}
    for _, opt := range opts {
        if err := opt(c); err != nil {
            return nil, err
        }
    }
    return c, nil
}
```

### Storage: Etcd

#### Key Structure
```
/sensu.io/
├── checks/{namespace}/{name}
├── entities/{namespace}/{name}
├── events/{namespace}/{entity}/{check}
├── handlers/{namespace}/{name}
├── filters/{namespace}/{name}
├── mutators/{namespace}/{name}
└── rings/{namespace}/{subscription}
```

#### Store Interface
**Location**: `backend/store/store.go`

```go
type Store interface {
    // Checks
    GetCheckConfigByName(ctx context.Context, name string) (*corev2.CheckConfig, error)
    UpdateCheckConfig(ctx context.Context, check *corev2.CheckConfig) error
    DeleteCheckConfigByName(ctx context.Context, name string) error
    
    // Events
    GetEventByEntityCheck(ctx context.Context, entity, check string) (*corev2.Event, error)
    UpdateEvent(ctx context.Context, event *corev2.Event) error
    
    // Entities
    GetEntityByName(ctx context.Context, name string) (*corev3.EntityConfig, error)
    UpdateEntity(ctx context.Context, entity *corev3.EntityConfig) error
    
    // ... more methods
}
```

#### Watchers
```go
// Watch for resource changes
watcher := store.Watch(ctx, "/sensu.io/checks/")
for event := range watcher {
    switch event.Type {
    case store.WatchCreate:
        handleCreate(event.Object)
    case store.WatchUpdate:
        handleUpdate(event.Object)
    case store.WatchDelete:
        handleDelete(event.Object)
    }
}
```

### Security

#### Authentication
**Location**: `backend/authentication/`

- **JWT Tokens**: Used for API and agent authentication
- **mTLS**: Client certificates for agents
- **Username/Password**: Basic auth for initial token exchange

#### Authorization (RBAC)
**Location**: `backend/authorization/`

**Verbs**:
- get
- list
- create
- update
- delete

**Resources**:
- checks, entities, events, handlers, filters, mutators, namespaces, users, roles, etc.

**Evaluation**:
```go
func Authorize(ctx context.Context, attrs *Attributes) (bool, error) {
    // Extract user/groups from context
    // Load roles for user/groups
    // Check if any role grants permission
    return authorized, nil
}
```

### Error Handling

#### Error Propagation
```go
// Daemon error channels
func (d *Daemon) Err() <-chan error {
    return d.errChan
}

// Backend monitors all daemon errors
for _, daemon := range b.Daemons {
    go func(d Daemon) {
        for err := range d.Err() {
            logger.WithError(err).Errorf("%s error", d.Name())
        }
    }(daemon)
}
```

#### Retry with Backoff
```go
backoff := retry.ExponentialBackoff{
    InitialDelayInterval: 10 * time.Millisecond,
    MaxDelayInterval:     10 * time.Second,
    Multiplier:           10,
}

err := backoff.Retry(func(retry int) (bool, error) {
    err := operation()
    if err == nil {
        return true, nil  // Success
    }
    if isTransient(err) {
        return false, nil // Retry
    }
    return true, err     // Fatal error
})
```

### Metrics (Prometheus)

#### Common Metric Types

**Counters**:
```go
messageCounter := prometheus.NewCounterVec(
    prometheus.CounterOpts{
        Name: "sensu_go_messages_total",
        Help: "Total messages processed",
    },
    []string{"type", "status"},
)

messageCounter.WithLabelValues("event", "success").Inc()
```

**Gauges**:
```go
activeGauge := prometheus.NewGaugeVec(
    prometheus.GaugeOpts{
        Name: "sensu_go_active_sessions",
        Help: "Number of active sessions",
    },
    []string{"namespace"},
)

activeGauge.WithLabelValues("default").Set(10)
```

**Summaries** (Latency):
```go
latencySummary := prometheus.NewSummaryVec(
    prometheus.SummaryOpts{
        Name:       "sensu_go_operation_duration",
        Help:       "Operation latency distribution",
        Objectives: map[float64]float64{0.5: 0.05, 0.9: 0.01, 0.99: 0.001},
    },
    []string{"operation"},
)

then := time.Now()
// ... operation ...
duration := time.Since(then)
latencySummary.WithLabelValues("store_get").Observe(float64(duration))
```

### Logging

**Structured Logging with Logrus**:

```go
logger.WithFields(logrus.Fields{
    "component": "agentd",
    "agent":     agentName,
    "namespace": namespace,
}).Info("agent connected")

logger.WithError(err).Error("failed to process event")
```

---

## Summary Table

| # | Topic | Key Location | Main Flow |
|---|-------|--------------|-----------|
| 1 | Agent-Backend Communication | transport/client.go, backend/agentd/ | WebSocket bidirectional messaging |
| 2 | Backend Initialization | backend/backend.go | Sequential daemon startup |
| 3 | Agent Connection Reception | backend/agentd/session.go | WebSocket upgrade → Session creation |
| 4 | Protocols | Various | WebSocket, gRPC, HTTP, Protobuf/JSON |
| 5 | Check Scheduler | backend/schedulerd/ | Etcd watch → Schedule → Publish request |
| 6 | Event Generation | agent/check_handler.go | Execute → Parse → Create event → Send |
| 7 | Event Reception | backend/eventd/ | Receive → Process → Store → Publish |
| 8 | Pipeline Processing | backend/pipelined/ | Filter → Mutate → Handle |
| 9 | WizardBus Pub-Sub | backend/messaging/ | Publish → Topic → Fan-out → Subscribers |

---

## Quick Reference

### Message Flow
```
Agent Check Execution:
Scheduler → Bus → Agent → Execute → Event → Session → Bus → EventD → Bus → Pipelined → Handler

Keepalive Flow:
Agent → Session → Bus → EventD → Liveness Update

Entity Config Update:
Watcher → Bus → Session → Agent
```

### Key Ports
- **8080**: REST API (apid)
- **8081**: Agent WebSocket (agentd)
- **2379**: Etcd client
- **3031**: Agent local API

### Important Daemons
1. **Agentd**: Agent connections
2. **EventD**: Event processing
3. **SchedulerD**: Check scheduling
4. **Pipelined**: Handler execution
5. **Keepalived**: Keepalive monitoring

