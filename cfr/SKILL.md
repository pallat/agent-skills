---
description: Codebase Fitness Review — มาตรฐานร่วมสำหรับ review Go template repos (go-ms-otel, go-ms-otel-simple, go-kafka-otel-clean, go-kafka-otel-simple). Use when auditing, reviewing, or creating Go service repos under techcoach/template.
---

## Template Repository Checklist — มาตรฐานร่วม 4 Repos

> Repos: `go-ms-otel` · `go-ms-otel-simple` · `go-kafka-otel-clean` · `go-kafka-otel-simple`

---

## 1. Go Module & Dependencies

- [ ] **go.mod ใช้ module path รูปแบบ `gitdev.devops.krungthai.com/techcoach/template/<repo-name>.git`**
- [ ] **Go version ล่าสุด** — ตอนนี้คือ `go 1.27.x` ต้องตรงกับ Dockerfile builder image เสมอ
- [ ] **ใช้ lib เวอร์ชันล่าสุดเสมอ** — ตรวจด้วย `go list -m -u all` เป็น periodic task
- [ ] **ใช้ stdlib ก่อนเสมอ** — ถ้า stdlib ทำได้ ไม่เพิ่ม external dependency (เช่น `net/http` ก่อน gin, `slog` ก่อน zap)
- [ ] **ใช้ newer Go features** — generics, `range over func`, `copyloopvar`, `intrange` ถ้าเหมาะสม
- [ ] **ไม่มี deprecated lib** — ตรวจด้วย `gomodguard_v2` ใน golangci-lint (block `golang/protobuf`, `satori/go.uuid`, `gofrs/uuid`)
- [ ] **`go mod tidy` ผ่าน** — ไม่มี unused require ค้างไว้
- [ ] **`go mod verify` ผ่าน** — checksum ถูกต้อง
- [ ] **`govulncheck ./...` ผ่าน** — ไม่มี known CVE ใน code ที่เรียกใช้จริง

---

## 2. Build, Test, Vet

- [ ] **`go build ./...` ผ่าน** — compile ทุก package ได้
- [ ] **`go vet ./...` ผ่าน** — ไม่มี suspicious construct
- [ ] **`go test -race ./...` ผ่าน** — ทุก test green, ไม่มี data race
- [ ] **มี test ครอบคลุม domain layer เสมอ** — service_test.go อย่างน้อย 1 ไฟล์
- [ ] **ไม่มี dead code** — ตรวจด้วย `deadcode ./...` หรือ `unused` linter ใน golangci-lint
- [ ] **test ที่ fail ต้องเขียนก่อนแก้ bug** — TDD: RED → GREEN → REFACTOR

---

## 3. Project Structure

- [ ] **Screaming architecture** — domain package ตั้งชื่อตาม business domain (`permit`, `consumer`, `interpermit`) ไม่ใช่ technical role (`handler`, `service`, `util`)
- [ ] **`main.go` เป็น composition root** — wire dependencies ที่นี่ ไม่มี business logic
- [ ] **`internal/` บรรจุ domain + usecase + adapter** — ไม่ expose ออกนอก module
- [ ] **`pkg/` บรรจุ reusable tech packages** — config, otel, logger, probe, kafka เป็นต้น
- [ ] **Clean architecture (ถ้า repo แบบ clean)** — domain package imports เฉพาะ stdlib, ไม่มี sarama/gin/otel ใน domain
- [ ] **Port/Adapter separation (ถ้า repo แบบ clean)** — `port.go` นิยาม interfaces, adapter subpackages implement

---

## 4. Dockerfile

- [ ] **Multi-stage build** — builder stage (golang:alpine) → runtime stage (alpine)
- [ ] **Alpine + tini** — ไม่ใช้ Chainguard, ใช้ tini สำหรับ SIGTERM จาก K8s
- [ ] **Non-root USER** — `adduser -D -H appuser` + `USER appuser`
- [ ] **HEALTHCHECK** — ใช้ `wget --spider http://localhost:8080/liveness`
- [ ] **`COPY --link`** — ใช้ copylink สำหรับ layer efficiency
- [ ] **Build cache mount** — `--mount=type=cache,target=/go/pkg/mod`
- [ ] **Static binary** — `CGO_ENABLED=0 go build -trimpath -ldflags="-s -w"`
- [ ] **`ARG GIT_COMMIT`** — ฝัง commit hash ผ่าน `-X main.commit=$GIT_COMMIT`
- [ ] **Image pin** — ระบุ tag ชัดเจน (ไม่ใช้ `:latest`) ถ้ามี digest ยิ่งดี
- [ ] **GO_VERSION ใน Dockerfile ตรงกับ go directive ใน go.mod**

---

## 5. Makefile

- [ ] **มี targets: `build`, `test`, `vet`, `lint`, `run`, `clean`, `deps`**
- [ ] **`test` ใช้ `-race -count=1`** — detect data race, ไม่ cache
- [ ] **`lint` ใช้ `golangci-lint run`**
- [ ] **`build` ใช้ `CGO_ENABLED=0 -trimpath -ldflags="-s -w"`**
- [ ] **มี `coverage` target** — `-coverprofile` + `go tool cover`
- [ ] **มี `bump-version version=vX.Y.Z` target**
- [ ] *(ขั้นสูง)* **มี `vuln` target** — `govulncheck ./...`
- [ ] *(ขั้นสูง)* **มี `precommit` target** — `all + vuln + go mod verify + go vet`
- [ ] *(ขั้นสูง)* **มี `ci` target** — `precommit + diff`
- [ ] *(ขั้นสูง)* **มี `docker` target** — ส่ง `GIT_COMMIT` + `VERSION` เป็น build-arg

---

## 6. .golangci.yaml

- [ ] **golangci-lint v2 format** — `version: "2"`
- [ ] **Linters ขั้นต่ำ:** `errcheck`, `govet`, `staticcheck`, `unused`, `gosec`, `revive`, `gocritic`, `bodyclose`
- [ ] *(ขั้นสูง)* **Linters เพิ่ม:** `funlen` (≤100 lines), `gocognit` (≤20 complexity), `sloglint`, `perfsprint`, `unparam`, `whitespace`, `nestif`, `forbidigo`, `predeclared`, `usestdlibvars`, `copyloopvar`, `intrange`
- [ ] **`gomodguard_v2`** — block deprecated modules
- [ ] **Test files มี exclusion** — ผ่อน `funlen`, `gocognit`, `gosec` ใน `_test.go`
- [ ] **Generated code มี exclusion** — `openapi.gen.go` ผ่อน `unused`, `staticcheck`

---

## 7. .env.template

- [ ] **ไม่มี hardcoded secrets** — ใช้ placeholder หรือค่าว่าง
- [ ] **มี OTel config** — `OTEL_SERVICE_NAME`, `OTEL_EXPORTER_OTLP_ENDPOINT`, `OTEL_EXPORTER_OTLP_INSECURE`
- [ ] **มี `LOG_LEVEL`** — DEBUG/INFO/WARN/ERROR
- [ ] **มี health probe config** — `HEALTH_ADDR` หรือ `PORT`
- [ ] **OTEL_SERVICE_NAME ตรงกับชื่อ repo**
- [ ] **TLS/SASL config มี default ปลอดภัย** — `KAFKA_TLS_ENABLE=false`, `KAFKA_SASL_ENABLE=false`
- [ ] **ทุก variable map ไปยัง config struct** — ไม่มี orphan env var

---

## 8. .gitignore & .dockerignore

- [ ] **`.gitignore` มี:** `.env`, `bin/`, `coverage.out`, `.idea/`, `.vscode/`, `*.exe`, `.DS_Store`
- [ ] **`.dockerignore` มี:** `.git`, `.env`, `Makefile`, `docker-compose.yml`, `*.md` (ยกเว้น `!README.md`), `bin/`, `coverage.*`
- [ ] **Hermes artifacts ไม่ commit** — `.hermes/`, `.security-review-plan.md`

---

## 9. OpenTelemetry & Observability

- [ ] **OTel SDK setup** — traces + metrics ใน `pkg/otel/`
- [ ] **slog handler ฉีด trace_id** — log และ trace correlate
- [ ] **Prometheus `/metrics` endpoint** — ใช้ `prometheus/client_golang` หรือ OTel exporter
- [ ] **Health probes** — `/liveness` (K8s liveness), `/readiness` หรือ `/readyz`
- [ ] **`OTEL_TRACES_ENABLED` flag** — สามารถ disable ได้โดยยังมี tracer (NeverSample)
- [ ] **`OTEL_METRICS_ENABLED` flag** — สามารถ disable ได้
- [ ] **W3C traceparent propagation** — รับ inbound, ฉีด outbound

---

## 10. OpenAPI (ถ้าเป็น HTTP service)

- [ ] **`openapi/openapi.yaml`** — spec นิยาม API
- [ ] **`openapi/openapi.gen.go`** — generated code จาก spec
- [ ] **`openapi/codegen.yaml`** — oapi-codegen config
- [ ] **`make openapi-gen`** — regenerate ได้
- [ ] **`make openapi-check`** — fail ถ้า generated code stale (CI gate)
- [ ] **oapi-codegen version ถูก pin** — ป้องกัน drift ระหว่าง dev กับ CI

---

## 11. GitLab CI (ถ้ามี)

- [ ] **`gitlabci.yml` มี stages:** validate → test → containerize → scan → deploy
- [ ] **Validate:** tag validation, Dockerfile validation
- [ ] **Test:** coverage test (Go), secret detection, dependency scan (OWASP)
- [ ] **Containerize:** build Docker image
- [ ] **Scan:** SonarQube, Rapid7 (container scan), NexusIQ
- [ ] **`GO_VERSION` ใน CI ตรงกับ go directive ใน go.mod**
- [ ] **OpenAPI spec drift check** — ถ้ามี openapi/

---

## 12. README.md

- [ ] **อธิบาย architecture** — layout, dependency rule, domain purity (ถ้า clean)
- [ ] **Quickstart** — `make setup`, `cp .env.template .env`, `docker compose up`, `make run`
- [ ] **OTel section** — อธิบาย traces, metrics, logs correlation
- [ ] **Project structure tree** — แสดงโครงสร้างไดเรกทอรี
- [ ] **Environment variables** — อ้างอิง `.env.template`
- [ ] **ชื่อ repo ใน README ตรงกับชื่อจริง**

---

## 13. CHANGELOG.md & CONTRIBUTING.md

- [ ] **CHANGELOG.md มี entry ล่าสุด** — วันที่, สรุปการเปลี่ยนแปลง
- [ ] **CONTRIBUTING.md มี:** development setup, before-commit steps, วิธี add domain
- [ ] **CONTRIBUTING.md อธิบาย clean architecture rules** (ถ้า repo แบบ clean)

---

## 14. Scripts & Infrastructure Files

- [ ] **`.scripts/setup.sh`** — install deps + tools
- [ ] **`.scripts/bump-version.sh`** — bump VERSION + tag
- [ ] **`.scripts/observability/prometheus.yml`** — scrape config, `job_name` ตรงกับ repo
- [ ] *(ขั้นสูง)* **`.scripts/install-deps.sh`** — ติดตั้ง dev dependencies
- [ ] *(ขั้นสูง)* **`.scripts/setup-pre-commit.sh`** — pre-commit hooks
- [ ] *(ขั้นสูง)* **`.scripts/commit-msg.sh`** — commit message convention
- [ ] *(ขั้นสูง)* **`.scripts/colorize`** — สี output ของ `make test`
- [ ] **ทุก script มี `chmod +x`** — executable
- [ ] **Scripts ไม่มี hardcoded secrets** — รับจาก env var

---

## 15. Security (Secured Dev Perspective)

- [ ] **ไม่มี hardcoded credentials** ใน code, config, Dockerfile, scripts
- [ ] **Docker non-root user** — `USER appuser`
- [ ] **ไม่มี `InsecureSkipVerify: true`** ใน TLS config
- [ ] **`.env.template` ไม่มี password จริง** — placeholder เท่านั้น
- [ ] **DLQ headers ไม่ leak sensitive data** — ไม่ฉีด full error stack ลง `x-error` header
- [ ] **govulncheck ผ่าน** — ไม่มี called vulnerability
- [ ] *(ขั้นสูง)* **SOPS + age** — encrypt `.env` → `.env.enc` (Makefile targets: `env-keygen`, `env-decrypt`, `env-encrypt`)
- [ ] *(ขั้นสูง)* **gosec linter** — ผ่านใน golangci-lint
- [ ] *(ขั้นสูง)* **Secret detection ใน CI** — GitLab secret_detection

---

## 16. Version & Release

- [ ] **`VERSION` file** — รูปแบบ `vX.Y.Z` (semver)
- [ ] **VERSION ใช้ใน Docker label** — `LABEL version="${VERSION}"`
- [ ] **VERSION ใช้ใน health probe response** — ตอบกลับใน `/liveness`
- [ ] **`make bump-version version=vX.Y.Z`** — อัปเดต VERSION + git tag

---

## 17. docker-compose.yml

- [ ] **มี services ที่จำเป็น** — Kafka (KRaft), Jaeger, Prometheus (ตามประเภท service)
- [ ] **ชื่อ service/cluster_id อ้างอิงชื่อ repo**
- [ ] **Volume mount สำหรับ persistence** (ถ้าจำเป็น)
- [ ] **Network isolation** — ไม่ expose port ออกนอก localhost โดยไม่จำเป็น

---

## 18. HTTP Status Convention (ถ้าเป็น HTTP service)

- [ ] **14 status codes:** 200/201/204, 400/401/403/404/409/422/429, 500/502/503/504
- [ ] **4xx = no retry, 5xx = retry**
- [ ] **ทุก error response มี `error_code` ใน body**
- [ ] **Orchestration เช็คแค่ 3 groups:** 2xx/4xx/5xx

---

## วิธีใช้

1. Clone repo ที่ต้อง review
2. วิ่งทุก section ตาม checklist — tick `[x]` สำหรับข้อที่ผ่าน, note gap สำหรับข้อที่ fail
3. รันคำสั่ง verify จริง: `go build ./...`, `go vet ./...`, `go test -race ./...`, `golangci-lint run`, `govulncheck ./...`
4. สรุป gap ที่เหลือ + แนะนำวิธีปิด
