# Agent: Kernel Bridge

## Mission
Implement Jupyter kernel protocol support in marimo's frontend.

## Scope
- Understand marimo's current execution model (marimo/_runtime/)
- Understand Jupyter kernel wire protocol (ZMQ, see kernel-protocol repo)
- Design adapter: marimo cell → kernel protocol message → kernel → result → marimo cell
- Cell type detection: Python cells use marimo runtime, %%magic cells route to kernels

## Key Files
- marimo/_runtime/ — current execution engine
- marimo/_server/ — websocket server
- kernel-protocol/docs/messaging.rst — the wire spec

## Constraints
- Don't break marimo's reactive model for Python cells
- Kernel cells may not be reactive (R/Rust don't have dep tracking)
- Variable sharing between Python and kernel cells via Arrow IPC
