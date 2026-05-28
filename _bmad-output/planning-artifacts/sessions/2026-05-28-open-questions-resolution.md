# Open Questions Resolution

**Date:** 2026-05-28
**Status:** All resolved
**Session refs:**
- `_bmad-output/planning-artifacts/sessions/2026-05-28-topic-1-text-extraction.md`
- `_bmad-output/planning-artifacts/sessions/2026-05-28-topic-2-artifact-classification.md`

---

## OQ#1: Block Grouping Algorithm

**Decision:** Hybrid C — spatial clustering (A) primary, vision clustering (B) on low confidence.

- A groups by font + distance heuristics (fast, free)
- B renders page → vision model outlines blocks → maps back to pdfjs-dist items (only when A is uncertain)
- **Error context:** When A fails, store in Elasticsearch: page image, raw items, A's output, B's corrected output with rationale

---

## OQ#2: Vision Prompt Template

**Decision:** Single JSON-output prompt for GLM 4.6V Flash and Qwen3 VL 8B.

- Role taxonomy aligned with artifact labels (body, heading, caption, sidebar, etc.)
- Visual cues list guides reasoning (position, font size, whitespace, alignment, repetition)
- "Uncertain" escape hatch for low-confidence cases
- JSON output constraint for parseable results
- **Pending:** Test 5 sample pages across both models for output consistency

---

## OQ#3: Confidence Threshold for Vision Fallback

**Decision:** Per-page boolean triggers — any one fires → route to vision OCR.

| Signal | Threshold |
|--------|-----------|
| Item count | < 5 items on non-empty page |
| Replacement char ratio | > 5% (□, �, invalid Unicode). Single occurrence always logged to ES |
| Mean char width | < 0.5 units |
| Empty after artifact removal | > 90% classified as artifact |

**Removed:** Invisible char ratio (invisible things don't count).
**Added:** Single replacement char → always document to Elasticsearch.
