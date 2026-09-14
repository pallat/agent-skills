---
description: Go clean architecture — hexagonal, onion, screaming, DCI. Use when creating or modifying Go domain packages, ports, adapters, handlers, or wiring.
---

## Hexagonal — ports & adapters
A **port** is an interface declared in the domain. An **adapter** is a concrete that implements it.
- Driving port: how the outside calls in (handler method on a `Context` interface)
- Driven port: how the domain calls out (`UserFinder`, `Repository`)
- The domain never sees the adapter; it only sees the port. Dependency inverted.

## Onion — dependency flows inward only
`adapter → domain ← adapter` but never `domain → adapter`.
- Domain parent imports ONLY stdlib (no gin, pgx, sarama, no own pkg/)
- Driven adapters import domain (for types) + tech (for impl) — they are outside
- `main.go` is the composition root: the only place that knows concrete types
- `pkg/` helpers are outside the onion; `pkg/` never imports `internal/`

## Screaming — the structure tells you what the system does
Read every package name out loud. If it names the **business** (`permit`, `loan`, `booking`) it passes. If it names a **layer or pattern** (`core`, `service`, `handler`, `consumer`, `adapter`) it fails.
- Name adapters by **intent** (`userfinder`, `userstorage`) not **tech** (`client`, `postgres`)
- No `internal/adapter/` — adapters are **subpackages of the domain**: `internal/<domain>/userfinder/`
- Opening `internal/<domain>/` shows everything about that domain

## DCI — Context is the role the domain declares
The domain defines a `Context` interface with only the methods it needs (`ShouldBindJSON`, `JSON`, `context.Context`). The framework fills the role at runtime.
- `pkg/framework/NewHandler[C]` casts `*gin.Context` → `C` — the sole gin import
- Domain validates its own invariants (no `binding:"required"` from a spec generator)
- Tests pass a `stubContext` — no gin, no router, no HTTP stack
- `DomainError` is the domain's error vocabulary; adapters translate foreign errors into it at the boundary (anti-corruption layer)

## Illustration
```
main.go                         ← wires concretes, no logic
internal/<domain>/              ← onion center: entity, port, service, handler
internal/<domain>/<adapter>/    ← driven adapters (hexagonal outside)
pkg/framework/                  ← fills the Context role (DCI)
```

## Checklist
- Domain parent: stdlib only (+ pkg/framework in handler.go)
- No `internal/adapter/` — adapters are domain subpackages
- Package names are business terms, not architecture terms
- Handler depends on port interface, not concrete `*Service`
- Adapter errors → `*DomainError` at the boundary
- `var _ Port = (*Impl)(nil)` compile-time proof in adapter tests
- `go build ./... && go vet ./... && go test ./...` pass
