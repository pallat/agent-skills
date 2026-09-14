# go-clean

Claude Code skill สำหรับเขียน Go ตาม Clean Architecture ที่สกัดจาก go-ms-otel.

## ที่มา

สกัดแพทเทิร์นจริงจาก `gitdev.devops.krungthai.com/techcoach/template/go-ms-otel` —
Go microservice template ที่ใช้ Clean Architecture (onion/hexagonal/screaming) ใน Krungthai TechCoach.

## โครงสร้าง

```
go-clean/
├── README.md     ← ไฟล์นี้
└── SKILL.md      ← Claude Code skill (frontmatter + compact instructions)
```

## วิธีติดตั้ง

คัดลอก `go-clean/` ไปไว้ใน `.claude/skills/` ของ project:

```bash
cp -r go-clean/ <project>/.claude/skills/go-clean/
```

Claude Code จะโหลด skill อัตโนมัติเวลาเขียน/แก้ Go code หรือ invoke ด้วย `/go-clean`.

## แพทเทิร์นหลัก

| แพทเทิร์น | รายละเอียด |
|---|---|
| Handler-in-domain | HTTP handler อยู่ใน domain package, ใช้ `Context` interface |
| Framework bridge | `pkg/framework/NewHandler[C]` — จุดเดียวที่ import gin |
| Driven adapters | subpackage ของ domain (`internal/<domain>/userfinder/`) |
| DomainError | adapter แปลง error ที่ boundary (anti-corruption layer) |
| Screaming arch | package name = business term (`permit`), ไม่ใช่ role (`service`) |
| Stub testing | test ผ่าน stub Context ไม่ต้องมี gin/router |
