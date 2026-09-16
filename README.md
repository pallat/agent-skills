# agent-skills

รวม Claude Code skills ส่วนตัว — เอาไปวางใน `<project>/.claude/skills/<skill>/` เพื่อใช้.

## Skills

| Skill | ใช้ทำอะไร |
| --- | --- |
| [go-clean](go-clean/) | สอน Go clean architecture (hexagonal, onion, screaming, DCI) ตอนเขียน/แก้ domain package |
| [go-clean-validate](go-clean-validate/) | ตรวจ Go project ที่มีอยู่แล้วว่า clean ตาม 6 checks แค่ไหน — คู่กับ `go-clean` |
| [cfr](cfr/) | audit Go service repo แบบ Cross Functional Requirements (consistency, orphans, dangling refs, gates, security) |

## วิธีติดตั้ง

```bash
cp -r <skill>/ <project>/.claude/skills/<skill>/
```
