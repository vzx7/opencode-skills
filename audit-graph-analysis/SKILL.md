---
name: audit-graph
description: Dependency graph analyzer — cycles, topology, fan-in/out, connectivity, and blast radius. Replaces audit-dependencies and audit-graph-analysis.
license: MIT
compatibility: opencode
metadata:
  scope: global
  type: standalone-self-audit
---

## What I do

I guide the model through dependency graph analysis. I do NOT require MCP tool output — the model constructs the import graph from source files and applies the analysis framework below.

**Independent from `tool-*` skills.** I am designed for standalone use when the user asks to "find dependency cycles" or "analyze import graph" without invoking MCP triggers.

## Pipeline

### Phase 1 — Data collection

1. **Collect all source files** — find non-test, non-vendor source files. Exclude: `vendor/`, `node_modules/`, `.venv/`, `__pycache__/`, `target/`.

2. **Extract module prefix** — for Go, read the module name from `go.mod`. For Python/TS, infer from directory structure.

3. **Build import graph** — for each source file, extract internal imports (imports that start with the module prefix or are relative paths within the project):

   **Go approach:**
   ```
   For each .go file:
     - Parse import statements
     - Filter: keep only imports matching the module prefix from go.mod
     - Strip the module prefix to get the internal package path
     - Edge: file's package → imported package
   ```

   **Alternative (manual) approach:**
   ```
   Use grep to extract import blocks:
   grep -rh '"[^"]*"' <project>/ --include="*.go" | 
     grep "module-prefix" | 
     sort -u
   ```

   Build map: `edges[from_package] = [to_package1, to_package2, ...]`

4. **Deduplicate** — merge per-file edges into per-package edges (remove duplicates).

### Phase 2 — Analysis

#### 1. Cycle detection
- **Direct cycles**: A imports B, B imports A
- **Indirect cycles**: A → B → C → A (3+ hops)
- For each cycle, report the full path and identify the cheapest edge to break
- Shorter cycles are more critical than longer ones

#### 2. Topology classification
Classify the overall graph structure:
- **Star**: one central hub imported by many packages, imports few/none → fragile foundation
- **Chain**: A → B → C → D → ... → long serial dependency chains → change propagation risk
- **Mesh**: many cross-links between packages → high coupling, low cohesion
- **Layered**: clear one-way dependency direction → good

#### 3. Fan-in / Fan-out
- **Fan-in**: how many packages import this package? High fan-in = breaking change risk
- **Fan-out**: how many packages does this package import? High fan-out = instability (inherits upstream risk)
- Report top 3 by each metric
- Packages with **both** high fan-in and high fan-out are the most fragile

#### 4. Connectivity
- **Orphan packages**: no incoming edges, no outgoing edges to internal packages → dead code candidate
- **Disconnected subgraphs**: isolated clusters with no connection to the main dependency graph
- **Root packages**: only outgoing edges, no incoming — entry points or utility packages

#### 5. Blast radius
- For each package, compute **transitive dependents**: the set of all packages that depend on it, directly or indirectly
- Top 3 packages by transitive dependents = highest blast radius
- Estimate: how many packages are downstream of each candidate?

### Phase 3 — Output

Format findings as:

```
## Dependency Graph Audit: [project]

### Score: X / 100 [RATING]

Cycles: [0-3 critical, 4-7 medium, 8+ low — or "None found ✓"]
Topology: [star | chain | mesh | layered]

### Import Graph (simplified)

package/a → package/b, package/c
package/b → package/d
...

### Metrics

| Package | Fan-in | Fan-out | Transitive Dependents | Blast Radius |
|---|---|---|---|---|
| top/package | 12 | 3 | 25 | HIGH |
| ... | ... | ... | ... | ... |

### Issues

[SEVERITY] <finding>
  Location: <package or cycle path>
  Category: <cycle | topology | fan-in/out | orphan | blast-radius>
  Suggestion: <concrete fix>
```

**Scoring:**
- Start at 100
- Each direct cycle: -20
- Each indirect cycle: -10
- High fan-in + fan-out package: -5 each
- Orphan package: -3 each
- Mesh topology flag: -10

---

## When to use me

Use when:
- user asks about "dependency cycles", "import graph", "package dependencies"
- impact analysis of a change is needed ("what breaks if I change X?")
- system has more than 3–4 internal packages

Do NOT use me for:
- cohesion and coupling design principles → use `audit-core`
- complexity hotspots and maintainability → use `audit-complexity`
- security patterns → use `audit-security-basic`

---

## Relationship to other audit skills

This skill focuses on **dependency structure and topology**. For complementary dimensions:

- `audit-core` — cohesion, coupling, SRP, dependency direction (uses graph as input)
- `audit-complexity` — change hotspots (uses fan-in × lines metric)
- `audit-security-basic` — unsafe dependency flows (uses graph to trace untrusted input paths)

Load multiple audit skills for a comprehensive review. Each operates independently.
