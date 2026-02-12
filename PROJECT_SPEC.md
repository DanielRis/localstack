# LocalStack Project Specification

> A comprehensive technical specification of the LocalStack project — its architecture, design
> patterns, module organization, and operational guidelines. This document is intended as a
> reference for building a similar cloud-service emulation platform from scratch.

---

## Table of Contents

1. [Project Overview](#1-project-overview)
2. [Technology Stack](#2-technology-stack)
3. [Repository Structure](#3-repository-structure)
4. [Core Architecture](#4-core-architecture)
5. [HTTP Request Lifecycle](#5-http-request-lifecycle)
6. [Service Implementation Pattern](#6-service-implementation-pattern)
7. [Plugin System](#7-plugin-system)
8. [State Management (Stores)](#8-state-management-stores)
9. [AWS Protocol Handling](#9-aws-protocol-handling)
10. [Runtime & Lifecycle](#10-runtime--lifecycle)
11. [Configuration System](#11-configuration-system)
12. [Package Management](#12-package-management)
13. [Extension System](#13-extension-system)
14. [Testing Framework](#14-testing-framework)
15. [Docker & Deployment](#15-docker--deployment)
16. [CLI](#16-cli)
17. [DNS Resolution](#17-dns-resolution)
18. [Code Quality & CI/CD](#18-code-quality--cicd)
19. [Design Principles & Guidelines](#19-design-principles--guidelines)
20. [Glossary](#20-glossary)

---

## 1. Project Overview

### What It Is

LocalStack is a **cloud service emulator** that runs entirely on a local machine (or in a
container), providing functional implementations of AWS APIs. Developers use it to develop and
test AWS-based applications without needing a real AWS account or incurring cloud costs.

### Key Goals

| Goal | Description |
|---|---|
| **AWS API Parity** | Reproduce the behavior of real AWS services as faithfully as possible |
| **Local-First** | Run entirely offline — no AWS credentials or network access needed |
| **Fast Feedback** | Start in seconds, run tests in-process, avoid deployment latency |
| **Multi-Service** | Support 40+ AWS services behind a single gateway endpoint |
| **Multi-Account/Region** | Emulate account and region isolation as AWS does |
| **Extensible** | Plugin-based architecture allowing new services and overrides |

### Supported Services (Community Edition)

ACM, API Gateway, CloudFormation, CloudWatch, Config, DynamoDB, EC2, ECR, ECS,
Elasticsearch/OpenSearch, EventBridge, Firehose, IAM, Kinesis, KMS, Lambda, CloudWatch Logs,
Redshift, Route53, S3, S3 Control, Scheduler, Secrets Manager, SES, SNS, SQS, SSM,
Step Functions, STS, SWF, Transcribe, and more.

### Licensing

- **Community Edition**: Apache License 2.0
- **Pro Edition**: Proprietary (extends community with additional services and features)

---

## 2. Technology Stack

### Core Runtime

| Component | Technology | Purpose |
|---|---|---|
| Language | **Python 3.10+** (targets 3.13) | Primary implementation language |
| HTTP Server | **Hypercorn** (ASGI, default), Werkzeug (WSGI), Twisted | Serves the gateway endpoint |
| HTTP Framework | **Rolo** (custom micro-framework built on Werkzeug) | Gateway, handler chain, routing |
| Plugin Framework | **Plux** | Service discovery, lazy loading, entrypoint management |
| AWS SDK Parsing | **Botocore** | Service specs, request/response parsing, protocol details |
| AWS Fallback | **Moto** (moto-ext fork) | Fallback implementations for partially-implemented services |
| Serialization | **Werkzeug**, custom serializers | HTTP request/response serialization per AWS protocol |
| Configuration | **python-dotenv**, env vars, YAML | Hierarchical configuration management |

### Supporting Technologies

| Component | Technology |
|---|---|
| Containerization | Docker (multi-stage builds) |
| JavaScript Runtime | Node.js v22 (Kinesis, some test tooling) |
| Java Runtime | JDK (DynamoDB Local, StepFunctions) |
| Package Manager | LPM (LocalStack Package Manager) |
| Code Formatting | Ruff |
| Type Checking | mypy |
| Testing | pytest + custom snapshot framework |
| CI/CD | GitHub Actions |

### Key Python Dependencies

| Dependency | Purpose |
|---|---|
| `boto3` / `botocore` | AWS SDK, service models, protocol definitions |
| `plux` | Plugin discovery and loading |
| `rolo` | HTTP gateway micro-framework |
| `werkzeug` | HTTP utilities, WSGI compatibility |
| `hypercorn` | ASGI server |
| `cryptography` | TLS, signing |
| `pydantic` | Data validation |
| `jsonschema` | Schema validation |
| `xmltodict` | XML processing |
| `docker` | Docker API client |
| `click` | CLI framework |
| `rich` | Terminal formatting |

---

## 3. Repository Structure

```
localstack/
├── localstack-core/              # Main Python package source
│   └── localstack/
│       ├── aws/                  # AWS gateway, protocol, API definitions
│       │   ├── api/              # Auto-generated service API interfaces (DO NOT EDIT)
│       │   ├── data/             # AWS service model data / patches
│       │   ├── handlers/         # Request/response handler chain components
│       │   ├── protocol/         # AWS protocol parsers and serializers
│       │   ├── serving/          # Server bootstrap and configuration
│       │   ├── app.py            # LocalstackAwsGateway (central orchestrator)
│       │   ├── gateway.py        # Base gateway class
│       │   ├── chain.py          # Handler chain types
│       │   ├── skeleton.py       # Dispatch table and Skeleton pattern
│       │   ├── spec.py           # AWS service specification loading
│       │   └── scaffold.py       # Service scaffolding utilities
│       ├── services/             # AWS service implementations (41+ services)
│       │   ├── <service>/        # Per-service directory
│       │   │   ├── provider.py   # Service provider (API implementation)
│       │   │   ├── models.py     # State store definitions
│       │   │   ├── exceptions.py # Service-specific exceptions
│       │   │   ├── utils.py      # Service-specific utilities
│       │   │   └── resource_providers/  # CloudFormation resource handlers
│       │   ├── plugins.py        # Service plugin infrastructure
│       │   ├── providers.py      # Service registration (@aws_provider decorators)
│       │   └── stores.py         # Store base classes (BaseStore, AccountRegionBundle)
│       ├── runtime/              # Application runtime & lifecycle
│       │   ├── runtime.py        # LocalstackRuntime class
│       │   ├── main.py           # Entry point
│       │   ├── hooks.py          # Lifecycle hooks (on_infra_start, on_shutdown, etc.)
│       │   ├── events.py         # Runtime events
│       │   ├── components.py     # Runtime component factory
│       │   ├── init.py           # Initialization scripts
│       │   └── server/           # Server implementations
│       ├── http/                 # HTTP infrastructure
│       │   ├── hypercorn.py      # Hypercorn ASGI server
│       │   ├── proxy.py          # HTTP proxy
│       │   ├── router.py         # URL routing
│       │   └── websocket.py      # WebSocket support
│       ├── cli/                  # Command-line interface
│       │   └── lpm.py            # LocalStack Package Manager CLI
│       ├── utils/                # Utility modules (46+ modules)
│       ├── state/                # State persistence & snapshots
│       ├── testing/              # Test fixtures, pytest plugins, snapshot testing
│       ├── extensions/           # Extension API & patterns
│       ├── dns/                  # DNS resolution for local services
│       ├── packages/             # Package discovery & installation
│       ├── logging/              # Structured logging
│       ├── dev/                  # Development utilities
│       ├── config.py             # Configuration management (~67KB)
│       ├── constants.py          # Global constants
│       ├── plugins.py            # OpenAPI spec plugin infrastructure
│       └── deprecations.py       # Deprecation warnings
├── tests/
│   ├── aws/services/             # Per-service integration tests
│   ├── unit/                     # Unit tests
│   ├── integration/              # Integration tests
│   ├── bootstrap/                # Container startup tests
│   └── performance/              # Performance tests
├── bin/                          # Executable scripts
│   ├── docker-entrypoint.sh      # Container entrypoint
│   └── localstack-supervisor     # Process supervisor
├── docs/                         # Documentation
│   ├── CONTRIBUTING.md           # Contribution guidelines
│   └── development-environment-setup/
├── scripts/                      # CI/build helper scripts
├── Dockerfile                    # Multi-stage production build
├── Dockerfile.s3                 # S3-only optimized image
├── docker-compose.yml            # Community edition compose
├── docker-compose-pro.yml        # Pro edition compose
├── pyproject.toml                # Python project configuration
├── Makefile                      # Build automation
├── plux.ini                      # Plugin entrypoint declarations (~20KB)
├── AGENTS.md                     # AI/Agent development guidelines
└── LICENSE.txt                   # Apache 2.0
```

### Key Conventions

- Service implementations live in `localstack/services/<service_name>/`
- Auto-generated API interfaces live in `localstack/aws/api/<service_name>/` — **never edit manually**
- Tests mirror the service structure: `tests/aws/services/<service_name>/`
- Plugin entrypoints are declared in `plux.ini` with the namespace `localstack.aws.provider`

---

## 4. Core Architecture

### High-Level Architecture Diagram

```
                                ┌─────────────────────┐
                                │   Client (awslocal,  │
                                │   boto3, AWS CLI)    │
                                └─────────┬───────────┘
                                          │ HTTP :4566
                                          ▼
                              ┌───────────────────────┐
                              │    HTTP Server         │
                              │  (Hypercorn / Werkzeug │
                              │   / Twisted)           │
                              └───────────┬───────────┘
                                          │
                                          ▼
                              ┌───────────────────────┐
                              │  LocalstackAwsGateway  │
                              │  (Handler Chain)       │
                              └───────────┬───────────┘
                                          │
                    ┌─────────────────────┼─────────────────────┐
                    │                     │                     │
                    ▼                     ▼                     ▼
           ┌──────────────┐    ┌──────────────┐    ┌──────────────┐
           │ Request       │    │ Service       │    │ Response      │
           │ Handlers      │    │ Router        │    │ Handlers      │
           │ (parse, auth, │    │               │    │ (serialize,   │
           │  validate)    │    │               │    │  CORS, log)   │
           └──────────────┘    └───────┬──────┘    └──────────────┘
                                       │
                                       ▼
                              ┌──────────────────┐
                              │   Skeleton        │
                              │ (Dispatch Table)  │
                              └────────┬─────────┘
                                       │
                    ┌──────────────────┼──────────────────┐
                    │                  │                  │
                    ▼                  ▼                  ▼
           ┌──────────────┐  ┌──────────────┐  ┌──────────────┐
           │ S3Provider    │  │ SQSProvider   │  │ LambdaProvider│
           │               │  │               │  │               │
           │  Store ◄──────│  │  Store ◄──────│  │  Store ◄──────│
           │ (per-account  │  │ (per-account  │  │ (per-account  │
           │  per-region)  │  │  per-region)  │  │  per-region)  │
           └──────────────┘  └──────────────┘  └──────────────┘
```

### Core Design Patterns

| Pattern | Where Used | Purpose |
|---|---|---|
| **Handler Chain** | `LocalstackAwsGateway` | Sequential processing of request/response through composable handlers |
| **Skeleton / Dispatch Table** | `skeleton.py` | Map AWS operation names to provider method calls |
| **Plugin-based Service Discovery** | `plux` + `plugins.py` | Lazy-load service implementations on first request |
| **Descriptor-based Store** | `stores.py` | Region/account-scoped state with cross-region/cross-account sharing |
| **Protocol Abstraction** | `protocol/` | Parse and serialize requests per AWS protocol type |
| **Fallback Dispatcher** | `MotoFallbackDispatcher` | Delegate unimplemented operations to Moto |

---

## 5. HTTP Request Lifecycle

Every AWS API call to LocalStack traverses a precisely ordered handler chain. This is the
**most critical architectural concept** — all requests flow through the same pipeline.

### 5.1 Request Handler Chain (in execution order)

```python
# Phase 1: Internal & Metrics
add_internal_request_params       # Parse internal cross-service parameters
handle_runtime_shutdown           # Reject requests during shutdown
metric_collector                  # Begin metric collection

# Phase 2: Service Loading & Preprocessing
load_service_for_data_plane       # Lazy-load data-plane services
preprocess_request                # Custom request preprocessing hooks
enforce_cors                      # CORS enforcement
content_decoder                   # Decompress/decode request body (gzip, etc.)
validate_request_schema           # OpenAPI schema validation

# Phase 3: LocalStack Internal Routes
serve_localstack_resources        # Serve /_localstack/* endpoints (health, plugins)
serve_edge_router_rules           # Legacy edge routing rules

# Phase 4: AWS Request Parsing
parse_service_name                # Determine target AWS service (S3, EC2, SQS...)
parse_pre_signed_url_request      # Handle S3 pre-signed URLs
inject_auth_header_if_missing     # Add Authorization header if absent
add_region_from_header            # Extract region from request headers/auth
rewrite_region                    # Rewrite region if configured
add_account_id                    # Extract/validate AWS account ID from auth
parse_trace_context               # Parse X-Ray / trace headers
parse_service_request             # Parse operation name + parameters from HTTP

# Phase 5: Service Dispatch
metric_collector.record           # Record parsed request metrics
serve_custom_handlers             # Execute plugin-registered custom handlers
load_service                      # Load service plugin if not yet loaded
service_request_router            # Route to the service provider's handler

# Phase 6: Fallback
EmptyResponseHandler(404)         # Return 404 if no handler matched
```

### 5.2 Response Handler Chain

```python
validate_response_schema          # Validate response against OpenAPI schema
modify_service_response           # Apply response modifications
parse_service_response            # Serialize service response to HTTP
run_custom_response_handlers      # Execute plugin response handlers
add_cors_response_headers         # Add CORS headers
log_response                      # Log the response
count_service_request             # Increment service counters
update_metric_collection          # Finalize metric collection
```

### 5.3 Exception Handler Chain

```python
log_exception                     # Log the exception
serve_custom_exception_handlers   # Plugin exception handlers
handle_service_exception          # Serialize ServiceException to HTTP error response
handle_internal_failure           # Handle unexpected internal errors
```

### 5.4 Finalizers (always execute)

```python
set_close_connection_header       # Set Connection: close if needed
run_custom_finalizers             # Plugin-registered cleanup
```

### 5.5 RequestContext Object

The `RequestContext` is a mutable object that flows through the entire chain, accumulating
parsed data at each step:

```python
class RequestContext:
    request: Request               # Raw HTTP request
    service: ServiceModel          # Botocore service model (set by parse_service_name)
    protocol: ProtocolName         # AWS protocol type (json, query, rest-xml, etc.)
    operation: OperationModel      # AWS operation model (set by parse_service_request)
    region: str                    # AWS region (set by add_region_from_header)
    account_id: str                # AWS account ID (set by add_account_id)
    request_id: str                # Unique request ID
    service_request: dict          # Parsed request parameters (set by parse_service_request)
    service_response: dict         # Response from service handler
    service_exception: Exception   # Exception if handler raised one
    trace_context: dict            # X-Ray tracing context
    internal_request_params: dict  # Cross-service communication parameters
```

---

## 6. Service Implementation Pattern

### 6.1 The Provider Pattern

Every AWS service is implemented as a **Provider** class that inherits from:
1. An auto-generated **API interface** (e.g., `SqsApi`) containing abstract method stubs
2. `ServiceLifecycleHook` for lifecycle callbacks

```python
# localstack/services/sqs/provider.py

from localstack.aws.api.sqs import SqsApi, CreateQueueResult, ...
from localstack.services.plugins import ServiceLifecycleHook

class SqsProvider(SqsApi, ServiceLifecycleHook):

    @handler("CreateQueue")
    def create_queue(
        self,
        context: RequestContext,
        queue_name: str,
        attributes: dict = None,
        tags: dict = None,
        **kwargs,
    ) -> CreateQueueResult:
        store = sqs_stores[context.account_id][context.region]
        # ... implementation ...
        return CreateQueueResult(QueueUrl=queue_url)

    @handler("SendMessage")
    def send_message(
        self,
        context: RequestContext,
        queue_url: str,
        message_body: str,
        **kwargs,
    ) -> SendMessageResult:
        # ... implementation ...
```

### 6.2 The `@handler` Decorator

Methods are decorated with `@handler(operation_name)` which marks them for the dispatch table:

```python
@handler(operation, context=True, expand=True)
```

| Parameter | Type | Default | Description |
|---|---|---|---|
| `operation` | `str` | required | AWS operation name (e.g., `"CreateBucket"`) |
| `context` | `bool` | `True` | Pass `RequestContext` as first argument |
| `expand` | `bool` | `True` | Expand request dict into keyword arguments |

When `expand=False`, the raw request dict is passed as a single argument instead.

### 6.3 The Skeleton & Dispatch Table

The **Skeleton** wraps a provider and creates a mapping from operation names to handler methods:

```
Skeleton
  ├── service: ServiceModel (botocore model)
  └── dispatch_table: {
        "CreateQueue":   ServiceRequestDispatcher(provider.create_queue),
        "SendMessage":   ServiceRequestDispatcher(provider.send_message),
        "DeleteQueue":   ServiceRequestDispatcher(provider.delete_queue),
        ...
      }
```

The `create_dispatch_table()` function scans the provider's MRO (Method Resolution Order) for
all `@handler`-decorated methods and builds this table automatically.

**Dispatch Flow:**
1. `Skeleton.invoke(context)` is called
2. It looks up `dispatch_table[context.operation.name]`
3. `ServiceRequestDispatcher.__call__()` transforms parameters and calls the provider method
4. The result is serialized back to an HTTP response

### 6.4 The MotoFallbackDispatcher

For services with partial implementations, a `MotoFallbackDispatcher` wraps the dispatch table:

```python
@aws_provider()
def ec2():
    provider = Ec2Provider()
    return Service.for_provider(provider, dispatch_table_factory=MotoFallbackDispatcher)
```

This allows:
- Operations implemented in the provider to be handled directly
- Unimplemented operations to fall back to Moto's implementation
- Gradual migration from Moto to native implementations

### 6.5 Service Registration

Services are registered via the `@aws_provider` decorator in `providers.py`:

```python
# localstack/services/providers.py

@aws_provider()
def sqs():
    from localstack.services.sqs.provider import SqsProvider
    provider = SqsProvider()
    return Service.for_provider(provider)

@aws_provider(api="lambda", name="default")
def lambda_():
    from localstack.services.lambda_.provider import LambdaProvider
    provider = LambdaProvider()
    return Service.for_provider(provider)
```

The decorator creates a `PluginSpec` in the namespace `"localstack.aws.provider"` with the
name `"<api>:<provider_name>"` (e.g., `"sqs:default"`, `"lambda:default"`).

### 6.6 Multi-Implementation Support

A single service can have multiple provider implementations:

```python
@aws_provider(api="events", name="default")
def events_default(): ...

@aws_provider(api="events", name="v1")
def events_v1(): ...

@aws_provider(api="events", name="v2")
def events_v2(): ...
```

The active provider is selected via configuration:
```
PROVIDER_OVERRIDE_EVENTS=v2
```

### 6.7 File Layout for a Service

```
localstack/services/<service>/
├── provider.py               # Main provider class (implements <Service>Api)
├── models.py                 # Store definition (subclass of BaseStore)
├── exceptions.py             # Service-specific exception classes
├── utils.py                  # Service-specific helpers
├── resource_providers/       # CloudFormation resource handlers
│   ├── aws_<service>_<resource>.py
│   └── aws_<service>_<resource>.schema.json
└── <feature>.py              # Feature-specific modules (e.g., notifications.py, cors.py)
```

---

## 7. Plugin System

### 7.1 Overview

LocalStack uses **Plux** — a lightweight plugin framework based on Python entry points — to
achieve lazy-loading of service implementations and extensibility.

### 7.2 Plugin Namespaces

| Namespace | Purpose | Example Entry |
|---|---|---|
| `localstack.aws.provider` | AWS service providers | `sqs:default`, `s3:default` |
| `localstack.cloudformation.resource_providers` | CloudFormation resource handlers | `AWS::S3::Bucket` |
| `localstack.openapi.spec` | OpenAPI specifications | `core` |

### 7.3 Plugin Declaration (`plux.ini`)

All plugin entrypoints are declared in `plux.ini` (~20KB), which maps plugin names to their
factory functions:

```ini
[localstack.aws.provider]
acm:default = localstack.services.providers:acm
dynamodb:default = localstack.services.providers:dynamodb
s3:default = localstack.services.providers:s3
lambda:default = localstack.services.providers:lambda_asf
...
```

### 7.4 Plugin Lifecycle

```
plux.ini declares entrypoints
        │
        ▼
ServicePluginManager scans namespace
        │
        ▼
First request for service arrives
        │
        ▼
ServicePluginManager.require("sqs")
        │
        ▼
PluginManager.load("sqs:default")
        │
        ▼
Factory function called → SqsProvider created
        │
        ▼
Service.for_provider(provider) → Skeleton + dispatch table
        │
        ▼
ServiceContainer wraps Service (tracks state: AVAILABLE → STARTING → RUNNING)
        │
        ▼
Service ready to handle requests
```

### 7.5 ServicePluginManager

The `ServicePluginManager` extends `ServiceManager` with plugin-based lazy loading:

- **`require(name)`**: Returns a running service; loads and starts it if needed
- **`get_state(name)`**: Returns the service's current state without loading it
- **`list_available()`**: Lists all services that have a configured, available provider
- **`get_active_provider(service)`**: Returns the configured provider name for a service
- **`stop_all_services()`**: Stops all running services

### 7.6 Service States

```
AVAILABLE ──start()──► STARTING ──check()──► RUNNING
    │                      │                    │
    │                      │ error              │ stop()
    ▼                      ▼                    ▼
DISABLED              ERROR              STOPPING ──► STOPPED
```

---

## 8. State Management (Stores)

### 8.1 Overview

Each service maintains its state in **Store** objects that provide account and region isolation,
mirroring how AWS manages resource scoping.

### 8.2 Store Definition

```python
# localstack/services/sqs/models.py

from localstack.services.stores import (
    AccountRegionBundle,
    BaseStore,
    LocalAttribute,
    CrossRegionAttribute,
    CrossAccountAttribute,
)

class SqsStore(BaseStore):
    # Per-account, per-region (most resources)
    queues: dict[str, SqsQueue] = LocalAttribute(default=dict)

    # Shared across regions within an account
    DELETED: dict[str, float] = CrossRegionAttribute(default=dict)

# Single global instance
sqs_stores = AccountRegionBundle("sqs", SqsStore)
```

### 8.3 Attribute Scoping

| Descriptor | Scope | Convention | Use Case |
|---|---|---|---|
| `LocalAttribute` | Per-account, per-region | `lowercase` | Most resources (queues, buckets, functions) |
| `CrossRegionAttribute` | Per-account, all regions | `UPPER_CASE` | IAM roles, Route53 hosted zones |
| `CrossAccountAttribute` | All accounts, all regions | `UPPER_CASE` | ARN-based cross-account resources |

### 8.4 Access Pattern

```python
# In a provider method:
def create_queue(self, context: RequestContext, queue_name: str, **kwargs):
    store = sqs_stores[context.account_id][context.region]
    store.queues[queue_name] = SqsQueue(name=queue_name, ...)
```

### 8.5 Store Hierarchy

```
AccountRegionBundle["sqs", SqsStore]
  │
  ├── ["111111111111"] → RegionBundle
  │     ├── ["us-east-1"] → SqsStore (queues={...})
  │     ├── ["eu-west-1"] → SqsStore (queues={...})
  │     └── _global = { DELETED: {...} }  ← shared across regions
  │
  ├── ["222222222222"] → RegionBundle
  │     ├── ["us-east-1"] → SqsStore (queues={...})
  │     └── _global = { DELETED: {...} }
  │
  └── _universal = { ... }  ← shared across all accounts
```

### 8.6 Store Descriptor Protocol

The descriptors use Python's descriptor protocol to transparently redirect attribute access:

- **`LocalAttribute`**: Stores data on the instance with a `attr_` prefix
- **`CrossRegionAttribute`**: Stores data in a shared `_global` dict on the `RegionBundle`
- **`CrossAccountAttribute`**: Stores data in a shared `_universal` dict on the `AccountRegionBundle`

Region and account validation is enforced — invalid region names or malformed account IDs raise `ValueError`.

---

## 9. AWS Protocol Handling

### 9.1 Supported Protocols

AWS services use different wire protocols. LocalStack supports all of them:

| Protocol | Services Using It | Request Format |
|---|---|---|
| **Query** | SQS, SNS, IAM, STS, CloudFormation | Form-urlencoded body, `Action` parameter |
| **JSON** (1.0 / 1.1) | DynamoDB, Lambda, KMS, Logs | JSON body, `X-Amz-Target` header |
| **REST-XML** | S3, CloudFront, Route53 | HTTP method + URI path + XML body |
| **REST-JSON** | API Gateway, Kinesis, EventBridge | HTTP method + URI path + JSON body |
| **EC2** | EC2 | Variant of Query protocol |
| **CBOR** | Kinesis (binary mode) | Binary CBOR encoding |
| **Smithy RPC v2** | Newer AWS services | Modern RPC protocol |

### 9.2 Parsing Pipeline

```
HTTP Request
    │
    ▼
ServiceNameParser determines service from:
  - Host header (s3.localhost.localstack.cloud)
  - Authorization header (Credential=.../s3/aws4_request)
  - X-Amz-Target header
  - URL path
    │
    ▼
ServiceRequestParser uses botocore model to:
  1. Determine the protocol type from the service model
  2. Create the appropriate protocol parser
  3. Parse the operation name from the request
  4. Extract parameters according to the operation's input shape
    │
    ▼
context.operation = OperationModel("CreateQueue")
context.service_request = {"QueueName": "my-queue", "Attributes": {...}}
```

### 9.3 Serialization Pipeline

```
service_response = {"QueueUrl": "http://..."}
    │
    ▼
ServiceResponseParser uses botocore model to:
  1. Select serializer for the service's protocol
  2. Serialize the response dict according to the operation's output shape
  3. Set appropriate Content-Type, status code, headers
    │
    ▼
HTTP Response (XML, JSON, etc.)
```

### 9.4 Error Serialization

Service exceptions are serialized per the protocol:

```python
# In a provider:
raise CommonServiceException("QueueDoesNotExist", "The queue does not exist.", status_code=400)

# Or using generated exception classes:
raise QueueDoesNotExist("The queue does not exist.")
```

Each protocol type serializes errors differently (XML for REST-XML, JSON for JSON protocol, etc.).

---

## 10. Runtime & Lifecycle

### 10.1 LocalstackRuntime

The runtime is the top-level coordinator for the entire application:

```python
class LocalstackRuntime:
    def run(self):
        self._init_filesystem()      # Create directory structure
        self._on_starting()          # Fire on_infra_start hooks
        self._init_gateway_server()  # Create and configure the gateway
        self._run_ready_monitor()    # Monitor until ready, then fire on_infra_ready
        self.components.runtime_server.run()  # Block until shutdown
        self._on_return()            # Fire on_infra_stop hooks, cleanup
```

### 10.2 Lifecycle Hooks

Hooks are plugin-based and allow code to execute at specific lifecycle stages:

```python
from localstack.runtime import hooks

@hooks.on_infra_start()
def setup_something():
    """Runs when infrastructure is starting."""
    ...

@hooks.on_infra_ready()
def after_ready():
    """Runs when all services are ready."""
    ...

@hooks.on_infra_shutdown()
def cleanup():
    """Runs during shutdown."""
    ...
```

### 10.3 Service Lifecycle Hooks

Individual services can implement lifecycle callbacks:

```python
class SqsProvider(SqsApi, ServiceLifecycleHook):
    def on_after_init(self):
        """Called after provider is instantiated."""

    def on_before_start(self):
        """Called before the service starts."""

    def on_before_stop(self):
        """Called before the service stops."""
```

### 10.4 Runtime Events

```python
# Observable events
events.infra_starting    # Emitted when infrastructure begins starting
events.infra_ready       # Emitted when infrastructure is fully ready
events.infra_stopping    # Emitted when infrastructure begins stopping
events.infra_stopped     # Emitted when infrastructure has fully stopped
```

### 10.5 Directory Structure (Runtime Filesystem)

```
/var/lib/localstack/         # DEFAULT_VOLUME_DIR — mounted from host
├── cache/                   # Ephemeral persistent cache
├── tmp/                     # Container-only temp files
├── state/                   # Persisted state (snapshots)
├── logs/                    # Log files
└── lib/                     # Runtime-computed binaries

/usr/lib/localstack/         # Static libs (installed at build time)
/tmp/localstack/             # Shared temp directory
```

---

## 11. Configuration System

### 11.1 Overview

Configuration is primarily environment-variable driven, managed in `config.py` (~67KB).
This is one of the largest files in the project and handles all runtime configuration.

### 11.2 Key Configuration Variables

| Variable | Default | Description |
|---|---|---|
| `GATEWAY_LISTEN` | `0.0.0.0:4566` | Gateway bind address |
| `SERVICES` | (all) | Comma-separated list of services to enable |
| `DEBUG` | `0` | Enable debug mode |
| `LS_LOG` | `warning` | Log level (`trace-internal`, `trace`, `debug`, `info`, `warn`, `error`) |
| `DATA_DIR` | (empty) | Directory for persistent data |
| `PERSISTENCE` | `0` | Enable state persistence |
| `EAGER_SERVICE_LOADING` | `0` | Load all services at startup (vs. lazy) |
| `GATEWAY_SERVER` | `hypercorn` | HTTP server backend (`hypercorn`, `werkzeug`, `twisted`) |
| `ALLOW_NONSTANDARD_REGIONS` | `0` | Accept non-standard AWS region names |
| `DNS_ADDRESS` | `0.0.0.0` | DNS server bind address (`false` to disable) |
| `LAMBDA_EXECUTOR` | (varies) | Lambda execution mode |
| `PROVIDER_OVERRIDE_<SERVICE>` | `default` | Override provider for a specific service |

### 11.3 Directories Configuration

The `Directories` class manages filesystem paths:

```python
class Directories:
    static_libs: str    # /usr/lib/localstack (container-only binaries)
    var_libs: str       # /var/lib/localstack/lib (runtime binaries)
    cache: str          # /var/lib/localstack/cache
    tmp: str            # /tmp/localstack (container-only)
    mounted_tmp: str    # /var/lib/localstack/tmp (shared with host)
    functions: str      # Lambda function volume
    data: str           # Persistent state directory
    config: str         # Configuration directory (host-only)
    init: str           # Initialization scripts
    logs: str           # Log files
```

### 11.4 ServiceProviderConfig

Controls which provider implementation is used for each service:

```python
SERVICE_PROVIDER_CONFIG = ServiceProviderConfig(
    default_value="default",
    override_prefix="PROVIDER_OVERRIDE_",
    # Reads from environment: PROVIDER_OVERRIDE_S3=v2, etc.
)
```

---

## 12. Package Management

### 12.1 LPM (LocalStack Package Manager)

LocalStack ships with a package manager for installing optional runtime dependencies:

```bash
# Install a package
localstack packages install lambda-runtime

# List available packages
localstack packages list
```

### 12.2 Default Packages

The Docker image pre-installs these packages at build time:

| Package | Purpose |
|---|---|
| `lambda-runtime` | Lambda function execution environment |
| `jpype-jsonata` | JSONata support (Java-based) |
| `dynamodb-local` | DynamoDB Local (Java-based) |

### 12.3 Package Installation

Packages are installed into isolated virtual environments under `/var/lib/localstack/lib/`,
then symlinked into the main Python environment.

---

## 13. Extension System

### 13.1 Overview

LocalStack provides an extension API for adding custom functionality:

```
localstack/extensions/
├── api/          # Extension API definitions
└── patterns/     # Common extension patterns
```

Extensions can:
- Add custom request/response handlers to the gateway chain
- Register custom service implementations
- Hook into lifecycle events
- Add custom HTTP endpoints under `/_localstack/extensions/`

---

## 14. Testing Framework

### 14.1 Test Organization

```
tests/
├── aws/services/<service>/    # Per-service integration tests
│   ├── test_<service>.py      # Main test file
│   ├── test_<feature>.py      # Feature-specific tests
│   └── conftest.py            # Service-specific fixtures
├── unit/                      # Unit tests (no running LocalStack needed)
├── integration/               # Integration tests
├── bootstrap/                 # Container startup tests
└── performance/               # Performance benchmarks
```

### 14.2 Pytest Plugins

LocalStack registers a comprehensive set of pytest plugins:

| Plugin | Purpose |
|---|---|
| `localstack.testing.pytest.fixtures` | Core AWS client fixtures (~96KB) |
| `localstack.testing.pytest.container` | Docker container management |
| `localstack_snapshot.pytest.snapshot` | Snapshot testing framework |
| `localstack.testing.pytest.filters` | Test filtering |
| `localstack.testing.pytest.marking` | Custom test markers |
| `localstack.testing.pytest.in_memory_localstack` | In-memory server for tests |
| `localstack.testing.pytest.validation_tracking` | Validation tracking |

### 14.3 Key Test Fixtures

```python
# Session-scoped AWS client factory
aws_client              # Convenience fixture — aws_client.s3, aws_client.sqs, etc.
aws_client_factory      # Factory for creating clients with custom config
aws_session             # Raw boto3 Session
aws_http_client_factory # HTTP client with SigV4 signing

# Per-test fixtures
account_id              # Returns TEST_AWS_ACCOUNT_ID ("123456789012")
aws_region              # Returns TEST_AWS_REGION_NAME ("us-east-1")

# Secondary account support (for cross-account testing)
secondary_aws_client
secondary_aws_client_factory
secondary_aws_session
```

### 14.4 Snapshot Testing

Tests validate AWS parity by recording real AWS responses and comparing against them:

```python
@markers.aws.validated
def test_create_queue(snapshot, aws_client):
    response = aws_client.sqs.create_queue(QueueName="test-queue")
    snapshot.match("create_queue", response)
```

**Workflow:**
1. Write test with `snapshot.match()` assertions
2. Run against real AWS with `SNAPSHOT_UPDATE=1` to capture baseline
3. Run against LocalStack — snapshot comparison verifies parity
4. Snapshot files (`*.snapshot.json`) are committed to the repo and **never edited manually**

### 14.5 Test Markers

```python
@markers.aws.validated          # Test is validated against real AWS
@markers.aws.only_localstack    # Test only runs against LocalStack
@markers.aws.needs_fixing       # Known failing test
```

### 14.6 Testing Best Practices

1. **Always use fixtures** for resource creation — never create resources directly in test bodies
2. **Always use `snapshot.match()`** — never use plain `assert` for response validation
3. **Never hardcode** account IDs (`123456789012`) or region names — use fixtures
4. **Add transformers** for non-deterministic values (ARNs, timestamps, request IDs)
5. **Clean up resources** in fixture teardown with error logging (not raising)
6. **Run against real AWS first** to establish baseline behavior

### 14.7 Test Commands

```bash
# Run a specific test
pytest tests/aws/services/sqs/test_sqs.py

# Run a specific test function
pytest tests/aws/services/sqs/test_sqs.py -k test_create_queue

# Run against real AWS and update snapshots
AWS_PROFILE=ls-sandbox TEST_TARGET=AWS_CLOUD SNAPSHOT_UPDATE=1 pytest <path>

# Run with coverage
make test-coverage
```

---

## 15. Docker & Deployment

### 15.1 Multi-Stage Build (Full Image)

```
Stage 1: base
  └─ Python 3.13-slim + system deps (curl, git, Node.js v22, etc.)
  └─ Creates localstack user and directory hierarchy

Stage 2: builder
  └─ Extends base with build deps (gcc, g++)
  └─ Installs Python dependencies into .venv

Stage 3: final
  └─ Extends base (no build deps)
  └─ Copies .venv from builder
  └─ Installs LocalStack package
  └─ Pre-generates service catalog cache
  └─ Pre-installs default packages (lambda-runtime, dynamodb-local, etc.)
```

### 15.2 S3-Only Image (`Dockerfile.s3`)

An optimized, smaller image for S3-only workloads:
- Minimal system dependencies
- `[base-runtime]` dependencies only (no full runtime)
- Deletes botocore specs for non-S3 services (~80MB savings)
- Sets `SERVICES=s3`, `EAGER_SERVICE_LOADING=1`, `GATEWAY_SERVER=twisted`

### 15.3 Ports

| Port | Purpose |
|---|---|
| `4566` | Main gateway (all AWS APIs) |
| `4510-4559` | External service ports (reserved range) |
| `5678` | Debug port (debugpy) |
| `443` | HTTPS gateway (Pro only) |

### 15.4 Docker Compose

```yaml
services:
  localstack:
    image: localstack/localstack
    ports:
      - "127.0.0.1:4566:4566"
      - "127.0.0.1:4510-4559:4510-4559"
    environment:
      - DEBUG=${DEBUG:-0}
    volumes:
      - "./volume:/var/lib/localstack"
      - "/var/run/docker.sock:/var/run/docker.sock"
```

### 15.5 Health Check

```bash
localstack status services --format=json
```

### 15.6 Image Tags

| Tag | Description |
|---|---|
| `latest` | Latest tested commit on main (may have breaking changes) |
| `stable` | Latest release |
| `<major>` | Latest of major version (e.g., `3`) |
| `<major>.<minor>` | Latest of minor version (e.g., `3.0`) |
| `<major>.<minor>.<patch>` | Exact version (immutable) |

---

## 16. CLI

### 16.1 LocalStack CLI

The CLI (`localstack`) is the primary user interface:

```bash
localstack start [-d]       # Start LocalStack (optionally detached)
localstack stop             # Stop LocalStack
localstack status services  # Show service status
localstack config validate  # Validate docker-compose configuration
localstack logs             # View logs
```

### 16.2 LPM CLI

```bash
localstack packages install <name>   # Install an optional package
localstack packages list             # List available packages
```

### 16.3 awslocal

A convenience wrapper around the AWS CLI that automatically sets the endpoint URL:

```bash
awslocal s3 mb s3://my-bucket
awslocal sqs create-queue --queue-name my-queue
```

---

## 17. DNS Resolution

LocalStack includes a built-in DNS server to resolve AWS-style hostnames locally:

- `*.localhost.localstack.cloud` → `127.0.0.1`
- S3 virtual-hosted style: `my-bucket.s3.localhost.localstack.cloud`
- Configurable via `DNS_ADDRESS` environment variable
- Can be disabled with `DNS_ADDRESS=false`

---

## 18. Code Quality & CI/CD

### 18.1 Code Formatting & Linting

| Tool | Purpose | Command |
|---|---|---|
| **Ruff** | Linting + formatting (replaces flake8, isort, black) | `make lint` / `make format` |
| **mypy** | Static type checking | Runs via pre-commit |
| **openapi-spec-validator** | Validates OpenAPI specs | Runs via pre-commit |
| **deptry** | Dependency checking | `make lint` |

### 18.2 Ruff Configuration

```toml
[tool.ruff]
target-version = "py313"
line-length = 100
select = ["B", "C", "E", "F", "I", "W", "T", "UP"]
```

### 18.3 Pre-commit Hooks

1. **Ruff** linting and formatting
2. **mypy** type checking
3. **Standard hooks**: no-commit-to-branch, end-of-file-fixer, trailing-whitespace, check-json
4. **Custom**: check-pinned-deps-for-needed-upgrade
5. **OpenAPI spec validation**

### 18.4 CI/CD Pipelines (GitHub Actions)

| Workflow | Trigger | Purpose |
|---|---|---|
| `aws-main.yml` | Push to main, scheduled, PRs | Full build, test, push pipeline |
| `aws-tests.yml` | Reusable | Integration test suite |
| `aws-tests-mamr.yml` | Reusable | Multi-architecture (ARM) testing |
| `aws-tests-s3-image.yml` | Reusable | S3-only image testing |
| `tests-pro-integration.yml` | PRs | Pro integration tests |
| `tests-bin.yml` | PRs | CLI/binary tests |
| `asf-updates.yml` | Scheduled | Auto-update AWS service framework |
| `update-cfn-resources.yml` | Scheduled | Update CloudFormation resources |

### 18.5 Semver Labels (PR Requirements)

Every PR must have a semver label:

| Label | When to Use |
|---|---|
| `semver: patch` | Small, non-breaking changes (bug fixes) |
| `semver: minor` | Bigger, non-breaking changes (features, refactoring) |
| `semver: major` | Breaking changes |

### 18.6 Contributor License Agreement (CLA)

Required for all contributors. Enforced via `pr-cla.yml` workflow.

---

## 19. Design Principles & Guidelines

### 19.1 Architecture Principles

1. **Single Gateway, Multiple Services**: All AWS APIs are served through a single port (4566).
   The gateway determines the target service from request metadata (headers, auth, path).

2. **Lazy Loading**: Services are loaded on first request, not at startup (unless
   `EAGER_SERVICE_LOADING=1`). This keeps startup fast.

3. **Protocol Agnostic Core**: The handler chain is protocol-agnostic. Protocol-specific parsing
   and serialization happen in dedicated handlers, so providers deal only with typed Python dicts.

4. **Account/Region Isolation**: State is always scoped to an account and region, matching AWS
   behavior. The store system enforces this.

5. **Composition over Inheritance**: The handler chain, plugin system, and dispatch table all
   favor composition. Services are assembled from independent pieces.

6. **Graceful Degradation**: Unknown operations return informative error messages. The
   `MotoFallbackDispatcher` provides fallback behavior for partial implementations.

### 19.2 Development Guidelines

1. **Never edit auto-generated files** in `localstack/aws/api/` — regenerate with `make asf-regenerate`
2. **Never edit snapshot files** (`*.snapshot.json`) manually — regenerate with `SNAPSHOT_UPDATE=1`
3. **Always run `make format` and `make lint`** before committing
4. **Write tests first** — follow the test-driven workflow described in AGENTS.md
5. **Use fixtures for resource creation** — ensures cleanup happens
6. **Separate unrelated changes** into multiple PRs
7. **Add `pydoc` documentation** for public methods and classes
8. **Don't add dependencies** without explicit approval

### 19.3 Naming Conventions

| Entity | Convention | Example |
|---|---|---|
| Service directory | `snake_case` (AWS name) | `services/lambda_/`, `services/s3/` |
| Provider class | `PascalCase` + `Provider` | `SqsProvider`, `S3Provider` |
| Store class | `PascalCase` + `Store` | `SqsStore`, `S3Store` |
| Store instance | `snake_case` + `_stores` | `sqs_stores`, `s3_stores` |
| Local attributes | `lowercase` | `queues`, `buckets` |
| Cross-region/account attributes | `UPPER_CASE` | `DELETED`, `GLOBAL_TABLE` |
| Plugin name | `<api>:<provider>` | `sqs:default`, `lambda:v2` |
| Test file | `test_<service>.py` | `test_sqs.py`, `test_s3.py` |

### 19.4 Error Handling

- Use `ServiceException` subclasses for AWS-compatible errors
- Use `CommonServiceException(code, message, status_code)` for ad-hoc errors
- `NotImplementedError` results in a 501 with an informative message
- Internal errors are caught by the exception handler chain and logged

---

## 20. Glossary

| Term | Definition |
|---|---|
| **ASF** | AWS Service Framework — the code generation and scaffolding system for service APIs |
| **Dispatch Table** | A mapping from AWS operation names to handler functions |
| **Edge** | The single gateway endpoint (port 4566) that all requests flow through |
| **Gateway** | The `LocalstackAwsGateway` that orchestrates the handler chain |
| **Handler Chain** | An ordered list of handlers that process each request sequentially |
| **LPM** | LocalStack Package Manager — manages optional runtime packages |
| **Moto** | Third-party AWS mock library used as a fallback for unimplemented operations |
| **Plux** | Plugin framework based on Python entry points |
| **Provider** | A class that implements the API for a specific AWS service |
| **RequestContext** | Mutable object carrying all parsed request data through the handler chain |
| **Rolo** | Micro-framework for HTTP gateways, built on Werkzeug |
| **Skeleton** | Wrapper around a provider that creates and manages the dispatch table |
| **Snapshot Testing** | Testing approach that records real AWS responses and compares against them |
| **Store** | Per-service state container scoped by account and region |
| **ServiceContainer** | Wrapper around a `Service` that tracks lifecycle state |

---

*This spec was generated by analyzing the LocalStack codebase at its current state. For the
most up-to-date information, refer to the [official documentation](https://docs.localstack.cloud)
and the source code.*
