---
description: Validate a Go project's structure against Clean Architecture principles (Screaming, Onion, Hexagonal, Go idiom). Use when asked to audit, review, or grade how "clean" a Go codebase is, or to check compliance with the go-clean architecture rules.
---

Validate the Go project at the target directory (from `args`, or the current directory if not given) against Clean Architecture principles — see the `go-clean` skill for the underlying concepts (hexagonal, onion, screaming, DCI).

## How to run

1. Determine the target directory: use the path passed in `args` if given, otherwise use `.`
2. Run each check below in sequence. Collect findings.
3. Print a final report.

---

## Check 1 — Screaming Architecture

List top-level packages under `internal/` or the module root (excluding `cmd/`, `main.go`, `.git`, `vendor`).

**Pass** if package names are business-domain words (e.g. `order`, `user`, `payment`, `loan`, `book`).
**Fail** if they are technical-layer names: `models`, `views`, `controllers`, `handlers`, `services`, `repositories`, `domain`, `port`, `core`, `adapter`, `usecase`.

Command to list:
```bash
find <dir> -mindepth 1 -maxdepth 2 -type d \
  ! -path '*/.git/*' ! -name '.git' ! -name 'vendor' \
  ! -name 'cmd' ! -name 'testdata'
```

---

## Check 2 — Dependency Direction (Onion rule)

Run:
```bash
cd <dir> && go list -f '{{.ImportPath}}: {{join .Imports " "}}' ./... 2>/dev/null
```

Classify each package as **domain** (business name) or **adapter** (infrastructure name: `memory`, `postgres`, `mysql`, `redis`, `http`, `grpc`, `kafka`, `handler`, `framework`).

**Pass** if adapters import domain packages but domain packages do NOT import adapter packages.
**Fail** if a domain package imports an adapter package (dependency inversion violated).
**Warn** if `go list` fails (module not initialized).

---

## Check 3 — Interface Placement (Go idiom)

Search for `interface` keyword declarations:
```bash
grep -rn "interface {" <dir> --include="*.go" | grep -v "_test.go" | grep -v "vendor"
```

**Pass** if interfaces are declared inside domain packages (same folder as the service/entity that uses them).
**Fail** if interfaces live in a centralized package named `port`, `ports`, `interfaces`, or `contract`.

---

## Check 4 — Circular Imports

```bash
cd <dir> && go build ./... 2>&1
```

**Pass** if build succeeds or errors are unrelated to imports.
**Fail** if output contains `import cycle not allowed`.
Extract and report the cycle path.

---

## Check 5 — God Package

Count `.go` files per domain package directory:
```bash
find <dir> -name "*.go" ! -name "*_test.go" | \
  awk -F/ '{OFS="/"; NF--; print}' | sort | uniq -c | sort -rn
```

**Warn** if a single package has more than ~10 `.go` files AND its name is a layer word (`domain`, `service`, `model`).
**Pass** otherwise.

---

## Check 6 — Cross-domain Coupling via unexported symbols

Search for unexported function calls that cross domain boundaries (heuristic: same package containing multiple struct types from clearly different domains).

Look for packages where multiple unrelated struct types coexist (e.g., `User` and `Order` and `Payment` all in package `domain`):
```bash
grep -rn "^type .* struct" <dir> --include="*.go" | grep -v vendor
```

Group by package path. **Warn** if a single package defines structs from 3+ different business domains.

---

## Report Format

Print the result as a markdown table, then a summary paragraph.

```
## Clean Architecture Validation Report
Directory: <path>

| # | Check | Status | Details |
|---|---|---|---|
| 1 | Screaming Architecture | ✅ / ❌ / ⚠️ | ... |
| 2 | Dependency Direction   | ✅ / ❌ / ⚠️ | ... |
| 3 | Interface Placement    | ✅ / ❌ / ⚠️ | ... |
| 4 | Circular Imports       | ✅ / ❌      | ... |
| 5 | God Package            | ✅ / ⚠️      | ... |
| 6 | Cross-domain Coupling  | ✅ / ⚠️      | ... |

### Summary
<one paragraph: overall verdict, most critical issue to fix first>

### Reference
Clean structure: https://gitlab.com/gophernment/clean-architecture/demos
```

Legend: ✅ Pass · ❌ Fail · ⚠️ Warning
