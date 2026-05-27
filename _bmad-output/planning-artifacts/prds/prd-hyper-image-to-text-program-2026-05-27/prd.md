---
title: Hyper Image-to-Text Program — Publisher-Quality PDF-to-EPUB Converter
created: 2026-05-27
updated: 2026-05-27
status: draft
---

# PRD: Hyper Image-to-Text Program
*Publisher-quality PDF-to-EPUB3 conversion pipeline.*

## 0. Document Purpose

This PRD defines the product vision, target users, feature requirements, and success metrics for a pipeline that converts PDFs into publisher-quality EPUB3 and HTML output — genuinely readable on Kobo, Kindle, PocketBook, and browser-based readers. It is intended for the product team, engineering, UX, and QA stakeholders. Vocabulary is Glossary-anchored; features are grouped with globally numbered FRs; assumptions are tagged inline and indexed in §9. This PRD builds on the existing `product.md` specification document.

## 1. Vision

PDFs are designed as fixed-size rendering instructions — placing glyphs and images at absolute coordinates for visual perfection on one specific page size. E-readers are a fundamentally different medium: text-first, variable-viewport, reflow-driven. Existing converters mechanically carry over the broken positioning, inject artifacts, and produce output no publisher would ship.

This product solves that mismatch. It parses a PDF not as a set of rendering instructions but as a semantic document — extracting text cleanly, classifying images, restructuring tables, detecting chapters, and producing EPUB3 that genuinely reflows on any device. The output is indistinguishable from a commercial publisher EPUB of the same title.

The pipeline also handles the hardest cases: multi-column academic layouts, magazine-style designs with heavy image use, flowcharts that need native diagram reconstruction, and tables that must be broken into readable prose. Where confidence is low, the system flags sections for human review rather than silently producing bad output.

## 2. Target User

### 2.1 Jobs To Be Done

- **Publisher production teams** need to take a designer's print PDF and ship an EPUB without manual rewrite or expensive toolchains
- **Academic institutions** need to convert research papers and multi-column journals into readable formats for e-reader study
- **Self-publishers and authors** need a single PDF upload that produces store-ready EPUB without technical expertise
- **Content archivists** need to preserve print documents in a reflowable, accessible, standards-compliant format
- **Accessibility teams** need EPUB output with semantic structure, alt text, and navigable TOC for screen reader users

### 2.2 Non-Users (v1)

- Users with scanned/image-only PDFs requiring OCR — out of scope for v1
- Users needing batch/folder processing — single-PDF upload only in v1
- Users needing API/programmatic access — web UI only in v1

### 2.3 Key User Journeys

- **UJ-1. A publisher production editor converts a print-ready novel to EPUB.**
  - **Persona + context:** Jamie, production editor at a mid-size publisher, receives the designer's print PDF of a 300-page novel.
  - **Entry state:** Logged into the web UI. Has the PDF on their local machine.
  - **Path:** Uploads the PDF → pipeline runs (text extraction, chapter detection, image classification) → preview is ready in under 2 minutes → reviews flagged sections (2 low-confidence chapter boundaries only) → approves all → downloads EPUB3.
  - **Climax:** Opens the EPUB on a Kobo — full TOC, chapter skip works, text reflows cleanly at any font size.
  - **Resolution:** Ships to the Kindle store after Calibre KFX conversion. **Edge case:** A pull-quote in chapter 3 was duplicated (once in-flow, once as overlay) — the system flagged it, Jamie approved the removal on review.

- **UJ-2. An academic researcher converts a multi-column journal PDF for e-reader study.**
  - **Persona + context:** Dr. Wei, postdoc, downloads a two-column IEEE paper PDF.
  - **Entry state:** Logged into the web UI, has the PDF.
  - **Path:** Uploads → pipeline detects multi-column layout → extracts in correct reading order (left column, top to bottom, then right column) → 4 tables detected: two are 3-column (retained as HTML), one is 7-column (broken into sub-tables), one is 10-column (rewritten as structured prose) → 3 diagrams detected: one flowchart reconstructed as Mermaid, two complex diagrams retained as images with AI-generated alt text → flagged sections presented for review.
  - **Climax:** Reviews and approves the diagram conversion; edits the alt text on one retained image.
  - **Resolution:** Downloads EPUB3, opens on PocketBook — reads cleanly, tables are legible, no horizontal scrolling. **Edge case:** A sidebar note was initially extracted mid-paragraph — the artifact removal pass correctly moved it to a marginal note block.

- **UJ-3. A self-publisher uploads a magazine-style PDF with heavy image use.**
  - **Persona + context:** Elena, self-publishing a photo-rich travel magazine.
  - **Entry state:** Web UI, has the 80-page magazine PDF with multi-column overlapping layouts.
  - **Path:** Uploads → pipeline detects magazine layout type → uses dedicated multi-column parsing path → decorative borders and background textures are stripped automatically → 45 content images are extracted and sized to viewport percentage → 3 flowcharts are reconstructed as Mermaid → 2 complex architectural diagrams are retained as high-res images with detailed alt text → 22 sections flagged for review (mostly caption quality checks).
  - **Climax:** Reviews flagged sections: approves 20, edits 2 AI-generated captions for accuracy. Downloads EPUB3.
  - **Resolution:** Opens on Kobo — images resize correctly with viewport, text reflows, magazine reads like a native publication.

## 3. Glossary

- **PDF** — Portable Document Format. A fixed-layout rendering instruction set with no semantic structure.
- **EPUB3** — The current version of the EPUB standard. Reflowable HTML-based document format with full CSS3 support.
- **IDM (Intermediate Document Model)** — A clean, queryable JSON representation produced after parsing and before export. Contains metadata, chapters with sections and blocks, and flagged items.
- **Artifact** — Any element in the PDF text layer that is not part of the intended reading content: page numbers, running headers, watermarks, anti-copy garbage characters, duplicated pull-quotes, decorative images.
- **Decorative Image** — An image with no semantic content (divider, background texture, logo, watermark). Stripped from output.
- **Content Image** — An image that carries meaning referenced by the text (photograph, diagram, illustration). Retained and block-positioned.
- **Diagram** — A flowchart, technical illustration, or architectural diagram that the pipeline attempts to reconstruct as Mermaid or D2 markup.
- **Confidence Score** — A 0.0–1.0 value assigned to each AI-derived decision (chapter boundary, diagram reconstruction, caption quality). Below-threshold scores trigger human review flags.
- **Flag** — A section marked for mandatory human review during QA, based on confidence thresholds.
- **Mermaid** — A JavaScript-based diagramming and charting tool that renders Markdown-inspired text definitions to SVG. Target format for flowcharts, sequence diagrams, ERDs, state machines.
- **D2** — A declarative diagramming language. Target format for complex architectural diagrams; supports animation.
- **KFX/MOBI** — Amazon Kindle device formats. Produced via post-conversion step (Calibre CLI) from EPUB3.
- **QA Review UI** — In-browser HTML preview of the converted output with flagged section highlighting, side-by-side PDF vs. converted comparison, and approve/edit/revert/skip actions.

## 4. Features

### 4.1 PDF Upload and Parsing

**Description:** Single-PDF upload via web UI. The system reads the PDF, extracts the text layer, inventories images, detects layout type (book, academic, magazine), and scores parsing confidence. Realizes UJ-1, UJ-2, UJ-3.

**Functional Requirements:**

#### FR-1: Single PDF Upload

User can upload a single PDF file (any size up to 200MB) via the web UI. System validates that the file is a valid PDF with a text layer. Realizes UJ-1.

**Consequences (testable):**
- System returns a clear error message for non-PDF files
- System rejects files over 200MB with messaging
- System detects absence of text layer and returns message (scanned PDF detection)
- Upload progress is visible to the user

**Out of Scope:**
- Batch/folder upload
- Drag-and-drop reordering of multiple files

#### FR-2: Layout Type Detection

System classifies uploaded PDF into a layout type (book, academic/multi-column, magazine) and selects the appropriate parsing path. Realizes UJ-2, UJ-3.

**Consequences (testable):**
- Single-column continuous text detected as book type
- Multi-column with footnotes detected as academic type
- Multi-column with overlapping elements and heavy images detected as magazine type
- Layout type displayed on pipeline status screen
- Fallback to book layout when type is ambiguous

#### FR-3: Text Layer Extraction

System extracts all text from the PDF text layer with positional metadata (coordinates, font size, font weight, page number). Realizes UJ-1, UJ-2, UJ-3.

**Consequences (testable):**
- Extraction preserves reading order for single-column layouts
- Multi-column text is extracted in correct column-first reading order
- Positional data (x, y, page) preserved for artifact classification
- Extraction completes within 30 seconds for a 300-page document

**Out of Scope:**
- OCR for scanned/image-only PDFs

### 4.2 Artifact Removal and Text Cleaning

**Description:** A pre-processing pass that classifies extracted text elements into content vs. artifact, then applies targeted cleaning. Artifacts include headers, footers, page numbers, watermarks, anti-copy garbage, duplicated pull-quotes, and ligature/encoding issues. Realizes UJ-1.

**Functional Requirements:**

#### FR-4: Artifact Classification

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

#### FR-5: Artifact Removal and Flow Correction

System removes classified artifacts and reconstructs clean paragraph flow. When removal exceeds 15% of a paragraph, the section is always flagged. Realizes UJ-1.

**Consequences (testable):**
- Page numbers and running headers removed from body text
- Anti-copy garbage characters stripped
- Duplicated pull-quotes resolved (one instance retained)
- Ligature normalization applied (fi, fl, ffi → Unicode equivalents)
- Mid-word hyphenation from line breaks resolved
- Flagged when >15% of paragraph content is removed

#### FR-6: Low-Confidence Handling

System does not silently drop or mangle content below confidence thresholds — it flags for human review. Realizes UJ-1.

**Consequences (testable):**
- Flagged items presented in QA Review UI with original and cleaned versions
- User can approve, edit, revert to original, or skip each flag
- Non-flagged content is not surfaced in review

### 4.3 Image and Figure Processing

**Description:** Image inventory, classification, and treatment. Three categories: decorative (stripped), content images (retained + recaptioned), and diagrams (reconstructed as Mermaid/D2). Realizes UJ-1, UJ-3.

**Functional Requirements:**

#### FR-7: Image Inventory and Classification

System inventories all images in the PDF and classifies each as decorative, content, or diagram. Realizes UJ-1, UJ-3.

**Consequences (testable):**
- Images below a configurable size threshold classified as decorative
- Images outside main text flow (watermarks, logos) classified as decorative
- Images repeated across pages classified as decorative
- Photographs and illustrations classified as content images
- Flowcharts and structured visual diagrams classified as diagrams
- Low-information-entropy images classified as decorative

#### FR-8: Decorative Image Stripping

Decorative images are removed from output. Realizes UJ-1.

**Consequences (testable):**
- No decorative images appear in EPUB3 or HTML output
- Stripping is logged for audit but not flagged for review
- File size reduction from stripping is measurable

#### FR-9: Content Image Placement and Captioning

Content images are repositioned as block-level elements sized to viewport percentage (not fixed pixels). Missing or ambiguous captions are AI-generated. Realizes UJ-1, UJ-3.

**Consequences (testable):**
- Images rendered at configurable viewport width percentage (default 80%)
- No image exceeds viewport width on any target device
- AI-generated caption present when no original caption exists
- AI-generated caption confidence scored; flagged if below 0.85
- Original caption retained when present and unambiguous

#### FR-10: Diagram Reconstruction

Flowcharts and technical diagrams are reconstructed as Mermaid (flowcharts, sequence diagrams, ERDs, state machines) or D2 (complex architectural diagrams) markup. Realizes UJ-2, UJ-3.

**Consequences (testable):**
- AI interprets flat PNG image and outputs Mermaid/D2 source
- Reconstruction confidence scored; flagged if below 0.88
- Complex or ambiguous diagrams fall back to retained high-res image with detailed AI-generated alt text
- Mermaid output renders correctly as SVG in EPUB3
- D2 output renders correctly as SVG (animation optional)

**Out of Scope:**
- Interactive diagram editing in the review UI

### 4.4 Table Processing

**Description:** Intelligent table handling based on column count and content type. Narrow tables retained as HTML, medium tables broken into sub-tables, wide tables rewritten as structured prose. Realizes UJ-2.

**Functional Requirements:**

#### FR-11: Table Classification

System detects tables in the PDF and classifies them by column count, content type (data vs. layout), and multi-page status. Realizes UJ-2.

**Consequences (testable):**
- Tables detected and extracted with cell content and structure
- Column count determined and recorded
- Content type classified as data (numbers, stats) or layout
- Multi-page tables detected and tracked

#### FR-12: Table Transformation

System applies the appropriate strategy per table: ≤4 columns → HTML table with responsive CSS; 5-8 columns → broken into logical sub-tables; >8 columns or dense data → rewritten as structured prose or bullet summary with original moved to appendix. Realizes UJ-2.

**Consequences (testable):**
- ≤4 column tables render as HTML tables legible on 600px viewport
- 5-8 column tables split into sub-tables with clear grouping rationale
- >8 column tables converted to structured prose with data summary
- Multi-page tables have column headers re-injected at each continuation
- Data tables offer chart/visual representation as alternative view
- Table-to-prose transformation always flagged for human review
- No table requires horizontal scrolling on e-reader

**Out of Scope:**
- Editable table reconstruction in review UI

### 4.5 Chapter and Structure Detection

**Description:** Semantic structure detection using PDF bookmarks (priority 1), embedded TOC page (priority 2), AI analysis of heading patterns (priority 3), and human review fallback (priority 4). Produces the navigable TOC. Realizes UJ-1.

**Functional Requirements:**

#### FR-13: PDF Bookmark Extraction

System reads the PDF bookmark/named destination tree and uses it as the primary chapter structure. Realizes UJ-1.

**Consequences (testable):**
- Bookmark tree extracted and mapped to page ranges
- Bookmark tree used directly when confidence > 0.95
- Missing or incomplete bookmarks trigger fallback to TOC page detection

**Out of Scope:**
- Editing bookmark data in the review UI

#### FR-14: TOC Page Parsing

System detects and parses the table of contents page(s) when PDF bookmarks are absent or incomplete. Validates or supplements bookmark data. Realizes UJ-1.

**Consequences (testable):**
- TOC page detected by positional and typographic patterns
- TOC entries parsed with page number references
- Confidence scored on TOC-to-content alignment

#### FR-15: AI Structural Analysis

For PDFs without bookmarks or a clear TOC page, system uses AI analysis of heading size, font weight, spacing, and positional patterns to infer chapter/section boundaries. Realizes UJ-1, UJ-2.

**Consequences (testable):**
- Chapter boundaries detected with confidence scores
- Confidence < 0.90 flagged for human review
- Heading hierarchy (chapter → section → subsection) inferred and recorded
- Section blocks mapped to parent chapters

#### FR-16: Navigable TOC Generation

System produces a navigable table of contents in the EPUB3 output. Realizes UJ-1.

**Consequences (testable):**
- EPUB3 TOC present and navigable on Kobo, PocketBook, Kindle (after conversion)
- Chapter skip works on device
- Section-level navigation present in the EPUB3 spine
- Heading hierarchy reflected in TOC nesting

### 4.6 QA Review UI

**Description:** In-browser HTML preview of the converted output with flagged section highlighting. Side-by-side comparison of original PDF rendering vs. converted output. Per-flag actions: approve, edit, revert to original, skip. Realizes UJ-1, UJ-2, UJ-3.

**Functional Requirements:**

#### FR-17: Flagged Section Review

System presents all flagged sections in the QA Review UI with highlighting. Realizes UJ-1.

**Consequences (testable):**
- Flagged sections visually highlighted in HTML preview
- Flag reason and confidence score displayed per flag
- Non-flagged sections not shown in review (but accessible for context)
- User can navigate between flags sequentially

#### FR-18: Side-by-Side Comparison

System renders original PDF excerpt alongside converted output for each flagged section. Realizes UJ-1.

**Consequences (testable):**
- PDF excerpt rendered from original source
- Converted output rendered as it appears in EPUB
- Side-by-side view responsive on browser

#### FR-19: Per-Flag Actions

User can approve, edit, revert to original, or skip each flag. Realizes UJ-1, UJ-2, UJ-3.

**Consequences (testable):**
- Approve: flag dismissed, current output retained
- Edit: inline text editor opens, user modifies content, change recorded
- Revert to original: restores the un-modified extracted content for this section
- Skip: flag deferred, output unchanged
- All actions logged for audit

### 4.7 Export Pipeline

**Description:** Multi-format export: EPUB3 (primary), HTML (review format), and KFX/MOBI (via Calibre CLI). Each export reads from the IDM. Realizes UJ-1, UJ-2, UJ-3.

**Functional Requirements:**

#### FR-20: EPUB3 Export

System generates a standards-compliant EPUB3 with CSS3 styling, semantic HTML, navigable TOC, and embedded images. Realizes UJ-1.

**Consequences (testable):**
- EPUB3 passes epubcheck validation
- TOC navigable on Kobo and PocketBook
- Images render at responsive viewport widths
- CSS3 media queries respect device preferences (dark mode, font size)
- No absolute positioning in output CSS

**Out of Scope:**
- Fixed-layout EPUB
- EPUB2 compatibility

#### FR-21: HTML Reading View Export

System generates a standalone HTML document optimized for browser reading view. Realizes UJ-1.

**Consequences (testable):**
- HTML renders correctly in Chrome, Firefox, Safari reading modes
- Responsive at all viewport widths
- No external CSS dependencies (self-contained)

#### FR-22: KFX/MOBI Export

System invokes Calibre CLI to convert EPUB3 to KFX or MOBI format for Kindle devices. Realizes UJ-1.

**Consequences (testable):**
- Calibre CLI called with correct conversion profile (Kobo → Kindle)
- KFX/MOBI output renders on Kindle devices
- Conversion failure surfaced to user with error message

**Out of Scope:**
- Direct KFX generation without Calibre dependency

### 4.8 Post-Processing Module (Future)

**Description:** Read-only operations on the IDM to generate derivatives: chapter summaries, daily newsletters, reading plans, and style rewrites. Scope and architecture TBD — share pipeline or separate module. Realizes long-term vision.

**Functional Requirements:**

#### FR-23: Chapter Summary Generation

System generates chapter-by-chapter summaries from the IDM section block trees.

**Consequences (testable):**
- Summary per chapter with key takeaways
- Summary length configurable (brief / detailed)
- Summary does not modify EPUB output

**Out of Scope:**
- v1 scope — TBD

**Feature-specific NFRs:**
- Post-processing operations must be read-only on the IDM — they cannot modify the stored model or the EPUB output.

## 5. Non-Goals (Explicit)

- **Not an OCR engine** — scanned/image-only PDFs are out of scope for v1
- **Not a batch processor** — single-PDF upload only in v1
- **Not an API** — web UI only; no programmatic access in v1
- **Not a PDF editor** — the product converts PDFs; it does not modify them
- **Not a storefront** — no direct publishing integration with Kindle, Kobo, or Apple Books
- **Not an EPUB editor** — the QA Review UI handles flagged sections only; full EPUB editing is out of scope
- **Not a diagram editor** — Mermaid/D2 output is generated; the UI does not provide interactive diagram editing
- **Not multi-language in v1** — English-language PDFs only; RTL language support is deferred

## 6. MVP Scope

### 6.1 In Scope

- Single PDF upload via web UI (text-layer PDFs only)
- Text extraction with artifact removal (headers, footers, page numbers, anti-copy garbage, ligature normalization)
- Layout type detection (book, academic, magazine)
- Image classification and treatment (decorative strip, content retain + recaption, diagram reconstruct)
- Table classification and transformation (≤4/5-8/>8 column strategies)
- Chapter detection via bookmarks, TOC page, and AI analysis
- EPUB3 and HTML export
- QA Review UI with flagged section highlighting, side-by-side comparison, and per-flag actions
- Confidence scoring and flag system for all AI-derived decisions

### 6.2 Out of Scope for MVP

- Scanned PDF / OCR support — requires separate strategy and significant compute
- Batch/folder processing — deferred to v2
- API endpoint — deferred to v2
- KFX/MOBI export via Calibre — deferred to v1.1 (toolchain decision in progress)
- Post-processing features (summaries, newsletters, reading plans, style rewriting) — TBD scope and architecture
- Right-to-left language support — deferred to v2+
- Mathematical equation rendering (LaTeX → MathML) — deferred to v2+

## 7. Success Metrics

**Primary**
- **SM-1**: Publisher-quality output — output EPUB3 passes epubcheck validation with zero errors. Validates FR-20.
- **SM-2**: Readability — no garbled text, injected artifacts, or broken encoding in output. Validates FR-4, FR-5, FR-6.
- **SM-3**: Navigability — TOC present in EPUB3, chapter skip works on Kobo and PocketBook devices. Validates FR-16.

**Secondary**
- **SM-4**: Conversion time — 300-page PDF processes in under 3 minutes end-to-end. Validates FR-3, FR-4, FR-7, FR-15.
- **SM-5**: Flag accuracy — no more than 10% false positives in flagged items (items flagged unnecessarily). Validates confidence scoring system (cross-cutting).

**Counter-metrics (do not optimize)**
- **SM-C1**: Flag rate — do not optimize for zero flags. Higher flag rates are acceptable if they indicate proper caution. Counterbalances SM-5.
- **SM-C2**: Processing speed — do not sacrifice output quality for faster conversion. Counterbalances SM-4.

## 8. Open Questions

1. **Post-processing module architecture** — same pipeline or separate module? Affects IDM storage and versioning. Decision needed before architecture phase.
2. **Magazine layout parsing strategy** — multi-column overlapping layouts need a dedicated parsing path. Requires a spike before committing to a single parser library.
3. **Kindle conversion toolchain** — Calibre CLI vs. Kindle Previewer 3 vs. other. KindleGen is deprecated. Calibre is the practical choice but adds a deployment dependency.
4. **D2 vs. Mermaid decision boundary** — when does a diagram type warrant D2 over Mermaid? Needs a taxonomy of diagram types with format mapping.
5. **AI diagram reconstruction failure mode** — when AI cannot reliably reconstruct a diagram, the fallback is retained image + detailed alt text. Confirm this is the correct behavior for all cases.
6. **Calibre dependency management** — if KFX/MOBI export is included, Calibre must be available in the deployment environment. Determine containerization strategy.

## 9. Assumptions Index

- Assumption from §4.1 (FR-1): 200MB file size limit is sufficient. Adjust based on user feedback.
- Assumption from §4.2 (FR-4): Artifact classification heuristics (position, repetition, font properties) are sufficient for >90% of cases.
- Assumption from §4.2 (FR-5): >15% paragraph removal threshold is a correct balance of sensitivity vs. false positives. Calibrate during beta.
- Assumption from §4.3 (FR-9): Viewport-width percentage (80% default) is the correct image sizing approach for e-readers.
- Assumption from §4.3 (FR-10): AI image-to-diagram reconstruction is feasible for a majority of flowchart and technical diagram types.
- Assumption from §4.4 (FR-12): 4-column and 8-column thresholds for table strategies are correct. Calibrate during beta.
- Assumption from §4.5 (FR-15): AI structural analysis can reliably detect chapter boundaries from typographic patterns in the absence of bookmarks.
- Assumption from §4.6 (FR-17): Side-by-side PDF vs. converted comparison is sufficient for QA review. No interactive overlay or diff view required.
- Assumption from §7 (SM-5): 10% false-positive flag rate is an acceptable target for confidence scoring calibration.

## 10. Cross-Cutting NFRs

- **Performance:** 300-page PDF processed end-to-end in under 3 minutes. QA Review UI loads in under 2 seconds.
- **Reliability:** Pipeline does not silently produce bad output — any low-confidence decision is flagged. System handles malformed PDFs gracefully with clear error messages.
- **Security:** Uploaded PDFs are isolated per session. Files are not accessible between user sessions. Processed output is available for download for 24 hours then deleted.
- **Observability:** Pipeline stages log duration, confidence scores, and flag counts. Failed conversions surface error details to user and system logs.

## 11. Constraints and Guardrails

- **Cost:** AI enrichment (recaptioning, diagram reconstruction, structural analysis) uses an LLM API. Per-document cost must not exceed $0.50 for a 300-page PDF. If cost exceeds threshold, fall back to non-AI paths where available.
- **Privacy:** Uploaded PDFs are not stored beyond the 24-hour download window. PDF content is not used for model training. AI API calls do not include PII in prompts.
- **Output quality:** The pipeline must never produce output worse than the original PDF reading experience. When confidence is low, flag rather than guess.

## 12. Why Now

The existing PDF-to-EPUB converter ecosystem is stagnant — tools do mechanical conversion that publishers cannot ship. Meanwhile, LLM capabilities for image understanding, diagram reconstruction, and structural analysis have crossed a quality threshold that makes intelligent conversion feasible for the first time. The combination of accessible AI APIs + mature EPUB3 standard + growing e-reader market (Kobo, PocketBook, Kindle) creates a clear window for a publisher-quality converter.

## 13. Platform

Web application (responsive browser UI). No mobile app or native desktop client in v1. Deployment as a containerized web service (Docker). Single-page application frontend with API backend.
