# Topic 1: Text Extraction — Deep Dive

**Date:** 2026-05-28
**Participants:** Prajwal, John (PM), Winston (Architect)
**Status:** Resolved
**Session ref:** `_bmad-output/planning-artifacts/sessions/2026-05-28-architect-stress-test.md`

---

## Decisions

### Accepted

| # | Decision | Rationale |
|---|----------|-----------|
| D1 | **Primary extraction: pdfjs-dist** | Confirmed L1 (text) + L2 (x/y/width/height per item) via Mozilla's `page.getTextContent()` API. Proven on real PDFs — see GitHub issue #18201 for raw JSON output proof. |
| D2 | **Red-box highlight → vision model for L3 (semantic role)** | Multiple peer-reviewed papers validate this (SoM, VP-Bench, BBVPE). Overlay red semi-transparent fill (α=0.15-0.3) + 3px outline on extracted blocks, pass annotated page to vision model for classification. |
| D3 | **Vendor-agnostic vision layer (LangChain)** | No hard SDK binding. Vision slot must support local models (Ollama) and cloud models interchangeably. Target: GLM 4.6V Flash or Qwen3 VL 8B local-first. |
| D4 | **Two-tier fallback: pdfjs-dist → vision OCR** | If pdfjs-dist per-page confidence < threshold (broken encoding, empty output, anti-copy chars), render page via Sharp → send to vision OCR as fallback. Keeps costs low (fallback only) while handling damaged PDFs. |
| D5 | **Group pdfjs-dist items into blocks before vision pass** | Raw items are per-character/per-word. Group by proximity + font + line height into paragraphs, headings, captions before red-box overlay. Single vision call per block (or per page with multiple highlighted regions). |
| D6 | **Confidence scoring per extraction page** | Heuristic: <10 chars extracted, >50% invisible chars, high encoding error rate → low confidence → route to vision fallback. Thresholds calibrated during iteration. |

### Deferred

| # | Idea | Why Deferred | Trigger |
|---|------|--------------|---------|
| X1 | **Vision-first (full page OCR via vision model)** | High latency (2-5s/page × 300 = 10-25 min). No native positional data. Cost adds up. But may work if model quality improves — test later as alternative path. | Run benchmark: if vision-first can match pdfjs-dist positional accuracy within acceptable speed, reconsider. |
| X2 | **Corpus audit (20 PDFs)** | Action item from last session — not yet executed. Needed to determine what % of real PDFs trigger the vision fallback. | Before we tune fallback thresholds, run a 20-PDF corpus audit against pdfjs-dist. |

---

## Research Evidence

### pdfjs-dist L1 + L2 — Proven

| Property | Source | Evidence |
|----------|--------|----------|
| `item.str` (L1 — text) | Mozilla PDF.js `page.getTextContent()` | Official type defs: `types/src/display/api.d.ts` |
| `item.transform[4]` (L2 — x) | GitHub issue #12031 | Confirmed by PDF.js contributors |
| `item.transform[5]` (L2 — y) | GitHub discussion #20234 | Real console output with viewport transform |
| `item.width`, `item.height` (L2) | Official type defs | Part of `TextItem` interface |
| Real PDF output | GitHub issue #18201 | "Johns Email" raw JSON with per-character coords |

### Red-Box Overlay — Validated

| Paper | Year | Finding |
|-------|------|---------|
| **Set-of-Mark (SoM)** — Microsoft | 2023 | Zero-shot labeled boxes beat fine-tuned models on RefCOCOg |
| **VP-Bench** — Xu et al. | 2025 | Red bounding boxes are most reliably understood visual prompt across 28 MLLMs |
| **BBVPE** — Woo et al., NAACL | 2025 | Box overlays significantly reduce object hallucination |
| **Visual In-Context Prompting** — CVPR | 2024 | Strokes, boxes, points as valid prompts for referring segmentation |

### Best Practice (from research)

1. Semi-transparent red fill (α=0.2) + 3px outline — better than outline-only
2. Alphanumeric labels ("A", "B") inside boxes for multi-region queries
3. Crop + full-image combined for small regions (<20px)
4. GPT-4o slightly better than Claude for bounding-box grounding; our stack uses LangChain to remain model-agnostic
5. 2-5 labeled regions per image is the reliable range

---

## Architecture Sketch

```
                          ┌──────────────┐
         PDF buffer ────► │  pdfjs-dist  │
                          └──────┬───────┘
                                 │
                                 ▼
                     ┌─────────────────────┐
                     │  Group items into    │
                     │  blocks by proximity │
                     │  + font + line height│
                     └────────┬────────────┘
                              │
                              ▼
                    ┌──────────────────┐
                    │  Confidence check │
                    │  per extraction   │
                    └──┬───────────────┘
                       │          │
                High conf.   Low conf.
                       │          │
                       ▼          ▼
                ┌─────────┐ ┌──────────────┐
                │ Use as  │ │ Sharp render  │
                │ is      │ │ → Vision OCR  │
                └────┬────┘ └──────┬───────┘
                     │             │
                     ▼             ▼
              ┌────────────────────────────┐
              │  Render page to PNG        │
              │  Overlay red box(es) on    │
              │  each block                │
              └────────┬───────────────────┘
                       │
                       ▼
              ┌────────────────────────────┐
              │  Vision model (LangChain)  │
              │  → semantic classification │
              │  GLM / Qwen (local)        │
              └────────┬───────────────────┘
                       │
                       ▼
              ┌────────────────────────────┐
              │  IDM Seed:                 │
              │  { blocks, roles,          │
              │    layoutRegions,          │
              │    intentRationale }       │
              └────────────────────────────┘
```

---

## Output Contract

```typescript
interface ExtractedBlock {
  text: string
  bbox: { x1: number; y1: number; x2: number; y2: number }
  items: RawPdfjsItem[]        // original pdfjs-dist items for this block
  source: 'pdfjs' | 'vision'   // which extraction path
  confidence: number           // 0.0-1.0
  role?: string                // 'body' | 'heading' | 'caption' | 'sidebar' | 'footer' | 'header'
  intentRationale?: string     // vision model's reasoning for the role
}

interface PageLayout {
  pageNumber: number
  blocks: ExtractedBlock[]
  regions: {
    type: 'body' | 'sidebar' | 'header' | 'footer' | 'figure-area' | 'toc'
    bbox: { x1: number; y1: number; x2: number; y2: number }
    rationale: string
  }[]
}
```

---

## Open Items (for Topic 1 calibration)

1. **Block grouping algorithm** — distance threshold? Same font+size heuristic? Need prototype to validate.
2. **Confidence threshold for fallback** — 10 chars? 50% invisible? Needs corpus audit (X2).
3. **Vision model prompt template** — need to craft and test the "examine the red-highlighted region" prompt for both GLM and Qwen.
