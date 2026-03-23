# Scope A Findings: marimo Runtime Architecture

This document captures the core runtime architecture of marimo for transcoding to Rust.
All code references are from the marimo repository at `/home/user/marimo/marimo/`.

---

## 1. Dependency Tracking Model

marimo uses **static AST analysis** to extract refs (references) and defs (definitions) from each cell, then builds a directed graph of cell dependencies.

### AST Analysis Pipeline

Cell code is compiled and analyzed in `_ast/compiler.py:compile_cell()`:

```python
# _ast/compiler.py:294-295
v = ScopedVisitor("cell_" + cell_id)
v.visit(module)
```

The `ScopedVisitor` (in `_ast/visitor.py`) walks the Python AST to extract:
- **`v.defs`**: Names defined/assigned in the cell (at module scope)
- **`v.refs`**: Names referenced but not defined locally (external dependencies)
- **`v.deleted_refs`**: Names deleted with `del` that are defined elsewhere
- **`v.variable_data`**: Rich metadata per definition (kind, required_refs, import_data, etc.)
- **`v.sql_refs`**: SQL-specific hierarchical references

Local variables (names starting with `_`) are separated into `temporaries` and excluded from the dependency graph:

```python
# _ast/compiler.py:355-361
nonlocals = {name for name in v.defs if not is_local(name)}
temporaries = v.defs - nonlocals
variable_data = {
    name: v.variable_data[name]
    for name in nonlocals
    if name in v.variable_data
}
```

### Key Data Structures

**`Name`** is just `str` (`_ast/visitor.py:31`).

**`VariableData`** (`_ast/visitor.py:63-100`):
```python
@dataclass
class VariableData:
    kind: Union[
        Literal["function", "class", "import", "variable", "temporary"],
        SQLKind,  # "table", "view", "schema", "catalog"
    ] = "variable"
    required_refs: set[Name] = field(default_factory=set)
    unbounded_refs: set[Name] = field(default_factory=set)
    annotation_data: Optional[AnnotationData] = None
    import_data: Optional[ImportData] = None
    qualified_name: Optional[str] = None

    @property
    def language(self) -> Language:  # "python" | "sql"
```

**`CellImpl`** (`_ast/cell.py:154-176`) -- the internal cell representation:
```python
@dataclasses.dataclass(frozen=True)
class CellImpl:
    key: int               # hash of code
    code: str
    mod: ast.Module
    defs: set[Name]        # non-local definitions
    refs: set[Name]        # external references
    sql_refs: dict[Name, SQLRef]
    temporaries: set[Name] # local-only definitions
    variable_data: dict[Name, list[VariableData]]
    deleted_refs: set[Name]
    body: Optional[CodeType]      # compiled body (all but last expr)
    last_expr: Optional[CodeType] # compiled last expression (the cell output)
    language: Language     # "python" | "sql"
    cell_id: CellId_t
    # Mutable fields:
    config: CellConfig
    _status: RuntimeState          # idle/queued/running/disabled-transitively
    _run_result_status: RunResultStatus  # success/exception/cancelled/interrupted/marimo-error/disabled
    _stale: CellStaleState
    _output: CellOutput
```

### Graph Structure

**`DirectedGraph`** (`_runtime/dataflow/graph.py:32`) coordinates:
- **`topology: MutableGraphTopology`** -- nodes (cells) and edges (parent/child)
- **`definition_registry: DefinitionRegistry`** -- maps `Name -> set[CellId_t]`
- **`cycle_tracker: CycleTracker`** -- detects cycles

**`MutableGraphTopology`** (`_runtime/dataflow/topology.py:46`):
```python
@dataclass
class MutableGraphTopology:
    _cells: dict[CellId_t, CellImpl]
    _children: dict[CellId_t, set[CellId_t]]  # edge (u,v) means v refs something u defines
    _parents: dict[CellId_t, set[CellId_t]]    # reverse edges
```

**`DefinitionRegistry`** (`_runtime/dataflow/definitions.py:16`):
```python
@dataclass
class DefinitionRegistry:
    definitions: dict[Name, set[CellId_t]]                    # name -> defining cells
    typed_definitions: dict[tuple[Name, str], set[CellId_t]]  # (name, kind) -> cells
    definition_types: dict[Name, set[str]]                    # name -> set of kinds
```

### Edge Computation

When a cell is registered (`graph.register_cell()`), edges are computed by `_runtime/dataflow/edges.py:compute_edges_for_cell()`:

1. **Children (dependents)**: For each name this cell defines, find all other cells that reference it.
2. **Parents (dependencies)**: For each name this cell references, find the cell(s) that define it.
3. **Delete edges**: If a cell deletes a variable defined elsewhere, it becomes a child of cells referencing that variable.

SQL has special handling: SQL variables don't leak to Python cells, but Python variables leak to SQL cells. SQL supports hierarchical references (schema.table).

---

## 2. Execution Order Algorithm

### Topological Sort with Registration-Order Tiebreaking

Located in `_runtime/dataflow/__init__.py:97-132`:

```python
def topological_sort(
    graph: GraphTopology, cell_ids: Collection[CellId_t]
) -> list[CellId_t]:
    """Sort cell_ids in topological order using a heap queue.
    Ties broken by registration order (cells registered earlier first)."""
    from heapq import heapify, heappop, heappush

    registration_order = list(graph.cells.keys())
    top_down_keys = {key: idx for idx, key in enumerate(registration_order)}

    parents, children = induced_subgraph(graph, cell_ids)
    in_degree = {cid: len(parents[cid]) for cid in cell_ids}

    heap = [(top_down_keys[cid], cid) for cid in cell_ids if in_degree[cid] == 0]
    heapify(heap)

    sorted_cell_ids: list[CellId_t] = []
    while heap:
        _, cid = heappop(heap)
        sorted_cell_ids.append(cid)
        for child in children[cid]:
            in_degree[child] -= 1
            if in_degree[child] == 0:
                heappush(heap, (top_down_keys[child], child))
    return sorted_cell_ids
```

### Cells-to-Run Computation

The `Runner` class (`_runtime/runner/cell_runner.py:124`) computes which cells to run:

```python
# cell_runner.py:181-221
@staticmethod
def compute_cells_to_run(graph, roots, excluded_cells, execution_mode):
    # Always run stale ancestors
    cells_to_run = roots.union(
        dataflow.transitive_closure(
            graph, roots, children=False, inclusive=False,
            predicate=lambda cell: cell.stale,
        )
    )

    if execution_mode == "autorun":
        # In autorun mode, also run all descendants
        cells_to_run = dataflow.transitive_closure(
            graph, cells_to_run,
            relatives=dataflow.get_import_block_relatives(graph),
        )

    sorted_cells = dataflow.topological_sort(
        graph, cells_to_run - excluded_cells,
    )
    return sorted_cells
```

### Transitive Closure

`_runtime/dataflow/__init__.py:22-68` -- BFS traversal of the graph in either direction:

```python
def transitive_closure(
    graph, cell_ids, *, children=True, inclusive=True,
    relatives=None, predicate=None,
) -> set[CellId_t]:
    """BFS to find all descendants (children=True) or ancestors (children=False)."""
    result = cell_ids.copy() if inclusive else set()
    seen = cell_ids.copy()
    queue = deque(cell_ids)
    while queue:
        cid = queue.popleft()
        for relative in (graph.children[cid] if children else graph.parents[cid]) - seen:
            if predicate(graph.cells[relative]):
                result.add(relative)
            seen.add(relative)
            queue.append(relative)
    return result
```

### Execution Loop

`Runner.run_all()` (`cell_runner.py:686-811`) iterates through the topologically sorted queue:

1. Run **preparation hooks**.
2. For each cell in queue:
   - Skip if cancelled (ancestor raised exception), disabled, or transitively disabled.
   - Run **pre-execution hooks** (set status to "running", remove old UI elements).
   - Execute the cell via `Runner.run()` which calls the executor.
   - Run **post-execution hooks** (broadcast output, update variables, set status).
3. Run **on-finish hooks**.

Cell execution itself (`Runner.run()`, line 463) calls `executor.execute_cell()` or `execute_cell_async()` depending on whether the cell is a coroutine.

---

## 3. Cell State Model

### Runtime States (`_ast/cell.py:97-99`)

```python
RuntimeStateType = Literal[
    "idle",                    # cell has run with latest inputs
    "queued",                  # cell is queued to run
    "running",                 # cell is running
    "disabled-transitively",   # cell disabled because a parent is disabled
]
```

### Run Result Statuses (`_ast/cell.py:112-118`)

```python
RunResultStatusType = Literal[
    "success",       # cell ran successfully
    "exception",     # cell raised an exception
    "cancelled",     # ancestor raised an exception
    "interrupted",   # user interrupted (Ctrl+C)
    "marimo-error",  # cell prevented from executing (e.g., cycle)
    "disabled",      # skipped because cell is disabled
]
```

### Stale State

Cells become **stale** when their dependencies change but they haven't been re-executed. Set via `graph.set_stale()` which propagates through the transitive closure. In lazy mode (`on_cell_change="lazy"`), stale cells are marked but not automatically re-run.

### State Transitions

1. **User edits cell** -> `ExecuteCellsCommand` sent -> cell registered in graph -> set to `"queued"` -> `"running"` -> `"idle"` (or exception status).
2. **Dependency changes** in autorun mode -> descendants queued automatically.
3. **Dependency changes** in lazy mode -> descendants marked `stale`.
4. **Cell disabled** -> `"disabled-transitively"` propagated to all descendants.
5. **Exception** in cell -> cell gets `"exception"` status -> all descendants `"cancelled"`.

### Cell Inputs/Outputs

- **Inputs**: `cell.refs` (set of variable names the cell reads from other cells)
- **Outputs**: `cell.defs` (set of variable names the cell defines for other cells)
- **Visual output**: The last expression of the cell (compiled separately as `last_expr`)
- **Console output**: stdout/stderr captured per-cell

---

## 4. Server Framework

marimo uses **Starlette** (ASGI framework) with **uvicorn** as the ASGI server.

Evidence from `_server/router.py`:
```python
from starlette.responses import FileResponse, Response
from starlette.routing import Mount, Router

class APIRouter(Router):
    def post(self, path, ...): ...
    def websocket(self, path, ...): ...
```

From `_server/asgi.py`:
```python
from starlette.types import ASGIApp, Receive, Scope, Send
```

From `_server/api/endpoints/ws_endpoint.py`:
```python
from starlette.websockets import WebSocket, WebSocketState
```

The app uses Starlette's `Router` for HTTP endpoints and native WebSocket support. Serialization uses **msgspec** (not pydantic or dataclasses-json).

---

## 5. WebSocket Protocol

### Wire Format

Messages are sent as JSON text over WebSocket in this envelope format:

**Kernel -> Frontend** (from `_server/api/endpoints/ws/ws_formatter.py`):
```json
{"op": "<operation-name>", "data": { ... notification fields ... }}
```

The `op` field is the `tag` value from the msgspec struct (e.g., `"cell-op"`, `"kernel-ready"`).
The `data` field contains the full serialized notification struct (which also includes the `"op"` field internally due to msgspec tagged unions).

**Frontend -> Kernel**: Commands are sent as HTTP POST requests to REST endpoints, NOT over WebSocket. The WebSocket is read-only from the kernel's perspective (the frontend only sends a disconnect signal over it).

From `ws_message_loop.py:118`:
```python
# Wait for disconnection (frontend doesn't send messages over WS)
await self.websocket.receive_text()
```

### Command Message Format (HTTP POST)

Commands use msgspec tagged unions with `tag_field="type"` and `rename="camel"`:

```python
class Command(msgspec.Struct, rename="camel", tag_field="type", tag=kebab_case):
    pass
```

The tag is automatically derived from the class name by stripping "Command" suffix and converting to kebab-case. Field names are serialized as camelCase.

Example `ExecuteCellsCommand`:
```json
{
  "type": "execute-cells",
  "cellIds": ["cell-abc123", "cell-def456"],
  "codes": ["x = 1", "y = x + 1"],
  "timestamp": 1711234567.89
}
```

Example `UpdateUIElementCommand`:
```json
{
  "type": "update-ui-element",
  "objectIds": ["ui-slider-1"],
  "values": [42],
  "token": "uuid-string"
}
```

### Key Notification Types (Kernel -> Frontend)

All notifications extend `Notification(msgspec.Struct, tag_field="op")`.

**`CellNotification`** (tag=`"cell-op"`) -- the most frequent message:
```python
class CellNotification(Notification, tag="cell-op"):
    cell_id: CellId_t
    output: Optional[CellOutput]           # cell visual output
    console: Optional[Union[CellOutput, list[CellOutput]]]  # stdout/stderr
    status: Optional[RuntimeStateType]     # "idle"/"queued"/"running"/...
    stale_inputs: Optional[bool]
    run_id: Optional[RunId_t]
    timestamp: float
```

**`KernelReadyNotification`** (tag=`"kernel-ready"`) -- sent once on connection:
```python
class KernelReadyNotification(Notification, tag="kernel-ready"):
    cell_ids: tuple[CellId_t, ...]
    codes: tuple[str, ...]
    names: tuple[str, ...]
    layout: Optional[LayoutConfig]
    configs: tuple[CellConfig, ...]
    resumed: bool
    ui_values: Optional[dict[str, JSONType]]
    last_executed_code: Optional[dict[CellId_t, str]]
    last_execution_time: Optional[dict[CellId_t, float]]
    app_config: _AppConfig
    kiosk: bool
    capabilities: KernelCapabilitiesNotification
    auto_instantiated: bool
```

**`VariablesNotification`** (tag=`"variables"`) -- dataflow graph for the variables panel:
```python
class VariablesNotification(Notification, tag="variables"):
    variables: list[VariableDeclarationNotification]

class VariableDeclarationNotification(msgspec.Struct):
    name: str
    declared_by: list[CellId_t]
    used_by: list[CellId_t]
```

**`VariableValuesNotification`** (tag=`"variable-values"`):
```python
class VariableValuesNotification(Notification, tag="variable-values"):
    variables: list[VariableValue]

class VariableValue(BaseStruct):
    name: str
    value: Optional[str]    # string representation
    datatype: Optional[str]
```

### Complete List of Command Types (Frontend -> Kernel)

All defined in `_runtime/commands.py` as the `CommandMessage` union:

| Command Class | Tag (type field) | Key Fields |
|---|---|---|
| `CreateNotebookCommand` | `create-notebook` | |
| `RenameNotebookCommand` | `rename-notebook` | `filename` |
| `CodeCompletionCommand` | `code-completion` | `id`, `document`, `cell_id` |
| `ExecuteCellsCommand` | `execute-cells` | `cell_ids`, `codes` |
| `ExecuteScratchpadCommand` | `execute-scratchpad` | `code` |
| `ExecuteStaleCellsCommand` | `execute-stale-cells` | |
| `DebugCellCommand` | `debug-cell` | `cell_id` |
| `DeleteCellCommand` | `delete-cell` | `cell_id` |
| `SyncGraphCommand` | `sync-graph` | `cells`, `run_ids`, `delete_ids` |
| `UpdateCellConfigCommand` | `update-cell-config` | `configs` |
| `InstallPackagesCommand` | `install-packages` | `packages`, `manager` |
| `UpdateUIElementCommand` | `update-ui-element` | `object_ids`, `values`, `token` |
| `ModelCommand` | `model` | `model_id`, `message` |
| `InvokeFunctionCommand` | `invoke-function` | `function_call_id`, `namespace`, `function_name`, `args` |
| `UpdateUserConfigCommand` | `update-user-config` | `config` |
| `PreviewDatasetColumnCommand` | `preview-dataset-column` | `table_name`, `column_name`, `source_type` |
| `PreviewSQLTableCommand` | `preview-sql-table` | `request_id`, `table`, `connection` |
| `ListSQLTablesCommand` | `list-sql-tables` | `request_id`, `connection` |
| `ListDataSourceConnectionCommand` | `list-data-source-connection` | |
| `ValidateSQLCommand` | `validate-sql` | `request_id`, `code`, `cell_id` |
| `StorageListEntriesCommand` | `storage-list-entries` | `request_id`, `namespace`, `prefix` |
| `StorageDownloadCommand` | `storage-download` | `request_id`, `namespace`, `key` |
| `ListSecretKeysCommand` | `list-secret-keys` | |
| `RefreshSecretsCommand` | `refresh-secrets` | |
| `ClearCacheCommand` | `clear-cache` | |
| `GetCacheInfoCommand` | `get-cache-info` | |
| `StopKernelCommand` | `stop-kernel` | |

### Complete List of Notification Types (Kernel -> Frontend)

All in `NotificationMessage` union (`_messaging/notification.py:788-838`):

| Notification Class | Tag (op field) |
|---|---|
| `CellNotification` | `cell-op` |
| `FunctionCallResultNotification` | `function-call-result` |
| `UIElementMessageNotification` | `send-ui-element-message` |
| `ModelLifecycleNotification` | `model-lifecycle` |
| `RemoveUIElementsNotification` | `remove-ui-elements` |
| `ReloadNotification` | `reload` |
| `ReconnectedNotification` | `reconnected` |
| `InterruptedNotification` | `interrupted` |
| `CompletedRunNotification` | `completed-run` |
| `KernelReadyNotification` | `kernel-ready` |
| `CompletionResultNotification` | `completion-result` |
| `AlertNotification` | `alert` |
| `BannerNotification` | `banner` |
| `MissingPackageAlertNotification` | `missing-package-alert` |
| `InstallingPackageAlertNotification` | `installing-package-alert` |
| `StartupLogsNotification` | `startup-logs` |
| `KernelStartupErrorNotification` | `kernel-startup-error` |
| `VariablesNotification` | `variables` |
| `VariableValuesNotification` | `variable-values` |
| `QueryParamsSetNotification` | `query-params-set` |
| `QueryParamsAppendNotification` | `query-params-append` |
| `QueryParamsDeleteNotification` | `query-params-delete` |
| `QueryParamsClearNotification` | `query-params-clear` |
| `DatasetsNotification` | `datasets` |
| `DataColumnPreviewNotification` | `data-column-preview` |
| `SQLTablePreviewNotification` | `sql-table-preview` |
| `SQLTableListPreviewNotification` | `sql-table-list-preview` |
| `DataSourceConnectionsNotification` | `data-source-connections` |
| `ValidateSQLResultNotification` | `validate-sql-result` |
| `StorageNamespacesNotification` | `storage-namespaces` |
| `StorageEntriesNotification` | `storage-entries` |
| `StorageDownloadReadyNotification` | `storage-download-ready` |
| `SecretKeysResultNotification` | `secret-keys-result` |
| `CacheClearedNotification` | `cache-cleared` |
| `CacheInfoNotification` | `cache-info` |
| `FocusCellNotification` | `focus-cell` |
| `UpdateCellCodesNotification` | `update-cell-codes` |
| `UpdateCellIdsNotification` | `update-cell-ids` |

### KernelMessage Serialization

`KernelMessage` is a `NewType("KernelMessage", bytes)` -- just raw JSON bytes.

Serialization uses msgspec:
```python
# _messaging/serde.py
def serialize_kernel_message(message: NotificationMessage) -> KernelMessage:
    return KernelMessage(encode_json_bytes(message))
```

The wire format wraps it:
```python
# ws_formatter.py
def format_wire_message(op: str, data: bytes) -> str:
    return f'{{"op": "{op}", "data": {data.decode("utf-8")}}}'
```

### Communication Architecture Summary

```
Frontend (TypeScript)
    |
    |--- HTTP POST /api/kernel/execute_cells  -----> Starlette REST endpoint
    |--- HTTP POST /api/kernel/set_ui_element -----> Starlette REST endpoint
    |                                                    |
    |                                            put_control_request()
    |                                                    |
    |                                                    v
    |                                            Kernel.handle_message()
    |                                                    |
    |                                            Runner.run_all()
    |                                                    |
    |<--- WebSocket /ws <--- message_queue <--- broadcast_notification()
```

The kernel runs in a separate process, connected to the server via `multiprocessing.connection`. Commands flow: HTTP endpoint -> session -> kernel process queue. Notifications flow: kernel process -> pipe -> session -> WebSocket message queue -> frontend.
