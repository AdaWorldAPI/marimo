# Progress

## Phase 1: Understand (current)
- [ ] Map marimo's execution model
- [ ] Map kernel protocol wire format
- [ ] Identify where kernel protocol hooks into marimo
- [ ] Inventory graph-notebook magics

## Phase 2: Kernel Protocol Bridge
- [ ] ZMQ client in marimo
- [ ] Cell type routing (Python → marimo runtime, %%magic → kernel)
- [ ] Message serialization/deserialization
- [ ] Kernel lifecycle (start/stop/restart)

## Phase 3: Graph Magics
- [ ] Port %%oc (openCypher)
- [ ] Port %%gremlin
- [ ] Port %%sparql
- [ ] Create %%nars (new, custom)
- [ ] vis.js graph rendering in marimo output

## Phase 4: Polyglot Kernels
- [ ] evcxr (Rust) kernel integration
- [ ] IRkernel (R) integration
- [ ] Arrow IPC variable sharing between cells

## Phase 5: lance-graph Local Engine
- [ ] %%cypher local path → lance-graph semiring planner
- [ ] Zero-copy results into marimo DataFrame
