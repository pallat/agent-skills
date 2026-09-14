---
description: Cross Functional Requirements — standards for Go service repos. Use when creating, auditing, or reviewing Go microservice repositories for structural and operational compliance.
---

# Cross Functional Requirements (CFR)

Standards that every Go service repository must satisfy. Each section defines mandatory rules; items marked *(Advanced)* are recommended but not required for compliance.

## 1. Go Module & Dependencies

- Module path must be a fully-qualified, organization-scoped path (no personal or placeholder domains).
- Go version in `go.mod` must match the builder image in Dockerfile and the CI runner.
- Prefer stdlib over external dependencies — add a third-party package only when stdlib cannot do the job.
- Pin all dependencies to concrete versions; no `latest` or floating tags.
- Run `go mod tidy`, `go mod verify`, and `govulncheck ./...` — all must pass with zero output.
- Block deprecated modules via linter configuration (e.g. `gomodguard`).
- Use modern language features (generics, `range over func`, `copyloopvar`) where they improve clarity.

## 2. Build, Test, Vet

- `go build ./...`, `go vet ./...`, and `go test -race ./...` must all pass.
- Every domain package must have at least one `_test.go` file covering core logic.
- No dead code — verify with `deadcode` or an `unused` linter.
- A failing test must be written before the fix that makes it pass (TDD: RED → GREEN → REFACTOR).

## 3. Project Structure

- Package names must reflect business domains (`permit`, `loan`, `booking`), never technical roles (`handler`, `service`, `util`).
- `main.go` is the composition root — it wires dependencies and contains no business logic.
- `internal/` holds domain, usecase, and adapter packages; nothing inside it is importable from outside the module.
- `pkg/` holds reusable technical packages; it must never import from `internal/`.
- In clean-architecture repos: domain packages import only stdlib; adapters live in domain subpackages and implement interfaces declared in `port.go`.

## 4. Dockerfile

- Multi-stage build: builder stage compiles, runtime stage runs the binary.
- Runtime base image must be minimal (Alpine or equivalent) with an init process (`tini` or `--init`) for proper signal handling.
- Run as a non-root user — create a dedicated user and switch to it.
- Include a `HEALTHCHECK` directive pointing at the liveness endpoint.
- Use `COPY --link` for layer efficiency and `--mount=type=cache` for module cache.
- Build a static binary: `CGO_ENABLED=0 go build -trimpath -ldflags="-s -w"`.
- Embed commit hash via build arg and `-ldflags -X`.
- Pin base image by digest or explicit tag — never `:latest`.
- Go version in the builder image must match `go.mod`.

## 5. Makefile

- Required targets: `build`, `test`, `vet`, `lint`, `run`, `clean`, `deps`.
- `test` must use `-race -count=1`.
- `build` must use `CGO_ENABLED=0 -trimpath -ldflags="-s -w"`.
- Include `coverage` target with `-coverprofile`.
- *(Advanced)* `vuln` (govulncheck), `precommit` (all checks + verify + vet), `ci` (precommit + diff), `docker` (with build args).

## 6. Linter Configuration

- Use golangci-lint v2 format (`version: "2"`).
- Minimum linters: `errcheck`, `govet`, `staticcheck`, `unused`, `gosec`, `revive`, `gocritic`, `bodyclose`.
- *(Advanced)* Add: `funlen` (≤100), `gocognit` (≤20), `sloglint`, `perfsprint`, `unparam`, `nestif`, `copyloopvar`, `intrange`.
- Exclude test files from `funlen`, `gocognit`, `gosec`.
- Exclude generated code from `unused`, `staticcheck`.

## 7. Environment Template

- No hardcoded secrets — use placeholders only.
- Include observability config keys (service name, exporter endpoint, insecure flag).
- Include `LOG_LEVEL` (DEBUG/INFO/WARN/ERROR).
- Include health probe config (address or port).
- Every variable must map to a config struct field — no orphan variables.
- TLS/SASL defaults must be safe (disabled by default).

## 8. Ignore Files

- `.gitignore` must exclude: secrets files, build output, coverage files, editor directories, OS artifacts.
- `.dockerignore` must exclude: `.git`, secrets, Makefile, compose files, docs (except README), build output.
- Agent/CI artifacts must not be committed.

## 9. Observability

- OTel SDK must set up traces and metrics in a reusable package.
- Structured logging must inject `trace_id` for log-trace correlation.
- Expose `/metrics` (Prometheus) and health probes (`/liveness`, `/readiness`).
- Traces and metrics must be individually disableable via env flags while keeping no-op handlers.
- Propagate W3C `traceparent` on both inbound and outbound calls.

## 10. OpenAPI *(HTTP services only)*

- Spec file, generated Go code, and codegen config must all exist under a dedicated directory.
- `make openapi-gen` regenerates; `make openapi-check` fails CI if generated code is stale.
- Codegen tool version must be pinned.

## 11. CI Pipeline

- Stages: validate → test → containerize → scan → deploy.
- Validate: tag and Dockerfile validation.
- Test: coverage, secret detection, dependency scan.
- Scan: container image scan and SAST.
- Go version in CI must match `go.mod`.
- *(Advanced)* OpenAPI spec drift check.

## 12. README

- Document architecture layout and dependency rules.
- Provide quickstart: setup, env copy, compose up, run.
- Include an observability section and a project structure tree.
- List all environment variables with references to the env template.
- Repository name in README must match the actual repo name.

## 13. CHANGELOG & CONTRIBUTING

- CHANGELOG must have a dated entry for the latest change.
- CONTRIBUTING must cover: dev setup, pre-commit steps, how to add a domain package.
- In clean-arch repos, CONTRIBUTING must explain the architecture rules.

## 14. Scripts

- Setup and version-bump scripts must exist and be executable.
- *(Advanced)* Dev dependency installer, pre-commit hook setup, commit message convention, test output colorizer.
- No script may contain hardcoded secrets — read from environment variables.

## 15. Security

- No hardcoded credentials anywhere in the repository.
- Docker must run as non-root.
- No `InsecureSkipVerify: true` in any TLS config.
- Env template must contain placeholders, never real credentials.
- DLQ or error headers must not leak stack traces or sensitive data.
- `govulncheck` must pass.
- *(Advanced)* SOPS/age encryption for env files, `gosec` in linter config, secret detection in CI.

## 16. Version & Release

- A `VERSION` file using semver (`vX.Y.Z`).
- Version must appear in Docker label and health probe response.
- A `make bump-version` target updates VERSION and creates a git tag.

## 17. docker-compose

- Include only services needed for local development of this repo.
- Service and cluster names must reference the repo name.
- *(Advanced)* Volume mounts for persistence; network isolation to prevent unintended port exposure.

## 18. HTTP Status Convention *(HTTP services only)*

- Use exactly 14 status codes: 200/201/204, 400/401/403/404/409/422/429, 500/502/503/504.
- 4xx errors are not retryable; 5xx errors are retryable.
- Every error response body must include a machine-readable `error_code` field.
- Orchestration layers should branch on three groups only: 2xx, 4xx, 5xx.
