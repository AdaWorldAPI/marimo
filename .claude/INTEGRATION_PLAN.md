# Project Runbook — Integration Plan

## The Product

A reactive polyglot notebook where graph queries are first-class citizens,
Rust kernels run SIMD-native, R analysts work in their language, and
the whole thing publishes to PDF without leaving the process.

## Repos

| Repo | Language | Role |
|------|----------|------|
| marimo | Python | Reactive frontend + execution engine |
| graph-notebook | Python | Graph magic implementations (%%oc, %%gremlin, %%sparql) |
| kernel-protocol | Spec | Jupyter wire protocol reference |
| quarto (TS) | TypeScript | Publishing CLI (upstream reference) |
| quarto-r | R | R bindings for Quarto (Bardioc/almato) |
| lance-graph | Rust | Local graph engine (Cypher→Semiring, blasgraph) |
| ndarray | Rust | SIMD kernel library |

## Phases

### Phase 1: Inventory (no code changes)
Read each codebase. Map the execution model, magic system, wire protocol,
and rendering pipeline. Understand before touching.

Deliverable: blackboard.md in each repo documents what exists and how it works.

### Phase 2: Kernel Protocol Bridge (marimo ← kernel-protocol)
Make marimo speak Jupyter kernel protocol for non-Python cells.
Python cells stay reactive in marimo's runtime. %%magic cells and
foreign-language cells route through kernel protocol to external kernels.

Deliverable: marimo can connect to evcxr (Rust) and IRkernel (R).

### Phase 3: Graph Magics (marimo ← graph-notebook)
Extract graph-notebook's magic implementations from Jupyter dependency.
Create standalone executors. Wire into marimo as cell types.

Deliverable: %%oc, %%gremlin, %%sparql cells work in marimo.
Graph results render as vis.js inline.

### Phase 4: Local Graph Engine (marimo ← lance-graph)
%%cypher cells can route locally to lance-graph's semiring planner.
No network hop. SIMD-native. Results as Arrow DataFrames.

Deliverable: %%cypher MATCH (a)-[:CAUSES]->(b) RETURN a, b
executes locally through blasgraph CSC engine.

### Phase 5: R Integration (marimo ← quarto-r ← IRkernel)
R cells work in marimo via IRkernel. Arrow IPC shares data between
R, Python, and Rust cells. quarto-r publishes the notebook.

Deliverable: Mixed R + Cypher + Python notebook publishes to PDF.
Bardioc/almato can use their existing R code.

### Phase 6: %%nars + Custom Magics
Add NARS truth-value query cells. Extensible magic system so anyone
can add new query languages.

Deliverable: %%nars cells query NARS endpoints, display truth values.

### Phase 7: quarto-rs (future)
Rust-native Quarto rendering. Pandoc AST manipulation in Rust.
Publish from inside lance-graph pipeline without shelling to Node.

Deliverable: `cargo add quarto` works. Notebook → PDF in-process.

## Dependency Graph

```
kernel-protocol (spec)
       ↓
marimo (frontend) ←── graph-notebook (magics)
   ↓         ↓
evcxr      IRkernel
(Rust)       (R)
   ↓         ↓
lance-graph  quarto-r
ndarray      Bardioc R code
```

## What NOT to Build

- Don't rewrite marimo in Rust. It's a Python app, keep it Python.
- Don't rewrite graph-notebook. Extract and adapt.
- Don't fork quarto CLI. Use quarto-r to call it.
- Don't build a new graph database. lance-graph IS the local engine.
- Don't build a new visualization library. vis.js exists.
