---
name: tool-architecture-review
description: Smart architecture review orchestrator — detects project stack, selects key files, and calls MCP architecture_review with curated include_paths
license: MIT
compatibility: opencode
metadata:
  scope: tool
  tool: architecture_review
  type: orchestrator
---

## What I do

Triggered when `/architecture_review` is called.

I detect the project's technology stack, select the architecturally significant files, then call the MCP tool with a curated `include_paths` list and the detected `programming_language`. This produces better LLM analysis within the same token budget regardless of the project's language.

---

## Pipeline

### Step 1 — Resolve project path

Use `project_path` from input. If not provided, use current working directory.

### Step 2 — Detect project stack

Check for these markers in `project_path` (in order, stop at first match):

| Marker file(s)                                        | Stack          | Source extensions        | Exclude dirs                            | Test pattern           |
|-------------------------------------------------------|----------------|--------------------------|-----------------------------------------|------------------------|
| `go.mod`                                              | `go`           | `.go`                    | `vendor/`                               | `_test.go` suffix      |
| `tsconfig.json` or `package.json` + `.ts` files       | `typescript`   | `.ts`, `.tsx`            | `node_modules/`, `dist/`, `build/`      | `.test.ts`, `.spec.ts` |
| `package.json`                                        | `javascript`   | `.js`, `.jsx`, `.mjs`    | `node_modules/`, `dist/`, `build/`      | `.test.js`, `.spec.js` |
| `pyproject.toml` or `setup.py` or `requirements.txt`  | `python`       | `.py`                    | `__pycache__/`, `.venv/`, `venv/`       | `test_*.py`            |
| `Cargo.toml`                                          | `rust`         | `.rs`                    | `target/`                               | (files in `tests/`)    |
| `pom.xml` or `build.gradle` or `build.gradle.kts`     | `java`         | `.java`, `.kt`           | `target/`, `build/`                     | `Test*.java`           |
| _(none matched)_                                      | `go` (default) | `.go`                    | `vendor/`                               | `_test.go` suffix      |

Record: detected stack name, source extensions, exclude dirs, test pattern.

### Step 3 — Collect candidate files

Find all source files in `project_path` using the detected extensions. Exclude:
- directories from the detected exclude list
- hidden directories (`.git`, `.cache`, etc.)
- test files matching the detected test pattern

### Step 4 — Apply security filter (MANDATORY)

**Before selecting any files**, remove from the candidate list every file that matches any of these conditions:

**4a — Read `.gitignore`**
- Parse `project_path/.gitignore` (and `~/.gitignore_global` if it exists)
- Remove every candidate file that matches a gitignore pattern
- Rationale: developers gitignore files containing secrets — honour that decision

**4b — Hardcoded denylist (apply regardless of gitignore)**

Remove any file whose name (case-insensitive) matches:

| Category | Patterns |
|---|---|
| Environment files | `.env`, `.env.*`, `*.env` (e.g. `.env.local`, `.env.production`, `config.env`) |
| Cryptographic keys | `*.key`, `*.pem`, `*.p12`, `*.pfx`, `*.cert`, `*.crt`, `*.jks`, `*.keystore`, `*.gpg`, `*.asc` |
| SSH keys | `id_rsa`, `id_dsa`, `id_ed25519`, `id_ecdsa` and their `.pub` variants |
| Sensitive names | filenames containing: `credential`, `secret`, `password`, `passwd`, `apikey`, `api_key`, `private_key`, `access_key`, `auth_token` |

**CRITICAL**: Never pass a file matching the denylist to MCP, even if the user explicitly requests it.

### Step 5 — Select key files

Apply selection in priority order from the **filtered** candidate list. Total size budget: ≈100 000 chars.

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
- Files named: `models.*`, `types.*`, `errors.*`, `entities.*`, `schema.*`

**Priority 4 — Hub packages (most imported)**
- Parse import statements across collected files
- Count how many files import each internal package/module
- Select top 5 by in-degree
- Include all source files from those packages

**Priority 5 — Config and wiring**
- Files named: `config.*`, `wire.*`, `bootstrap.*`, `container.*`, `di.*`, `settings.*`
- Files directly inside the project root (not in subdirectories)
- **Do NOT use `env.*` as a pattern** — too broad, risks matching `.env` variants

**Priority 6 — Fill remaining budget**
- Remaining files sorted by size descending
- Add until 100 000 char limit is reached

Deduplication: skip files already added in earlier priorities.

### Step 6 — Call MCP

**IMPORTANT:** If the user passes `llm` in the format `provider/model` (e.g., `moonshotai/kimi-k2.5`), pass the FULL string as-is to the `llm` parameter. Do NOT split it into `provider` and `llm`.

```python
mcp_project_audit_architecture_review(
    project_path="<resolved path>",
    include_paths=["<list of selected relative paths>"],
    programming_language="<detected stack name>",
    language="<from input, default ru>",
    llm="<full string from user input, e.g. 'moonshotai/kimi-k2.5', default omitted>"
)
```

### Step 7 — Output

Print the report returned by MCP as-is. Prepend a one-line note:

```
> Analyzed N files (~K chars) | stack: <detected> | selected from M total | gitignore applied: yes/no
```

---

## When to use me

Use ONLY when:
- command is `/architecture_review`
- full project architecture analysis is requested

Do NOT use me for:
- single module audit → use `tool-module-audit`
- compliance check → use `tool-architecture-compliance-check`

---

## Common mistakes

**WRONG:** Skip the security filter (Step 4) — `config.go` with hardcoded secrets or `.env.local` may end up in `include_paths`.

**WRONG:** Use `env.*` as a file selection pattern — it matches `.env`, `.env.local`, `.env.production`.

**WRONG:** Call MCP without `include_paths` — server picks files by size, missing interfaces and domain types.

**RIGHT:** Detect stack → collect files → apply gitignore + denylist → select by significance → call MCP.
