# 🔄 Request Flow

Deep dive into Bifrost's request processing pipeline - from transport layer ingestion through provider execution to response delivery.

---

## 📋 Processing Pipeline Overview

```mermaid
flowchart TD
    Client[Client Request] --> Transport{Transport Layer}
    Transport -->|HTTP| HTTP[HTTP Transport]
    Transport -->|SDK| SDK[Go SDK]

    HTTP --> Parse[Request Parsing]
    SDK --> Parse

    Parse --> Validate[Request Validation]
    Validate --> Route[Request Routing]

    Route --> PrePlugin[Pre-Processing Plugins]
    PrePlugin --> MCPDiscover[MCP Tool Discovery]
    MCPDiscover --> MemoryPool[Memory Pool Acquisition]

    MemoryPool --> KeySelect[API Key Selection]
    KeySelect --> Queue[Provider Queue]
    Queue --> Worker[Worker Assignment]

    Worker --> ProviderCall[Provider API Call]
    ProviderCall --> MCPExec[MCP Tool Execution]
    MCPExec --> PostPlugin[Post-Processing Plugins]

    PostPlugin --> Response[Response Formation]
    Response --> MemoryReturn[Memory Pool Return]
    MemoryReturn --> ClientResponse[Client Response]
```

---

## 🚪 Stage 1: Transport Layer Processing

### **HTTP Transport Flow**

```mermaid
sequenceDiagram
    participant Client
    participant HTTPTransport
    participant Router
    participant Validation

    Client->>HTTPTransport: POST /v1/chat/completions
    HTTPTransport->>HTTPTransport: Parse Headers
    HTTPTransport->>HTTPTransport: Extract Body
    HTTPTransport->>Validation: Validate JSON Schema
    Validation->>Router: BifrostRequest
    Router-->>HTTPTransport: Processing Started
    HTTPTransport-->>Client: HTTP 200 (async processing)
```

**Key Processing Steps:**

1. **Request Reception** - FastHTTP server receives request
2. **Header Processing** - Extract authentication, content-type, custom headers
3. **Body Parsing** - JSON unmarshaling with schema validation
4. **Request Transformation** - Convert to internal `BifrostRequest` schema
5. **Context Creation** - Build request context with metadata

**Performance Characteristics:**

- **Parsing Time:** ~2.1μs for typical requests
- **Validation Overhead:** ~400ns for schema checks
- **Memory Allocation:** Zero-copy where possible

### **Go SDK Flow**

```mermaid
sequenceDiagram
    participant Application
    participant SDK
    participant Core
    participant Validation

    Application->>SDK: bifrost.ChatCompletion(req)
    SDK->>SDK: Type Validation
    SDK->>Core: Direct Function Call
    Core->>Validation: Schema Validation
    Validation-->>Core: Validated Request
    Core-->>SDK: Processing Result
    SDK-->>Application: Typed Response
```

**Advantages:**

- **Zero Serialization** - Direct Go struct passing
- **Type Safety** - Compile-time validation
- **Lower Latency** - No HTTP/JSON overhead
- **Memory Efficiency** - No intermediate allocations

---

## 🎯 Stage 2: Request Routing & Load Balancing

### **Provider Selection Logic**

```mermaid
flowchart TD
    Request[Incoming Request] --> ModelCheck{Model Available?}
    ModelCheck -->|Yes| ProviderDirect[Use Specified Provider]
    ModelCheck -->|No| ModelMapping[Model → Provider Mapping]

    ProviderDirect --> KeyPool[API Key Pool]
    ModelMapping --> KeyPool

    KeyPool --> WeightedSelect[Weighted Random Selection]
    WeightedSelect --> HealthCheck{Provider Healthy?}

    HealthCheck -->|Yes| AssignWorker[Assign Worker]
    HealthCheck -->|No| CircuitBreaker[Circuit Breaker]

    CircuitBreaker --> FallbackCheck{Fallback Available?}
    FallbackCheck -->|Yes| FallbackProvider[Try Fallback]
    FallbackCheck -->|No| ErrorResponse[Return Error]

    FallbackProvider --> KeyPool
```

**Key Selection Algorithm:**

```go
// Weighted random key selection
type KeySelector struct {
    keys    []APIKey
    weights []float64
    total   float64
}

func (ks *KeySelector) SelectKey() *APIKey {
    r := rand.Float64() * ks.total
    cumulative := 0.0

    for i, weight := range ks.weights {
        cumulative += weight
        if r <= cumulative {
            return &ks.keys[i]
        }
    }
    return &ks.keys[len(ks.keys)-1]
}
```

**Performance Metrics:**

- **Key Selection Time:** ~10ns (constant time)
- **Health Check Overhead:** ~50ns (cached results)
- **Fallback Decision:** ~25ns (configuration lookup)

---

## 🔌 Stage 3: Plugin Pipeline Processing

### **Pre-Processing Hooks**

```mermaid
sequenceDiagram
    participant Request
    participant AuthPlugin
    participant RateLimitPlugin
    participant TransformPlugin
    participant Core

    Request->>AuthPlugin: ProcessRequest()
    AuthPlugin->>AuthPlugin: Validate API Key
    AuthPlugin->>RateLimitPlugin: Authorized Request

    RateLimitPlugin->>RateLimitPlugin: Check Rate Limits
    RateLimitPlugin->>TransformPlugin: Allowed Request

    TransformPlugin->>TransformPlugin: Modify Request
    TransformPlugin->>Core: Final Request
```

**Plugin Execution Model:**

```go
type PluginManager struct {
    plugins []Plugin
}

func (pm *PluginManager) ExecutePreHooks(
    ctx BifrostContext,
    req *BifrostRequest,
) (*BifrostRequest, *BifrostError) {
    for _, plugin := range pm.plugins {
        modifiedReq, err := plugin.ProcessRequest(ctx, req)
        if err != nil {
            return nil, err
        }
        req = modifiedReq
    }
    return req, nil
}
```

**Plugin Types & Performance:**

| Plugin Type           | Processing Time | Memory Impact | Failure Mode           |
| --------------------- | --------------- | ------------- | ---------------------- |
| **Authentication**    | ~1-5μs          | Minimal       | Reject request         |
| **Rate Limiting**     | ~500ns          | Cache-based   | Throttle/reject        |
| **Request Transform** | ~2-10μs         | Copy-on-write | Continue with original |
| **Monitoring**        | ~200ns          | Append-only   | Continue silently      |

---

## 🛠️ Stage 4: MCP Tool Discovery & Integration

### **Tool Discovery Process**

```mermaid
flowchart TD
    Request[Request with Model] --> MCPCheck{MCP Enabled?}
    MCPCheck -->|No| SkipMCP[Skip MCP Processing]
    MCPCheck -->|Yes| ClientLookup[MCP Client Lookup]

    ClientLookup --> ToolFilter[Tool Filtering]
    ToolFilter --> ToolInject[Inject Tools into Request]

    ToolFilter --> IncludeCheck{Include Filter?}
    ToolFilter --> ExcludeCheck{Exclude Filter?}

    IncludeCheck -->|Yes| IncludeTools[Include Specified Tools]
    IncludeCheck -->|No| AllTools[Include All Tools]

    ExcludeCheck -->|Yes| RemoveTools[Remove Excluded Tools]
    ExcludeCheck -->|No| KeepFiltered[Keep Filtered Tools]

    IncludeTools --> ToolInject
    AllTools --> ToolInject
    RemoveTools --> ToolInject
    KeepFiltered --> ToolInject

    ToolInject --> EnhancedRequest[Request with Tools]
    SkipMCP --> EnhancedRequest
```

**Tool Integration Algorithm:**

```go
func (mcpm *MCPManager) EnhanceRequest(
    ctx BifrostContext,
    req *BifrostRequest,
) (*BifrostRequest, error) {
    // Extract tool filtering from context
    includeClients := ctx.GetStringSlice("mcp-include-clients")
    excludeClients := ctx.GetStringSlice("mcp-exclude-clients")
    includeTools := ctx.GetStringSlice("mcp-include-tools")
    excludeTools := ctx.GetStringSlice("mcp-exclude-tools")

    // Get available tools
    availableTools := mcpm.getAvailableTools(includeClients, excludeClients)

    // Filter tools
    filteredTools := mcpm.filterTools(availableTools, includeTools, excludeTools)

    // Inject into request
    if req.Params == nil {
        req.Params = &ModelParameters{}
    }
    req.Params.Tools = append(req.Params.Tools, filteredTools...)

    return req, nil
}
```

**MCP Performance Impact:**

- **Tool Discovery:** ~100-500μs (cached after first request)
- **Tool Filtering:** ~50-200ns per tool
- **Request Enhancement:** ~1-5μs depending on tool count

---

## 💾 Stage 5: Memory Pool Management

### **Object Pool Lifecycle**

```mermaid
stateDiagram-v2
    [*] --> PoolInit: System Startup
    PoolInit --> Available: Objects Pre-allocated

    Available --> Acquired: Request Processing
    Acquired --> InUse: Object Populated
    InUse --> Processing: Worker Processing
    Processing --> Completed: Processing Done
    Completed --> Reset: Object Cleanup
    Reset --> Available: Return to Pool

    Available --> Expansion: Pool Exhaustion
    Expansion --> Available: New Objects Created

    Reset --> GC: Pool Full
    GC --> [*]: Garbage Collection
```

**Memory Pool Implementation:**

```go
type MemoryPools struct {
    channelPool  sync.Pool
    messagePool  sync.Pool
    responsePool sync.Pool
    bufferPool   sync.Pool
}

func (mp *MemoryPools) GetChannel() *ProcessingChannel {
    if ch := mp.channelPool.Get(); ch != nil {
        return ch.(*ProcessingChannel)
    }
    return NewProcessingChannel()
}

func (mp *MemoryPools) ReturnChannel(ch *ProcessingChannel) {
    ch.Reset() // Clear previous data
    mp.channelPool.Put(ch)
}
```

---

## ⚙️ Stage 6: Worker Pool Processing

### **Worker Assignment & Execution**

```mermaid
sequenceDiagram
    participant Queue
    participant WorkerPool
    participant Worker
    participant Provider
    participant Circuit

    Queue->>WorkerPool: Enqueue Request
    WorkerPool->>Worker: Assign Available Worker
    Worker->>Circuit: Check Circuit Breaker
    Circuit->>Provider: Forward Request

    Provider-->>Circuit: Response/Error
    Circuit->>Circuit: Update Health Metrics
    Circuit-->>Worker: Provider Response
    Worker-->>WorkerPool: Release Worker
    WorkerPool-->>Queue: Request Completed
```

**Worker Pool Architecture:**

```go
type ProviderWorkerPool struct {
    workers    chan *Worker
    queue      chan *ProcessingJob
    config     WorkerPoolConfig
    metrics    *PoolMetrics
}

func (pwp *ProviderWorkerPool) ProcessRequest(job *ProcessingJob) {
    // Get worker from pool
    worker := <-pwp.workers

    go func() {
        defer func() {
            // Return worker to pool
            pwp.workers <- worker
        }()

        // Process request
        result := worker.Execute(job)
        job.ResultChan <- result
    }()
}
```

---

## 🌐 Stage 7: Provider API Communication

### **HTTP Request Execution**

```mermaid
sequenceDiagram
    participant Worker
    participant HTTPClient
    participant Provider
    participant CircuitBreaker
    participant Metrics

    Worker->>HTTPClient: PrepareRequest()
    HTTPClient->>HTTPClient: Add Headers & Auth
    HTTPClient->>CircuitBreaker: CheckHealth()
    CircuitBreaker->>Provider: HTTP Request

    Provider-->>CircuitBreaker: HTTP Response
    CircuitBreaker->>Metrics: Record Metrics
    CircuitBreaker-->>HTTPClient: Response/Error
    HTTPClient-->>Worker: Parsed Response
```

**Request Preparation Pipeline:**

```go
func (w *ProviderWorker) ExecuteRequest(job *ProcessingJob) *ProviderResponse {
    // Prepare HTTP request
    httpReq := w.prepareHTTPRequest(job.Request)

    // Add authentication
    w.addAuthentication(httpReq, job.APIKey)

    // Execute with timeout
    ctx, cancel := context.WithTimeout(context.Background(), job.Timeout)
    defer cancel()

    httpResp, err := w.httpClient.Do(httpReq.WithContext(ctx))
    if err != nil {
        return w.handleError(err, job)
    }

    // Parse response
    return w.parseResponse(httpResp, job)
}
```

---

## 🔄 Stage 8: Tool Execution & Response Processing

### **MCP Tool Execution Flow**

```mermaid
sequenceDiagram
    participant Provider
    participant MCPProcessor
    participant MCPServer
    participant ToolExecutor
    participant ResponseBuilder

    Provider->>MCPProcessor: Response with Tool Calls
    MCPProcessor->>MCPProcessor: Extract Tool Calls

    loop For each tool call
        MCPProcessor->>MCPServer: Execute Tool
        MCPServer->>ToolExecutor: Tool Invocation
        ToolExecutor-->>MCPServer: Tool Result
        MCPServer-->>MCPProcessor: Tool Response
    end

    MCPProcessor->>ResponseBuilder: Combine Results
    ResponseBuilder-->>Provider: Enhanced Response
```

**Tool Execution Pipeline:**

```go
func (mcp *MCPProcessor) ProcessToolCalls(
    response *ProviderResponse,
) (*ProviderResponse, error) {
    toolCalls := mcp.extractToolCalls(response)
    if len(toolCalls) == 0 {
        return response, nil
    }

    // Execute tools concurrently
    results := make(chan ToolResult, len(toolCalls))
    for _, toolCall := range toolCalls {
        go func(tc ToolCall) {
            result := mcp.executeTool(tc)
            results <- result
        }(toolCall)
    }

    // Collect results
    toolResults := make([]ToolResult, 0, len(toolCalls))
    for i := 0; i < len(toolCalls); i++ {
        toolResults = append(toolResults, <-results)
    }

    // Enhance response
    return mcp.enhanceResponse(response, toolResults), nil
}
```

---

## 📤 Stage 9: Post-Processing & Response Formation

### **Plugin Post-Processing**

```mermaid
sequenceDiagram
    participant CoreResponse
    participant LoggingPlugin
    participant CachePlugin
    participant MetricsPlugin
    participant Transport

    CoreResponse->>LoggingPlugin: ProcessResponse()
    LoggingPlugin->>LoggingPlugin: Log Request/Response
    LoggingPlugin->>CachePlugin: Response + Logs

    CachePlugin->>CachePlugin: Cache Response
    CachePlugin->>MetricsPlugin: Cached Response

    MetricsPlugin->>MetricsPlugin: Record Metrics
    MetricsPlugin->>Transport: Final Response
```

**Response Enhancement Pipeline:**

```go
func (pm *PluginManager) ExecutePostHooks(
    ctx BifrostContext,
    req *BifrostRequest,
    resp *BifrostResponse,
) (*BifrostResponse, error) {
    for _, plugin := range pm.plugins {
        enhancedResp, err := plugin.ProcessResponse(ctx, req, resp)
        if err != nil {
            // Log error but continue processing
            pm.logger.Warn("Plugin post-processing error", "plugin", plugin.Name(), "error", err)
            continue
        }
        resp = enhancedResp
    }
    return resp, nil
}
```

### **Response Serialization**

```mermaid
flowchart TD
    Response[BifrostResponse] --> Format{Response Format}
    Format -->|HTTP| JSONSerialize[JSON Serialization]
    Format -->|SDK| DirectReturn[Direct Go Struct]

    JSONSerialize --> Compress[Compression]
    DirectReturn --> TypeCheck[Type Validation]

    Compress --> Headers[Set Headers]
    TypeCheck --> Return[Return Response]

    Headers --> HTTPResponse[HTTP Response]
    HTTPResponse --> Client[Client Response]
    Return --> Client
```

---

## 🔗 Related Architecture Documentation

- **[🌐 System Overview](./system-overview.md)** - High-level architecture components
- **[⚙️ Concurrency Model](./concurrency.md)** - Worker pools and threading details
- **[🔌 Plugin System](./plugins.md)** - Plugin execution and lifecycle
- **[🛠️ MCP System](./mcp.md)** - Tool discovery and execution internals
- **[📊 Benchmarks](../benchmarks.md)** - Detailed performance analysis
- **[💡 Design Decisions](./design-decisions.md)** - Why this flow was chosen

---

**🎯 Next Step:** Deep dive into the concurrency model in **[Concurrency](./concurrency.md)**.
