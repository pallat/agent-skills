# go-clean

Claude Code skill สอน Go clean architecture ผ่าน 4 แกน concept: hexagonal, onion, screaming, DCI.

## ที่มา

สกัดจาก go-ms-otel และ clean-architecture workshop (coffeeshop-clean) —
ใช้ concept คุม ไม่ใช่ structure คุม. ตัวอย่าง structure เป็น illustration เล็กๆ.

## โครงสร้างไฟล์

```
go-clean/
├── README.md     ← ไฟล์นี้
└── SKILL.md      ← Claude Code skill (43 lines, concept-driven)
```

## 4 แกน

| แกน | core concept |
|---|---|
| **Hexagonal** | Port = interface ใน domain, adapter = concrete ข้างนอก, dependency inverted |
| **Onion** | dependency ไหลเข้าศูนย์เท่านั้น, domain import stdlib อย่างเดียว |
| **Screaming** | อ่านชื่อ package แล้วรู้ว่าระบบทำอะไร — business term ไม่ใช่ layer/pattern |
| **DCI** | Context = role ที่ domain ประกาศ, framework เติม implementation ตอน runtime |

## วิธีติดตั้ง

```bash
cp -r go-clean/ <project>/.claude/skills/go-clean/
```

Claude Code โหลดอัตโนมัติเวลาเขียน Go code หรือ invoke ด้วย `/go-clean`.
