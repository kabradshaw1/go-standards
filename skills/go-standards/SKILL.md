---
name: go-standards
description: Load the full Go service architecture standards for cross-layer review or greenfield implementation
allowed-tools: [Read, Glob, Grep, Edit, Write, Bash]
---

Load the full Go service architecture standards. Use this for greenfield service implementation or cross-layer review.

# Service Architecture — Overview

This document describes the top-level structure for a service in this project. Before building any layer, read this file. Then read the spec file for the specific layer you are implementing.

---

## Directory Structure

Every service follows this layout:

```
services/<name>/
├── main.go         # Env validation, dependency wiring, server start
├── handler/
│   └── handler.go  # Transport layer
├── service/
│   └── service.go  # Business logic
├── database/
│   └── database.go # Data access
├── client/
│   └── client.go   # Outbound clients to other services or APIs
├── types/
│   └── types.go    # Local interfaces for every dependency
└── Dockerfile
```

Each directory is its own Go package named after the directory.

---

## Build Order

Implement layers in this order. Each layer depends on the one before it.

1. `types/` — define all interfaces first so every other layer has a contract to depend on
2. `database/` — implement the data access layer against the `Database` interface
3. `client/` — implement any outbound clients against their interfaces
4. `service/` — implement business logic using only the interfaces from `types/`
5. `handler/` — implement the transport layer using only the `Service` interface from `types/`
6. `main.go` — wire all concrete implementations together and start the server

---

## Core Principles

**Depend on interfaces, not implementations.** Every layer holds its dependencies as interfaces from `types/`. Nothing is imported directly from sibling packages except in `main.go`.

**Inject all dependencies through constructors.** All dependencies are passed into `New*` functions. Nothing is instantiated inside the service or handler.

**One responsibility per layer.** Transport logic does not touch the database. Business logic does not build queries. Queries do not make outbound calls.

**Crash on bad config, log on recoverable errors.** Missing environment variables are fatal at startup. Cache failures and best-effort side effects are logged and swallowed. Anything that prevents a correct response is returned as an error.

**Shared infrastructure lives in `services/common`.** Reusable clients, interfaces, and domain models are defined there and imported. Never copy infrastructure code into a service.

---

# Layer Spec — `types/types.go`

## Responsibility

Define the interfaces that every other layer in the service depends on. This file is the contract layer — it describes what each dependency must provide without specifying how.

---

## Rules

- One interface per dependency: `Service`, `Database`, `SomeClient`, etc.
- The `Service` interface must exactly match the public methods implemented on the service struct — the handler depends on it
- No concrete types, no structs, no implementation details — interfaces only
- Shared domain models used across multiple services (e.g. common database models, queue message types) live in `services/common` and are imported here as needed
- Do not import from sibling packages (`handler`, `service`, `database`, `client`) — only from `services/common` and standard libraries

---

## Pattern

```go
package types

import (
    "context"

    commontypes "github.com/kabradshaw1/story/services/common"
    "github.com/kabradshaw1/story/services/common/genproto/<name>"
)

type Service interface {
    MethodOne(ctx context.Context, req *proto.Request) (*proto.Response, error)
    MethodTwo(ctx context.Context, req *proto.Request) (*proto.Response, error)
}

type Database interface {
    FindRecord(ctx context.Context, id int) (*commontypes.SomeModel, error)
    SaveRecord(ctx context.Context, record commontypes.SomeModel) error
}

type SomeClient interface {
    Notify(ctx context.Context, payload SomePayload) error
}
```

---

## Checklist

- [ ] One interface defined for each dependency the service layer requires
- [ ] `Service` interface matches the service struct's public methods exactly
- [ ] No concrete types or structs defined here
- [ ] All shared domain types imported from `services/common`, not redefined

---

# Layer Spec — `database/database.go`

## Responsibility

Execute all data access operations. This is the only layer that communicates with the database.

---

## Rules

- No business logic — only queries and result mapping
- A `Connect` function (or equivalent) accepts a connection string, establishes the connection, and returns a typed `*DB` struct; it calls `log.Fatal` on failure
- Every method accepts `context.Context` as its first argument and passes it through to the underlying query
- The service layer never constructs a query directly — all data access goes through this layer
- Does not call outbound clients or touch the cache

---

## Pattern

```go
package database

type DB struct {
    client *SomeClient
}

func Connect(connectionString string) *DB {
    // connect, fatal on error
    return &DB{client: client}
}

func (d *DB) FindRecord(ctx context.Context, id int) (*commontypes.SomeModel, error) {
    // query using ctx
}

func (d *DB) SaveRecord(ctx context.Context, record commontypes.SomeModel) error {
    // insert using ctx
}
```

---

## Mutation Authorization

For any mutation scoped to a user, always verify ownership before modifying data:

1. Fetch the record by ID
2. Confirm `record.User == requestingUser` — return a descriptive error if not
3. Perform the update

---

## Avoiding Extra Round-Trips

After an update query, do not re-fetch the record to return it. Instead, update the already-fetched struct in memory:

```go
// fetch → verify → update → return modified struct
record.Field = newValue
return &record, nil
```

---

## Error Handling

- Distinguish "record not found" from other errors and return a clear, specific message for each
- Wrap all errors with context before returning: `fmt.Errorf("failed to find record: %w", err)`

---

## Checklist

- [ ] `Connect` accepts a connection string and calls `log.Fatal` on failure
- [ ] Every method takes `context.Context` as its first parameter
- [ ] No business logic, no cache access, no outbound calls
- [ ] User-scoped mutations verify ownership before updating
- [ ] Updates modify the in-memory struct and return it — no redundant re-fetch
- [ ] "Not found" errors are distinguished from other errors
- [ ] All returned errors are wrapped with context

---

# Layer Spec — `client/client.go`

## Responsibility

Handle all outbound calls to external services or third-party APIs. This is the only layer that makes outbound network requests.

---

## Rules

- Wraps the underlying transport (HTTP client, gRPC connection, SDK, etc.) in a struct
- A `New*` constructor initializes and returns the client struct
- Every method accepts `context.Context` as its first argument and passes it to the underlying call
- Validate required inputs before making any call — return an error immediately for missing or invalid fields
- Return a descriptive error for any non-success response from the remote
- No business logic — only request construction, sending, and response validation

---

## Pattern

```go
package client

type SomeClient struct {
    // underlying transport (http.Client, grpc.ClientConn, etc.)
}

func New() *SomeClient {
    return &SomeClient{
        // initialize transport
    }
}

func (c *SomeClient) Notify(ctx context.Context, payload SomePayload) error {
    if payload.RequiredField == "" {
        return fmt.Errorf("RequiredField is required")
    }
    // construct and send request using ctx
    // check response and return error on failure
    return nil
}
```

---

## Checklist

- [ ] Underlying transport is wrapped in a struct, not used directly
- [ ] `New*` constructor initializes the transport
- [ ] Every method accepts and passes through `context.Context`
- [ ] Required inputs are validated before the call is made
- [ ] Non-success responses from the remote return a descriptive error
- [ ] No business logic, no database access, no cache access

---

# Layer Spec — `service/service.go`

## Responsibility

Implement all business logic. Orchestrate calls to the database, cache, and outbound clients to fulfill a request.

---

## Rules

- The struct holds all dependencies as interfaces from `types/` — never concrete implementations
- The `New*` constructor accepts every dependency as an argument — nothing is instantiated inside
- This is the only layer that decides what to validate, what to cache, and what order to call things in
- Does not build queries directly — calls `types.Database` methods only
- Does not know about the transport mechanism

---

## Pattern

```go
package service

type Service struct {
    db     types.Database
    cache  commontypes.Cache
    client types.SomeClient
}

func NewService(db types.Database, cache commontypes.Cache, client types.SomeClient) *Service {
    return &Service{db: db, cache: cache, client: client}
}
```

---

## Cache Pattern

Use a read-through cache on any method that reads data. Keys must use `:` as a separator between all components to prevent collisions.

```
cacheKey := "resource:" + userID + ":" + entityType
```

Order of operations:
1. Check the cache; if hit, unmarshal and return
2. On miss, query the database
3. Build the response
4. Marshal and write to cache with a TTL
5. Return the response

Cache errors (get, set, unmarshal, marshal) are always logged and swallowed — a cache failure must never prevent a valid response from being returned.

```go
cacheData, err := s.cache.Get(ctx, cacheKey)
if err == nil && cacheData != "" {
    if err := json.Unmarshal([]byte(cacheData), &response); err != nil {
        log.Printf("<service>/service: Method: failed to unmarshal cache: %v", err)
    } else {
        return &response, nil
    }
}
// ... query db, build response ...
serializedData, err := json.Marshal(&response)
if err != nil {
    log.Printf("<service>/service: Method: failed to marshal for cache: %v", err)
} else {
    if err := s.cache.Set(ctx, cacheKey, string(serializedData), 10*time.Minute); err != nil {
        log.Printf("<service>/service: Method: failed to set cache: %v", err)
    }
}
```

---

## Error Handling

- Return errors for anything required to produce a correct response
- Wrap all returned errors with context: `fmt.Errorf("description: %w", err)`
- Log and continue only for genuine best-effort side effects (e.g. an audit write that must not block the response); always add a comment explaining the intent

---

## Log Message Format

All log messages must identify the service and method:

```
"<service>/service: MethodName: description of what failed"
```

---

## Checklist

- [ ] Struct holds only `types.*` interfaces and `commontypes.*` interfaces — no concrete types
- [ ] `New*` constructor accepts all dependencies as arguments
- [ ] Cache keys use `:` separators between all variable components
- [ ] Cache errors are logged, never returned
- [ ] All returned errors are wrapped with `fmt.Errorf`
- [ ] Log messages follow the `<service>/service: Method: description` format
- [ ] Best-effort side effects have a comment explaining why failure is intentionally non-fatal

---

# Layer Spec — `handler/handler.go`

## Responsibility

Receive inbound requests and delegate to the service. This is the only layer that knows about the transport mechanism (HTTP, gRPC, queue consumer, webhook, etc.).

---

## Rules

- The handler struct holds a `types.Service` interface — never a concrete `*service.Service`
- Methods are thin pass-throughs: accept a request, call the service method, return the result
- No business logic here — no validation, no conditionals on the data, no error wrapping
- No direct access to the database, cache, or any client
- The `New*` constructor accepts `types.Service` and registers or returns the handler

---

## Pattern

```go
package handler

type Handler struct {
    svc types.Service
}

func New(svc types.Service) *Handler {
    return &Handler{svc: svc}
}

func (h *Handler) MethodOne(ctx context.Context, req *Request) (*Response, error) {
    return h.svc.MethodOne(ctx, req)
}
```

---

## Checklist

- [ ] Handler struct holds `types.Service`, not `*service.Service`
- [ ] `New*` constructor accepts `types.Service`
- [ ] Every method is a direct delegation to the service with no added logic
- [ ] No imports from `service/`, `database/`, or `client/`

---

# Layer Spec — Testing

## Expectations

Write tests for every layer and run them before considering any implementation complete. Do not skip or comment out failing tests — fix the underlying issue.

---

## Unit Tests

Unit tests use mocks for all dependencies. They require no running infrastructure and should always pass locally.

**File placement:** alongside the file being tested, in the same package.

```
handler/handler_test.go
service/service_test.go
database/database_test.go
client/client_test.go
```

**Run with:**
```
go test ./...
```

### What to test per layer

**`service/service_test.go` — highest priority**

For each service method:
- Happy path — correct inputs return the expected response
- Each validation failure — missing or invalid fields return the expected error
- Database error — the db mock returns an error; confirm it propagates
- Cache hit — cache returns data; confirm the database mock is never called
- Cache miss — cache returns nothing; confirm the database is called and the result is written to cache
- Best-effort failure — if a side effect is intentionally non-fatal, confirm the method still succeeds when it fails

**`handler/handler_test.go`**

For each handler method:
- Correct service method is called with the correct arguments
- Service response is returned unmodified
- Service error is returned as-is

**`database/database_test.go`**

For each database method:
- Happy path returns the expected record(s)
- Record not found returns the expected error
- For user-scoped mutations: an unauthorized user receives an error and the record is not modified

**`client/client_test.go`**

For each client method:
- Happy path — remote returns success; confirm the correct payload was sent
- Input validation — missing required fields return an error before a call is made
- Remote error — remote returns a failure; confirm an error is returned

### Mocking

Because every dependency is an interface from `types/`, mocks can be written by hand or generated. A mock must:
- Implement the interface
- Record what arguments it was called with
- Return configurable values (success value and error) so a single mock struct covers all cases

---

## Integration Tests

Integration tests verify that two or more real components work correctly together. No mocks — real infrastructure only.

**File placement:**
```
integration/integration_test.go
```

**What to cover:**
- The database layer against a real test database: queries return correct data, constraints are enforced, ownership checks work
- The client layer against a real or local stub server: correct request is assembled and sent, response is handled
- The service and database together: confirm the full read/write flow without mocking the database

**Run with:**
```
go test ./integration/... -tags integration
```

Only run when the required infrastructure is available (local Docker, CI services, etc.).

---

## End-to-End Tests

End-to-end tests start the full service — real transport, database, and cache — and exercise it as a client would.

**File placement:**
```
e2e/e2e_test.go
```

**What to cover:**
- The complete request lifecycle from transport input to response, including database writes and cache population
- Cross-layer error paths — invalid input that the handler receives but the database never sees
- Server starts, accepts connections, and shuts down cleanly

**Setup:** the e2e test starts the server with a real test configuration and tears it down after the suite. Seed required data before tests run and clean it up afterward.

**Run with:**
```
go test ./e2e/... -tags e2e
```

Run in CI on pull requests and before releases.

---

## Summary

| Type | Mocks | Infrastructure needed | When to run |
|---|---|---|---|
| Unit | All dependencies | None | Always |
| Integration | None | Test DB / server | When infrastructure is available |
| End-to-end | None | Full service stack | CI / pre-release |
