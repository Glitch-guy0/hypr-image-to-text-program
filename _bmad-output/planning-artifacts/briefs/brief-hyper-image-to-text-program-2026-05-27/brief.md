---
title: Hyper Image-to-Text Program
status: draft
created: 2026-05-27
updated: 2026-05-27
---

# Product Brief: Hyper Image-to-Text Program

## Executive Summary

PDFs are rendering instructions, not documents — glyphs and images positioned at absolute coordinates for a single page size. E-readers (Kobo, Kindle, PocketBook) are a fundamentally different medium: text-first, reflow-driven, variable-viewport. Every existing PDF-to-EPUB converter carries over the broken positioning, injects page numbers mid-paragraph, produces microscopic images, and generates output no publisher would ship.

This product builds a pipeline that genuinely understands PDF content — extracting text with semantic awareness, classifying images (decorative vs. content vs. diagram), restructuring tables for small screens, detecting chapter structure, and producing EPUB3 output indistinguishable from a commercial publisher's own file. It uses AI not as a gimmick but as a practical tool: reconstructing flat flowchart images into native Mermaid diagrams, generating captions for uncaptioned figures, and flagging low-confidence decisions for human review rather than silently producing bad output.

The conversion pipeline is accessible via web UI — no local tooling required. Upload a PDF, review flagged sections in the browser preview, and download a publisher-quality EPUB3 or HTML reading view.

## The Problem

Anyone who has tried to read a PDF on an e-reader knows the pain: microscopic text that does not reflow, images that are either gigantic or invisible, page numbers and running headers injected into the middle of paragraphs, tables that require a magnifying glass, and flowcharts that are completely illegible. Existing converter tools — Calibre's built-in conversion, online PDF-to-EPUB services — produce output with the same mechanical flaws because they treat PDF as document rather than as rendering instructions.

The cost of the status quo:
- **Publishers** manually rebuild EPUBs from print PDFs, adding days of production time per title
- **Academic researchers** cannot comfortably read multi-column journal papers on their e-readers
- **Self-publishers** are locked into technical toolchains (Calibre CLI, KindleGen) and still get poor results
- **Accessibility** suffers — screen readers cannot navigate untagged, artifact-ridden output

## The Solution

A web-based pipeline that converts PDFs to publisher-quality EPUB3 through six stages:

1. **Parse and classify** — text extraction with positional metadata, image inventory, layout type detection
2. **Artifact removal** — strip headers, footers, page numbers, anti-copy garbage, decorative images; clean text flow
3. **Semantic structuring** — detect chapters via bookmarks, TOC, or AI analysis; classify tables and images
4. **AI enrichment** — recaption images, reconstruct diagrams as Mermaid/D2, rewrite wide tables as structured prose
5. **Human QA review** — browser preview with flagged section highlighting, side-by-side comparison, approve/edit/revert
6. **Export** — EPUB3, HTML reading view, and (future) KFX/MOBI via Calibre

The key insight: the pipeline does not need human review of every page — only low-confidence sections. Confidence scoring determines what gets flagged, and the review UI makes those decisions fast.

## What Makes This Different

- **Semantic understanding, not mechanical conversion** — we parse PDF as what it is (rendering instructions) but reconstruct the semantic document the author intended
- **AI-native architecture** — LLMs are used not for text generation but for structural tasks (diagram reconstruction, image captioning, chapter detection, table rewriting) that directly improve output quality
- **Confidence-scored flagging** — the system knows when it is unsure and asks for help, rather than silently producing bad output
- **Diagram-native output** — flat flowchart PNGs become Mermaid/D2 markup — zoomable, accessible, dark-mode capable
- **Publisher-quality bar** — the output must be indistinguishable from a commercial EPUB of the same title

## Who This Serves

- **Primary: Publisher production teams** — need to convert designer print PDFs to EPUB without manual rebuild. Success: EPUB passes epubcheck, renders perfectly on Kobo, ships to store.
- **Primary: Academic researchers** — need to read multi-column journal papers on e-readers. Success: paper is legible at any font size, tables readable without horizontal scroll.
- **Primary: Self-publishers** — need store-ready EPUB from a single PDF upload. Success: upload → review → download → publish, no technical skills required.
- **Secondary: Content archivists and accessibility teams** — need standards-compliant, navigable, screen-reader-friendly EPUB output.

## Success Criteria

1. **Readable** — no garbled text, injected artifacts, or broken encoding in output
2. **Navigable** — TOC present, chapter skip works, semantic headings throughout
3. **Visually appropriate** — images sized for e-reader viewports, no overflow
4. **Tables legible** — no horizontal scroll on e-ink devices, text above readable size
5. **Publisher-comparable** — output indistinguishable from commercial EPUB of same title
6. **Diagrams native** — flowcharts and technical diagrams rendered as Mermaid/D2 markup

## Scope

**In (v1):**
- Single PDF upload (text-layer PDFs only), web UI
- Text extraction with artifact removal
- Layout type detection (book, academic, magazine)
- Image classification and treatment (decorative strip, content retain + recaption, diagram reconstruction)
- Table classification and transformation
- Chapter detection (bookmarks, TOC, AI)
- EPUB3 and HTML export
- QA Review UI with flagged section review

**Out (v1):**
- Scanned PDF/OCR — separate strategy needed
- Batch/folder processing — v2
- API access — v2
- KFX/MOBI export — v1.1
- Post-processing (summaries, newsletters, style rewriting) — TBD
- RTL language support — v2+
- Math equation rendering — v2+

## Vision

This product becomes the standard pipeline for professional PDF-to-EPUB conversion — relied on by publishing houses, academic institutions, and self-publishers alike. In 2-3 years, the Intermediate Document Model (IDM) becomes the foundation for a content services platform: generate chapter summaries, daily reading plans, newsletter digests, and style rewrites as read-only derivatives of the same semantic model. The IDM is the asset; EPUB export is just one of many outputs.
