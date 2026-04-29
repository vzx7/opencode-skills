---
name: tool-architecture-compliance-check
description: Smart architecture compliance orchestrator — detects stack and docs directory, selects key source files, calls MCP with curated docs + include_paths + programming_language
license: MIT
compatibility: opencode
metadata:
  scope: tool
  tool: architecture_compliance_check
  type: orchestrator
---

## What I do

Triggered when `/architecture_compliance_check` is called.

I detect the project stack and architecture documentation directory, select the architecturally significant source files, then call MCP with `docs`, `include_paths`, and `programming_language`. The LLM receives both the intended architecture (docs) and the actual implementation (key files) in one context.

---

## Pipeline

### Step 1 — Resolve project path

Use `project_path` from input. If not provided, use current working directory.

### Step 2 — Detect project stack

Check for these markers in `project_path` (stop at first match):

| Marker file(s)                                        | `programming_language` | Source extensions     | Exclude dirs                         | Test pattern           |
|-------------------------------------------------------|------------------------|-----------------------|--------------------------------------|------------------------|
| `go.mod`                                              | `go`                   | `.go`                 | `vendor/`                            | `_test.go` suffix      |
| `tsconfig.json` or `package.json` + `.ts` files       | `typescript`           | `.ts`, `.tsx`         | `node_modules/`, `dist/`, `build/`   | `.test.ts`, `.spec.ts` |
| `package.json`                                        | `javascript`           | `.js`, `.jsx`, `.mjs` | `node_modules/`, `dist/`, `build/`   | `.test.js`, `.spec.js` |
| `pyproject.toml` or `setup.py` or `requirements.txt`  | `python`               | `.py`                 | `__pycache__/`, `.venv/`, `venv/`    | `test_*.py`            |
| `Cargo.toml`                                          | `rust`                 | `.rs`                 | `target/`                            | (files in `tests/`)    |
| `pom.xml` or `build.gradle` or `build.gradle.kts`     | `java`                 | `.java`, `.kt`        | `target/`, `build/`                  | `Test*.java`           |
| _(none matched)_                                      | `go` (default)         | `.go`                 | `vendor/`                            | `_test.go` suffix      |

Record: detected stack, source extensions, exclude dirs, test pattern.

### Step 3 — Detect docs directory

Check in order (stop at first existing directory):

1. `docs` argument from input — use as-is if provided (relative to project root)
2. `docs/arch/`
3. `docs/`
4. `documentation/`

If none found, proceed without `docs`. Note it explicitly in the output.

> **Note:** If the found directory contains `.architecture.json`, the MCP server loads it automatically as the rules source — no explicit `docs` argument needed when the file is at the default path `docs/arch/.architecture.json`.

### Step 4 — Collect candidate source files

Find all source files using the **detected extensions** from Step 2. Exclude:
- directories from the detected exclude list
- hidden directories (`.git`, `.cache`, etc.)
- test files matching the detected test pattern

### Step 5 — Apply security filter (MANDATORY)

**Before selecting any files**, remove from the candidate list every file that matches any of these conditions:

**5a — Read `.gitignore`**
- Parse `project_path/.gitignore` (and `~/.gitignore_global` if it exists)
- Remove every candidate file that matches a gitignore pattern
- Rationale: developers gitignore files containing secrets — honour that decision

**5b — Hardcoded denylist (apply regardless of gitignore)**

Remove any file whose name (case-insensitive) matches:

| Category | Patterns |
|---|---|
| Environment files | `.env`, `.env.*`, `*.env` (e.g. `.env.local`, `.env.production`, `config.env`) |
| Cryptographic keys | `*.key`, `*.pem`, `*.p12`, `*.pfx`, `*.cert`, `*.crt`, `*.jks`, `*.keystore`, `*.gpg`, `*.asc` |
| SSH keys | `id_rsa`, `id_dsa`, `id_ed25519`, `id_ecdsa` and their `.pub` variants |
| Sensitive names | filenames containing: `credential`, `secret`, `password`, `passwd`, `apikey`, `api_key`, `private_key`, `access_key`, `auth_token` |

**CRITICAL**: Never pass a file matching the denylist to MCP, even if the user explicitly requests it.

### Step 6 — Select key files

Apply in priority order from the **filtered** candidate list. Total size budget: **60 000 chars** (compliance check reserves room for docs).

**Priority 1 — Entry points**
- Go: files in `cmd/`, `cmd/*/`
- Python: `__main__.py`, `app.py`, `main.py`, `manage.py`
- JS/TS: `index.ts`, `index.js`, `server.ts`, `server.js`, `app.ts`, `app.js`
- Rust: `src/main.rs`, `src/lib.rs`
- Java: files in `src/main/java/` containing `public static void main`
- Universal: any file named `main.*` at the project root

**Priority 2 — Interfaces and contracts**
- Files whose name contains: `interface`, `contract`, `port`, `gateway`, `repo`, `repository`, `service`
- Go/TS/Java: files containing the keyword `interface` in source
- Python: files containing `ABC`, `Protocol`, or `@abstractmethod`
- Rust: files containing `trait`

**Priority 3 — Domain core**
- Files in: `domain/`, `core/`, `entities/`, `model/`, `models/`, `internal/domain/`
- Files named: `models.*`, `types.*`, `errors.*`, `entities.*`, `schema.*`, `domain.*`

**Priority 4 — Hub packages (most imported)**
- Parse import statements across collected files
- Count how many files import each internal package/module
- Select top 5 by in-degree
- Include all source files from those packages

**Priority 5 — Config and wiring**
- Files named: `config.*`, `wire.*`, `di.*`, `container.*`, `bootstrap.*`, `settings.*`
- Files directly in the project root (not in subdirectories)
- **Do NOT use `env.*` as a pattern** — too broad, risks matching `.env` variants

**Priority 6 — Fill remaining budget**
- Remaining files sorted by size descending, added until 60 000 char limit

Deduplication: skip files already added in earlier priorities.

### Step 7 — Call MCP

**IMPORTANT:** If the user passes `llm` in the format `provider/model` (e.g., `moonshotai/kimi-k2.5`), pass the FULL string as-is to the `llm` parameter. Do NOT split it into `provider` and `llm`.

```python
mcp_project_audit_architecture_compliance_check(
    project_path="<resolved path>",
    programming_language="<detected from Step 2>",
    docs="<detected docs dir, relative to project_path — omit if none found>",
    include_paths=["<list of selected relative paths>"],
    target_architecture=<from input, or omit>,
    language="<from input, default ru>",
    llm="<full string from user input, e.g. 'moonshotai/kimi-k2.5', default omitted>"
)
```

### Step 8 — Output

Print the MCP report as-is. Prepend:

```
> Compliance check | stack: <detected> | N source files (~K chars) | docs: <dir or "none"> (M files) | gitignore applied: yes/no
```

---

## When to use me

Use ONLY when:
- command is `/architecture_compliance_check`
- architecture compliance validation is requested

Do NOT use me for:
- full architecture review → use `tool-architecture-review`
- single module audit → use `tool-module-audit`

---

## Common mistakes

**WRONG:** Skip the security filter (Step 5) — `credentials.go` or `.env.local` may end up in `include_paths`.

**WRONG:** Use `env.*` as a file selection pattern — it matches `.env`, `.env.local`, `.env.production`.

**WRONG:** Call MCP without `docs` when `docs/arch/` exists but `.architecture.json` is absent — server falls back to generic default rules.

**WRONG:** Call MCP without `include_paths` — server builds snapshot by size, misses interfaces and domain core.

**RIGHT:** Detect stack → collect files → gitignore + denylist → select by significance → call MCP with all three.
