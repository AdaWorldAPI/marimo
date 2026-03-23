# Blackboard — marimo

> Single-binary architecture: marimo's reactive runtime transcoded to Rust.

## Overview

marimo is a **reactive Python notebook** with a graph-based dataflow runtime. Cells are Python (or SQL) code blocks that automatically re-execute when their dependencies change. The frontend is a TypeScript/React app communicating with a Python backend over WebSocket.

## Execution Model

### Reactive Runtime

- **Location**: `marimo/_runtime/runtime.py` (~138K)
- Cells compile to Python code objects and execute in a shared `globals` namespace
- The `Runtime` class tracks cell state: `queued → running → idle` (or `disabled-transitively`)
- Three execution modes:
  - **Relaxed** (default): auto-runs dependent cells on change
  - **Strict**: cells only see references from direct dependencies (prevents hidden state)
  - **Lazy**: marks cells as stale, user must manually re-run

### Dependency Graph

- **Location**: `marimo/_runtime/dataflow/`
- `DirectedGraph` (`graph.py`): cells as nodes, edges from variable defs→refs
- `edges.py`: determines dependencies by analyzing variable definitions vs references
- `topology.py`: thread-safe graph structure management
- `cycles.py`: circular dependency detection
- Topological sort determines execution order (`runner/cell_runner.py`)

### Cell Execution Flow

1. Frontend sends `ExecuteCellCommand` via WebSocket
2. Runtime compiles cell via `_ast.compiler.compile_cell()`
3. Executor chain runs cell (with optional strict-mode decorator)
4. Output captured, formatted as MIME, sent back as `CellNotification`

## Cell Types

- **Python cells**: standard code with last-expression rendering
- **SQL cells**: queries against registered DB engines; can reference Python variables
- **Markdown cells**: rich text content
- **Setup cell**: special init cell (`SETUP_CELL_NAME`), no external refs
- **Test cells**: `test_`-prefixed functions, runnable with pytest

### Cell Definition (`_ast/cell.py`)

```python
CellImpl:
  code: str             # Source code
  defs: Set[str]        # Variables defined
  refs: Set[str]        # Variables referenced
  temporaries: Set[str] # _prefixed locals
  language: "python"|"sql"
  cell_id: CellId_t
  config: CellConfig    # disabled, hide_code, column
```

## Magic System

**marimo does NOT support IPython `%` / `%%` magics.**

Error: `"IPython magic commands (starting with %) are not supported."`

Migration path: `_convert/ipynb/to_ir.py` converts `%%sql` → SQL cell type; other magics flagged for manual handling.

## Wire Protocol

### Transport

WebSocket over ASGI/Starlette at `/ws/kernel`

### Messages

- **Commands** (Frontend → Backend): `msgspec.Struct` with kebab-case tags, discriminator field `type`
  - `ExecuteCellCommand`, `UpdateUIElementCommand`, `InstallPackagesCommand`, `SyncGraphCommand`
- **Notifications** (Backend → Frontend): tagged union, discriminator field `op`
  - `CellNotification` (output + status), `VariablesNotification`, `CompletionResultNotification`, `KernelReadyNotification`

### Output Format

```
CellNotification:
  cell_id, output (MIME), console, status, stale_inputs, run_id, timestamp
```

MIME types: `text/plain`, `text/html`, `image/png`, `application/vnd.plotly.v1+json`, `application/vnd.vega.v5+json`, etc.

## Kernel Architecture

1. **In-Process** (default edit mode): same process as server
2. **Isolated Process** (sandbox): separate Python process, ZMQ IPC (`_ipc/`)
3. **App Host** (production run mode): one kernel per notebook, horizontal scaling

**No non-Python kernel support currently.** Architecture is extensible via entrypoints.

## Rendering Pipeline

1. Object → MIME (via FormatterRegistry in `_output/formatting.py`)
2. Serialize to JSON via `msgspec`
3. Send over WebSocket
4. Frontend renders based on MIME type

Priority: registered formatters > `_mime_()` method > default repr > plain text

Built-in formatters for: Pandas/Polars DataFrames, Matplotlib/Plotly/Altair plots, JSON, rich tables.

## Plugin / Extension System

### Frontend Plugins (`frontend/src/plugins/`)

- **Stateful** (UI elements): sliders, inputs, dataframes — have persistent state
- **Stateless** (displays): images, tables, charts — read-only
- **Layout**: grid, sidebar, tabs

### Backend Plugins (`_plugins/`)

- 100+ UI components in `_plugins/ui/_impl/`
- Base class: `UIElement` in `_plugins/ui/_core/ui_element.py`

### Entrypoint Registry (`_entrypoints/`)

```python
KnownEntryPoint = Literal[
    "marimo.cell.executor",
    "marimo.cache.store",
    "marimo.kernel.lifespan",
    "marimo.server.asgi.lifespan",
    "marimo.server.asgi.middleware",
]
```

Register via `pyproject.toml` entry-points.

## Integration Points (for Polyglot Notebook)

### Adding Non-Python Kernels Would Require

1. New language in `Language` enum (`_ast/visitor.py`)
2. Parser in `_ast/parse.py`
3. Compiler changes in `_ast/compiler.py`
4. New executor via `marimo.cell.executor` entrypoint
5. Frontend support for cell type detection

### Key Paths for Kernel Protocol Bridge (Phase 2)

- `_runtime/executor.py` — cell execution interface (plug in external kernel)
- `_entrypoints/registry.py` — entrypoint discovery
- `_messaging/notification.py` — output messages (map Jupyter display_data → CellNotification)
- `_ast/cell.py` — cell language field (add "rust", "r", etc.)

### Key Paths for Graph Magics (Phase 3)

- `_plugins/ui/_impl/` — add graph visualization widget
- `_output/formatting.py` — register graph result formatters
- `_sql/` — pattern for non-Python cell execution (SQL already does this)

## Key Files

| File | Role |
|------|------|
| `_runtime/runtime.py` | Main execution engine |
| `_runtime/dataflow/graph.py` | Dependency graph |
| `_runtime/runner/cell_runner.py` | Cell execution orchestration |
| `_ast/cell.py` | Cell definition |
| `_ast/compiler.py` | Code compilation |
| `_messaging/notification.py` | Backend→Frontend messages |
| `_server/asgi.py` | ASGI app builder |
| `_session/session.py` | WebSocket session handler |
| `_plugins/ui/_core/ui_element.py` | UI element base class |
| `_output/formatting.py` | MIME formatter registry |
| `_entrypoints/registry.py` | Plugin discovery |
| `_sql/engines/` | SQL execution (pattern for foreign cells) |
