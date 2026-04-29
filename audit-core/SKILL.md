---
name: audit-core
description: Fundamental design quality evaluator — cohesion, coupling, SRP, and dependency direction. Language-agnostic baseline for any audit.
license: MIT
compatibility: opencode
metadata:
  scope: global
  type: standalone-self-audit
---

## What I do

I guide the model through a self-contained design quality audit. I do NOT require MCP tool output — the model reads source files directly and applies the analysis framework below.

**Independent from `tool-*` skills.** I am designed for standalone use when the user asks to "analyze this code" or "evaluate design quality" without invoking MCP triggers.

## Pipeline

### Phase 1 — Data collection

1. **Detect stack** — check for `go.mod`, `package.json`, `pyproject.toml`, etc. in project root. Record the language.
2. **Collect source files** — find all non-test, non-vendor source files using the detected extensions. Exclude: `vendor/`, `node_modules/`, `.venv/`, `__pycache__/`, `target/`, `build/`, `dist/`.
3. **Security filter (always apply)** — remove files matching: `.env*`, `*.key`, `*.pem`, `*.cert`, `credentials*`, `secrets*`, `password*`. Also honor `.gitignore`.
4. **Read architecture docs** — if `docs/arch/.architecture.json` exists, read it as the intended dependency rules. Otherwise infer rules from package structure.
5. **Read key files** — prioritize: entry points (`main.*`, `cmd/`), interfaces/contracts, domain models, config files. Read enough to understand module boundaries and responsibilities.

### Phase 2 — Analysis

Apply these four criteria to every module/package found:

#### 1. Cohesion
- Do the exported symbols in this package share a single, clear theme?
- Flag: packages whose name suggests one thing but exports symbols from unrelated domains
- Flag: "utility", "common", "helpers", "shared" packages — are they dumping grounds?
- Check: does every file in the package relate to the package name?

#### 2. Coupling
- Count imports per package. How many other internal packages does each import?
- Identify **tight coupling clusters**: groups of packages that always import each other
- Flag: packages that import from layers above them (dependency inversion)
- Flag: packages imported by >5 other packages — potential bottleneck
- Check for indirect coupling: two sibling packages using the same shared types

#### 3. Single Responsibility
- Does each file have one clear, nameable purpose?
- Flag: files >300 lines — likely mixing concerns
- Flag: files with >10 exported functions + types — possible god-object
- Flag: package name mismatch with primary exported symbols

#### 4. Dependency direction
- Build a mental dependency graph: which package imports which?
- Verify: do dependencies flow in one direction (high-level → low-level)?
- Flag: lower-level packages importing from higher-level packages
- Flag: sibling packages importing each other
- If `.architecture.json` exists, check every import against `forbidden_dependencies`

### Phase 3 — Output

Format findings as:

```
## Design Quality Audit: [project]

### Score: X / 100 [RATING]

| Dimension | Score | Notes |
|---|---|---|
| Cohesion | X/25 | ... |
| Coupling | X/25 | ... |
| Single Responsibility | X/25 | ... |
| Dependency Direction | X/25 | ... |

### Issues

[SEVERITY] <finding>
  Location: <file or package>
  Suggestion: <concrete fix>
```

Severity levels: `critical` | `high` | `medium` | `low`

Rating scale: 0-40 CRITICAL, 41-60 LOW, 61-75 MEDIUM, 76-90 HIGH, 91-100 EXCELLENT

---

## When to use me

Use when:
- user asks to "analyze code quality", "evaluate design", "audit architecture"
- performing an initial design review on unfamiliar code
- evaluating whether a refactoring improved design quality

Do NOT use me for:
- cycle detection and graph topology → use `audit-graph`
- complexity hotspots and maintainability → use `audit-complexity`
- security patterns → use `audit-security-basic`

---

## Relationship to other audit skills

This skill focuses on **design principles** (cohesion, coupling, SRP, dependency direction). For complementary dimensions:

- `audit-graph` — import graph construction, cycle detection, fan-in/fan-out, blast radius
- `audit-complexity` — change hotspots, over-abstraction, responsibility sprawl, abstraction gaps
- `audit-security-basic` — secret exposure, input validation, unsafe dependency flows, insecure defaults

Load multiple audit skills for a comprehensive review. Each operates independently.
