# Session: Architect Stress-Test & Pipeline Planning

**Date:** 2026-05-28
**Participants:** Prajwal, John (PM), Winston (Architect)
**Status:** Paused

---

## Summary

Reviewed PRD from technical architecture perspective. Winston stress-tested each pipeline stage against feasibility, cost, and accuracy constraints. Seven architecture topics mapped in dependency order. Only Topic 1 was opened before session paused.

---

## Decisions

- **Cost guardrail updated:** If AI spend exceeds $0.30, the additional cost must be justified by measurable accuracy gain — do not fall back to lower-quality paths for cost savings.
- **Conversion time removed** as primary metric. **Structural accuracy** (content order, heading hierarchy, paragraph grouping, list/table structure vs. hand-annotated gold corpus) is the new primary metric.
- **Topic order** agreed: Text Extraction → Artifact Tracker → Image Detection → Image-Paragraph Context → (Tables: parked) → Semantic Structure → Test Harness.
- **Winston + John tag-team** for each topic: architect on feasibility, PM on value/risk.

---

## Topic 1: Text Extraction (Opened)

**Debate:** pdfjs-dist (native text layer extraction) vs. render-to-image + vision model OCR.

**Winston's recommendation:** Primary path = pdfjs-dist (free, native positional data, fast). Fallback path = render-to-PNG + vision model OCR for pages where pdfjs-dist extraction confidence is low.

**Open question:** What % of the target corpus has broken text layers? If >5%, the fallback cost model needs recalculation.

**Action item (deferred):** Run a 20-PDF corpus audit — grab random academic PDFs, run pdfjs-dist, measure extraction confidence and failure patterns.

---

## Files Changed

- `prd.md` — SM-4 redefined, time constraints removed, cost guardrail updated, output quality guard rewritten
- `parked-features.md` — created (11 deferred features)
- `.decision-log.md` — scope narrowing + accuracy refocus entries

**Commit:** `f297ea2` (PRD accuracy refocus)

---

## Queued for Next Session

1. Topic 1 deep dive: pdfjs-dist extraction architecture, fallback pipeline design, corpus audit plan
2. Topic 2: artifact classification + iteration/tracker design
3. Topic 3: image detection via vision model + positioning problem
4. Topic 4: image-paragraph context tracking
5. Topic 6: semantic structure (graph vs. markdown storage)
6. Topic 7: human-in-loop testing harness
7. Epics & Stories creation (CE) once architecture is validated
