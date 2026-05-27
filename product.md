## Overview

PDFs are designed for fixed-size screens with free zoom — every element (text, image, table) is absolutely positioned for visual perfection at a specific page size. E-readers are an entirely different medium: text-first, low-refresh-rate, fixed-viewport devices where absolute positioning breaks entirely.

Existing PDF-to-EPUB converters do a mechanical conversion — they carry over the broken positioning, inject unwanted artifacts, and produce output that no publisher would ship. The goal of this project is to build a pipeline that produces **publisher-quality EPUB3 and HTML output** that genuinely reads well on Kobo, Kindle, and browser-based readers.

A PDF is not a document. It is a set of rendering instructions — "place this glyph at coordinate (x, y)", "draw this image at (x2, y2) with width w and height h". There is no semantic meaning: no concept of paragraphs, no concept of "this text is a heading", no concept of "this image belongs to this caption". Everything that looks structured to a human reader is purely visual coincidence of positioning.

An EPUB, by contrast, is a semantic HTML document. It reflows. The reader controls font size, line spacing, and margin. The device controls viewport width. Absolute positioning is not just useless — it is actively harmful.

This mismatch is the root cause of every problem in the existing converter ecosystem.

---

## Input Scope

### PDF types handled

| Type | Notes |
|---|---|
| Text-based PDFs | Searchable text layer present — primary target |
| Academic papers / research | Multi-column layouts, footnotes, citations, equations |
| Long-form books | Chapters, TOC, consistent heading hierarchy |
| Magazine-style layouts | Heavy image use, multi-column, overlapping elements — highest complexity |

> **Out of scope (v1):** Scanned/image-only PDFs requiring OCR. These need a separate strategy and significantly more compute.

### Output targets

| Format | Platform | Notes |
|---|---|---|
| EPUB3 | Kobo, PocketBook | Native format — full CSS3 support |
| EPUB3 → KFX/MOBI | Kindle | Requires post-conversion step via Calibre or KindleGen; stricter formatting rules apply |
| HTML | Browser reading view | Responsive, optimised for reading — also serves as the review format during QA |

### User Interaction

- **Interface:** Web UI / dashboard — no local tooling, accessible to non-technical users
- **Input:** Single PDF upload per session (v1)
- **Review:** In-browser HTML preview with flagged section highlighting
- **Output:** Download EPUB3, KFX/MOBI, or HTML

---

## Core Issues to Solve

### 1. Text extraction & artifact removal

PDF text extraction is not a solved problem. Even with a clean text-layer PDF, extraction produces:

- **Inconsistent text flow** — columns extracted in the wrong order, mid-word line breaks from hyphenation, merged words where inter-word spacing was visual rather than encoded
- **Header/footer injection** — page numbers, running headers, copyright notices, and watermarks get injected into the body text mid-paragraph
- **Anti-copy artifacts** — some publishers embed invisible characters, reorder Unicode codepoints, or split words across invisible layers to defeat clipboard copying. These appear as garbled text after extraction
- **Author typographic decisions** — drop caps extracted as isolated single characters, pull quotes duplicated (once in-flow, once as a positioned overlay), sidebar text injected mid-paragraph
- **Ligature and encoding issues** — `fi`, `fl`, `ffi` ligatures extracted as single codepoints that don't match standard Unicode, leading to broken search and broken text-to-speech

**Strategy:**
- Classify artifact types during a pre-processing pass
- Apply targeted cleaning rules per artifact class
- Flag low-confidence regions for human review rather than silently dropping or mangling content

---

### 2. Images & figures

Images in PDFs are absolutely positioned with pixel coordinates. Converters that carry these coordinates over produce images that are either microscopic (a full-page figure rendered at 80×60px on a 6-inch screen) or enormous (a thumbnail that fills the entire viewport).

**Beyond positioning, there are three categories of images that need different handling:**

#### Decorative images
Images that are purely presentational — dividers, background textures, publisher logos, watermarks, decorative borders. These should be **stripped entirely**. They add file size, increase e-reader rendering load, and contribute nothing to reading.

**Detection:** Low information-entropy images, images outside the main text flow, images under a size threshold, images that repeat across pages.

#### Content images (photographs, diagrams, figures)
Images that carry meaning the text references. These should be:
- Retained and repositioned as block-level elements in the text flow
- Resized to a percentage of viewport width (not fixed pixels)
- **AI-recaptioned** if no caption is present or if the existing caption is ambiguous — the caption should describe what the image shows, not just "Figure 3.2"

#### Flowcharts, diagrams, and technical illustrations
These are the hardest and most valuable case. A flowchart saved as a flat PNG is:
- Unzoomable on e-ink without quality loss
- Not accessible (no alt text, no structure)
- Not dark-mode capable
- Static

The target output is **Mermaid or D2 native diagrams**:
- Mermaid for flowcharts, sequence diagrams, entity relationships, state machines
- D2 for more complex architectural diagrams; D2 also supports animation for flowcharts
- AI interprets the image, reconstructs the semantic structure, and outputs the diagram markup
- Falls back to a high-quality retained image with detailed AI-generated alt text if the diagram is too complex or ambiguous to reconstruct reliably

---

### 3. Tables

Tables are one of the worst experiences on e-readers. A 12-column financial table that looks fine at full A4 width is completely illegible at 600px — text shrinks to 6px, columns overlap, and there is no horizontal scroll on e-ink.

**Strategy — chosen based on column count and content type:**

| Scenario | Strategy |
|---|---|
| ≤ 4 columns, simple data | Retain as HTML table with responsive CSS |
| 5–8 columns | Break into multiple sub-tables grouped by logical relationship |
| > 8 columns, or dense data | AI rewrites as structured prose or bullet summary, table moved to appendix |
| Headers missing on continuation pages | Detect multi-page tables, re-inject column headers at each continuation |
| Data tables (numbers, stats) | Consider chart/visual representation as an alternative |

The key principle: **it is better to have readable prose than an unreadable table**. A table is a display optimisation, not a semantic requirement — the information can always be expressed another way.

---

### 4. Chapter & structure detection

Semantic structure (chapters, sections, subsections) is what makes an EPUB navigable. Without it, there is no table of contents, no chapter-skip on the device, and no way to generate derivatives (summaries, newsletters) downstream.

**Detection strategy (in priority order):**

1. **PDF bookmarks / named destinations** — if the PDF has a bookmark tree, use it directly. Most publisher PDFs have this.
2. **Embedded TOC page** — detect and parse the table of contents page if present; use it to validate or supplement bookmark data
3. **AI structural analysis** — for PDFs with no bookmarks, use heading size, font weight, spacing, and positional patterns to infer chapter boundaries. Outputs a confidence score per detected boundary.
4. **Human review fallback** — low-confidence boundaries are flagged in the review UI for manual confirmation or correction

---

## Intermediate Document Model

A clean, queryable intermediate representation produced after parsing and before export:

```json
{
  "metadata": {...},
  "chapters": [
    {
      "id": "...",
      "title": "...",
      "confidence": ...,
      "sections": [
        {
          "id": "...",
          "heading": "...",
          "heading_level": ...,
          "blocks": [
            { "type": "paragraph", "text": "..." },
            { "type": "image", "src": "...", "caption": "...", "original_type": "..." },
            { "type": "table", "strategy": "...", "original_cols": ..., "content": "..." },
            { "type": "diagram", "format": "mermaid", "source": "...", "confidence": ... }
          ]
        }
      ]
    }
  ],
  "flagged": [
    { "ref": "...", "reason": "...", "confidence": ... }
  ]
}
```
## Processing Pipeline

1. **Parse & classify** — Text extraction, image inventory, layout type detection, confidence scoring
2. **Artifact removal** — Header/footer strip, anti-copy cleanup, decorative image strip, ligature normalisation
3. **Semantic structuring** — Chapter detection, heading hierarchy, table classification, image classification
4. **AI enrichment** — Image recaptioning, diagram → Mermaid/D2, table → prose/sub-tables, confidence flagging
5. **Human QA review** — Review UI (HTML preview), approve / correct / skip (flagged sections only)
6. **Export** — EPUB3 generation, Kindle KFX/MOBI (Calibre), HTML reading view
7. **Post-processing (TBD)** — Summary generation, newsletter format, daily reading plan, style rewriting

---

## QA Review System

The pipeline does not require human review of every page — only flagged sections. Confidence scoring determines what gets flagged.

**Flag triggers:**

| Trigger | Threshold |
|---|---|
| Chapter boundary detected by AI (no PDF bookmark) | confidence < 0.90 |
| Diagram reconstructed as Mermaid/D2 | confidence < 0.88 |
| Table rewritten as prose | always flagged (structural change) |
| Artifact removal removed > 15% of a paragraph | always flagged |
| Image AI-recaptioned with no original caption | confidence < 0.85 |

**Review UI:**
- HTML preview of the book, flagged sections highlighted
- Side-by-side: original PDF rendering vs converted output
- Actions per flag: approve, edit, revert to original, skip
- Non-technical users access via web UI; no local tooling required

---

## Post-Processing Module

> **Status: scope not yet decided** — may be same pipeline, separate module, or separate product. Architectural decision needed before build.

Once the intermediate document model exists, the following are straightforward to build:

| Feature | Description |
|---|---|
| **Book summary** | Chapter-by-chapter summary with key takeaways, generated from section block trees |
| **Daily newsletter** | Digestible excerpt per day, respecting narrative flow — not random chunks |
| **Daily reading plan** | Suggested reading order with time estimates per session |
| **Style rewriting** | Rewrite content in a different voice (academic → conversational, verbose → concise) while preserving meaning and structure |

All of these are read-only operations on the intermediate model — they do not modify the EPUB output.

---

## Open Questions & Risks

### High priority — resolve before build

- [ ] **Post-processing module architecture** — same pipeline or separate? Affects how the intermediate model is stored and versioned
- [ ] **Magazine layout strategy** — multi-column, overlapping elements need a dedicated parsing path separate from book-style PDFs. Needs a spike before committing to a single parser

### Medium priority

- [ ] **Kindle conversion toolchain** — Calibre CLI vs KindleGen (deprecated) vs Kindle Previewer 3. KindleGen is no longer officially supported; Calibre is the practical choice but adds a dependency
- [ ] **D2 vs Mermaid decision boundary** — when does a diagram warrant D2 over Mermaid? Need a taxonomy of diagram types and a mapping to formats
- [ ] **AI diagram reconstruction failure mode** — what is the output when AI cannot reliably reconstruct a diagram? Retain flat image + detailed alt text, or drop and flag?

### Low priority / future scope

- [ ] Scanned PDF / OCR support
- [ ] Batch processing (folder of PDFs)
- [ ] API endpoint for programmatic access
- [ ] Right-to-left language support (Arabic, Hebrew, Urdu)
- [ ] Mathematical equation rendering (LaTeX → MathML for EPUB3)

---

## Success Criteria

A successful conversion should meet the following bar:

1. **Readable** — no garbled text, injected artifacts, or broken encoding
2. **Navigable** — table of contents present, chapter skip works on device, semantic headings
3. **Visually appropriate** — images sized for viewport, no overflow or microscopic figures
4. **Tables legible** — no horizontal scroll or text below 12px effective size
5. **Publisher-comparable** — indistinguishable from commercial EPUB of same title
6. **Diagrams native** — flowcharts/technical diagrams as Mermaid/D2 where possible

---

*Last updated: May 2026*