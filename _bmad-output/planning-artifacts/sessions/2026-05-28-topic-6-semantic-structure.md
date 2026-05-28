# Topic 6: Semantic Structure — Graph vs. Markdown

**Date:** 2026-05-28
**Participants:** Prajwal, Winston (Architect)
**Status:** Resolved
**Session refs:**
- All prior topic session documents

---

## Decisions

| # | Decision | Detail |
|---|----------|--------|
| D26 | **Neo4j graph DB in v1** | No interim step. Neo4j from day one as the canonical IDM store. |
| D27 | **Nodes: all atomic units** | Chapters, sections, paragraphs, images, tables — each is a Neo4j node with typed labels. |
| D28 | **Edges: all relationships** | `contains` (parent→child), `references` (text→image), `placed-before` (image→next-block), `next-sibling` (paragraph→paragraph). |
| D29 | **Markdown is derived** | Generated from Neo4j via Cypher traversal → linearized → Extended GFM. Never the source of truth. |
| D30 | **Export path** | Graph (Neo4j) → Markdown (Cypher traversal) → HTML (markdown renderer). |
| D31 | **Markdown standard** | Extended GFM — math (LaTeX), Mermaid diagrams, footnotes, definition lists, tables, task lists, strikethrough, autolinks. |
| D32 | **MongoDB scope** | Extraction run metadata, artifact tracker decisions, vision cache. **Not** graph storage. |

## Architecture

```
                     ┌───────────┐
Topics 1-4 ────────► │   Neo4j   │
                     │  (Graph)  │
                     │  Canonic. │
                     └─────┬─────┘
                           │
                     Cypher traversal →
                     linearize to markdown
                           │
                           ▼
                     ┌───────────┐
                     │  Markdown  │
                     │  (Ext. GFM)│
                     │  Derived   │
                     └─────┬─────┘
                           │
                     Render to HTML
                           │
                           ▼
                     ┌───────────┐
                     │   HTML    │
                     │  Export   │
                     └───────────┘
```

## Rationale

- Product core value = creating and maintaining relations between elements. Neo4j's relationship-first model is purpose-built for this.
- Markdown is a *derived view* for export — never the source of truth.
- Building on Neo4j from v1 avoids a data migration later. The relational model shapes extraction design from the start.

## Open Items

- **Node label taxonomy** — need to define all node types (Document, Chapter, Section, Paragraph, Image, Table, List, CodeBlock, Footnote, MathBlock) and their properties before writing Cypher queries.
- **Edge type taxonomy** — `CONTAINS`, `REFERENCES`, `PLACED_BEFORE`, `NEXT_SIBLING`, `FOOTNOTE_REF`, `MARGINALIA`. Any others?
- **Cypher traversal for markdown** — the query that linearizes a document for markdown generation needs design. Order by `CONTAINS` hierarchy + `NEXT_SIBLING` chain.
