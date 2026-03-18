---
name: go-handler
description: Implement or review the handler layer for a Go service following architecture standards
argument-hint: <ServiceName>
allowed-tools: [Read, Glob, Grep, Edit, Write, Bash]
---

Implement or review the handler layer for $ARGUMENTS following the specs below exactly.

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

Order of operations:
1. Check the cache; if hit, unmarshal and return
2. On miss, query the database
3. Build the response
4. Marshal and write to cache with a TTL
5. Return the response

Cache errors are always logged and swallowed — a cache failure must never prevent a valid response from being returned.

---

## Log Message Format

```
"<service>/service: MethodName: description of what failed"
```

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
