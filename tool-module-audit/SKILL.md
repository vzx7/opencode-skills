---
name: tool-module-audit
description: Module-level audit orchestrator — detects project stack, calls MCP module_audit, optionally deepens with audit skills
license: MIT
compatibility: opencode
metadata:
  scope: tool
  tool: module_audit
  type: orchestrator
---

## What I do

Triggered when `/module_audit` is called.

I detect the project stack, call MCP `module_audit`, and output the report. If the user requests deeper analysis (asks for coupling/security/complexity breakdown), I additionally apply audit skills on the MCP output.

---

## Pipeline

### Step 1 — Resolve paths

- `project_path`: from input, or current working directory if not provided
- `module_path`: from input (required — a file or directory to audit)

### Step 2 — Detect project stack

Check for these markers in `project_path` (stop at first match):

| Marker file(s)                                        | `programming_language` |
|-------------------------------------------------------|------------------------|
| `go.mod`                                              | `go`                   |
| `tsconfig.json` or `package.json` + `.ts` files       | `typescript`           |
| `package.json`                                        | `javascript`           |
| `pyproject.toml` or `setup.py` or `requirements.txt`  | `python`               |
| `Cargo.toml`                                          | `rust`                 |
| `pom.xml` or `build.gradle` or `build.gradle.kts`     | `java`                 |
| _(none matched)_                                      | `go` (default)         |

### Step 3 — Validate module_path (security check)

Before calling MCP, verify `module_path` does not point to a sensitive file.

Refuse and explain if `module_path` matches any of these:
- `.env`, `.env.*`, `*.env`
- `*.key`, `*.pem`, `*.p12`, `*.pfx`, `*.cert`, `*.crt`, `*.jks`
- `id_rsa`, `id_dsa`, `id_ed25519`, `id_ecdsa`
- filename contains: `credential`, `secret`, `password`, `passwd`, `apikey`, `api_key`, `private_key`, `access_key`, `auth_token`

Also check `project_path/.gitignore`: if `module_path` is gitignored, warn the user and ask for confirmation before proceeding.

### Step 4 — Call MCP

**IMPORTANT:** If the user passes `llm` in the format `provider/model` (e.g., `moonshotai/kimi-k2.5`), pass the FULL string as-is to the `llm` parameter. Do NOT split it into `provider` and `llm`.

```python
mcp_project_audit_module_audit(
    project_path="<resolved project path>",
    module_path="<from input>",
    programming_language="<detected>",
    language="<from input, default ru>",
    llm="<full string from user input, e.g. 'moonshotai/kimi-k2.5', default omitted>"
)
```

### Step 5 — Output

Print the MCP report as-is. Prepend:

```
> Module audit: <module_path> | stack: <detected>
```

### Step 6 — Deep mode (optional)

Apply only if the user asks for deeper analysis beyond the MCP report. Perform the following checks directly (no MCP calls — read files if needed and apply these criteria):

#### Cohesion & Coupling
- Does the module have a single, clear responsibility? Can you name it in 3 words?
- Count internal imports: how many other project packages does it depend on?
- Are there imports from layers above? (e.g., a domain package importing from tools)
- Are there sibling-package imports that create hidden coupling?

#### Security
- Are there hardcoded credentials, tokens, or API keys?
- Is all external input validated before use?
- Does any user-controlled value reach `exec.Command`, `os.Open`, or SQL without sanitization?

#### Complexity
- File size: flag files >300 lines — likely mixing concerns
- Exported symbols: flag >10 exported functions/types — possible god-object
- Over-abstraction: flag thin wrappers that only re-export from another module
- Responsibility sprawl: flag "manager", "handler", "helper" packages

Append findings under `## Deep Analysis` after the MCP report.

---

## When to use me

Use when:
- command is `/module_audit`
- a specific file or directory path is provided for audit

Do NOT use me for:
- full project architecture review → use `tool-architecture-review`
- architecture compliance validation → use `tool-architecture-compliance-check`

---

## Common mistakes

**WRONG:** Call MCP without `programming_language` — server defaults to Go, misses files in Python/TS projects.

**WRONG:** Manually read and analyze source files before calling MCP — MCP runs its own LLM analysis.

**RIGHT:** Detect stack → call MCP → output → optionally deepen with audit skills.
