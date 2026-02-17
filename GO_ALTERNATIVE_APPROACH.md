# Building an AWS Cloud Emulator in Go

> Architectural approach for a Go-based alternative to LocalStack.

---

## Table of Contents

1. [Why Go Is a Good Fit](#1-why-go-is-a-good-fit)
2. [Strategic Positioning](#2-strategic-positioning)
3. [High-Level Architecture](#3-high-level-architecture)
4. [Module Structure](#4-module-structure)
5. [Core Subsystems — Detailed Design](#5-core-subsystems--detailed-design)
6. [Service Implementation Strategy](#6-service-implementation-strategy)
7. [AWS Protocol Engine](#7-aws-protocol-engine)
8. [State & Storage Layer](#8-state--storage-layer)
9. [Testing Strategy](#9-testing-strategy)
10. [Distribution & Packaging](#10-distribution--packaging)
11. [Implementation Roadmap](#11-implementation-roadmap)
12. [What LocalStack Gets Wrong (and How to Fix It)](#12-what-localstack-gets-wrong-and-how-to-fix-it)
13. [Risk Assessment](#13-risk-assessment)

---

## 1. Why Go Is a Good Fit

| Advantage | Why It Matters |
|---|---|
| **Single binary** | No Python, no virtualenvs, no pip — `curl \| sh` install |
| **Fast startup** | Go binaries start in <100ms vs. LocalStack's 10-30 seconds |
| **Low memory** | No GIL, no interpreter overhead — 50-100MB vs. LocalStack's 500MB-2GB |
| **Native concurrency** | Goroutines handle thousands of concurrent requests cheaply |
| **Strong typing** | AWS APIs are heavily typed — Go's type system catches errors at compile time |
| **Cross-compilation** | Build for linux/amd64, linux/arm64, darwin/arm64 from one machine |
| **No Docker required** | Can run as a plain binary — Docker becomes optional, not mandatory |
| **AWS SDK ecosystem** | `aws-sdk-go-v2` provides service models, signers, and protocol implementations |

### Where Go Has Tradeoffs

| Challenge | Mitigation |
|---|---|
| No botocore-style universal parser | Build a code generator from Smithy/botocore models |
| More boilerplate than Python | Code generation reduces this significantly |
| No Moto equivalent for fallback | Forces clean implementations — actually a feature |
| Reflection is clunky | Use generics (Go 1.18+) and code generation instead |

---

## 2. Strategic Positioning

### Differentiation from LocalStack

| Aspect | LocalStack | Your Project |
|---|---|---|
| Language | Python | Go |
| Distribution | Docker image (1-3GB) | Single binary (~50MB) + optional Docker |
| Startup time | 10-30s | <1s |
| Memory usage | 500MB-2GB | 50-200MB |
| License | BSL 1.1 (changed from Apache 2.0) | Apache 2.0 / MIT |
| Java dependencies | Required (DynamoDB, StepFunctions) | Pure Go reimplementations |
| Node.js dependencies | Required (some services) | None |
| Install | `pip install` + Docker | `brew install` / `go install` / single binary |
| Service coverage | 80+ (Pro) / 40+ (Community) | Start with 10-15 most-used, grow organically |
| AWS parity | High but inconsistent | Focus on correctness for supported operations |

### Target Users

1. **Developers** who want fast local AWS testing without Docker
2. **CI/CD pipelines** where startup speed matters (every second counts)
3. **Resource-constrained environments** (laptops, small CI runners, ARM devices)
4. **Teams burned by LocalStack's license change** who want a stable OSS option

### Initial Service Targets (MVP)

Focus on the services that cover 80% of real-world usage:

| Tier | Services | Rationale |
|---|---|---|
| **P0** | S3, SQS, DynamoDB, Lambda (invoke-only), SNS, STS | Used by nearly every AWS app |
| **P1** | Secrets Manager, SSM Parameter Store, KMS, CloudWatch Logs | Config/secrets/observability |
| **P2** | IAM, EventBridge, Step Functions, API Gateway | Orchestration and auth |
| **P3** | Kinesis, Firehose, EC2 (basic), ECS (basic) | Streaming, compute |

---

## 3. High-Level Architecture

```
                    ┌──────────────────────────────┐
                    │         CLI / Binary          │
                    │   (awslocal-go / cloudmock)   │
                    └──────────────┬───────────────┘
                                   │
                    ┌──────────────▼───────────────┐
                    │       Gateway (net/http)      │
                    │     Single port :4566         │
                    └──────────────┬───────────────┘
                                   │
                    ┌──────────────▼───────────────┐
                    │       Middleware Chain         │
                    │  ┌─────────────────────────┐  │
                    │  │ Logging                  │  │
                    │  │ Metrics                  │  │
                    │  │ CORS                     │  │
                    │  │ Auth Parser              │  │
                    │  │ Service Router           │  │
                    │  │ Protocol Decoder         │  │
                    │  │ Request Validator        │  │
                    │  └─────────────────────────┘  │
                    └──────────────┬───────────────┘
                                   │
                    ┌──────────────▼───────────────┐
                    │      Service Dispatcher       │
                    │   service + operation → fn    │
                    └──────────────┬───────────────┘
                                   │
            ┌──────────┬───────────┼───────────┬──────────┐
            ▼          ▼           ▼           ▼          ▼
        ┌──────┐  ┌──────┐   ┌──────┐   ┌──────┐   ┌──────┐
        │  S3  │  │ SQS  │   │ DDB  │   │ SNS  │   │ ...  │
        └──┬───┘  └──┬───┘   └──┬───┘   └──┬───┘   └──┬───┘
           │         │          │          │          │
           ▼         ▼          ▼          ▼          ▼
        ┌─────────────────────────────────────────────────┐
        │              State Store Layer                   │
        │  AccountRegionStore[ServiceState]                │
        │  (in-memory, optional persistence to disk)       │
        └─────────────────────────────────────────────────┘
```

### Key Architectural Decisions

| Decision | Choice | Rationale |
|---|---|---|
| HTTP framework | `net/http` (stdlib) | No need for a framework; middleware pattern is trivial in Go |
| Routing | Custom service router based on request introspection | AWS routing is not URL-based — it's header/body based |
| Serialization | Code-generated per protocol type | Botocore models define the shape; generate Go structs |
| State | In-memory with optional SQLite/bbolt persistence | Fast, embedded, no external dependencies |
| Concurrency | Per-request goroutines, `sync.RWMutex` per store | Simple, correct, scalable |
| Code gen | Custom tool reading Smithy/botocore JSON models | The single highest-leverage investment |
| Plugin system | Go plugin interface + build tags | Compile-time composition (no runtime reflection) |

---

## 4. Module Structure

```
cloudmock/                        # Project root (or whatever you name it)
├── cmd/
│   ├── cloudmock/                # Main binary
│   │   └── main.go
│   └── codegen/                  # Code generation tool
│       └── main.go
├── internal/
│   ├── gateway/                  # HTTP gateway & middleware
│   │   ├── gateway.go            # Main gateway server
│   │   ├── middleware.go         # Middleware chain
│   │   ├── cors.go               # CORS handling
│   │   └── router.go             # Service routing (header/path-based)
│   ├── protocol/                 # AWS protocol engine
│   │   ├── detect.go             # Protocol detection from request
│   │   ├── query/                # Query protocol (SQS, IAM, etc.)
│   │   │   ├── parser.go
│   │   │   └── serializer.go
│   │   ├── json/                 # JSON protocol (DynamoDB, Lambda, etc.)
│   │   │   ├── parser.go
│   │   │   └── serializer.go
│   │   ├── restxml/              # REST-XML protocol (S3, Route53, etc.)
│   │   │   ├── parser.go
│   │   │   └── serializer.go
│   │   ├── restjson/             # REST-JSON protocol (API Gateway, etc.)
│   │   │   ├── parser.go
│   │   │   └── serializer.go
│   │   └── ec2/                  # EC2 protocol variant
│   │       ├── parser.go
│   │       └── serializer.go
│   ├── auth/                     # SigV4 parsing (not validation)
│   │   ├── sigv4.go              # Parse account, region, service from auth
│   │   └── presigned.go          # Pre-signed URL handling
│   ├── services/                 # Service implementations
│   │   ├── registry.go           # Service registry
│   │   ├── s3/
│   │   │   ├── service.go        # S3 service definition + operation routing
│   │   │   ├── buckets.go        # Bucket operations
│   │   │   ├── objects.go        # Object operations
│   │   │   ├── multipart.go      # Multipart upload
│   │   │   ├── store.go          # S3 state store
│   │   │   └── types.go          # S3 types (generated)
│   │   ├── sqs/
│   │   │   ├── service.go
│   │   │   ├── queue.go          # Queue operations
│   │   │   ├── message.go        # Message operations
│   │   │   ├── store.go
│   │   │   └── types.go          # (generated)
│   │   ├── dynamodb/
│   │   │   ├── service.go
│   │   │   ├── table.go
│   │   │   ├── item.go
│   │   │   ├── query.go          # Query/Scan engine
│   │   │   ├── expression.go     # Filter/projection expressions
│   │   │   ├── store.go
│   │   │   └── types.go          # (generated)
│   │   └── .../
│   ├── store/                    # Generic state store infrastructure
│   │   ├── account_region.go     # AccountRegionStore[T]
│   │   ├── persistence.go        # Optional disk persistence
│   │   └── snapshot.go           # State snapshot/restore
│   ├── config/                   # Configuration management
│   │   └── config.go
│   └── errors/                   # AWS-compatible error types
│       ├── errors.go             # Base error types
│       └── codes.go              # Standard AWS error codes
├── pkg/                          # Public API (for embedding/library use)
│   ├── server/                   # Embeddable server for use in tests
│   │   └── server.go             # cloudmock.NewServer(services...)
│   └── client/                   # Optional: pre-configured AWS SDK client
│       └── client.go
├── codegen/                      # Code generation engine
│   ├── smithy/                   # Smithy model loader
│   │   └── loader.go
│   ├── templates/                # Go templates for codegen
│   │   ├── types.go.tmpl
│   │   ├── service.go.tmpl
│   │   └── parser.go.tmpl
│   └── generator.go
├── models/                       # AWS service model JSON files (from botocore/smithy)
│   ├── s3/
│   │   └── service-2.json
│   ├── sqs/
│   │   └── service-2.json
│   └── .../
├── testutil/                     # Test utilities
│   ├── aws.go                    # Pre-configured AWS SDK clients for tests
│   ├── fixtures.go               # Common test fixtures
│   └── snapshot.go               # Snapshot testing helpers
├── go.mod
├── go.sum
├── Makefile
├── Dockerfile                    # Small, optional Docker image
└── README.md
```

---

## 5. Core Subsystems — Detailed Design

### 5.1 Gateway

The gateway is a standard `net/http` server with a middleware chain:

```go
// internal/gateway/gateway.go

type Gateway struct {
    server   *http.Server
    router   *ServiceRouter
    services *ServiceRegistry
    config   *config.Config
}

func (g *Gateway) ServeHTTP(w http.ResponseWriter, r *http.Request) {
    ctx := NewRequestContext(r)

    // Middleware chain (executed in order)
    chain := []Middleware{
        RecoveryMiddleware,        // Panic recovery
        LoggingMiddleware,         // Request/response logging
        CORSMiddleware,            // CORS headers
        AuthParserMiddleware,      // Parse SigV4 → account, region, service
        ServiceRouterMiddleware,   // Route to service handler
    }

    handler := buildChain(chain, g.dispatchToService)
    handler(ctx, w, r)
}
```

### 5.2 Request Context

```go
// internal/gateway/context.go

type RequestContext struct {
    // Parsed from SigV4 Authorization header
    AccountID   string
    Region      string
    ServiceName string

    // Parsed by protocol decoder
    Operation   string            // e.g., "CreateQueue"
    Params      map[string]any    // Decoded request parameters

    // Metadata
    RequestID   string            // Generated UUID
    TraceID     string            // X-Amzn-Trace-Id
    Protocol    ProtocolType      // query, json, rest-xml, rest-json, ec2
}
```

### 5.3 Service Router

AWS routing is NOT URL-based. The service is determined from:

1. **Host header**: `sqs.us-east-1.localhost:4566` or `bucket.s3.localhost:4566`
2. **Authorization header**: `Credential=.../sqs/aws4_request`
3. **X-Amz-Target header**: `DynamoDB_20120810.GetItem`
4. **Path prefix**: `/@services/sqs/...` (optional explicit routing)

```go
// internal/gateway/router.go

func (r *ServiceRouter) ResolveService(req *http.Request) (string, error) {
    // 1. Check Host header for virtual-hosted style
    if service := r.fromHost(req.Host); service != "" {
        return service, nil
    }

    // 2. Check X-Amz-Target header (JSON protocol services)
    if target := req.Header.Get("X-Amz-Target"); target != "" {
        return r.fromAmzTarget(target)
    }

    // 3. Check Authorization header
    if auth := req.Header.Get("Authorization"); auth != "" {
        return r.fromAuthHeader(auth)
    }

    // 4. Check query string (Query protocol)
    if action := req.URL.Query().Get("Action"); action != "" {
        return r.fromAction(req)
    }

    return "", ErrServiceNotFound
}
```

### 5.4 Service Interface

```go
// internal/services/registry.go

// Service is the interface every AWS service implementation must satisfy.
type Service interface {
    // Name returns the AWS service name (e.g., "sqs", "s3").
    Name() string

    // Protocol returns the AWS protocol type for this service.
    Protocol() protocol.Type

    // Operations returns the set of supported operations.
    Operations() map[string]OperationHandler
}

// OperationHandler handles a single AWS operation.
type OperationHandler func(ctx *gateway.RequestContext, input any) (any, error)

// ServiceRegistry holds all registered services.
type ServiceRegistry struct {
    mu       sync.RWMutex
    services map[string]Service
}

func (r *ServiceRegistry) Register(s Service) {
    r.mu.Lock()
    defer r.mu.Unlock()
    r.services[s.Name()] = s
}

func (r *ServiceRegistry) Dispatch(ctx *gateway.RequestContext) (any, error) {
    r.mu.RLock()
    svc, ok := r.services[ctx.ServiceName]
    r.mu.RUnlock()

    if !ok {
        return nil, &errors.ServiceNotFound{Service: ctx.ServiceName}
    }

    handler, ok := svc.Operations()[ctx.Operation]
    if !ok {
        return nil, &errors.OperationNotImplemented{
            Service:   ctx.ServiceName,
            Operation: ctx.Operation,
        }
    }

    return handler(ctx, ctx.Params)
}
```

### 5.5 Example Service Implementation (SQS)

```go
// internal/services/sqs/service.go

type SQSService struct {
    stores *store.AccountRegionStore[*SQSStore]
}

func New() *SQSService {
    return &SQSService{
        stores: store.NewAccountRegionStore[*SQSStore](func() *SQSStore {
            return &SQSStore{Queues: make(map[string]*Queue)}
        }),
    }
}

func (s *SQSService) Name() string             { return "sqs" }
func (s *SQSService) Protocol() protocol.Type   { return protocol.Query }

func (s *SQSService) Operations() map[string]OperationHandler {
    return map[string]OperationHandler{
        "CreateQueue":  s.CreateQueue,
        "DeleteQueue":  s.DeleteQueue,
        "SendMessage":  s.SendMessage,
        "ReceiveMessage": s.ReceiveMessage,
        "GetQueueUrl":  s.GetQueueUrl,
        // ...
    }
}

func (s *SQSService) CreateQueue(ctx *gateway.RequestContext, input any) (any, error) {
    params := input.(*CreateQueueInput)  // generated type
    st := s.stores.Get(ctx.AccountID, ctx.Region)

    st.mu.Lock()
    defer st.mu.Unlock()

    if _, exists := st.Queues[params.QueueName]; exists {
        return nil, &errors.AWSError{
            Code:    "QueueAlreadyExists",
            Message: "A queue with this name already exists.",
            Status:  400,
        }
    }

    queue := NewQueue(ctx.AccountID, ctx.Region, params.QueueName, params.Attributes)
    st.Queues[params.QueueName] = queue

    return &CreateQueueOutput{
        QueueUrl: queue.URL(),
    }, nil
}
```

---

## 6. Service Implementation Strategy

### 6.1 Code Generation (the highest-leverage investment)

The single most important piece of infrastructure is a **code generator** that reads AWS service
model JSON files (from botocore or Smithy) and produces:

1. **Go types** for every input/output shape (structs with JSON/XML tags)
2. **Protocol parsers** that decode HTTP requests into typed Go structs
3. **Protocol serializers** that encode Go structs into HTTP responses
4. **Service interface stubs** with all operations as method signatures
5. **Operation routing tables** mapping operation names to handler functions

```
models/sqs/service-2.json ──► codegen ──► internal/services/sqs/types_gen.go
                                     ──► internal/services/sqs/parser_gen.go
                                     ──► internal/services/sqs/serializer_gen.go
                                     ──► internal/services/sqs/interface_gen.go
```

**This eliminates 60-70% of the work** for each new service. The developer only writes the
business logic — all protocol handling is generated.

### 6.2 Where to Get AWS Service Models

| Source | Format | Completeness |
|---|---|---|
| `botocore/data/` (pip install botocore) | JSON service-2.json | Complete, well-tested |
| `aws-sdk-go-v2/codegen/sdk-codegen/aws-models/` | Smithy JSON AST | Complete, Go-native |
| AWS Smithy GitHub | Smithy IDL | Authoritative but complex |

**Recommendation**: Use **botocore's `service-2.json`** files. They are the most battle-tested,
contain all the protocol metadata you need, and are used by LocalStack itself. You can vendor
them or download them at build time.

### 6.3 Per-Service Complexity Tiers

| Tier | Services | Estimated Effort | Notes |
|---|---|---|---|
| **Simple** | STS, Secrets Manager, SSM, CloudWatch Logs | 1-2 weeks each | CRUD operations, simple state |
| **Medium** | SQS, SNS, KMS, EventBridge | 2-4 weeks each | Message passing, pub/sub patterns |
| **Complex** | S3, DynamoDB | 1-3 months each | Rich query engines, streaming, multipart |
| **Very Complex** | Lambda, Step Functions, API Gateway, CloudFormation | 3-6 months each | Runtime execution, state machines |

### 6.4 DynamoDB Implementation Notes

DynamoDB is one of the hardest services to implement correctly. Key challenges:

- **Expression engine**: `FilterExpression`, `ProjectionExpression`, `ConditionExpression`, `KeyConditionExpression`
- **Type system**: DynamoDB has its own type system (`S`, `N`, `B`, `SS`, `NS`, `BOOL`, `NULL`, `L`, `M`)
- **Query planner**: Efficient scanning with filters
- **Secondary indexes**: GSI and LSI with eventual consistency semantics
- **Streams**: DynamoDB Streams (change data capture)
- **Transactions**: `TransactWriteItems`, `TransactGetItems`

**Approach**: Build a proper expression parser using a recursive descent parser or PEG grammar.
Store items as native Go maps with DynamoDB-typed values. Use sorted maps (or B-trees) for range
key queries.

Consider using **SQLite** as the backing store for DynamoDB — it gives you indexing, transactions,
and efficient range queries for free.

### 6.5 S3 Implementation Notes

S3 is the other very complex service:

- **Object storage**: Byte-range reads, multipart uploads, ETags, content types
- **Bucket operations**: Versioning, lifecycle, replication configuration, notifications
- **Auth**: Virtual-hosted style (`bucket.s3.amazonaws.com`), path style, pre-signed URLs
- **Streaming**: `Transfer-Encoding: chunked`, `aws-chunked` encoding

**Approach**: Use the filesystem for object storage (one file per object, metadata in a sidecar
JSON file or embedded SQLite). This gives you large-object support without memory pressure.

---

## 7. AWS Protocol Engine

### 7.1 Protocol Detection

```go
type ProtocolType int

const (
    ProtocolQuery   ProtocolType = iota  // SQS, SNS, IAM, STS, CloudFormation
    ProtocolJSON10                        // DynamoDB
    ProtocolJSON11                        // Lambda, KMS, Logs
    ProtocolRestXML                       // S3, CloudFront, Route53
    ProtocolRestJSON                      // API Gateway, EventBridge
    ProtocolEC2                           // EC2
)
```

The protocol type is **determined by the service model**, not by inspecting the request.
Each service's `service-2.json` has a `"protocol"` field.

### 7.2 Protocol Parser Interface

```go
type RequestParser interface {
    // Parse decodes an HTTP request into an operation name and typed input struct.
    Parse(r *http.Request, service *ServiceModel) (operation string, input any, err error)
}
```

### 7.3 Protocol Serializer Interface

```go
type ResponseSerializer interface {
    // Serialize encodes a service response into an HTTP response.
    Serialize(w http.ResponseWriter, operation string, output any, meta ResponseMeta) error

    // SerializeError encodes a service error into an HTTP error response.
    SerializeError(w http.ResponseWriter, operation string, err *AWSError) error
}
```

### 7.4 Query Protocol Example

```
POST / HTTP/1.1
Content-Type: application/x-www-form-urlencoded

Action=CreateQueue&QueueName=my-queue&Attribute.1.Name=VisibilityTimeout&Attribute.1.Value=30
```

Parser extracts `Action` → operation name, then maps form fields to the operation's input shape.

### 7.5 JSON Protocol Example

```
POST / HTTP/1.1
X-Amz-Target: DynamoDB_20120810.GetItem
Content-Type: application/x-amz-json-1.0

{"TableName": "MyTable", "Key": {"pk": {"S": "123"}}}
```

Parser extracts `X-Amz-Target` → operation name, then unmarshals JSON body into the input shape.

---

## 8. State & Storage Layer

### 8.1 AccountRegionStore (Generic)

```go
// internal/store/account_region.go

// AccountRegionStore provides account-and-region-scoped state storage.
// It is the Go equivalent of LocalStack's AccountRegionBundle.
type AccountRegionStore[T any] struct {
    mu      sync.RWMutex
    stores  map[string]map[string]T   // account_id → region → T
    factory func() T                   // creates a new empty store
}

func NewAccountRegionStore[T any](factory func() T) *AccountRegionStore[T] {
    return &AccountRegionStore[T]{
        stores:  make(map[string]map[string]T),
        factory: factory,
    }
}

func (s *AccountRegionStore[T]) Get(accountID, region string) T {
    s.mu.RLock()
    if regions, ok := s.stores[accountID]; ok {
        if store, ok := regions[region]; ok {
            s.mu.RUnlock()
            return store
        }
    }
    s.mu.RUnlock()

    // Upgrade to write lock for initialization
    s.mu.Lock()
    defer s.mu.Unlock()

    if _, ok := s.stores[accountID]; !ok {
        s.stores[accountID] = make(map[string]T)
    }
    if _, ok := s.stores[accountID][region]; !ok {
        s.stores[accountID][region] = s.factory()
    }

    return s.stores[accountID][region]
}

// Reset clears all state (useful for testing).
func (s *AccountRegionStore[T]) Reset() {
    s.mu.Lock()
    defer s.mu.Unlock()
    s.stores = make(map[string]map[string]T)
}
```

### 8.2 Per-Service Store Example

```go
// internal/services/sqs/store.go

type SQSStore struct {
    mu     sync.RWMutex
    Queues map[string]*Queue
}

type Queue struct {
    Name       string
    URL        string
    ARN        string
    Attributes QueueAttributes
    Messages   []*Message      // Simple slice; use a proper queue data structure in production
    Created    time.Time
}
```

### 8.3 Optional Persistence

For users who want state to survive restarts:

```go
// internal/store/persistence.go

type PersistenceBackend interface {
    Save(key string, data []byte) error
    Load(key string) ([]byte, error)
    Delete(key string) error
    List(prefix string) ([]string, error)
}

// Implementations:
// - FilesystemBackend (JSON files on disk)
// - SQLiteBackend (embedded SQLite)
// - BoltBackend (bbolt key-value store)
```

---

## 9. Testing Strategy

### 9.1 Three-Layer Testing

```
Layer 1: Unit Tests
  └─ Test individual operations in isolation
  └─ No HTTP server needed — call handlers directly
  └─ Fast, run in <1s

Layer 2: Integration Tests
  └─ Start the full server, use real AWS SDK client
  └─ Test full request lifecycle (HTTP → parse → handle → serialize)
  └─ Run in <10s

Layer 3: Parity Tests (the differentiator)
  └─ Run the same test against both your emulator AND real AWS
  └─ Compare responses (snapshot testing)
  └─ This is how you guarantee correctness
```

### 9.2 Embeddable Server for Testing

This is a **massive advantage** over LocalStack — users can embed the emulator directly in their
Go tests without starting a separate process:

```go
// pkg/server/server.go

func NewServer(services ...services.Service) *Server { ... }
func (s *Server) Start() (endpoint string, cleanup func()) { ... }

// Usage in user's tests:
func TestMyApp(t *testing.T) {
    endpoint, cleanup := cloudmock.NewServer(
        sqs.New(),
        s3.New(),
    ).Start()
    defer cleanup()

    // Use endpoint with AWS SDK
    cfg, _ := awsconfig.LoadDefaultConfig(context.Background(),
        awsconfig.WithEndpointResolverWithOptions(/* endpoint */),
    )
    client := s3client.NewFromConfig(cfg)
    // ... test your application code ...
}
```

### 9.3 Parity Test Framework

```go
// testutil/parity.go

type ParityTest struct {
    Name string
    Run  func(t *testing.T, client *aws.Client)
}

// RunParity runs a test against both the emulator and (optionally) real AWS,
// then compares the results.
func RunParity(t *testing.T, tests []ParityTest) {
    for _, tt := range tests {
        t.Run(tt.Name+"/emulator", func(t *testing.T) {
            client := newEmulatorClient(t)
            tt.Run(t, client)
        })

        if os.Getenv("TEST_AWS_REAL") == "1" {
            t.Run(tt.Name+"/aws", func(t *testing.T) {
                client := newRealAWSClient(t)
                tt.Run(t, client)
            })
        }
    }
}
```

---

## 10. Distribution & Packaging

### 10.1 Distribution Channels

| Channel | Command | Notes |
|---|---|---|
| **Go install** | `go install github.com/you/cloudmock/cmd/cloudmock@latest` | Go developers |
| **Homebrew** | `brew install cloudmock` | macOS/Linux |
| **Binary releases** | GitHub Releases (goreleaser) | All platforms |
| **Docker** (optional) | `docker run cloudmock/cloudmock` | ~20MB image (scratch + binary) |
| **npm** (wrapper) | `npx cloudmock` | JS ecosystem compatibility |

### 10.2 Docker Image (if desired)

```dockerfile
FROM scratch
COPY cloudmock /cloudmock
ENTRYPOINT ["/cloudmock"]
```

Total image size: **~20-30MB** (vs. LocalStack's 1-3GB).

### 10.3 goreleaser Configuration

Use goreleaser for cross-platform builds:

- `linux/amd64`, `linux/arm64`
- `darwin/amd64`, `darwin/arm64`
- `windows/amd64`

---

## 11. Implementation Roadmap

### Phase 1: Foundation (Months 1-2)

**Goal**: Working gateway that can handle a single service end-to-end.

| Week | Deliverable |
|---|---|
| 1-2 | Code generator: Read botocore `service-2.json` → Go types |
| 3-4 | Protocol engine: Query protocol parser + serializer |
| 5 | Gateway: HTTP server, auth parser, service routing |
| 6 | STS service: `GetCallerIdentity`, `AssumeRole` (simplest service) |
| 7 | SQS service: `CreateQueue`, `SendMessage`, `ReceiveMessage`, `DeleteQueue` |
| 8 | Testing infrastructure: Embeddable server, parity test framework |

**Milestone**: `aws sqs create-queue --endpoint-url http://localhost:4566` works.

### Phase 2: Core Services (Months 3-5)

| Month | Deliverable |
|---|---|
| 3 | S3: Bucket CRUD, object CRUD, multipart upload, virtual-hosted routing |
| 4 | DynamoDB: Table CRUD, `PutItem`, `GetItem`, `Query`, `Scan`, expression engine |
| 5 | SNS + Secrets Manager + SSM Parameter Store + KMS |

**Milestone**: A typical web application's integration tests pass.

### Phase 3: Ecosystem (Months 6-8)

| Month | Deliverable |
|---|---|
| 6 | CloudWatch Logs, EventBridge, IAM (basic) |
| 7 | Lambda invoke (execute pre-built containers or binaries) |
| 8 | Persistence, snapshot/restore, CLI polish, docs, first public release |

**Milestone**: Public v0.1.0 release.

### Phase 4: Growth (Months 9+)

- Step Functions (state machine interpreter)
- API Gateway (HTTP proxy with transformations)
- CloudFormation (resource orchestrator) — consider this carefully, it's a massive undertaking
- Community contributions for additional services

---

## 12. What LocalStack Gets Wrong (and How to Fix It)

These are the pain points that create an opportunity for an alternative:

### 12.1 Startup Time

**Problem**: LocalStack takes 10-30 seconds to start, even with lazy loading.
**Root Cause**: Python interpreter startup, plugin scanning, dependency imports.
**Fix**: Go binary starts in <100ms. No plugin scanning needed — services are compiled in.

### 12.2 Memory Usage

**Problem**: LocalStack uses 500MB-2GB RAM, making it painful on laptops and CI.
**Root Cause**: Python runtime, Moto, Java processes (DynamoDB Local), Node.js.
**Fix**: Pure Go implementations. 50-200MB total. No JVM, no Node.

### 12.3 Docker Dependency

**Problem**: LocalStack practically requires Docker to run.
**Root Cause**: Complex dependency tree (Python + Java + Node.js + system libs).
**Fix**: Single binary. Docker is optional, not required.

### 12.4 Inconsistent Parity

**Problem**: Some services are well-emulated, others are barely stubs.
**Root Cause**: Moto fallback masks gaps; no systematic parity testing against real AWS.
**Fix**: Every operation is either properly implemented or explicitly returns "not implemented" with a clear error. No silent fallback to broken behavior. Mandatory parity tests.

### 12.5 Configuration Complexity

**Problem**: 100+ environment variables, many undocumented or with subtle interactions.
**Root Cause**: Organic growth over years, legacy compatibility.
**Fix**: Minimal, well-documented configuration. Sensible defaults. Config file support from day one.

### 12.6 Flaky Service Interactions

**Problem**: Cross-service integrations (S3 notifications → SQS, Lambda triggers) are unreliable.
**Root Cause**: Services are isolated plugins that communicate via internal HTTP calls.
**Fix**: Services share a process and can call each other directly via Go interfaces. Event bus for cross-service notifications.

### 12.7 License

**Problem**: LocalStack moved from Apache 2.0 to BSL 1.1 — cannot be used in competing products.
**Root Cause**: Business decision.
**Fix**: Apache 2.0 or MIT from day one. Commit to it publicly. This alone drives adoption.

---

## 13. Risk Assessment

### High Risk

| Risk | Impact | Mitigation |
|---|---|---|
| DynamoDB expression engine is hard | Blocks a P0 service | Start early; consider using sqlite as backend |
| S3 compatibility surface is huge | Endless edge cases | Focus on top 20 operations; add as users report gaps |
| Keeping up with AWS API changes | Drift over time | Automate: nightly CI job compares botocore models |

### Medium Risk

| Risk | Impact | Mitigation |
|---|---|---|
| Code generator bugs | Incorrect types/parsers | Extensive parity tests catch these |
| Community adoption | Project dies without users | Ship fast, solve real pain (startup time), great docs |
| Scope creep | Never ship | Strict MVP: 6 services, then release |

### Low Risk

| Risk | Impact | Mitigation |
|---|---|---|
| Go performance | Go is fast by default | Not a concern — even naive Go will outperform Python |
| Cross-platform | Go cross-compiles trivially | Use goreleaser from day one |

---

## Summary: The Minimum Viable Stack

To ship a usable v0.1.0, you need exactly these pieces:

```
1. Code Generator          → reads botocore JSON, outputs Go types + protocol code
2. Protocol Engine          → query + json parsers/serializers (covers 80% of services)
3. Gateway                  → net/http + auth parser + service router
4. AccountRegionStore[T]    → generic state store with account/region scoping
5. 6 Services               → STS, SQS, S3, DynamoDB, SNS, Secrets Manager
6. Embeddable Test Server   → cloudmock.NewServer().Start() for Go tests
7. CLI                      → cloudmock start / cloudmock stop
8. Parity Tests             → prove you match real AWS behavior
```

Everything else is nice to have. Ship this, get users, iterate.
