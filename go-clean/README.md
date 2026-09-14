# go-clean

Claude Code skill สำหรับเขียน Go ตาม Clean Architecture ที่สกัดจาก go-ms-otel.

## ที่มา

สกัดแพทเทิร์นจริงจาก `gitdev.devops.krungthai.com/techcoach/template/go-ms-otel` —
Go microservice template ที่ใช้ Clean Architecture (onion/hexagonal/screaming) ในการ Krungthai TechCoach.

## วิธีใช้

### ติดตั้งใน project

คัดลอก `go-clean-architecture.md` ไปไว้ใน `.claude/skills/` ของ project:

```bash
cp go-clean-architecture.md <project>/.claude/skills/go-clean-architecture.md
```

Claude Code จะโหลด skill อัตโนมัติทุก session แล้วเขียน Go ตาม convention นี้.

### ใช้กับ Claude Code (print mode)

```bash
claude -p "Create a new domain package for loan processing" \
  --allowedTools 'Read,Write,Edit,Bash' \
  --max-turns 15
```

Claude Code จะอ่าน skill แล้วสร้าง:
- `internal/loan/entity.go` — domain types
- `internal/loan/port.go` — interfaces
- `internal/loan/service.go` — business logic + DomainError
- `internal/loan/handler.go` — HTTP handler (handler-in-domain)
- `internal/loan/<adapter>/` — driven adapters as subpackages

## แพทเทิร์นหลัก

| แพทเทิร์น | รายละเอียด |
|---|---|
| **Layer structure** | `internal/<domain>/` (stdlib only) + `pkg/` (tech) + `main.go` (wiring) |
| **Handler-in-domain** | HTTP handler อยู่ใน domain package, ไม่มี `internal/adapter/http/` |
| **Framework bridge** | `pkg/framework/NewHandler[C]` — จุดเดียวที่ import gin |
| **Driven adapters** | subpackage ของ domain — `internal/<domain>/userfinder/`, ไม่ใช่ `internal/adapter/` |
| **DomainError** | domain error codes + `mapDomainErrorToHTTP` — adapter แปลง error ที่ boundary |
| **Anti-corruption layer** | adapter แปลง HTTP/transport error → DomainError, domain ไม่เห็น transport type |
| **Screaming architecture** | package name = business term (`permit`, `loan`), ไม่ใช่ technical role (`service`, `handler`) |
| **Testing** | stub Context (no gin, no router) — test domain logic บริสุทธิ์ |

## โครงสร้างไฟล์

```
go-clean/
├── README.md                    ← ไฟล์นี้
└── go-clean-architecture.md     ← skill สำหรับ Claude Code
```

## ข้อกำหนด

- Go 1.22+ ( generics สำหรับ `NewHandler[C]`)
- gin-gonic/gin ( HTTP framework)
- ไม่จำเป็ดต้องใช้ go-ms-otel — skill สอนแพทเทิร์น, ใช้กับ project ไหนก็ได้
