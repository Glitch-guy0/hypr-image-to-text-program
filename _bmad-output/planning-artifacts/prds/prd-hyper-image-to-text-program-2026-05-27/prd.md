---
title: Hyper Image-to-Text Program — PDF-to-Semantic-HTML Converter
created: 2026-05-27
updated: 2026-05-27
status: draft
scope_narrowed: 2026-05-27 — Removed EPUB/KFX/MOBI export, image retention, diagram reconstruction (Mermaid/D2), magazine layout path, post-processing module. Output is now purely semantic HTML with alt-text labels only. English-only text-layer PDFs.
---

# PRD: Hyper Image-to-Text Program
*PDF-to-semantic-HTML conversion pipeline — accuracy-first, structured text only, English, text-layer PDFs.*

## 0. Document Purpose

This PRD defines the product vision, target users, feature requirements, and success metrics for a pipeline that converts PDFs into clean, semantic HTML output — readable in any browser, reading-mode compatible, with structured text flow. The focus is narrow: English-language text-layer PDFs only, with images replaced by AI-generated alt-text labels. No EPUB, no image files in output, no diagram reconstruction. It is intended for the product team, engineering, UX, and QA stakeholders. Vocabulary is Glossary-anchored; features are grouped with globally numbered FRs; assumptions are tagged inline and indexed in §9. This PRD builds on the existing `product.md` specification document.

## 1. Vision

PDFs are designed as fixed-size rendering instructions — placing glyphs and images at absolute coordinates for visual perfection on one specific page size. Browsers and reading-mode views are a fundamentally different medium: text-first, variable-viewport, reflow-driven. Existing converters mechanically carry over the broken positioning, inject artifacts, and produce output that is barely readable.

This product solves that mismatch. It parses a PDF not as a set of rendering instructions but as a semantic document — extracting text cleanly, classifying images for alt-text labeling, restructuring tables, detecting chapters, and producing clean HTML that genuinely reflows in any browser reading mode. Images are not retained; instead, AI generates a concise text summary of each image placed inline as a label where the image would appear. The output is clean, navigable, structured text — no fixed layout, no broken positioning, no artifact noise.

Where confidence is low, the system flags sections for human review rather than silently producing bad output.

## 2. Target User

### 2.1 Jobs To Be Done

- **Academic/independent researchers** need to convert PDF papers and books into clean, readable HTML for study, annotation, and screen-reader use
- **Content archivists** need to preserve print documents in a reflowable, accessible, text-first format with semantic structure
- **Accessibility teams** need HTML output with semantic structure, alt-text labels, and navigable TOC for screen reader users

### 2.2 Non-Users (v1)

- Users with scanned/image-only PDFs requiring OCR — not in scope
- Users needing batch/folder processing — not in scope
- Users needing API/programmatic access — not in scope
- Users needing EPUB or Kindle output — not in scope (HTML only)
- Users needing images retained in output — not in scope (alt-text labels only)
- Users needing diagram reconstruction — not in scope

### 2.3 Key User Journeys

- **UJ-1. A researcher converts a PDF textbook chapter for screen-reader study.**
  - **Persona + context:** Dr. Wei, postdoc, has a 30-page PDF chapter from an academic textbook with diagrams, tables, and two-column layout.
  - **Entry state:** Web UI open. Has the PDF on their local machine.
  - **Path:** Uploads the PDF → pipeline detects text layer → extracts text with positional metadata → artifact removal strips headers, footers, page numbers → text reflowed in correct reading order → 4 tables detected: two 3-column (HTML tables), one 7-column (structured prose), one >8-column (summary prose) → 6 images detected: AI generates inline alt-text labels for each → chapter boundaries detected via heading analysis → output preview ready → reviews 3 flagged sections (1 low-confidence chapter boundary, 2 alt-text labels below 0.85).
  - **Climax:** Reviews flagged items — approves the chapter boundary, edits one alt-text label for accuracy. Downloads clean semantic HTML.
  - **Resolution:** Opens in browser reading mode — TOC navigable, text reflows cleanly, tables legible, image labels provide context. **Edge case:** A sidebar note was initially extracted mid-paragraph — the artifact removal pass correctly moved it to a marginal note block.

## 3. Glossary

- **PDF** — Portable Document Format. A fixed-layout rendering instruction set with no semantic structure.
- **IDM (Intermediate Document Model)** — A clean, queryable JSON representation produced after parsing and before export. Contains metadata, chapters with sections and blocks, and flagged items.
- **Artifact** — Any element in the PDF text layer that is not part of the intended reading content: page numbers, running headers, watermarks, anti-copy garbage characters, duplicated pull-quotes.
- **Decorative Image** — An image with no semantic content (divider, background texture, logo, watermark). No label generated; silently skipped.
- **Content Image** — An image that carries meaning referenced by the text (photograph, diagram, illustration). Not retained in output; replaced with an AI-generated inline alt-text label.
- **Alt-Text Label** — A concise AI-generated text summary of a content image, placed inline in the HTML where the image would appear. No image file is embedded.
- **Confidence Score** — A 0.0–1.0 value assigned to each AI-derived decision (chapter boundary, alt-text quality, artifact classification). Below-threshold scores trigger human review flags.
- **Flag** — A section marked for mandatory human review during QA, based on confidence thresholds.
- **QA Review UI** — In-browser HTML preview of the converted output with flagged section highlighting, side-by-side PDF vs. converted comparison, and approve/edit/revert/skip actions.

## 4. Features

### 4.1 PDF Upload and Parsing

**Description:** Single-PDF upload via web UI. The system reads the PDF, validates it has a text layer, extracts text with positional metadata, inventories images, and scores parsing confidence. Realizes UJ-1.

**Functional Requirements:**

#### FR-1: Single PDF Upload

User can upload a single PDF file via the web UI. System validates that the file is a valid PDF with a text layer. Realizes UJ-1.

**Consequences (testable):**
- System returns a clear error message for non-PDF files
- System rejects files over a configurable size limit with messaging
- System detects absence of text layer and returns message (scanned PDF detection)
- Upload progress is visible to the user

**Out of Scope:**
- Batch/folder upload

#### FR-2: Text Layer Extraction

System extracts all text from the PDF text layer with positional metadata (coordinates, font size, font weight, page number). Realizes UJ-1.

**Consequences (testable):**
- Extraction preserves reading order for single-column layouts
- Multi-column text is extracted in correct column-first reading order
- Positional data (x, y, page) preserved for artifact classification
- Extraction preserves correct reading order: single-column → top-to-bottom; multi-column → column-first, left-to-right
- Extraction completions within 60 seconds for a 300-page equivalent document; speed is a secondary concern

**Out of Scope:**
- OCR for scanned/image-only PDFs
- Magazine/overlapping layout detection

### 4.2 Artifact Removal and Text Cleaning

**Description:** A pre-processing pass that classifies extracted text elements into content vs. artifact, then applies targeted cleaning. Artifacts include headers, footers, page numbers, watermarks, anti-copy garbage, duplicated pull-quotes, and ligature/encoding issues. Realizes UJ-1.

**Functional Requirements:**

#### FR-3: Artifact Classification

System classifies each extracted text element as content or artifact based on position, repetition, font properties, and content heuristics. Realizes UJ-1.

**Consequences (testable):**
- Running headers detected by position + repetition pattern and classified as artifact
- Page numbers detected by position + numeric pattern and classified as artifact
- Anti-copy garbage (invisible characters, reordered Unicode) detected and classified as artifact
- Duplicated elements (pull-quotes appearing both in-flow and as overlay) detected and merged
- Classification confidence score assigned per element
- Elements with confidence < 0.85 flagged for review

**Out of Scope:**
- Language-specific artifact patterns beyond English

#### FR-4: Artifact Removal and Flow Correction

System removes classified artifacts and reconstructs clean paragraph flow. When removal exceeds 15% of a paragraph, the section is always flagged. Realizes UJ-1.

**Consequences (testable):**
- Page numbers and running headers removed from body text
- Anti-copy garbage characters stripped
- Duplicated pull-quotes resolved (one instance retained)
- Ligature normalization applied (fi, fl, ffi → Unicode equivalents)
- Mid-word hyphenation from line breaks resolved
- Flagged when >15% of paragraph content is removed

#### FR-5: Low-Confidence Handling

System does not silently drop or mangle content below confidence thresholds — it flags for human review. Realizes UJ-1.

**Consequences (testable):**
- Flagged items presented in QA Review UI with original and cleaned versions
- User can approve, edit, revert to original, or skip each flag
- Non-flagged content is not surfaced in review

### 4.3 Image Processing — Alt-Text Label Generation

**Description:** Image inventory and classification. Two categories: decorative (skipped entirely) and content (AI-generated alt-text label placed inline; no image file in output). Diagram reconstruction is not in scope. Realizes UJ-1.

**Functional Requirements:**

#### FR-6: Image Inventory and Classification

System inventories all images in the PDF and classifies each as decorative or content. Realizes UJ-1.

**Consequences (testable):**
- Images below a configurable size threshold classified as decorative
- Images outside main text flow (watermarks, logos) classified as decorative
- Images repeated across pages classified as decorative
- Photographs, illustrations, and all semantically meaningful images classified as content
- Low-information-entropy images classified as decorative
- Classification logged for audit

#### FR-7: Decorative Image Skipping

Decorative images are silently skipped — no label, no output entry. Realizes UJ-1.

**Consequences (testable):**
- No decorative images referenced in HTML output
- Skipping is logged for audit but not flagged for review

#### FR-8: Content Image Alt-Text Label Generation

For each content image, the system generates a concise AI-produced alt-text label describing what the image depicts. The label is placed inline in the HTML where the image would appear. No image file is embedded in output. Realizes UJ-1.

**Consequences (testable):**
- AI generates a text summary (1-3 sentences) for each content image
- Alt-text label confidence scored; flagged if below 0.85
- Label placed inline as a `<span>` or `<aside>` with `aria-label` semantics
- Surrounding text context is used to improve label relevance
- User can view/edit the alt-text label in the QA review

**Out of Scope:**
- Image file retention in output
- Diagram reconstruction (Mermaid/D2)
- Image caption formatting or styling
- Multi-image layout preservation

### 4.4 Table Processing

**Description:** Table detection and transformation for HTML output. Narrow tables retained as semantic HTML tables, wide tables rewritten as structured prose. Realizes UJ-1.

**Functional Requirements:**

#### FR-9: Table Detection and Classification

System detects tables in the PDF and classifies them by column count, content type, and multi-page status. Realizes UJ-1.

**Consequences (testable):**
- Tables detected and extracted with cell content and structure
- Column count determined and recorded
- Content type classified as data (numbers, stats) or layout
- Multi-page tables detected and tracked

#### FR-10: Table Transformation

System applies the appropriate strategy per table: ≤4 columns → semantic HTML `<table>` with responsive styling; 5-8 columns → broken into logical sub-tables; >8 columns or dense data → rewritten as structured prose with key data points. Realizes UJ-1.

**Consequences (testable):**
- ≤4 column tables render as HTML tables legible on 600px viewport
- 5-8 column tables split into sub-tables with clear grouping rationale
- >8 column tables converted to structured prose with data summary
- Multi-page tables have column headers re-injected at each continuation
- Table-to-prose transformation always flagged for human review

**Out of Scope:**
- Editable table reconstruction in review UI

### 4.5 Chapter and Structure Detection

**Description:** Semantic structure detection using PDF bookmarks (priority 1), embedded TOC page (priority 2), AI analysis of heading patterns (priority 3), and human review fallback (priority 4). Produces a navigable in-HTML TOC. Realizes UJ-1.

**Functional Requirements:**

#### FR-11: PDF Bookmark Extraction

System reads the PDF bookmark/named destination tree and uses it as the primary chapter structure. Realizes UJ-1.

**Consequences (testable):**
- Bookmark tree extracted and mapped to page ranges
- Bookmark tree used directly when confidence > 0.95
- Missing or incomplete bookmarks trigger fallback to TOC page detection

**Out of Scope:**
- Editing bookmark data in the review UI

#### FR-12: TOC Page Parsing

System detects and parses the table of contents page(s) when PDF bookmarks are absent or incomplete. Validates or supplements bookmark data. Realizes UJ-1.

**Consequences (testable):**
- TOC page detected by positional and typographic patterns
- TOC entries parsed with page number references
- Confidence scored on TOC-to-content alignment

#### FR-13: AI Structural Analysis

For PDFs without bookmarks or a clear TOC page, system uses AI analysis of heading size, font weight, spacing, and positional patterns to infer chapter/section boundaries. Realizes UJ-1.

**Consequences (testable):**
- Chapter boundaries detected with confidence scores
- Confidence < 0.90 flagged for human review
- Heading hierarchy (chapter → section → subsection) inferred and recorded
- Section blocks mapped to parent chapters

#### FR-14: Navigable In-HTML TOC Generation

System produces a navigable table of contents in the HTML output using anchor links. Realizes UJ-1.

**Consequences (testable):**
- HTML TOC is navigable via anchor link jumping
- Section-level navigation present
- Heading hierarchy reflected in TOC nesting
- TOC renders correctly in browser reading mode

### 4.6 QA Review UI

**Description:** In-browser HTML preview of the converted output with flagged section highlighting. Side-by-side comparison of original PDF excerpt vs. converted HTML. Per-flag actions: approve, edit, revert to original, skip. Realizes UJ-1.

**Functional Requirements:**

#### FR-15: Flagged Section Review

System presents all flagged sections in the QA Review UI with highlighting. Realizes UJ-1.

**Consequences (testable):**
- Flagged sections visually highlighted in HTML preview
- Flag reason and confidence score displayed per flag
- Non-flagged sections not shown in review (but accessible for context)
- User can navigate between flags sequentially

#### FR-16: Side-by-Side Comparison

System renders original PDF excerpt alongside converted HTML output for each flagged section. Realizes UJ-1.

**Consequences (testable):**
- PDF excerpt rendered from original source
- Converted HTML rendered as it appears in final output
- Side-by-side view responsive on browser

#### FR-17: Per-Flag Actions

User can approve, edit, revert to original, or skip each flag. Realizes UJ-1.

**Consequences (testable):**
- Approve: flag dismissed, current output retained
- Edit: inline text editor opens, user modifies content, change recorded
- Revert to original: restores the un-modified extracted content for this section
- Skip: flag deferred, output unchanged
- All actions logged for audit

### 4.7 Export Pipeline

**Description:** Single-format export: standalone semantic HTML document. Reads from the IDM. Realizes UJ-1.

**Functional Requirements:**

#### FR-18: HTML Export

System generates a self-contained, standalone HTML document with semantic HTML5 structure, reading-mode-optimized CSS, and navigable in-document TOC. Realizes UJ-1.

**Consequences (testable):**
- HTML renders correctly in Chrome, Firefox, Safari reading modes
- Responsive and reflows at all viewport widths
- No external CSS or JS dependencies (self-contained)
- Semantic HTML5 elements used (`<nav>`, `<article>`, `<section>`, `<h1>`-`<h6>`, `<table>`, `<figure>`)
- Alt-text labels rendered as accessible inline text
- TOC linked via anchor navigation
- No absolute positioning in output CSS

**Out of Scope:**
- EPUB3 export
- KFX/MOBI export
- Multi-format output


## 5. Non-Goals (Explicit)

- **Not an OCR engine** — scanned/image-only PDFs are not in scope
- **Not a batch processor** — single-PDF upload only
- **Not an API** — web UI only; no programmatic access
- **Not a PDF editor** — the product converts PDFs to HTML; it does not modify them
- **Not an EPUB/ebook converter** — output is HTML only; no EPUB, KFX, or MOBI
- **Not an image renderer** — images are not retained in output; only alt-text labels
- **Not a diagram reconstructor** — no Mermaid/D2; diagrams are labeled only
- **Not an HTML editor** — the QA Review UI handles flagged sections only; full HTML editing is out of scope
- **Not multi-language** — English-language PDFs only; RTL and other languages not supported

## 6. MVP Scope

### 6.1 In Scope

- Single PDF upload via web UI (text-layer PDFs only)
- Text extraction with positional metadata
- Artifact removal (headers, footers, page numbers, anti-copy garbage, duplicated pull-quotes, ligature normalization)
- Reading order recovery (single-column and basic multi-column)
- Image inventory and classification (decorative skip, content → alt-text label)
- AI-generated alt-text labels for content images (confidence-scored, flagged if <0.85)
- Table detection and transformation (≤4 col → HTML table, 5-8 → sub-tables, >8 → structured prose)
- Chapter detection via bookmarks, TOC page, or AI analysis
- Navigable in-document TOC with anchor links
- Standalone semantic HTML export (self-contained, reading-mode compatible)
- QA Review UI with flagged section highlighting, side-by-side comparison, and per-flag actions
- Confidence scoring and flag system for all AI-derived decisions

### 6.2 Out of Scope

> Full rationale for each feature below is documented in [`parked-features.md`](parked-features.md).

- Scanned PDF / OCR support
- Batch/folder processing
- API endpoint
- EPUB3, KFX, or MOBI export
- Image file retention in output
- Diagram reconstruction (Mermaid/D2)
- Magazine/overlapping layout detection
- Post-processing features (summaries, newsletters, reading plans)
- Right-to-left or non-English language support
- Mathematical equation rendering (LaTeX → MathML)

## 7. Success Metrics

**Primary**
- **SM-1**: Clean output — no garbled text, injected artifacts, or broken encoding. Validates FR-3, FR-4, FR-5 (artifact removal).
- **SM-2**: Navigability — in-document TOC with working anchor links, heading hierarchy correct. Validates FR-14.
- **SM-4**: Structural accuracy — HTML output preserves the original PDF's content order, heading hierarchy, paragraph grouping, and list/table structure. Measured against a hand-annotated gold corpus. Validates FR-2, FR-3, FR-6, FR-13.
- **SM-5**: Flag accuracy — no more than 10% false positives in flagged items (items flagged unnecessarily). Validates confidence scoring system (cross-cutting).

**Counter-metrics (do not optimize)**
- **SM-C1**: Flag rate — do not optimize for zero flags. Higher flag rates are acceptable if they indicate proper caution. Counterbalances SM-5.
- **SM-C2**: Processing speed — never sacrifice structural accuracy for faster conversion.

## 8. Open Questions

1. **Alt-text label format** — inline `<span>` with `aria-label`, `<aside>` with descriptive text, or a bracketed notation like `[Image: description]`? Needs UX input on how labels read in reading mode.
2. **Multi-column reading order** — basic left-to-right, top-to-bottom column flow is in scope. What's the fallback for complex layouts where columns interleave? Flag for human review?
3. **Confidence threshold calibration** — 0.85 for alt-text, 0.90 for chapter boundaries, 0.85 for artifact classification. Do these need early calibration against a test corpus before release?

## 9. Assumptions Index

- Assumption from §4.1 (FR-1): PDF size limit is configurable and defaults to a reasonable threshold. Adjust based on user feedback.
- Assumption from §4.2 (FR-4): Artifact classification heuristics (position, repetition, font properties) are sufficient for >90% of cases.
- Assumption from §4.2 (FR-5): >15% paragraph removal threshold is a correct balance of sensitivity vs. false positives. Calibrate during beta.
- Assumption from §4.3 (FR-8): AI-generated alt-text from a flat image is sufficiently accurate and relevant for >80% of content images.
- Assumption from §4.4 (FR-10): 4-column and 8-column thresholds for table strategies are correct. Calibrate during beta.
- Assumption from §4.5 (FR-13): AI structural analysis can reliably detect chapter boundaries from typographic patterns in the absence of bookmarks.
- Assumption from §4.6 (FR-15): Side-by-side PDF vs. converted comparison is sufficient for QA review. No interactive overlay or diff view required.
- Assumption from §7 (SM-5): 10% false-positive flag rate is an acceptable target for confidence scoring calibration.

## 10. Cross-Cutting NFRs

- **Performance:** QA Review UI loads in under 2 seconds. Pipeline throughput is a secondary concern — accuracy is the primary objective.
- **Reliability:** Pipeline does not silently produce bad output — any low-confidence decision is flagged. System handles malformed PDFs gracefully with clear error messages.
- **Security:** Uploaded PDFs are isolated per session. Files are not accessible between user sessions. Processed output is available for download for 24 hours then deleted.
- **Observability:** Pipeline stages log duration, confidence scores, and flag counts. Failed conversions surface error details to user and system logs.

## 11. Constraints and Guardrails

- **Cost:** AI enrichment (alt-text generation, structural analysis) uses an LLM API. Per-document cost should not exceed $0.30 for a 300-page equivalent PDF under normal operation. If cost exceeds threshold, the additional spend must be justified by measurable accuracy gain — do not fall back to lower-quality paths purely for cost savings.
- **Privacy:** Uploaded PDFs are not stored beyond the 24-hour download window. PDF content is not used for model training. AI API calls do not include PII in prompts.
- **Output quality:** The pipeline must never produce output worse than the original PDF's semantic structure. When confidence is low, flag rather than guess. Accuracy is the primary objective; speed is a secondary concern.

## 12. Why Now

The existing PDF-to-HTML converter ecosystem is stagnant — tools produce broken, artifact-laden output that fails in reading mode. Meanwhile, LLM capabilities for image understanding and structural analysis have crossed a quality threshold that makes intelligent conversion feasible for the first time. The combination of accessible AI APIs + universal browser reading-mode support creates a clear window for a clean, semantic HTML converter.

## 13. Platform

Web application (responsive browser UI). No mobile app or native desktop client. Deployment as a containerized web service (Docker). Single-page application frontend with API backend.
