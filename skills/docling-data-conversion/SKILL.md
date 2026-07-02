---
name: docling-data-conversion
description: >
  Use this skill whenever the user wants to parse, convert, extract, chunk, or analyse documents
  using Docling. Triggers include: converting PDF, DOCX, PPTX, XLSX, HTML, EPUB, or images to
  Markdown, JSON, HTML, or plain text; extracting tables or figures from documents; chunking
  documents for RAG pipelines; analysing document structure (headings, page counts, table counts);
  running OCR on scanned PDFs or images; or using a VLM pipeline for complex layouts, handwriting,
  or formulas. Also triggers on phrases like "parse this PDF", "extract tables from", "chunk for
  RAG", "convert to markdown", "docling", or "document intelligence". Do NOT use for non-document
  data tasks (CSV analysis, database queries) or for creating/editing documents (use the docx/pptx
  skills instead).
script: scripts/process_data.py
reference: https://docling-project.github.io/docling/
---

# Docling Data Conversion Skill

---

## Scope

| Task | Covered |
|---|---|
| Parse PDF, DOCX, PPTX, XLSX, HTML, EPUB, images | ✅ CLI + Python API |
| Convert to Markdown | ✅ CLI + Python API |
| Export as structured JSON (DoclingDocument) | ✅ CLI + Python API |
| OCR for scanned PDFs and images | ✅ auto-enabled |
| Table extraction to DataFrame / Markdown | ✅ Python API |
| Figure / picture export | ✅ Python API |
| Chunk for RAG (hybrid: heading + token) | ✅ Python API only |
| Analyze document structure (headings, tables, figures) | ✅ Python API only |
| Batch / multi-source conversion | ✅ CLI + Python API |
| VLM pipeline (complex layouts, handwriting, formulas) | ✅ CLI flag + Python API |
| Audio/video transcription (ASR) | ✅ requires `asr` extra + ffmpeg |

---

## Installation

```bash
pip install docling docling-core

# For OpenAI tokenizer chunking:
pip install 'docling-core[chunking-openai]'

# For audio/video transcription:
pip install 'docling[asr]'
# Also requires ffmpeg for video: brew install ffmpeg / apt install ffmpeg
```

Verify installed versions (prefer importlib — docling may not set `__version__`):

```python
from importlib.metadata import version
print(version("docling"), version("docling-core"))
```

---

## Supported Input Formats

| Format | Notes |
|---|---|
| PDF | Born-digital and scanned; advanced layout, table, formula understanding |
| DOCX, XLSX, PPTX | MS Office Open XML formats |
| HTML, XHTML | Web pages |
| EPUB | E-books |
| Markdown, AsciiDoc, LaTeX | Markup / scientific formats |
| CSV | Tabular data |
| PNG, JPEG, TIFF, BMP, WEBP | Images (OCR applied) |
| WAV, MP3, M4A, AAC, OGG, FLAC | Audio — requires `asr` extra |
| MP4, AVI, MOV | Video — audio track extracted; requires `asr` + ffmpeg |
| WebVTT | Timed text tracks |
| DocLang XML, USPTO XML, JATS XML, XBRL XML | Schema-specific XML formats |
| Docling JSON | Re-ingest a previously exported DoclingDocument |

## Supported Output Formats

`md` (Markdown) · `json` (DoclingDocument lossless) · `html` · `txt` (plain text) · `doctags` · `doclang` · `vtt`

---

## Step-by-Step Instructions

### 1. Resolve the Input

Determine whether the user supplied a **local path** or a **URL** — the CLI and Python API both accept either directly.

```bash
docling path/to/file.pdf
docling https://example.com/report.pdf
```

### 2. Choose a Pipeline

| Pipeline | CLI flag | Best for | Trade-off |
|---|---|---|---|
| **Standard** (default) | `--pipeline standard` | Born-digital PDFs, speed | No GPU needed; auto-OCR for scanned pages |
| **VLM** | `--pipeline vlm` | Complex layouts, multi-column, handwriting, formulas | Needs GPU; slower; best fidelity |

**Decision rules:**
- Born-digital PDF with clear layout → Standard
- Scanned / image-only PDF → Standard with OCR (auto), or VLM for best quality
- Multi-column, complex layout, handwriting, formulas → VLM
- 500+ pages, speed is priority → Standard + `--no-tables` if tables not needed
- Apple Silicon → `--vlm-model granite_docling` uses MLX backend automatically

### 3. Convert the Document

#### CLI (preferred for straightforward conversions)

```bash
# Markdown (default)
docling report.pdf --output /tmp/

# JSON (structured, lossless)
docling report.pdf --to json --output /tmp/

# HTML output
docling report.pdf --to html --output /tmp/

# Plain text
docling report.pdf --to txt --output /tmp/

# VLM pipeline (best quality, needs GPU)
docling report.pdf --pipeline vlm --output /tmp/

# VLM with specific model
docling report.pdf --pipeline vlm --vlm-model granite_docling --output /tmp/

# Custom OCR engine
docling report.pdf --ocr-engine tesserocr --output /tmp/
# OCR engine options: easyocr (default), tesserocr, rapidocr, mac_native (Apple Silicon)

# Speed optimisations
docling report.pdf --no-ocr --output /tmp/     # skip OCR (born-digital only)
docling report.pdf --no-tables --output /tmp/  # skip table structure detection

# Password-protected PDF
docling report.pdf --pdf-password SECRET --output /tmp/

# Remote VLM services (vLLM / Ollama / LM Studio)
docling report.pdf --pipeline vlm --enable-remote-services --output /tmp/

# Batch: multiple files
docling file1.pdf file2.docx https://example.com/doc.pdf --output /tmp/
```

Output files are written to `--output` dir, named after the input (e.g. `report.pdf` → `report.md`).

**Full CLI reference:** https://docling-project.github.io/docling/reference/cli/

#### Python API (for advanced features)

Use when you need: chunking, remote VLM endpoint config, hybrid `force_backend_text` mode, or programmatic access to the `DoclingDocument` object.

**⚠️ API note (Docling 2.81+):** `DocumentConverter(format_options=...)` requires `dict[InputFormat, FormatOption]` — use `InputFormat.PDF` enum keys, not string keys like `"pdf"`. String keys cause `AttributeError: 'PdfPipelineOptions' object has no attribute 'backend'` at runtime.

**Standard pipeline:**

```python
from docling.document_converter import DocumentConverter, PdfFormatOption
from docling.datamodel.base_models import InputFormat
from docling.datamodel.pipeline_options import PdfPipelineOptions

# Minimal (uses all defaults — auto-OCR, table detection enabled)
converter = DocumentConverter()
result = converter.convert("report.pdf")

# With explicit options
converter = DocumentConverter(
    format_options={
        InputFormat.PDF: PdfFormatOption(
            pipeline_options=PdfPipelineOptions(
                do_ocr=True,
                do_table_structure=True,
            ),
        ),
    }
)
result = converter.convert("report.pdf")
```

**VLM pipeline — local (GraniteDocling via HuggingFace Transformers):**

```python
from docling.document_converter import DocumentConverter, PdfFormatOption
from docling.datamodel.base_models import InputFormat
from docling.datamodel.pipeline_options import VlmPipelineOptions
from docling.datamodel import vlm_model_specs
from docling.pipeline.vlm_pipeline import VlmPipeline

pipeline_options = VlmPipelineOptions(
    vlm_options=vlm_model_specs.GRANITEDOCLING_TRANSFORMERS,
    generate_page_images=True,
)
converter = DocumentConverter(
    format_options={
        InputFormat.PDF: PdfFormatOption(
            pipeline_cls=VlmPipeline,
            pipeline_options=pipeline_options,
        )
    }
)
result = converter.convert("report.pdf")
```

**VLM pipeline — remote API (vLLM / LM Studio / Ollama):**

```python
from docling.document_converter import DocumentConverter, PdfFormatOption
from docling.datamodel.base_models import InputFormat
from docling.datamodel.pipeline_options import VlmPipelineOptions
from docling.datamodel.pipeline_options_vlm_model import ApiVlmOptions, ResponseFormat
from docling.pipeline.vlm_pipeline import VlmPipeline

vlm_opts = ApiVlmOptions(
    url="http://localhost:8000/v1/chat/completions",
    params=dict(model="ibm-granite/granite-docling-258M", max_tokens=4096),
    prompt="Convert this page to docling.",
    response_format=ResponseFormat.DOCTAGS,
    timeout=120,
)
pipeline_options = VlmPipelineOptions(
    vlm_options=vlm_opts,
    generate_page_images=True,
    enable_remote_services=True,   # required — gates all outbound HTTP
)
converter = DocumentConverter(
    format_options={
        InputFormat.PDF: PdfFormatOption(
            pipeline_cls=VlmPipeline,
            pipeline_options=pipeline_options,
        )
    }
)
result = converter.convert("report.pdf")
```

**Hybrid mode (`force_backend_text`) — Python API only:**

Uses deterministic PDF text extraction for text regions, VLM for images and tables. Reduces hallucination on text-heavy pages while retaining VLM quality for visual elements.

```python
pipeline_options = VlmPipelineOptions(
    vlm_options=vlm_model_specs.GRANITEDOCLING_TRANSFORMERS,
    force_backend_text=True,
    generate_page_images=True,
)
```

`result.document` is a `DoclingDocument` in all cases.

### 4. Export Output

```python
doc = result.document

# Markdown
md_text = doc.export_to_markdown()

# JSON (lossless)
import json
json_str = json.dumps(doc.export_to_dict(), indent=2)

# HTML
html_str = doc.export_to_html()

# Plain text
txt = doc.export_to_text()

# Save to file
with open("output.md", "w") as f:
    f.write(md_text)
```

If the user does not specify a format, ask: *"Should I output Markdown or structured JSON (DoclingDocument)?"*

### 5. Extract Tables

```python
for i, table in enumerate(doc.tables):
    print(f"=== Table {i} ===")
    df = table.export_to_dataframe()   # pandas DataFrame
    print(df)
    print(table.export_to_markdown())  # Markdown table string
```

For CLI table export to CSV (one file per table):

```bash
docling report.pdf --to json --output /tmp/
# Then process tables from the JSON DoclingDocument programmatically
```

### 6. Extract Figures / Pictures

```python
from pathlib import Path

output_dir = Path("/tmp/figures")
output_dir.mkdir(exist_ok=True)

for i, picture in enumerate(doc.pictures):
    caption = picture.caption_text(doc)
    print(f"Figure {i}: {caption}")
    # Save image bytes if available
    if picture.image and picture.image.pil_image:
        picture.image.pil_image.save(output_dir / f"figure_{i}.png")
```

### 7. Analyze Document Structure

```python
doc = result.document

# Heading tree
for item, level in doc.iterate_items():
    if hasattr(item, 'label') and item.label.name == 'SECTION_HEADER':
        print(f"{'#' * level} {item.text}")

# Summary counts
print(f"Pages:   {len(doc.pages)}")
print(f"Tables:  {len(doc.tables)}")
print(f"Figures: {len(doc.pictures)}")

# All text items
for item, level in doc.iterate_items():
    if hasattr(item, 'text'):
        print(item.text)
```

### 8. Chunk for RAG

Chunking is only available via the Python API. The default **hybrid chunker** splits first by heading hierarchy, then subdivides oversized sections by token count — preserving semantic boundaries while respecting model context limits.

**⚠️ API change (docling-core 2.8.0+):** Pass a `BaseTokenizer` object, not a raw string, to `HybridChunker`.

**HuggingFace tokenizer (sentence-transformers / embedding models):**

```python
from docling.chunking import HybridChunker
from docling_core.transforms.chunker.tokenizer.huggingface import HuggingFaceTokenizer

tokenizer = HuggingFaceTokenizer.from_pretrained(
    model_name="sentence-transformers/all-MiniLM-L6-v2",
    max_tokens=512,
)
chunker = HybridChunker(tokenizer=tokenizer, merge_peers=True)
chunks = list(chunker.chunk(doc))

for chunk in chunks:
    embed_text = chunker.contextualize(chunk)  # heading breadcrumb prepended
    print(chunk.meta.headings)        # list of parent headings
    print(chunk.meta.origin.page_no)  # source page
    print(embed_text[:200])
```

**OpenAI tokenizer (for OpenAI embedding models):**

```python
import tiktoken
from docling_core.transforms.chunker.tokenizer.openai import OpenAITokenizer

tokenizer = OpenAITokenizer(
    tokenizer=tiktoken.encoding_for_model("text-embedding-3-small"),
    max_tokens=8192,
)
chunker = HybridChunker(tokenizer=tokenizer)
# Requires: pip install 'docling-core[chunking-openai]'
```

### 9. Batch Conversion

**CLI:**
```bash
docling file1.pdf file2.docx file3.html --output /tmp/batch/
```

**Python API:**
```python
from docling.document_converter import DocumentConverter
from pathlib import Path

converter = DocumentConverter()
sources = [
    "report.pdf",
    "presentation.pptx",
    "https://example.com/article.html",
]

results = converter.convert_all(sources)
for result in results:
    out_path = Path("/tmp") / (result.input.file.stem + ".md")
    out_path.write_text(result.document.export_to_markdown())
    print(f"Converted: {result.input.file.name} → {out_path}")
```

---

## Common Edge Cases

| Situation | Handling |
|---|---|
| Scanned / image-only PDF | Standard + auto-OCR, or `--pipeline vlm` for best quality |
| Password-protected PDF | `--pdf-password PASSWORD`; raises `ConversionError` if wrong |
| 500+ page document | Standard + `--no-tables` for speed |
| Complex layout / multi-column | `--pipeline vlm`; standard may misorder reading flow |
| Handwriting or formulas | `--pipeline vlm` only — standard OCR won't handle these |
| URL behind auth | Pre-download to temp file; pass local path |
| Tables with merged cells | `table.export_to_markdown()` handles spans; VLM often more accurate |
| Non-UTF-8 encoding | Docling normalises internally; no special handling needed |
| VLM hallucinating text | `force_backend_text=True` via Python API for hybrid mode |
| VLM API call blocked | `--enable-remote-services` (CLI) or `enable_remote_services=True` (Python) |
| Apple Silicon | `--vlm-model granite_docling` automatically uses MLX backend |
| `\ufffd` replacement characters | Try `--ocr-engine tesserocr` or `--pipeline vlm` |
| Repeated lines in output | `--pipeline vlm` or `force_backend_text=True` (Python API) |
| Near-empty Markdown output | Enable OCR or switch to VLM pipeline |

---

## Quality Evaluation Loop

After any conversion where fidelity matters, run an evaluate → refine loop (max 3 iterations):

**Step A — Produce JSON + Markdown:**
```bash
docling "<source>" --to json --output /tmp/
docling "<source>" --to md   --output /tmp/
```

**Step B — Manual quality checklist (if no automated evaluator):**

| Check | Action if bad |
|---|---|
| Page count matches source (roughly) | Re-run; try `--pipeline vlm` for complex layout |
| Markdown is not near-empty | Enable OCR or switch to VLM |
| Tables missing when visually present | Remove `--no-tables`; try `--pipeline vlm` |
| `\ufffd` replacement characters | Change `--ocr-engine` or `--pipeline vlm` |
| Same line repeated many times | `--pipeline vlm` or `force_backend_text=True` |
| Figures missing or corrupt | `--pipeline vlm`; check `doc.pictures` count |

**Step C — Refinement (apply ONE change per iteration):**
1. Apply the primary `recommended_action` (e.g. switch pipeline, change OCR engine).
2. Re-convert and re-evaluate.
3. Stop when output passes all checks, or after 3 iterations — then summarise what worked and any remaining issues.

---

## Output Conventions

- Always report the number of pages and conversion status.
- For Markdown output: render directly unless the user needs to copy/paste it — only then wrap in a fenced code block.
- For JSON output: pretty-print with `indent=2` unless the user says otherwise.
- For chunks: report total count, min/max/avg token counts.
- For structure analysis: summarise heading tree + table count + figure count before detailing content.
- For batch: report per-file status (success / failure) and any `ConversionError` messages.

---

## Dependencies Reference

```
pip install docling docling-core
pip install 'docling-core[chunking-openai]'   # OpenAI tokenizer
pip install 'docling[asr]'                     # Audio/video transcription
# ffmpeg required for video: brew install ffmpeg / apt install ffmpeg
```

The `docling` CLI is bundled with the `docling` package — no separate install needed.

**Docs:** https://docling-project.github.io/docling/
**CLI reference:** https://docling-project.github.io/docling/reference/cli/
**Python API:** https://docling-project.github.io/docling/reference/document_converter/
**Supported formats:** https://docling-project.github.io/docling/usage/supported_formats/