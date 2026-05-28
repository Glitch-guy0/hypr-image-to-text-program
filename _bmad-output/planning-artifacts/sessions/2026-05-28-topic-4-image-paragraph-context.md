# Topic 4: Image-Paragraph Context

**Date:** 2026-05-28
**Participants:** Prajwal, John (PM), Winston (Architect), Mary (Analyst)
**Status:** Resolved
**Session refs:**
- `_bmad-output/planning-artifacts/sessions/2026-05-28-topic-3-image-detection.md`
- `_bmad-output/planning-artifacts/sessions/2026-05-28-open-questions-resolution.md`

---

## Problem Statement

PDFs place images at absolute coordinates, often mid-paragraph. The pipeline must determine **where each image belongs in the reading flow** and **how it relates to surrounding text**, then place the alt-text label at the correct semantic boundary.

---

## Three Document Cases

### Case A: Standalone Explainer
```
...end of previous paragraph.

[Diagram: Cell mitosis stages]

The diagram illustrates...
```
- Image describes itself. Reader can take it or leave it.
- **Label placement:** Standalone block in reading order.

### Case B: Text References the Image
```
...is shown in the diagram below.

[Diagram: Cell mitosis stages]

As you can see, the process begins with...
```
- Image provides context FOR the paragraph, not the other way around.
- **Label placement:** After-paragraph (attached to the referencing paragraph).

### Case C: Image Interrupts Paragraph (most common PDF issue)
```
...the process continues through mitosis,
[Diagram: Cell mitosis stages]
which eventually leads to cytokinesis.
```
- Image fragments an atomic knowledge unit (paragraph).
- **Decision:** Convert to **after-paragraph** — same treatment as Case A.
- **Rationale:** Reader gets complete paragraph first, then the image label. Never break the paragraph.

---

## Two-Axis Classification

### Axis 1: Placement in Reading Flow

| Value | Meaning |
|-------|---------|
| `before-paragraph` | Image precedes the paragraph it belongs to |
| `after-paragraph` | Image follows the paragraph it belongs to (or inline converted to after) |
| `standalone` | No clear anchor paragraph — image is its own block |

### Axis 2: Semantic Relationship

| Value | Meaning |
|-------|---------|
| `referenced` | Surrounding text explicitly references the image ("as shown in Fig 2.1") |
| `not-referenced` | Image is illustrative/standalone; text does not mention it |
| `ambiguous` | Cannot determine from immediate context |

### Decision Matrix

| | Referenced | Not referenced |
|---|---|---|
| **Before paragraph** | Image introduces the concept | Decorative / section opener |
| **After paragraph** | "as shown above" — attach to paragraph | Image illustrates next section |
| **Standalone** | Plate / full-page illustration | Pure explainer, no text dependency |

---

## Detection Strategy

### Step 1: Spatial Matching

For each detected image (from Topic 3), find candidate grouped blocks (from Topic 1):

```typescript
interface SpatialMatch {
  imageId: string
  candidateBlocks: {
    blockId: string
    relationship: 'above' | 'below' | 'left' | 'right' | 'wraps-around'
    verticalGap: number
  }[]
}
```

- Vertical adjacency: blocks directly above/below the image bounding box
- Same column check: blocks and image share the same x-range
- Gap measurement: px distance between block bbox and image bbox

### Step 2: Vision Context Prompt (Uncertain Cases)

When spatial matching is ambiguous, render page with:
- Image highlighted in **RED**
- Candidate paragraph block(s) highlighted in **BLUE**
- Prompt: *"What is the relationship between the red region and the blue region? Does the text reference the image? Where should the image label be placed?"*

### Step 3: Placement Decision

```
Spatial match clear (one adjacent block) + same column?
  → attach to that block

Spatial ambiguous (multiple candidates, across columns)?
  → vision model resolves: which block does the image belong to?

No clear match?
  → "standalone" — image label as its own block
```

---

## Context Window

- **Immediate block only** — not the full page
- The immediately adjacent paragraph (above or below) is sufficient context for determining relationship
- Vision prompt operates on: full page image (with highlights) + block-level text extracted via pdfjs-dist

---

## Output Contract

```typescript
interface ImageParagraphRelation {
  imageId: string
  imageDescription: string

  // Axis 1: Placement
  placement: 'before-paragraph' | 'after-paragraph' | 'standalone'
  anchorBlockId: string | null
  anchorBlockText: string
  anchorBlockRole: string    // 'body' | 'heading' | 'caption' | etc.

  // Axis 2: Semantic
  relationship: 'referenced' | 'not-referenced' | 'ambiguous'
  relationshipRationale: string

  // Final label position in output HTML
  labelPosition: 'start-of-paragraph' | 'end-of-paragraph'
               | 'standalone-block'

  confidence: number          // 0.0-1.0
  source: 'spatial' | 'vision' | 'merged'
}
```

### All Image Labels Tracked

Every image found through the pipeline maintains a traceable record:

```typescript
interface ImageLabel {
  imageId: string
  pageNumber: number
  bbox: { x1, y1, x2, y2 }
  source: 'triangulate' | 'gridzoom' | 'merged'
  description: string        // from GridZoom Level 3 or vision alt-text
  paragraphRelation: ImageParagraphRelation
}
```

Labels are:
- Stored in MongoDB as part of the extraction run
- Traceable back to detection algorithm and confidence
- Auditable: human can verify any image's label placement in QA UI

---

## Key Decisions

| # | Decision | Detail |
|---|----------|--------|
| D21 | **Case C (inline) → after-paragraph** | Interrupted paragraphs are repaired. Image label goes after the complete paragraph. |
| D22 | **Immediate block only** | Context window = single adjacent paragraph. Not full page. |
| D23 | **All images tracked** | Every detected image carries a label with source, confidence, placement, and relationship metadata. |
| D24 | **Two-axis classification** | Placement (before/after/standalone) × Relationship (referenced/not/ambiguous) |
| D25 | **RED + BLUE vision prompt** | Image in red, candidate paragraph in blue for relationship queries |

---

## Parked Feature (from this topic)

| Feature | Status | Reference |
|---------|--------|-----------|
| **Page Summary Stream** — per-page skim summary, 5-page rolling window, vision model working memory | Parked | `parked-features.md` entry added 2026-05-28 via Mary (Analyst) |
