# Go Clean Architecture — go-ms-otel Patterns

Follow these rules when writing, refactoring, or extending Go code in this project.
These patterns are extracted from go-ms-otel (gitdev.devops.krungthai.com/techcoach/template/go-ms-otel).

## Layer Structure

```
project/
├── main.go                     ← composition root ONLY — wiring, no business logic
├── internal/                   ← service-private: domain + driven adapters
│   └── <domain>/               ← core domain (onion center) — stdlib ONLY
│       ├── entity.go           → domain types, value objects
│       ├── port.go             → interfaces (ports) — depends on NOTHING external
│       ├── service.go          → business logic + DomainError — depends only on interfaces
│       ├── handler.go          → driving adapter: HTTP handler LIVES IN THE DOMAIN
│       ├── handler_test.go     → tests use stub Context (no gin, no router)
│       ├── <adapter>/          ← driven adapter: subpackage of the domain
│       │   └── <adapter>.go    → e.g., userfinder/, userstorage/
│       └── <adapter>/          ← another driven adapter if needed
├── pkg/                        ← reusable helpers & tech adapters — NO business logic
│   ├── config/                 → env parsing (caarlos0/env)
│   ├── framework/              → the ONLY place that imports gin — NewHandler[C] bridge
│   ├── middleware/             → CORS, security headers, body limit, timeout
│   ├── probe/                  → /liveness, /readiness, /metrics
│   ├── httpclient/             → outbound HTTP client + StatusError
│   ├── database/               → postgres/mysql/mongo connections
│   ├── logger/                 → slog with censor + GCP replacer
│   ├── otel/                   → OpenTelemetry setup
│   └── ...                     → redis, valkey, kafka, refid, reqlog
├── openapi/                    ← spec + Scalar docs
├── go.mod
└── go.sum
```

## Key Rules

1. **Domain package (`internal/<domain>/`) imports ONLY stdlib** — no gin, no pgx, no sarama, no pkg/ helpers
   - Exception: `handler.go` may import `pkg/framework` (the bridge) and `pkg/refid`
2. **Driven adapters are subpackages of the domain** — `internal/<domain>/userfinder/`, NOT `internal/adapter/http/`
3. **`pkg/` never imports `internal/`** — dependency flows inward only
4. **`main.go` is the ONLY place that wires concrete dependencies** — no business logic
5. **Domain package name = business term** — `permit`, `loan`, `booking`, NOT `service`, `handler`, `consumer`

## Domain Layer — entity.go

```go
package <domain>

import (
    "time"
)

// Request/response types owned by the domain
type SubmitTransactionRequest struct {
    TokenHash   string `json:"tokenHash" binding:"required"`
    // ... domain fields
}

// Entity — the domain owns it, adapters map to/from it
type UserProfile struct {
    CifNo     string    `json:"cifNo"`
    CitizenID string    `json:"citizenId"`
    CreatedAt time.Time `json:"createdAt"`
}
```

## Domain Layer — port.go

```go
package <domain>

import "context"

// Outbound port — adapter implements this
type UserFinder interface {
    FindUserByToken(ctx context.Context, tokenHash string) (*UserProfile, error)
}

// Persistence port
type Repository interface {
    SaveUser(ctx context.Context, user *UserProfile) error
}
```

## Domain Layer — service.go + DomainError

```go
package <domain>

import (
    "context"
    "errors"
    "fmt"
)

// Domain error codes
const (
    ErrCodeBadRequest          = 1
    ErrCodeUnauthorized        = 2
    ErrCodeForbidden           = 3
    ErrCodeNotFound            = 4
    ErrCodeConflict            = 5
    ErrCodeUpstreamUnavailable = 6
    ErrCodeTooManyRequests     = 7
    ErrCodeInternal            = 99
)

type DomainError struct {
    Code    int
    Message string
    Err     error
}

func (e *DomainError) Error() string {
    if e.Err != nil {
        return fmt.Sprintf("domain error code %d: %s: %v", e.Code, e.Message, e.Err)
    }
    return fmt.Sprintf("domain error code %d: %s", e.Code, e.Message)
}

func (e *DomainError) Unwrap() error { return e.Err }

func NewDomainError(code int, message string, cause error) *DomainError {
    return &DomainError{Code: code, Message: message, Err: cause}
}

// Service — depends only on ports
type Service struct {
    finder UserFinder
    store  Repository
}

func NewService(finder UserFinder, store Repository) *Service {
    return &Service{finder: finder, store: store}
}

func (s *Service) PermitTransaction(ctx context.Context, tokenHash string) error {
    user, err := s.finder.FindUserByToken(ctx, tokenHash)
    if err != nil {
        var domErr *DomainError
        if errors.As(err, &domErr) {
            return domErr
        }
        return NewDomainError(ErrCodeUpstreamUnavailable, "upstream service unavailable", err)
    }
    // ... business logic
    return nil
}
```

## Domain Layer — handler.go (Handler-in-Domain Pattern)

```go
package <domain>

import (
    "context"
    "errors"
    "log/slog"
    "net/http"

    "<module>/pkg/framework"
)

// Context — minimal interface, *gin.Context satisfies structurally
type Context interface {
    context.Context
    ShouldBindJSON(any) error
    JSON(code int, obj any)
}

type ErrorResponse struct {
    Error     string `json:"error"`
    ErrorCode string `json:"error_code"`
}

// Handler — driving adapter, lives IN the domain
type Handler struct {
    service PermitService  // interface, not *Service
}

type PermitService interface {
    PermitTransaction(ctx context.Context, tokenHash string) error
}

func NewHandler(service PermitService) *Handler {
    return &Handler{service: service}
}

func (h *Handler) PermitTransaction(c Context) {
    var req SubmitTransactionRequest
    if err := c.ShouldBindJSON(&req); err != nil {
        slog.WarnContext(framework.StdContext(c), "request bind failed", "error", err)
        c.JSON(http.StatusBadRequest, ErrorResponse{
            Error:     "invalid request body",
            ErrorCode: HTTPCodeValidation,
        })
        return
    }

    // Domain validates its own invariants
    if req.TokenHash == "" {
        c.JSON(http.StatusBadRequest, ErrorResponse{
            Error:     "tokenHash is required",
            ErrorCode: HTTPCodeValidation,
        })
        return
    }

    if err := h.service.PermitTransaction(framework.StdContext(c), req.TokenHash); err != nil {
        var domErr *DomainError
        if errors.As(err, &domErr) {
            statusCode, errorCode := mapDomainErrorToHTTP(domErr)
            c.JSON(statusCode, ErrorResponse{
                Error:     domErr.Message,
                ErrorCode: errorCode,
            })
            return
        }
        c.JSON(http.StatusInternalServerError, ErrorResponse{
            Error:     "internal server error",
            ErrorCode: "INTERNAL_ERROR",
        })
        return
    }

    c.JSON(http.StatusNoContent, nil)
}

// HTTP error code strings
const (
    HTTPCodeValidation          = "VALIDATION_ERROR"
    HTTPCodeNotFound            = "NOT_FOUND"
    HTTPCodeUnauthorized        = "UNAUTHORIZED"
    HTTPCodeForbidden           = "FORBIDDEN"
    HTTPCodeTooManyRequests     = "TOO_MANY_REQUESTS"
    HTTPCodeUpstreamUnavailable = "UPSTREAM_UNAVAILABLE"
    HTTPCodeInternal            = "INTERNAL_ERROR"
)

func mapDomainErrorToHTTP(err *DomainError) (statusCode int, errorCode string) {
    switch err.Code {
    case ErrCodeBadRequest:
        return http.StatusBadRequest, HTTPCodeValidation
    case ErrCodeUnauthorized:
        return http.StatusUnauthorized, HTTPCodeUnauthorized
    case ErrCodeForbidden:
        return http.StatusForbidden, HTTPCodeForbidden
    case ErrCodeNotFound:
        return http.StatusNotFound, HTTPCodeNotFound
    case ErrCodeConflict:
        return http.StatusConflict, "CONFLICT"
    case ErrCodeTooManyRequests:
        return http.StatusTooManyRequests, HTTPCodeTooManyRequests
    case ErrCodeUpstreamUnavailable:
        return http.StatusServiceUnavailable, HTTPCodeUpstreamUnavailable
    default:
        return http.StatusInternalServerError, HTTPCodeInternal
    }
}
```

## Framework Bridge — pkg/framework/gin.go

```go
package framework

import (
    "context"
    "github.com/gin-gonic/gin"
)

// NewHandler adapts any domain handler func(C) into gin.HandlerFunc
func NewHandler[C any](handler func(C)) gin.HandlerFunc {
    return func(ctx *gin.Context) {
        handler(any(ctx).(C))
    }
}

// StdContext unwraps *gin.Context → request context for middleware values
func StdContext(c context.Context) context.Context {
    if gc, ok := c.(*gin.Context); ok && gc.Request != nil {
        return gc.Request.Context()
    }
    return c
}
```

## Driven Adapter — internal/<domain>/userfinder/user_finder.go

```go
package userfinder

import (
    "context"
    "errors"
    "net/http"

    "<module>/internal/<domain>"
    "<module>/pkg/httpclient"
)

// Wire envelope stays in the adapter — domain never sees it
type getUserByTokenResponse struct {
    Code    int                    `json:"code"`
    Message string                 `json:"message"`
    Data    *<domain>.UserProfile  `json:"data,omitempty"`
}

type userFinder struct {
    url    string
    client *http.Client
}

func NewUserFinder(client *http.Client, baseURL string) <domain>.UserFinder {
    return &userFinder{client: client, url: baseURL}
}

// Anti-corruption layer: translate HTTP errors → DomainError at the boundary
func (f *userFinder) FindUserByToken(ctx context.Context, tokenHash string) (*<domain>.UserProfile, error) {
    resp, err := httpclient.Post[<domain>.SubmitTransactionRequest, getUserByTokenResponse](ctx, f.client, f.url, req)
    if err != nil {
        var statusErr *httpclient.StatusError
        if errors.As(err, &statusErr) {
            return nil, mapHTTPStatusToDomainError(statusErr)
        }
        return nil, <domain>.NewDomainError(<domain>.ErrCodeUpstreamUnavailable, "upstream service unavailable", err)
    }
    return resp.Response.Data, nil
}

func mapHTTPStatusToDomainError(err *httpclient.StatusError) *<domain>.DomainError {
    if err.Is4xx() {
        switch err.StatusCode {
        case http.StatusUnauthorized:
            return <domain>.NewDomainError(<domain>.ErrCodeUnauthorized, "unauthorized", err)
        case http.StatusNotFound:
            return <domain>.NewDomainError(<domain>.ErrCodeNotFound, "resource not found", err)
        // ... other 4xx
        default:
            return <domain>.NewDomainError(<domain>.ErrCodeBadRequest, err.Message, err)
        }
    }
    return <domain>.NewDomainError(<domain>.ErrCodeUpstreamUnavailable, "upstream service unavailable", err)
}
```

## Driven Adapter — internal/<domain>/userstorage/user_storage.go

```go
package userstorage

import (
    "context"
    "<module>/internal/<domain>"
    "github.com/jackc/pgx/v5/pgxpool"
)

type storage struct {
    db *pgxpool.Pool
}

func NewStorage(db *pgxpool.Pool) <domain>.Repository {
    return &storage{db: db}
}

func (s *storage) SaveUser(ctx context.Context, user *<domain>.UserProfile) error {
    _, err := s.db.Exec(ctx, "insert into users(cif_no, cid) values($1, $2)", user.CifNo, user.CitizenID)
    return err
}
```

## Composition Root — main.go

```go
package main

func main() {
    // 1. Config
    cfg := config.C(config.Env)

    // 2. OTel + Logger
    // ...

    // 3. Wire dependencies — ALL wiring here, NO logic
    r := gin.New()
    r.Use(middleware.SecurityHeaders, middleware.BodyLimit, otelgin.Middleware(...))

    // Probes
    r.GET("/liveness", probe.Liveness(version, commit))
    r.GET("/readiness", probe.Readiness())
    r.GET("/metrics", probe.Metrics())

    // Domain wiring
    httpClient := httpclient.NewHTTPClient()
    db := database.NewPostgresDB(cfg.Database.PostgresURL)
    userFinder := userfinder.NewUserFinder(httpClient, cfg.ServiceCoreDltAccountUrl)
    userStorage := userstorage.NewStorage(db)
    service := <domain>.NewService(userFinder, userStorage)
    h := <domain>.NewHandler(service)

    // framework.NewHandler is the ONLY bridge between gin and the domain
    r.POST("/interpermits", framework.NewHandler(h.PermitTransaction))

    // 4. Start + graceful shutdown
    srv := &http.Server{Addr: ":" + cfg.Server.Port, Handler: r}
    go gracefully(srv)
    srv.ListenAndServe()
}
```

## Testing — Stub Context (no gin, no router)

```go
package <domain>

import (
    "context"
    "encoding/json"
    "net/http"
)

type stubContext struct {
    ctx      context.Context
    body     string
    jsonCode int
    jsonBody any
}

func (s *stubContext) Deadline() (time.Time, bool) { return s.ctx.Deadline() }
func (s *stubContext) Done() <-chan struct{}        { return s.ctx.Done() }
func (s *stubContext) Err() error                   { return s.ctx.Err() }
func (s *stubContext) Value(key any) any            { return s.ctx.Value(key) }
func (s *stubContext) ShouldBindJSON(v any) error   { return json.Unmarshal([]byte(s.body), v) }
func (s *stubContext) JSON(code int, obj any)       { s.jsonCode = code; s.jsonBody = obj }

func TestHandler_BadRequest(t *testing.T) {
    svc := &stubService{}
    h := NewHandler(svc)
    c := &stubContext{ctx: context.Background(), body: `{}`}
    h.PermitTransaction(c)
    if c.jsonCode != http.StatusBadRequest {
        t.Errorf("got %d, want %d", c.jsonCode, http.StatusBadRequest)
    }
}
```

## Checklist Before Committing

- [ ] `internal/<domain>/*.go` imports only stdlib (+ pkg/framework in handler.go)
- [ ] No `internal/adapter/` directory — adapters are domain subpackages
- [ ] Domain package name is a business term, not an architecture term
- [ ] `main.go` has no business logic — only New, wiring, signal handling
- [ ] Handler depends on interface (PermitService), not concrete (*Service)
- [ ] Every adapter error returns *DomainError (anti-corruption layer)
- [ ] `var _ Port = (*Impl)(nil)` compile-time checks in adapter tests
- [ ] `go build ./... && go vet ./... && go test ./...` all pass

## HTTP Status Convention

14 codes: 200/201/204, 400/401/403/404/409/422/429, 500/502/503/504
- 4xx = no retry (except 429)
- 5xx = retry with backoff
- Every error response has `error_code` in body
