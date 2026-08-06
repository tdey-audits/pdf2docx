# PDF2DOCX Repository Inspection & Architectural Analysis

## 1. Executive Summary & Overview

**pdf2docx** is a high-fidelity, high-performance, and intelligent PDF-to-Word (`.docx`) converter written in Python. It is designed to preserve document design and structure—including fonts, sizes, colors, vector diagrams, table structures, and images—while specifically offering **native Word equations** (editable Office Math Markup Language objects) through multi-modal vision model transcription and LaTeX compilation.

The application operates as a standalone FastAPI web service with a single-page HTML interface, and can be integrated into broader document-processing workflows via its REST API. It supports three distinct layout conversion modes to accommodate different editing requirements:
1. **`structured` (Default)**: Reconstructs the PDF's logical structure into flowing paragraphs, headings, lists, tables, pictures, and native equations. It is fully editable, as text reflows naturally when content is added or modified.
2. **`replica`**: Reproduces every document element at its exact coordinate using absolute-positioned VML floating frames in Word. It is a visual facsimile where layouts do not reflow during editing.
3. **`flow`**: Uses a vision model to read each page image and rewrite it as ordinary, editable Markdown before rendering it to a clean `.docx` using standard flowing paragraphs.

---

## 2. Global Architecture & File Responsibilities

The codebase follows an elegant, modular design where tasks are cleanly separated across distinct layers. Below is an architectural diagram and file-by-file description:

```
                          ┌───────────────┐
                          │   Web / API   │
                          │   (main.py)   │
                          └───────┬───────┘
                                  │
                                  ▼
                          ┌───────────────┐
                          │ Pipeline Coord│
                          │ (pipeline.py) │
                          └───────┬───────┘
                                  │
         ┌────────────────────────┼────────────────────────┐
         ▼                        ▼                        ▼
┌─────────────────┐      ┌─────────────────┐      ┌─────────────────┐
│  REPLICA Mode   │      │ STRUCTURED Mode │      │    FLOW Mode    │
│  (Positioned)   │      │ (Reflow / Flow) │      │  (Pure Vision)  │
└────────┬────────┘      └────────┬────────┘      └────────┬────────┘
         │                        │                        │
         │                        │ ┌───────────────┐      │
         │                        │ │  Block Reflow │      │
         │                        │ │ (reflow.py)   │      │
         │                        │ └───────┬───────┘      │
         │                        │         │              │
         ▼                        ▼         ▼              ▼
┌───────────────────────────────────────────────────────────────────┐
│                        Helper Subsystems                          │
│                                                                   │
│  - PDF Extract (pdf_extract.py)      - Word Gen (docx_builder.py) │
│  - PDF Render (pdf_render.py)        - LaTeX -> OMML (latex_omml) │
│  - Vision Models (vision.py)         - Symbol Dec (symbols.py)    │
│  - Figure Cutouts (figures.py)       - XML Sanitizer (xml_text.py)│
└───────────────────────────────────────────────────────────────────┘
```

### Module Responsibilities

| Directory/File | Responsibility |
| :--- | :--- |
| **`app/main.py`** | FastAPI entry point exposing web routes and REST endpoints. Implements multi-threaded background job registry, history persistence, and server-restart crash mitigation. |
| **`app/config.py`** | Runtime settings loaded from environmental variables and `.env`. Controls concurrency, OpenRouter client config, image DPIs, token budgets, and history limits. |
| **`app/history.py`**| Persistent history manager. Writes background job statistics and metadata to `history.json` atomically via tmp-files and file replacement under thread-locks. |
| **`app/pipeline.py`**| Core pipeline orchestrator. Implements `convert_pdf_replica` (driving both the replica and structured paths) and `convert_pdf_flow`. Manages `ThreadPoolExecutor` workers. |
| **`app/pdf_extract.py`**| PyMuPDF-based extraction layer. Analyzes text spans, line heights, bounding boxes, vector strokes, table structures, and math equations. Maps font styles and sizes. |
| **`app/pdf_render.py`**| PDF rendering module. Rasterizes document pages with an adaptive zoom/resolution algorithm that respects high-resolution constraints and vision model token limits. |
| **`app/vision.py`**| OpenRouter / OpenAI API client. Handles system prompt generation, streaming completions, math OCR, and cost tracking with defensive error-handling. |
| **`app/figures.py`**| Cutout coordinator. Parses model-predicted bounding boxes and crops graphical figures directly from original vector pages at sharp diagram resolutions. |
| **`app/reflow.py`**| Layout analysis engine. Analyzes scattered blocks, tables, and absolute coordinates to reconstruct a flowing document model with paragraphs, headings, and indents. |
| **`app/docx_replica.py`**| Low-level Word document generator for replica mode. Translates positioned elements to absolutely positioned Microsoft Word VML shapes with custom character spacing. |
| **`app/docx_structured.py`**| Word document builder for structured mode. Transforms flowable page blocks (headings, paragraphs, tables, equations, lists) into real Word paragraph formatting. |
| **`app/docx_builder.py`**| Markdown-to-Word compilation engine. Lexes GitHub-Flavored Markdown into flowable paragraph elements, list hierarchies, blockquotes, code blocks, and tables. |
| **`app/latex_omml.py`**| LaTeX-to-OMML parser and compiler. Converts mathematical formulas into Word's native equation editor markup (XML schema). Includes fallback protections. |
| **`app/symbols.py`**| Character decoding bridge. Translates 8-bit Adobe Symbol Font Private Use Area (PUA) codes into standard UTF-8 characters and maps fonts to Cambria Math. |
| **`app/xml_text.py`**| XML-safe string sanitizer. Scrubs control characters and illegal surrogates that are rejected by OOXML specifications (such as InDesign's U+0007). |

---

## 3. End-to-End Pipelines & Data Flows

The repository executes three logical paths based on the requested output layout mode.

```
┌───────────────────────────────────────────────────────────────────────────────────────────────────┐
│ Common Initial Stage (All Modes):                                                                 │
│ 1. Multipart PDF upload received by `/api/convert`.                                               │
│ 2. Background task initialized; Job status set to `queued`.                                       │
└───────────────────────────────────────────────────────────────────────────────────────────────────┘
```

### Pipeline A: `replica` Mode

1. **Extraction (`app/pdf_extract.py`)**: PyMuPDF extracts page dimensions and reads native metadata. Spans of text are gathered, including characters, fonts, weights, colors, baseline origins, and coordinates.
2. **Clustering & Image Extraction**: Raster graphics (images) are extracted. Vector paths are grouped: axis-aligned lines become rules, and complex vector strokes are clustered. If vector stroke bounding boxes overlap with nearby text spans, labels are absorbed into the visual diagram boundary. The final bounded region is rasterized at high resolution with a transparent background.
3. **Math Detection**: Equations are detected by identifying segments written in math-oriented fonts. If the typical length of math characters on a page indicates true mathematical expressions, contiguous math spans are grouped. The system renders high-resolution PNG crops of these equation regions.
4. **Formula Transcription (`app/vision.py`)**: Equation PNG crops are batched into requests (up to 10 at a time) and sent to the vision model with instructions to return a JSON array containing pure LaTeX strings.
5. **XML Reconstruction (`app/docx_replica.py`)**:
    - Translates document margins, pages, and orientations.
    - Floating images and diagrams are mapped as absolute VML rectangles.
    - Inline equations are compiled to inline OMML objects using `app/latex_omml.py` and wrapped in VML textboxes.
    - Text lines are split to prevent spatial drift. Each text fragment is measured using Base-14 font metrics, and its character spacing is dynamically adjusted (`w:spacing`) to match the exact PDF layout coordinates.

---

### Pipeline B: `structured` Mode (Default)

1. **Extraction & Math Transcription**: Identical to steps 1–4 of the `replica` pipeline, extracting text, rules, diagrams, and transcribing math.
2. **Logical Flow Analysis (`app/reflow.py`)**:
    - Discards repeated background headers, footers, and page-number tabs across pages based on structural repetition thresholds.
    - Recombines separate lines on the same visual row (e.g., superscripts, footnotes) into a single logical text line.
    - Computes global document metrics (dominant font size, line pitch, margins, indents, list alignments).
    - Groups text lines into paragraph blocks, headings (sized based on font size ratios), and lists (capturing custom marker symbols like `1.` or `•`).
    - Stitches together paragraph blocks that span across a page break by identifying trailing hyphens and lowercase sentence continuations.
    - Folds equation numbers (e.g., `(1)`) set on the far right back onto the corresponding formula blocks.
3. **Document Flow Writing (`app/docx_structured.py`)**:
    - Computes the layout column width and adjusts margins.
    - Renders text blocks into flowing paragraphs. Strips hyphenations that crossed line breaks and joins words.
    - Renders equations natively into Word as block-level or inline mathematical formulas.
    - Inserts tables, lists, and images in-line inside the logical text column.

---

### Pipeline C: `flow` Mode

1. **Page Rasterization (`app/pdf_render.py`)**: Rasterizes every page of the uploaded PDF to a temporary high-resolution PNG image, scaling down the zoom dynamically if the long edge exceeds `settings.max_edge` to fit API bounds.
2. **Page Transcription (`app/vision.py`)**: The vision model transcribes each page image into structured GitHub-Flavored Markdown. The system prompt directs the model to output headings, text styling, pipe tables, lists, inline math `$`, display math `$$`, and figure descriptions with bounding-box tags (`<!--box: x0,y0,x1,y1-->`).
3. **Figure Extraction (`app/figures.py`)**: Extracts the coordinate tags from the Markdown, maps the 0–1000 coordinate grid back to the original vector PDF page dimensions, clips the figures out at sharp DPI resolutions, and saves them. It updates the Markdown references from text captions to local images (`![caption](path.png)`).
4. **Markdown-to-Word Parsing (`app/docx_builder.py`)**: Parses the compiled Markdown. A custom state-machine compiler maps headers, code blocks, tables, lists, and blockquotes to Word elements, compiling math strings to OMML.

---

## 4. Engineering Strengths & Elegant Patterns

Several remarkable design patterns make **pdf2docx** robust and precise:

1. **Base-14 Metric Space Compensation (Font Drift Correction)**:
   Because target machines rarely have the exact fonts used in a PDF, substitute fonts (like Arial or Calibri) will have different widths, causing letters to shift horizontally. `app/docx_replica.py` measures characters using Adobe's base-14 metrics. It dynamically calculates the drift between the natural width of the substitute font and the exact width in the PDF, and injects fine-grained character spacing adjustments (`w:spacing` XML nodes) into the Word runs to ensure the text stays perfectly aligned.
2. **Adobe Symbol Private Use Area (PUA) Decoding**:
   Math and Symbol fonts in PDFs frequently output private-use Unicode codes (0xF000–0xF0FF) in the text layer, which are normally unprintable or missing. `app/symbols.py` maps these codes back to standard Unicode characters and switches the rendering font to *Cambria Math* (which ships with MS Office and has comprehensive glyph support), ensuring that symbols like $\theta$, $\pi$, and stacked brackets render flawlessly.
3. **Failure-Safe Mathematical Fallbacks**:
   Math transcriptions can sometimes cut off or fail. `app/latex_omml.py` uses defensive checks (`looks_incomplete`) to check for unclosed braces or trailing operators. If an equation appears truncated or fails to compile to OMML, the system falls back to displaying the high-resolution pixel-exact visual crop of the original PDF, ensuring no information is lost.
4. **Clean, Validated XML Generation**:
   Office Open XML (OOXML) is highly sensitive to schema ordering and illegal control characters. The system uses a strict XML sanitizer (`app/xml_text.py`) to scrub characters like surrogates or InDesign layout symbols. When creating VML shapes and textboxes, it ensures the precise sequence of elements required by Word (e.g., `<m:dPr>` tags like `begChr`, `endChr`, `grow`) is strictly preserved to prevent Word from discarding document styles.
5. **Structural Page-Stitching and Noise Filtering**:
   The reflowing engine (`reflow.py`) uses smart statistical heuristics to remove headers, footers, and page numbers, and links paragraphs split across pages by checking line-ending punctuation and lowercase letters on the subsequent page.

---

## 5. Potential Enhancements & Recommendations

While the codebase is exceptionally high-quality and robust, there are several areas where it can be enhanced:

### Architectural & Scaling Improvements
* **Distributed Task Queue Integration**:
  The job registry is currently managed in-memory with thread-locks, and background tasks are run in the web-server process using FastAPI's `BackgroundTasks`. For production or high-volume usage, migrating to a task queue (e.g., Celery, Dramatiq) with a Redis or PostgreSQL broker would prevent job losses during restarts and allow scaling workers across multiple servers.
* **Persistent Object Storage Support**:
  Conversions currently write local files to a folder specified by `PDF2DOCX_DATA_DIR` (defaulting to `~/.pdf2docx`). Adding an abstraction layer for storage would allow storing PDFs, page images, and finished `.docx` files in cloud object stores (like AWS S3 or MinIO).

### Algorithmic & Usability Enhancements
* **OCR Support for Scanned Pages**:
  Currently, replica mode renders scanned pages as giant images with simple unaligned plain text placed underneath. Integrating local OCR engines (such as Tesseract or EasyOCR) would allow extracting character-level bounding boxes for scanned documents, enabling precise line-by-line replica positioning even for non-digital scans.
* **Local / Offline LLM Integration**:
  The vision pipeline relies on OpenRouter APIs. Supporting local vision models (such as LLaVA or Qwen-VL) via a unified API client interface (like Ollama or LocalAI) would allow offline, private, and zero-cost conversions for sensitive documents.
* **Table Grid Re-construction**:
  Table detection currently falls back to absolute-positioned elements if the grid lines are not fully continuous. Implementing a heuristic grid-reconstruction algorithm (interpreting cell coordinates to build a virtual table grid) would increase the number of tables successfully rendered as native editable tables.
