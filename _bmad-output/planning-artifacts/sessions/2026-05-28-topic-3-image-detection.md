# Topic 3: Image Detection & Positioning

**Date:** 2026-05-28
**Participants:** Prajwal, John (PM), Winston (Architect)
**Status:** Resolved
**Session refs:**
- `_bmad-output/planning-artifacts/sessions/2026-05-28-open-questions-resolution.md`
- All prior session documents

---

## Decisions

### Accepted

| # | Decision | Detail |
|---|----------|--------|
| D15 | **Two algorithms run in parallel** | **Triangulate** (three-source merge) and **GridZoom** (recursive 8×8 grid) both run on every page. Results compared, merged, logged. |
| D16 | **GridZoom algorithm** | 3-level recursive refinement. Level 1: full page into 8×8 grid → ask which cells contain images. Level 2: crop to found region → 8×8 again. Level 3: refine → also ask for image description. Multiple images can be labeled and tracked simultaneously per iteration. Previous region context passed forward to fine-tune. |
| D17 | **Triangulate algorithm** | Merge pdfjs-dist operator list (precise position, misses vector) + vision model layout classification + text-gap heuristic. |
| D18 | **GridZoom failure handling** | If any iteration fails → drop that image candidate, log full trace to ES (which level, cells tried, model output), move on. No retry. |
| D19 | **ES logging: success + failure** | Every GridZoom result logged to Elasticsearch — level traces, model output, bounding box, image description (Level 3). Both algorithms' outputs compared. |
| D20 | **Post-launch validation** | Run both in production. Metrics: agreement rate, coverage (which finds more), false positives, Triangulate misses caught by GridZoom. Kill worse performer once sufficient data collected. |

### Deferred

| # | Idea | Why Deferred |
|---|------|--------------|
| X4 | **Remove one algorithm** | Need real-world performance data first. Let both run, measure, then trim. |

---

## Architecture

### Per-page Flow

```
                    ┌──────────────┐
                    │  Render page  │
                    │  to PNG       │
                    └──────┬───────┘
                           │
              ┌────────────┴────────────┐
              ▼                         ▼
    ┌─────────────────┐       ┌─────────────────┐
    │  Triangulate     │       │  GridZoom        │
    │                  │       │                  │
    │  A: pdfjs ops    │       │  Level 1: 8×8    │
    │  B: vision       │       │  grid → cells    │
    │  C: text-gap     │       │  with images     │
    │                  │       │                  │
    │  Merge A+B+C     │       │  Level 2: crop   │
    │  → images[]      │       │  → 8×8 refine    │
    └────────┬────────┘       │                  │
             │                │  Level 3: crop   │
             │                │  → 8×8 refine    │
             │                │  + description   │
             │                └────────┬─────────┘
             │                         │
             └────────┬────────────────┘
                      ▼
            ┌─────────────────────┐
            │  Compare + Merge    │
            │  Log both to ES     │
            │  (success+failure)  │
            └──────────┬──────────┘
                       ▼
            ┌─────────────────────┐
            │  Detected images    │
            │  with bbox, type,   │
            │  confidence, source │
            └─────────────────────┘
```

### GridZoom Detail

```
Level 1 ─────────────────────────────────────
  Overlay 8×8 grid (red lines + red cell numbers)
  Prompt: "Which cells contain an image? 
           Label multiple images with A, B, C..."
  Output: { A: [D4, D5, E4, E5], B: [A1, A2, B1, B2] }

Level 2 ─────────────────────────────────────
  Crop page to region of each labeled image
  Overlay 8×8 grid on cropped region
  Include previous level's labels for context
  Prompt: "Refine the bounding box. Which cells 
           contain image A? Image B?"
  Output: refined rectangles per image

Level 3 ─────────────────────────────────────
  Crop to refined per-image rectangle
  Overlay 8×8 grid
  Prompt: "Final refinement. Also describe this image."
  Output: refined bbox + description
  Log: full trace (levels 1-3, model responses, final bbox) to ES
```

### Output Contract

```typescript
interface DetectedImage {
  id: string
  bbox: { x1: number; y1: number; x2: number; y2: number }
  positionPrecision: 'exact' | 'approximate'

  // Which algorithm produced this
  source: 'triangulate' | 'gridzoom' | 'merged'
  primarySource: 'triangulate' | 'gridzoom'  // which was preferred

  // Quality
  confidence: number
  triangulateConfidence?: number
  gridZoomConfidence?: number
  algorithmsAgreed: boolean  // both found same region

  // Content
  estimatedType: 'photo' | 'diagram' | 'illustration' | 'icon' | 'decoration'
  description?: string     // from GridZoom Level 3

  // Context
  adjacentBlockIds: string[]  // surrounding text blocks
  pageNumber: number

  // Trace (from GridZoom)
  gridTrace?: {
    level: number
    cells: string[]
    pageSection: string
  }[]
}
```

---

## Open Items

1. **GridZoom cell numbering scheme** — A1-H8? 1-64? Needs to be unambiguous to the vision model.
2. **Triangulate overlap threshold** — what IoU constitutes "same image" between the two algorithms? 0.5? 0.7?
3. **Multiple images in GridZoom** — how many simultaneous labels (A, B, C...) can the vision model track reliably per iteration? Needs testing.
