# Parked Features Register

*Features deferred from v1 scope. See PRD §6.2 (Out of Scope) and `.decision-log.md` for context.*

---

## EPUB3 Export (was FR-20)

**Why Parked:** HTML-first focus. EPUB adds epubcheck validation loop, device-specific testing, and CSS3 media query complexity that doesn't serve the core goal of structured text output.

**Trigger:** Revisit when ≥3 users explicitly request ebook format, or when HTML pipeline is stable with 10+ successful conversions.

**Notes:** Original PRD had epubcheck zero-error as SM-1. No dependency on other parked features — can be added as a post-processing export filter from the IDM.

---

## KFX/MOBI Export (was FR-22)

**Why Parked:** Dependent on EPUB export (must have EPUB before converting to Kindle formats). Adds Calibre CLI deployment dependency.

**Trigger:** Revisit after EPUB export is shipping and stable.

**Notes:** Calibre is the practical choice (KindleGen deprecated). Containerization strategy needed to manage the Calibre dependency.

---

## Image File Retention in Output (was part of FR-9)

**Why Parked:** Keeping actual image files in output adds storage, sizing logic, viewport-responsive CSS, and multi-image layout complexity. Alt-text labels alone serve the text-first use case.

**Trigger:** Revisit when users report that alt-text labels are insufficient for understanding document content.

**Notes:** Image classification pipeline (decorative vs. content) is still needed for alt-text label generation. Only the embedding step is parked.

---

## Diagram Reconstruction (Mermaid/D2) (was FR-10)

**Why Parked:** Technical risk — AI reliably interpreting flat PNG to produce semantically correct Mermaid/D2 is unproven at scale. High AI API cost per diagram. Unclear value for text-first use case where alt-text label may suffice.

**Trigger:** Revisit when a specific use case demonstrates that alt-text labels are insufficient for diagram-dependent content (e.g., software architecture docs, engineering manuals).

**Notes:** D2 vs. Mermaid decision boundary was never resolved. Original PRD threshold was <0.88 confidence for flagging.

---

## Magazine / Overlapping Layout Detection (was part of FR-2)

**Why Parked:** Requires a dedicated parsing spike to evaluate library options. Magazine layouts (overlapping elements, text-wrapped images, complex floats) are a fundamentally harder problem than single/basic-multi-column.

**Trigger:** Revisit when single-column and basic multi-column parsing reliably handles 90%+ of the target corpus, and magazine PDF demand is confirmed.

**Notes:** No decision was made on which parser library to use for this path.

---

## Post-Processing Module (was §4.8 / FR-23)

**Why Parked:** Scope was never finalized (chapter summaries, newsletters, reading plans, style rewrites). Architecture TBD — same pipeline or separate service.

**Trigger:** Revisit when core pipeline is stable with 3+ real users and a clear use case emerges for post-processing features.

**Notes:** Original PRD required post-processing to be read-only on the IDM. IDM storage and versioning decisions will affect how this is added later.

---

## OCR / Scanned PDF Support

**Why Parked:** Entirely different problem domain — requires OCR engine integration (Tesseract, OCRmyPDF, or cloud API), language model training data, and significantly more compute per document. Not a scope extension; it's a parallel product.

**Trigger:** Revisit if user research shows scanned PDFs as the primary format users need to convert, or if text-layer PDF demand plateaus.

**Notes:** No OCR capability exists in v1. Adding it would change the cost model ($0.30/document threshold cannot hold with OCR).

---

## Batch / Folder Processing

**Why Parked:** Adds UI complexity (drag-and-drop reordering, queue management, parallel processing), backend job scheduling, and progress tracking across multiple documents.

**Trigger:** Revisit when single-PDF workflow is validated with 20+ conversions and users consistently ask for batch upload.

**Notes:** Single-PDF upload only in v1. No file reordering or queue infrastructure planned.

---

## API / Programmatic Access

**Why Parked:** Needs authentication, rate limiting, API key management, request/response documentation, and SDK maintenance. Web UI only in v1.

**Trigger:** Revisit when web UI usage saturates and integration partners request programmatic access.

**Notes:** The underlying pipeline could be exposed as an API with moderate refactoring, but no endpoint contracts exist yet.

---

## RTL / Multi-Language Support

**Why Parked:** Cascading impact — text direction (CSS `direction: rtl`), font selection, hyphenation rules, encoding edge cases, artifact pattern detection for non-English languages, and testing corpus expansion.

**Trigger:** Revisit when English pipeline is solid and user research confirms demand for Arabic, Hebrew, or other RTL scripts.

**Notes:** Artifact classification heuristics (FR-3) explicitly exclude language-specific patterns beyond English. Adding RTL means reworking this system.

---

## Mathematical Equation Rendering (LaTeX → MathML)

**Why Parked:** Vertical feature serving a niche (academic STEM papers). Detection of math blocks in PDF text layer is unreliable. MathML rendering adds CSS/JS dependency.

**Trigger:** Revisit when the target corpus analysis shows >20% of uploaded PDFs contain mathematical equations, or a key academic user requests it.

**Notes:** No relationship to other parked features. Could be added independently when needed.

---

## Page Summary Stream (Post-Topic 4 feature idea)

**Why Parked:** Speculative enhancement — core pipeline accuracy must be established first. Adds a per-page LLM call whose marginal benefit to vision model coherence is unproven.

**Trigger:** Revisit after Topics 5-7 are designed and baseline accuracy (without summaries) is measured. Then run A/B test: with-summaries vs. without, measure downstream accuracy delta.

**Notes:**
- Concept: generate 1-2 sentence per-page skim summary → stack into rolling 5-page window → pass as vision model working memory
- Origin: Topic 4 (image-paragraph context) discussion. Hypothesis that cross-page narrative improves vision classification
- Concern: extra LLM call per page = cost additive; summary quality threshold unknown; storage and memory management TBD
- Also: could serve as end-user "skim mode" feature (future UI feature, not part of this pipeline)
- Dependencies: Topics 1-4 must be stable first
