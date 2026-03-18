# go-standards

A Claude Code plugin that provides skills for building Go services with a consistent, clean-architecture layered structure. Install it once in any service repo and agents will follow your exact standards — no copy-pasting, no drift.

---

## Installation

Add go-standards to your Go service repository so the whole team gets the plugin discoverable by default. This is the recommended approach when setting up a new Go service repo, and the right first step before using the Superpowers parallel agent workflow.

**Step 1 — Add `.claude/settings.json` to the consuming repo** (commit this):

```json
{
  "extraKnownMarketplaces": {
    "go-standards": {
      "source": {
        "source": "github",
        "repo": "kabradshaw1/go-standards"
      }
    }
  },
  "enabledPlugins": {
    "go-standards@go-standards": true
  }
}
```

**Step 2 — Each developer installs the plugin once** in the consuming repo:

```
/plugin install go-standards@go-standards
```

This fetches the plugin from GitHub and makes the skills available. The `enabledPlugins` flag in settings.json marks it enabled — but the install command must be run at least once per developer to fetch it. After that, the skills appear immediately on session start.

---

## Available Skills

| Skill | Invocation | Purpose |
|---|---|---|
| `go-standards` | `/go-standards:go-standards` | Full architecture overview + all layer specs. Use for greenfield services or cross-layer review. |
| `go-types` | `/go-standards:go-types <ServiceName>` | Implement or review the `types/` layer |
| `go-database` | `/go-standards:go-database <ServiceName>` | Implement or review the `database/` layer |
| `go-client` | `/go-standards:go-client <ServiceName>` | Implement or review the `client/` layer |
| `go-service` | `/go-standards:go-service <ServiceName>` | Implement or review the `service/` layer |
| `go-handler` | `/go-standards:go-handler <ServiceName>` | Implement or review the `handler/` layer |

---

## Service Structure

Every service produced by these skills follows this layout:

```
services/<name>/
├── main.go             # Env validation, dependency wiring, server start
├── handler/
│   └── handler.go      # Transport layer (HTTP, gRPC, queue, etc.)
├── service/
│   └── service.go      # Business logic
├── database/
│   └── database.go     # Data access
├── client/
│   └── client.go       # Outbound calls to external services or APIs
├── types/
│   └── types.go        # Local interfaces for every dependency
└── Dockerfile
```

Each directory is its own Go package named after the directory.

---

## Build Order

Layers must be implemented in this order — each depends on the one before it:

```
types/ → database/ → client/ → service/ → handler/ → main.go
```

Start with `types/` to establish the interfaces every other layer depends on.

---

## Core Principles

**Depend on interfaces, not implementations.** Every layer holds its dependencies as interfaces from `types/`. Nothing is imported directly from sibling packages except in `main.go`.

**Inject all dependencies through constructors.** All dependencies are passed into `New*` functions. Nothing is instantiated inside a layer.

**One responsibility per layer.** Transport logic does not touch the database. Business logic does not build queries. Queries do not make outbound calls.

**Crash on bad config, log on recoverable errors.** Missing environment variables are fatal at startup. Cache failures are logged and swallowed. Anything that prevents a correct response is returned as an error.

**Shared infrastructure lives in `services/common`.** Reusable clients, interfaces, and domain models are defined there and imported. Never copy infrastructure code into a service.

---

## Using with Superpowers

These skills are designed to be used as context carriers for agents. Each skill is self-contained — it includes the full architecture spec and the layer-specific rules, so an agent prompt only needs to name the skill and the service. No additional standard-pasting required.

### Pattern A — Sequential (one layer at a time)

Invoke the skill directly for the layer you are building:

```
/go-standards:go-types UserProfile
/go-standards:go-database UserProfile
/go-standards:go-service UserProfile
```

### Pattern B — Parallel agents (superpowers:dispatching-parallel-agents)

After the `types/` layer is complete (all other layers depend on it), the remaining layers can be dispatched in parallel. Use `superpowers:dispatching-parallel-agents` and give each agent a prompt like:

```
You are building the database layer for the UserProfile service.

Start by invoking the go-standards:go-database skill with argument "UserProfile".

Service path: services/user-profile/
Types are already defined in: services/user-profile/types/types.go

Do not modify types.go. Implement database/database.go only.
```

```
You are building the client layer for the UserProfile service.

Start by invoking the go-standards:go-client skill with argument "UserProfile".

Service path: services/user-profile/
Types are already defined in: services/user-profile/types/types.go

Do not modify types.go. Implement client/client.go only.
```

Each agent loads the skill, gets the full spec for its layer, and implements it to your exact standards.

### Full Parallel Dispatch Order

```
Phase 1 (sequential):  types/
Phase 2 (parallel):    database/ and client/ (independent of each other)
Phase 3 (sequential):  service/ (depends on database + client interfaces)
Phase 4 (sequential):  handler/ (depends on service interface)
Phase 5 (sequential):  main.go (wires everything together)
```

---

## Testing

Each skill includes a testing spec. The expected test layout per service:

```
handler/handler_test.go     # Unit: correct delegation, response returned unmodified
service/service_test.go     # Unit: happy path, validation, db errors, cache hit/miss
database/database_test.go   # Unit: happy path, not found, authorization
client/client_test.go       # Unit: happy path, input validation, remote errors
integration/                # Integration: real DB and service connections, no mocks
e2e/                        # E2E: full stack from transport to database
```

Run unit tests:
```
go test ./...
```

Run integration tests (requires infrastructure):
```
go test ./integration/... -tags integration
```

Run e2e tests (CI / pre-release):
```
go test ./e2e/... -tags e2e
```
