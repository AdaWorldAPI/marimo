# marimo — Reactive Notebook Frontend

## Role in Stack
marimo is the frontend. Its reactive execution model (cells auto-rerun when
dependencies change) eliminates Jupyter's hidden-state problem. This makes it
better for both humans (no stale cells) and AI agents (write a cell, deps propagate).

## Current State
- Python-only (no polyglot kernel support)
- No Jupyter kernel protocol — talks directly to its own Python runtime
- Stored as .py files (git-friendly, not .ipynb)
- Can deploy notebooks as web apps

## Our Fork's Mission
Bridge marimo's reactive frontend to the Jupyter kernel wire protocol so it can
drive: evcxr (Rust), IRkernel (R), and custom graph magics (%%cypher, %%gremlin,
%%sparql, %%nars).

## Integration Points
- **kernel-protocol** → ZMQ wire format spec (what we implement)
- **graph-notebook** → magics to extract and wire through kernel protocol
- **lance-graph** → local %%cypher executes via semiring planner, not remote DB
- **quarto-r** → R cells for Bardioc/almato
- **ndarray** → SIMD kernels available in Rust cells via evcxr

## Upstream Tracking
Fork of github.com/marimo-team/marimo. Track upstream main.
Our changes go on feature branches, not main.
