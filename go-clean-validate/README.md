# go-clean-validate

Claude Code skill ตรวจ Go project ว่า clean มากแค่ไหน ตามหลัก Clean Architecture (Screaming, Onion, Hexagonal, Go idiom) — คู่กับ [[go-clean]] ที่สอน concept.

## ที่มา

ดึงจาก `clean-arch-validate` slash command ใน clean-architecture workshop
(`demos/coffeeshop-clean/.claude/commands/clean-arch-validate.md`).

## โครงสร้างไฟล์

```
go-clean-validate/
├── README.md     ← ไฟล์นี้
└── SKILL.md      ← Claude Code skill (6 checks + report format)
```

## 6 checks

| # | check | ดู |
|---|---|---|
| 1 | Screaming Architecture | ชื่อ package เป็น business term ไหม |
| 2 | Dependency Direction | domain import adapter หรือเปล่า (onion rule) |
| 3 | Interface Placement | interface อยู่ใน domain หรือกอง `port`/`interfaces` |
| 4 | Circular Imports | `go build` ผ่านไหม |
| 5 | God Package | package เดียวมีไฟล์เยอะเกินไปไหม |
| 6 | Cross-domain Coupling | struct จากหลาย domain ปนกันไหม |

ผลออกเป็น markdown table + summary paragraph.

## วิธีติดตั้ง

```bash
cp -r go-clean-validate/ <project>/.claude/skills/go-clean-validate/
```

Claude Code invoke ด้วย `/go-clean-validate [path]` หรือให้ตรวจ current directory เฉยๆ.
