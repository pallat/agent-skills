---
description: Cross Functional Requirements — audit Go service repos for consistency, orphans, dangling references, and correctness. Use when reviewing or cleaning up Go microservice repositories.
---

# Cross Functional Requirements (CFR)

Audit framework for Go service repos. Every check asks: **does what exists agree with what else exists?** — not "does a specific file exist."

Five audit dimensions:

| Dimension | Question | Action |
|---|---|---|
| **Consistency** | A says X — does B agree? | Fix the mismatch |
| **Orphans** | A exists — does anything use it? | Remove or wire up |
| **Dangling** | A references B — does B exist? | Fix or remove reference |
| **Gates** | Do mandatory commands pass? | Fix until green |
| **Security** | Do forbidden patterns appear? | Eliminate |

*(Adv)* = recommended.

---

## 0. Prerequisites

Tools required before running this audit:

- `go` (toolchain matching `go.mod`)
- `golangci-lint` (v2)
- `govulncheck`
- `deadcode` *(Adv)*

Install missing tools before proceeding. A gate that cannot run is not a gate that passed.

---

## 1. Build & Test Gates

- `go build ./...` — all packages compile.
- `go vet ./...` — no suspicious constructs.
- `go test -race ./...` — all green, no data race.
- `go mod tidy` — no unused requires.
- `go mod verify` — checksums valid.
- `govulncheck ./...` — no called CVEs.
- `golangci-lint run` — passes (including `unused`, `deadcode` if enabled).
- Domain packages must have test coverage ≥ 60%. Verify with `go test -cover ./internal/...`.

---

## 2. Cross-File Consistency

Mismatch between files that should agree = bug.

- **Go version**: `go.mod` directive == Dockerfile builder image == CI `GO_VERSION`.
- **Module path**: `go.mod` module path == actual repo URL.
- **OTel service name**: `OTEL_SERVICE_NAME` default == repo name == Prometheus `job_name`.
- **Version**: `VERSION` file == Docker `LABEL version` == health probe response == latest git tag.
- **Env vars ↔ code**: every var in `.env.template` is read by code (config struct, `os.Getenv`, or OTel SDK); every env var read by code is in `.env.template`. No orphans either direction. Check `main.go` and all config packages, not just one.
- **README env vars ↔ `.env.template`**: documented vars match template exactly.
- **README repo name == actual repo name**.
- **README quickstart commands**: every `make <target>` and file path referenced in README exists.
- **OpenAPI spec ↔ generated code**: `openapi.gen.go` not stale vs `openapi.yaml` (`make openapi-check` exits 0).
- **Makefile targets ↔ CI**: every target CI invokes exists and works.
- **Health endpoints ↔ K8s/compose probes**: `/liveness`, `/readiness` paths in code match probe config.
- **Dockerfile `ARG` ↔ usage**: every declared `ARG` is consumed in a build stage.
- **Default values**: config defaults in code match documented defaults in README and `.env.template`.
- **CHANGELOG ↔ reality**: latest entry reflects actual changes. `[Unreleased]` section is acceptable; if it lists changes, they must match the diff since last tag. A dated released entry must exist for the current `VERSION`.

---

## 3. Orphan Detection — Exists But Unused

Things present in the repo that nothing references or invokes. Remove or wire up.

- **Env vars**: `.env.template` defines vars no code path reads → remove.
- **Scripts**: shell scripts never invoked by Makefile, CI, CONTRIBUTING, or another script → remove or wire up.
- **Makefile targets**: not called by CI, not in `make help`, not documented → question relevance.
- **Config struct fields**: populated from env but never read by business logic → remove.
- **Dependencies**: `go.mod` requires modules never imported → `go mod tidy`.
- **Dead code**: exported funcs/types never called outside own package → remove.
- **Linter exclusions**: `path`/`exclude` rules for directories that don't exist → remove stale rules.
- **`.gitignore` / `.dockerignore` entries**: patterns that never match anything → remove noise.
- **docker-compose services**: defined but not needed for local dev → remove.
- **Docker build stages**: never referenced by `COPY --from=` → remove.
- **Test files**: `_test.go` for packages that no longer exist → remove.

---

## 4. Dangling References — Points To Nothing

Things that reference targets that don't exist. Fix or remove the reference.

- **Scripts → files**: script references file paths (configs, binaries, other scripts) that don't exist.
- **Makefile → scripts/binaries**: target calls a script or binary not in repo or not on `PATH`.
- **CI → targets/images**: pipeline calls Makefile targets or container images that don't exist.
- **CONTRIBUTING → steps**: references setup scripts or targets that don't exist.
- **Code → config keys**: code reads env vars not in `.env.template` → add to template or remove code.
- **Imports → local packages**: code imports internal packages that don't exist or were moved.

---

## 5. Security — Forbidden Patterns

Must not appear anywhere in the repo.

- Hardcoded credentials in code, config, Dockerfile, scripts, or compose.
- Real passwords/tokens/keys in `.env.template` — placeholders only.
- `InsecureSkipVerify: true` in TLS config.
- `:latest` image tags in Dockerfile or compose — pin by digest or explicit tag.
- DLQ/error headers leaking full stack traces or sensitive data.
- Docker running as root — must have non-root `USER`.
- *(Adv)* SOPS/age encryption for `.env`; secret detection in CI; `gosec` in linter.

---

## 6. Dockerfile Correctness

- Multi-stage: builder compiles, runtime on minimal image (Alpine + `tini`/`--init`).
- Static binary: `CGO_ENABLED=0 go build -trimpath -ldflags="-s -w"`.
- `HEALTHCHECK` hits liveness endpoint. `COPY --link` for layers. `--mount=type=cache` for build cache.
- Commit hash embedded via `-ldflags -X main.commit=...`.

---

## 7. Observability Correctness

- OTel SDK (traces + metrics) in reusable package. Structured logging injects `trace_id`.
- `/metrics` (Prometheus), `/liveness`, `/readiness` exposed.
- Traces and metrics individually disableable via env flags (no-op when off, not crash). Flags may be read in `main.go` or the OTel package — both are valid.
- W3C `traceparent` propagated inbound and outbound.

---

## 8. Structural Conventions

- **Screaming architecture**: domain packages named by business domain (`permit`, `consumer`), not technical role (`handler`, `service`, `util`).
- `main.go` is composition root — wires deps, no business logic.
- `internal/` holds domain + usecase + adapter; `pkg/` holds reusable tech packages.
- Clean-arch repos: domain imports stdlib only. `port.go` defines interfaces; adapters implement.

---

## 9. Linter Config

- golangci-lint v2 (`version: "2"`).
- Minimum: `errcheck`, `govet`, `staticcheck`, `unused`, `gosec`, `revive`, `gocritic`, `bodyclose`.
- *(Adv)* `funlen` (≤100), `gocognit` (≤20), `sloglint`, `perfsprint`, `unparam`, `nestif`, `copyloopvar`, `intrange`.
- `gomodguard_v2` blocks deprecated modules.
- Test files excluded from `funlen`, `gocognit`, `gosec`; generated code excluded from `unused`, `staticcheck`.

---

## 10. HTTP Conventions *(HTTP only)*

- Must not use status codes outside this set: 200/201/204, 400/401/403/404/409/422/429, 500/502/503/504.
- 4xx = no retry, 5xx = retry. Every error body includes `error_code`.
- Orchestration branches on 2xx/4xx/5xx only.

---

## 11. CI Pipeline

- Stages: validate → test → containerize → scan → deploy.
- Validate: tag + Dockerfile validation. Test: coverage, secret detection, dep scan. Scan: container + SAST.
- *(Adv)* OpenAPI spec drift check. *(Adv)* Automated dependency update bot (e.g. Renovate, Dependabot) configured.
