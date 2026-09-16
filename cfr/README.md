# cfr

Claude Code skill audit Go service repo แบบ Cross Functional Requirements — เช็คว่าสิ่งที่มีอยู่ agree กับสิ่งอื่นที่มีอยู่ไหม ไม่ใช่แค่เช็คว่าไฟล์มีอยู่หรือเปล่า.

## 5 audit dimensions

| Dimension | คำถาม | Action |
|---|---|---|
| **Consistency** | A บอก X — B เห็นด้วยไหม | แก้ให้ตรงกัน |
| **Orphans** | A มีอยู่ — มีอะไรใช้ไหม | ลบหรือ wire ต่อ |
| **Dangling** | A อ้างถึง B — B มีจริงไหม | แก้หรือลบ reference |
| **Gates** | คำสั่งบังคับ (build/test/lint) ผ่านไหม | แก้จนเขียว |
| **Security** | มี forbidden pattern โผล่มาไหม | กำจัด |

ครอบคลุมถึง build/test gates, README/CHANGELOG/env consistency, Dockerfile, observability, screaming architecture, linter config, HTTP convention, CI pipeline.

## โครงสร้างไฟล์

```
cfr/
├── README.md     ← ไฟล์นี้
└── SKILL.md      ← Claude Code skill (11 sections)
```

## วิธีติดตั้ง

```bash
cp -r cfr/ <project>/.claude/skills/cfr/
```

Claude Code โหลดอัตโนมัติเวลา review/cleanup Go microservice repo หรือ invoke ด้วย `/cfr`.
