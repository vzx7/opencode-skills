---
name: audit-security-basic
description: Security risk analyzer — detects hardcoded secrets, missing input validation, and unsafe dependency flows in architecture and code structure.
license: MIT
compatibility: opencode
metadata:
  scope: global
  type: standalone-self-audit
---

## What I do

I guide the model through a security-focused code audit. I do NOT require MCP tool output — the model reads source files directly, searches for patterns, and applies the analysis framework below.

**Independent from `tool-*` skills.** I am designed for standalone use when the user asks to "check security" or "find vulnerabilities" without invoking MCP triggers.

## Pipeline

### Phase 1 — Data collection

1. **Identify entry points** — Read files that accept external input:
   - HTTP handlers (look for `http.HandleFunc`, `mux.Handle`, route definitions, `@app.route`, `@Get`/`@Post` decorators)
   - CLI entry points (`main.go`, `click`, `argparse`, `commander`)
   - File readers that accept paths from user input
   - Any code using `os.Getenv`, `.env` loading

2. **Read config/secrets code** — Read `config.*`, `settings.*`, `.env` loading code, and any code that handles API keys, tokens, or credentials.

3. **Grep for dangerous patterns** (run these searches):
   - Secrets: search for long string literals that look like tokens. Patterns by language:
     - `apiKey\s*[:=]\s*"[^"]{20,}"` — hardcoded API keys
     - `token\s*[:=]\s*"[^"]{8,}"` — hardcoded tokens
     - `password\s*[:=]\s*"[^"]+"` — hardcoded passwords
     - `secret\s*[:=]\s*"[^"]+"` — hardcoded secrets
   - Unsafe execution: search for `exec.Command`, `os/exec`, `subprocess`, `eval(`, `os.system(`
   - File I/O with variables: search for `os.Open(` followed by a variable (not literal), `ioutil.ReadFile`, `fs.readFileSync`
   - SQL without placeholders: `fmt.Sprintf.*SELECT`, `fmt.Sprintf.*INSERT`, string concatenation with query fragments

4. **Read auth/middleware code** — If auth middleware exists, read it. Check for disabled-by-default patterns.

### Phase 2 — Analysis

#### 1. Secret exposure
- Is there ANY hardcoded credential, token, key, or password in source code?
- Are API keys read from environment variables (good) or passed as plain strings (bad)?
- Are sensitive values ever logged or included in error messages?
- Check: `fmt.Printf` / `log.Printf` with variables named `key`, `token`, `secret`, `password`
- Check: error messages that include the raw config value

#### 2. Input validation
- At EVERY entry point (HTTP handler, CLI command, file reader), is input validated?
- Check for: path traversal (`../`), SQL injection (string concatenation in queries), command injection (unvalidated input passed to `exec.Command`), XSS (unvalidated input in HTML responses)
- Does validation happen BEFORE the input is used, or only after?
- Are there any input sources that are trusted without validation?

#### 3. Unsafe dependency flows
- Trace: can untrusted external input reach a sensitive operation (database, file system, OS command, external API) without passing through a validation layer?
- Identify trust boundaries: which packages handle external input, which packages perform sensitive operations?
- Flag: user input → file system without sanitization
- Flag: user input → shell command without validation
- Flag: user input → SQL query without parameterized queries

#### 4. Insecure defaults
- Is authentication disabled by default? (`auth = false`, empty token check, `if os.Getenv("DISABLE_AUTH") == "true"`)
- Is TLS disabled or configurable to off? (`tls: false`, `ssl: false`, HTTP-only mode)
- Are CORS settings overly permissive? (`Access-Control-Allow-Origin: *`)
- Are rate limits absent or set to unrealistically high values?
- Default provider/credentials: `mock` provider is fine for dev, but is it used in production?

### Phase 3 — Output

Format findings as:

```
## Security Audit: [project]

### Score: X / 100 [RATING]

| Dimension | Score | Notes |
|---|---|---|
| Secret exposure | X/25 | ... |
| Input validation | X/25 | ... |
| Unsafe flows | X/25 | ... |
| Secure defaults | X/25 | ... |

### Issues

[SEVERITY] <finding>
  Location: <file:line or function>
  Category: <secrets | validation | unsafe-flow | insecure-default>
  Suggestion: <concrete fix>
```

Severity levels: `critical` | `high` | `medium` | `low`

**Severity guidelines:**
- `critical`: hardcoded production credentials, RCE via unsanitized input, SQL injection
- `high`: missing input validation at entry points, disabled auth by default
- `medium`: secrets in error messages, overly permissive CORS, missing rate limits
- `low`: inconsistent validation patterns, theoretical concerns without exploit path

---

## When to use me

Use when:
- user asks to "check security", "find vulnerabilities", "security audit"
- the system accepts external input (HTTP, CLI, files, env)
- authentication or secrets management is part of the codebase

Do NOT use me for:
- design quality and cohesion → use `audit-core`
- import graph and cycles → use `audit-graph`
- complexity and maintainability → use `audit-complexity`

---

## Relationship to other audit skills

This skill focuses on **security risks**. For complementary dimensions:

- `audit-core` — cohesion, coupling, SRP, dependency direction
- `audit-graph` — import graph construction, cycle detection, fan-in/fan-out
- `audit-complexity` — change hotspots, over-abstraction, responsibility sprawl

Load multiple audit skills for a comprehensive review. Each operates independently.
