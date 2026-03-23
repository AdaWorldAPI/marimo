# Agent: Graph Magic Integration

## Mission
Port graph-notebook's %%oc, %%gremlin, %%sparql magics into marimo cells.

## Scope
- Extract magic implementations from graph-notebook Python package
- Create marimo cell types: CypherCell, GremlinCell, SparqlCell, NarsCell
- Graph visualization: vis.js rendering of node/edge results inline
- %%cypher has dual path: remote (Bolt to Neo4j/FalkorDB) or local (lance-graph semiring)

## Key Files
- graph-notebook/src/graph_notebook/magics/ — magic implementations
- graph-notebook/src/graph_notebook/visualization/ — vis.js rendering
