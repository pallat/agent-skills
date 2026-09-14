---
description: Cross Functional Requirements — standards for Go service repos. Use when creating, auditing, or reviewing Go microservice repositories for structural and operational compliance.
---

# Cross Functional Requirements (CFR)

Mandatory standards for every Go service repository. *(Adv)* = recommended.

## 1. Module & Dependencies
- Module path must be fully-qualified and organization-scoped.
- Go version in `go.mod` must match Dockerfile builder and CI runner.
- Prefer stdlib — add external deps only when stdlib cannot do the job.
- Pin all deps to concrete versions; no floating tags.
- `go mod tidy`, `go mod verify`, `govulncheck ./...` must pass clean.
- Block deprecated modules via linter config. Use modern features (generics, `range over func`) where they help.

## 2. Build, Test, Vet
- `go build ./...`, `go vet ./...`, `go test -race ./...` must pass.
- Every domain package needs ≥1 `_test.go` covering core logic.
- No dead code — verify via `deadcode` or `unused` linter.
- Failing test before fix (TDD: RED → GREEN → REFACTOR).

## 3. Dockerfile
- Multi-stage: builder compiles, runtime runs binary on minimal image (Alpine or equiv) with init process (`tini`/`--init`).
- Non-root user, `HEALTHCHECK` at liveness endpoint, `COPY --link`, `--mount=type=cache`.
- Static binary: `CGO_ENABLED=0 go build -trimpath -ldflags="-s -w"`. Embed commit hash via `-ldflags -X`.
- Pin base image by digest or explicit tag — never `:latest`.

## 4. Makefile
- Required targets: `build`, `test`, `vet`, `lint`, `run`, `clean`, `deps`, `coverage`.
- `test` uses `-race -count=1`; `build` uses `CGO_ENABLED=0 -trimpath -ldflags="-s -w"`.
- *(Adv)* `vuln`, `precommit`, `ci`, `docker` targets.

## 5. Linter Config
- golangci-lint v2 (`version: "2"`). Minimum: `errcheck`, `govet`, `staticcheck`, `unused`, `gosec`, `revive`, `gocritic`, `bodyclose`.
- *(Adv)* `funlen` (≤100), `gocognit` (≤20), `sloglint`, `perfsprint`, `unparam`, `nestif`, `copyloopvar`, `intrange`.
- Exclude test files from `funlen`, `gocognit`, `gosec`; exclude generated code from `unused`, `staticcheck`.

## 6. Environment Template
- No hardcoded secrets — placeholders only. Include observability keys, `LOG_LEVEL`, health probe config.
- Every variable maps to a config struct field — no orphans. TLS/SASL defaults disabled.

## 7. Ignore Files
- `.gitignore`: secrets, build output, coverage, editor dirs, OS artifacts.
- `.dockerignore`: `.git`, secrets, Makefile, compose files, docs (except README), build output.
- Agent/CI artifacts must not be committed.

## 8. Observability
- OTel SDK (traces + metrics) in a reusable package. Structured logging injects `trace_id`.
- Expose `/metrics` (Prometheus), `/liveness`, `/readiness`.
- Traces and metrics individually disableable via env flags (no-op handlers).
- Propagate W3C `traceparent` inbound and outbound.

## 9. OpenAPI *(HTTP only)*
- Spec, generated Go code, and codegen config in a dedicated directory.
- `make openapi-gen` regenerates; `make openapi-check` fails CI on stale code. Codegen tool version pinned.

## 10. CI Pipeline
- Stages: validate → test → containerize → scan → deploy.
- Validate: tag + Dockerfile validation. Test: coverage, secret detection, dep scan. Scan: container + SAST.
- CI Go version must match `go.mod`. *(Adv)* OpenAPI spec drift check.

## 11. README
- Document architecture layout, dependency rules, quickstart, observability section, and project structure tree.
- List all environment variables referencing the env template. Repo name must match actual name.

## 12. CHANGELOG & CONTRIBUTING
- CHANGELOG has a dated latest entry.
- CONTRIBUTING covers dev setup, pre-commit steps, and how to add a domain package.
- Clean-arch repos: CONTRIBUTING explains architecture rules.

## 13. Scripts
- `setup.sh` and `bump-version.sh` must exist and be executable. No script contains hardcoded secrets.
- *(Adv)* Dev dep installer, pre-commit setup, commit-msg convention, test colorizer.

## 14. Security
- No hardcoded credentials anywhere. Docker non-root; no `InsecureSkipVerify: true` in TLS config.
- Env template placeholders only; DLQ/error headers must not leak sensitive data. `govulncheck` passes.
- *(Adv)* SOPS/age env encryption, `gosec` in linter, secret detection in CI.

## 15. Version & Release
- `VERSION` file using semver (`vX.Y.Z`). Version in Docker label and health probe response.
- `make bump-version` updates VERSION and creates git tag.

## 16. docker-compose
- Include only services needed for local dev. Service/cluster names reference repo name.
- *(Adv)* Volume mounts for persistence; network isolation.

## 17. HTTP Status Convention *(HTTP only)*
- 14 codes: 200/201/204, 400/401/403/404/409/422/429, 500/502/503/504.
- 4xx = no retry, 5xx = retry. Every error body includes `error_code`. Orchestration branches on 2xx/4xx/5xx only.
