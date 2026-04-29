---
name: audit-complexity
description: Maintainability complexity analyzer — identifies what is hardest to change, over-abstracted, or responsibility-sprawling. Complements audit-core and audit-graph.
license: MIT
compatibility: opencode
metadata:
  scope: global
  type: standalone-self-audit
---

## What I do

I guide the model through a maintainability complexity audit. I identify the parts of the codebase that are hardest to change safely. I do NOT require MCP tool output — the model reads source files directly and applies the analysis below.

**Independent from `tool-*` skills.** I am designed for standalone use when the user asks to "find complexity hotspots" or "estimate refactoring scope" without invoking MCP triggers.

## Pipeline

### Phase 1 — Data collection

1. **Get file sizes** — run `wc -l` on all source files (excluding tests, vendor, generated code). Also get byte counts for later char-budget estimates.
2. **Count exported symbols** — for each file, count exported functions and types. For Go: identifiers starting with uppercase. For Python: `__all__` entries or top-level def/class without `_` prefix. For TS: `export` declarations.
3. **Collect import graph edges** — for each file, list which internal packages/modules it imports. This powers the coupling multiplier.
4. **Read key large files** — read files over 200 lines to understand their internal structure and count distinct concerns.

### Phase 2 — Analysis

#### 1. Change hotspots
Identify files that combine **high coupling** (many dependents) with **large size**:

- Compute for each file: `hotspot_score = (fan_in_count × lines) / 1000`
- `fan_in_count` = how many other files import this file's package
- Rank top 5 by `hotspot_score`
- These files are the riskiest to modify — any change has wide blast radius

#### 2. Over-abstraction
- Flag files <30 lines whose only purpose is to re-export symbols from another module (type aliases, `from X import *`, barrel exports)
- Flag interfaces/traits/ABCs with only 1 implementation
- Flag adapter/wrapper chains >2 hops: A wraps B wraps C — with no added domain logic at each hop

#### 3. Responsibility sprawl
- Flag files >400 lines
- For each large file, check: can you name its single responsibility in 3 words? If not → sprawl
- Read large files, identify distinct concerns by analyzing function names and import groupings
- Flag packages exporting symbols from 3+ unrelated domains (e.g., a package exporting file I/O + HTTP client + math utilities)
- Flag packages named "manager", "handler", "helper", "utils", "common" — these are sprawl attractors

#### 4. Abstraction gaps
- **Too thick**: layers/packages with >10 exported symbols — may be absorbing responsibilities of other layers
- **Too thin**: layers/packages with 0–1 exported symbols that are pure pass-through — dead weight
- **Missing domain layer**: business logic embedded directly in HTTP handlers, CLI commands, or database access code instead of a separate domain/service layer

### Phase 3 — Output

Format findings as:

```
## Complexity Audit: [project]

### Score: X / 100 [RATING]

### Top 5 Change Hotspots

| Rank | File | Lines | Fan-in | Hotspot Score |
|---|---|---|---|---|
| 1 | path/to/file | N | N | N |

### Issues

[SEVERITY] <finding>
  Location: <file or package>
  Category: <hotspot | over-abstraction | sprawl | abstraction-gap>
  Suggestion: <concrete fix>
```

Severity levels: `critical` | `high` | `medium` | `low`

Scoring: start at 100, deduct for each finding by severity (critical: -15, high: -10, medium: -5, low: -2)

---

## When to use me

Use when:
- user asks about "complexity", "maintainability", "refactoring scope", "hard to change"
- a module is frequently the source of bugs or painful to modify
- the codebase has grown and structural debt is suspected

Do NOT use me for:
- cohesion and coupling design quality → use `audit-core`
- import graph cycles and topology → use `audit-graph`
- security patterns → use `audit-security-basic`

---

## Relationship to other audit skills

This skill focuses on **maintainability and change risk**. For complementary dimensions:

- `audit-core` — cohesion, coupling, SRP, dependency direction
- `audit-graph` — import graph construction, cycle detection, fan-in/fan-out, blast radius
- `audit-security-basic` — secret exposure, input validation, unsafe dependency flows

Load multiple audit skills for a comprehensive review. Each operates independently.
