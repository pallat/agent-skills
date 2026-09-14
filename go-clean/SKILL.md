---
description: Go Clean Architecture patterns from go-ms-otel. Use when creating or modifying Go code — new domain packages, handlers, ports, adapters, or wiring in main.go.
---

## Structure

```
main.go                    ← wiring only, no logic
internal/<domain>/         ← stdlib ONLY (entity, port, service, handler)
internal/<domain>/<adapter>/ ← driven adapter subpackage (userfinder/, userstorage/)
pkg/                       ← tech helpers (framework/, config/, middleware/, probe/)
```

## Rules

1. Domain imports ONLY stdlib (+ pkg/framework in handler.go)
2. Adapters are domain subpackages — no `internal/adapter/`
3. `pkg/` never imports `internal/`
4. `main.go` wires only — no business logic
5. Package name = business term (`permit`, `loan`), not role (`service`, `handler`)

## Per-file essentials

**entity.go** — domain types + request/response structs (owns wire shape)
**port.go** — `UserFinder`, `Repository` interfaces — domain types only, no transport
**service.go** — `DomainError{Code,Message,Err}` + `NewDomainError(code,msg,cause)` + use case depending on ports
**handler.go** — `Context` interface (`context.Context` + `ShouldBindJSON` + `JSON`), `PermitService` interface, `mapDomainErrorToHTTP`, domain validates own invariants
**userfinder/user_finder.go** — anti-corruption: translate HTTP→DomainError at boundary, wire envelope stays unexported in adapter
**userstorage/user_storage.go** — implements `Repository`, takes entity not wire envelope

## Framework bridge (pkg/framework/gin.go)

`NewHandler[C any](handler func(C)) gin.HandlerFunc` — sole gin import; domain never imports gin.
`StdContext(c)` — unwraps `*gin.Context` → `request.Context()` for middleware values.

## main.go wiring order

config → otel/logger → gin+middleware → probes → adapters → service → handler → `framework.NewHandler(h.Method)` → start+shutdown

## Testing

Stub `Context` (no gin, no router): `stubContext{body, jsonCode, jsonBody}` implementing the domain interface. Test handler logic purely.

## Checklist

- Domain parent imports only stdlib (+ pkg/framework)
- No `internal/adapter/` dir
- Handler depends on interface, not `*Service`
- Adapter errors return `*DomainError`
- `var _ Port = (*Impl)(nil)` in adapter tests
- `go build ./... && go vet ./... && go test ./...` pass