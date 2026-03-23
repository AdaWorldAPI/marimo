# marimo — Transcode to Rust

Read the Python source in this repo. Read `.claude/SCOPE_A_FINDINGS.md`.
Transcode marimo's reactive cell runtime to a standalone Rust crate.

The crate must work on its own. No dependency on graph-notebook,
kernel-protocol, quarto, lance-graph, ndarray, or rs-graph-llm.

Output: a Rust crate in this repo that does reactive cell execution.
Someone adds it to their Cargo.toml, they get a reactive runtime.

Read first. Transcode. Test.
