# Topic 2: Artifact Classification & Iteration Tracker

**Date:** 2026-05-28
**Participants:** Prajwal, John (PM), Winston (Architect)
**Status:** Resolved
**Session refs:**
- `_bmad-output/planning-artifacts/sessions/2026-05-28-architect-stress-test.md`
- `_bmad-output/planning-artifacts/sessions/2026-05-28-topic-1-text-extraction.md`

---

## Decisions

### Accepted

| # | Decision | Detail |
|---|----------|--------|
| D7 | **Two-tier artifact classification** | Tier 1: Rules engine (position heuristics, char code analysis, spatial dedup, font analysis). Tier 2: Vision model for uncertain blocks only. |
| D8 | **Vision trigger: all uncertain blocks + cache dedup** | Every uncertain block goes to vision, UNLESS same region+font+size seen on ≥3 pages — then reuse cached result. |
| D9 | **Cache: MongoDB-first, Redis-compatible key** | Cache key = JSON serialized string `{regionX, regionY, fontName, fontSize}`. Structure designed so swapping MongoDB for Redis is a config change. |
| D10 | **Elasticsearch learning knowledge base** | Log every uncertain block with: what went wrong, edge case details, how vision model identified it. Update as iterations improve. Implementation delay approved — we use this *after* we understand failure patterns. |
| D11 | **Iteration tracker in MongoDB** | `decisions/` collection stores predicted label, human-corrected label, confidence, iteration batch, rationale. Enables per-class accuracy reports, confusion matrices, improvement metrics. |
| D12 | **All-blocks strategy first** | Send all uncertain blocks to vision. "Sample pages every 10th" is deferred until we understand failure patterns well enough to optimize. |
| D13 | **NestJS local web service** | Runs on user's machine. MongoDB + local filesystem for PDF storage. Web UI for QA review. |
| D14 | **LangChain abstraction** | Vendor-agnostic — vision model slot interchangeable between GLM 4.6V Flash, Qwen3 VL 8B (local), or cloud models. |

### Deferred

| # | Idea | Why Deferred | Trigger |
|---|------|--------------|---------|
| X3 | **Sample-page-only vision strategy** | Need failure pattern understanding first to know which pages to sample. | After Elasticsearch knowledge base has ≥500 logged uncertain blocks with iteration feedback. |

---

## Architecture Design

### Classification Flow

```
pdfjs-dist items
       │
       ▼
┌─────────────────────┐
│ Tier 1: Rules Engine │
│ - Position heuristics│
│ - Char code analysis │
│ - Spatial dedup      │
│ - Font analysis      │
└──────────┬──────────┘
           │
    ┌──────┴──────┐
    │             │
  Confident    Uncertain
    │             │
    │             ▼
    │    ┌──────────────────┐
    │    │ Cache lookup     │
    │    │ (MongoDB/Redis   │
    │    │ compatible key)  │
    │    └────┬─────────────┘
    │         │          │
    │      Cache       Cache
    │      hit        miss
    │         │          │
    │         ▼          ▼
    │    ┌────────┐ ┌──────────┐
    │    │ Reuse  │ │ Vision   │
    │    │ cached │ │ model    │
    │    │ result │ │ (LC)     │
    │    └────────┘ └────┬─────┘
    │                    │
    │                    ▼
    │           ┌────────────────┐
    │           │ Store in cache │
    │           │ + log to ES    │
    │           │ (learning KB)  │
    │           └────────────────┘
    │                    │
    └────────┬───────────┘
             ▼
    ┌────────────────┐
    │ Classified     │
    │ blocks with    │
    │ roles + conf   │
    └────────────────┘
```

### Feedback Loop

```
Classify → Output + Flags → Human Review (QA UI) → Store Decision
                                                          │
                                                          ▼
                                              ┌─────────────────────┐
                                              │ MongoDB: decisions/  │
                                              │ - predictedLabel    │
                                              │ - humanLabel        │
                                              │ - wasCorrect       │
                                              │ - iterationBatch   │
                                              │ - humanRationale   │
                                              └─────────┬───────────┘
                                                        │
                                                        ▼
                                              ┌─────────────────────┐
                                              │ Reports:            │
                                              │ - Per-class accuracy│
                                              │ - Confusion matrix  │
                                              │ - Correction rate   │
                                              │ - Improvement trend │
                                              └─────────────────────┘
```

---

## Data Contracts

### MongoDB Collections

```typescript
// documents/ — one per PDF
{
  _id: ObjectId,
  filename: string,
  filePath: string,          // local filesystem path
  pageCount: number,
  createdAt: Date,
  status: 'pending' | 'processing' | 'done' | 'failed'
}

// extractions/ — one per extraction run
{
  _id: ObjectId,
  documentId: ObjectId,
  version: string,            // extraction run version
  pipelineVersion: string,
  pages: ExtractedPage[],     // from Topic 1 IDM seed
  totalBlocks: number,
  artifactDecisions: ArtifactDecision[],
  metrics: {
    visionCalls: number,
    ruleHits: number,
    cacheHits: number,
    avgConfidence: number
  }
}

// decisions/ — iteration tracker (human feedback)
{
  _id: ObjectId,
  extractionId: ObjectId,
  blockId: string,
  classifierVersion: string,
  predictedLabel: string,       // 'content' | 'artifact-header' | 'artifact-footer' | ...
  predictionConfidence: number,
  humanLabel: string | null,    // null = not reviewed yet
  humanCorrectedAt: Date | null,
  humanRationale: string | null,
  wasCorrect: boolean | null,   // null = pending
  iterationBatch: number,
  edgeCaseNotes: string | null  // from ES cross-ref
}

// cache/ — vision classification cache
{
  _id: ObjectId,
  signature: string,           // JSON serialized: "{regionX, regionY, fontName, fontSize}"
  key: {
    regionX: number,           // approximate x region bucket
    regionY: number,           // approximate y region bucket
    fontName: string,
    fontSize: number
  },
  result: {
    role: string,
    confidence: number
  },
  hitCount: number,
  lastUsed: Date
}
```

### Artifact Label Taxonomy

```
content                    — body text
artifact-header            — running header (chapter title on every page)
artifact-footer            — running footer
artifact-page-number       — page number
artifact-garbage           — anti-copy chars, invisible text, broken encoding
artifact-duplicate         — duplicated content (pull-quotes, repeated headings)
design-sidebar             — marginal note, sidebar aside (author intent)
design-pull-quote          — highlighted quotation (author intent)
design-ornament            — decorative separator, divider (author intent)
design-toc-entries         — table of contents entries
design-folio-label         — section label in page margin (e.g., "§2.4")
```

---

## Open Items

1. **Rules engine thresholds** — need to define exact heuristics (y-repeat tolerance, font-name matching, invisible-char detection). Will calibrate during iteration.
2. **Vision prompt template for artifact classification** — needs to ask "is this highlighted block a header, footer, sidebar, body, or other?" with output constrained to the taxonomy above.
3. **Elasticsearch schema for learning KB** — deferred until post-MVP but needs rough design awareness now.
4. **Cache bucket sizing** — what granularity of `regionX, regionY, fontName, fontSize` constitutes a "match"? Needs experimentation.
