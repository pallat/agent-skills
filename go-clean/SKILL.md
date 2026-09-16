---
description: Go clean architecture — hexagonal, onion, screaming, DCI. Use when creating or modifying Go domain packages, ports, adapters, handlers, or wiring.
---

## Hexagonal — ports & adapters
A **port** is an interface declared in the domain. An **adapter** is a concrete outside it that implements it.
- Driving port: how the outside calls in
- Driven port: how the domain calls out
- The domain never sees the adapter, only the port — dependency inverted

## Onion — dependency flows inward only
`adapter → domain ← adapter`, never `domain → adapter`.
- Domain package imports stdlib only — no third-party packages, no own `pkg/` helpers
- Adapters import the domain (for types) plus their own tech — they sit outside
- The composition root is the only place that knows concrete types and wires them together

## Screaming — the structure tells you what the system does
Package name = business term, never a layer or pattern word.
- Name adapters by intent, not by the tech they wrap
- No shared `adapter/`/`port/`/`interfaces/` package — each lives beside the domain it serves
- Opening one domain's folder shows everything about that domain

## DCI — the domain declares the role it needs
The domain defines a minimal role interface for what it needs from the outside. The composition root supplies a concrete that fills that role at wiring time.
- Domain validates its own invariants — it never trusts an external contract to do that for it
- Tests fill the role with a stub — no framework, no real infra
- Foreign errors are translated into the domain's own error type at the boundary

## Illustration
```
main.go             ← composition root: wires concretes, no logic
internal/<domain>/  ← onion center: entity, port, service, handler
internal/<domain>/<driven-adapter>/ ← subpackage of the domain it serves
```

## Checklist
- Domain package imports stdlib only
- No shared `adapter/`/`port/`/`interfaces/` package
- Package names are business terms, not architecture terms
- Handler/service depends on an interface it declares, not a concrete
- Adapter errors are translated into the domain's error type at the boundary
- Compile-time proof an adapter satisfies its port (`var _ Port = (*Impl)(nil)`)
- `go build ./... && go vet ./... && go test ./...` pass
