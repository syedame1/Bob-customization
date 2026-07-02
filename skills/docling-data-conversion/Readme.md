# docling-data-conversion skill

> Parse, convert, extract, and chunk documents into structured, machine-readable formats using [Docling](https://docling-project.github.io/docling/).

---

## Overview

The `docling-data-conversion` skill gives a coding agent the ability to parse and convert documents into structured, machine-readable formats using the open-source Docling library. It handles PDFs (born-digital and scanned), Word, PowerPoint, HTML, EPUB, and images — outputting clean Markdown, JSON, or plain text. Beyond conversion, it extracts tables as CSVs, pulls out figures as PNGs, and chunks documents into RAG-ready JSONL. Everything is wired into a single `process_data.py` script with clear sub-commands, so the agent always knows exactly what to run.

---

## Folder Structure

```
docling-data-conversion/
├── README.md           ← You are here
├── SKILL.md            ← Agent instructions and decision rules
└── scripts/
    └── process_data.py ← Executable script (all sub-commands)
```

---

## Requirements

```bash
pip install docling docling-core pandas

# Optional — OpenAI tokenizer for chunking
pip install 'docling-core[chunking-openai]'

# Optional — audio/video transcription
pip install 'docling[asr]'
# Video also requires ffmpeg: brew install ffmpeg / apt install ffmpeg
```

---

## Quick Start

```bash
# Convert a PDF to Markdown
python scripts/process_data.py convert report.pdf

# Convert a URL to JSON
python scripts/process_data.py convert https://arxiv.org/pdf/2408.09869 --to json --output ./out

# Extract all tables as CSV
python scripts/process_data.py tables report.pdf --output ./out/tables

# Extract all figures as PNG
python scripts/process_data.py figures report.pdf --output ./out/figures

# Print heading tree + page/table/figure counts
python scripts/process_data.py structure report.pdf

# Chunk for RAG (HuggingFace tokenizer, 512 tokens)
python scripts/process_data.py chunk report.pdf --tokenizer hf --max-tokens 512 --output ./out/chunks

# Convert using VLM pipeline (complex layouts, scanned docs)
python scripts/process_data.py vlm report.pdf --output ./out
```

---

## Sub-commands

| Command | What it does | Key flags |
|---|---|---|
| `convert` | Convert file(s) or URL(s) → md / json / html / txt | `--to`, `--no-ocr`, `--no-tables`, `--password` |
| `tables` | Extract all tables → CSV + Markdown per table | `--output` |
| `figures` | Extract all embedded images → PNG files | `--output` |
| `structure` | Print heading tree, page/table/figure counts | `--text` |
| `chunk` | Hybrid RAG chunking → JSONL | `--tokenizer`, `--model`, `--max-tokens`, `--output` |
| `vlm` | VLM pipeline (local HF model or remote API) | `--remote-url`, `--remote-model`, `--timeout` |

---

## Supported Input Formats

| Format | Notes |
|---|---|
| PDF | Born-digital and scanned; auto-OCR enabled |
| DOCX, XLSX, PPTX | MS Office Open XML |
| HTML, EPUB | Web pages and e-books |
| PNG, JPEG, TIFF, BMP, WEBP | Images — OCR applied |
| Markdown, LaTeX, AsciiDoc | Markup and scientific formats |
| WAV, MP3, MP4, MOV | Audio/video — requires `asr` extra |

---

## Pipelines

| Pipeline | Best for | Requirement |
|---|---|---|
| **Standard** (default) | Born-digital PDFs, speed, large batches | CPU only |
| **VLM** | Complex layouts, multi-column, handwriting, formulas | GPU recommended |

Switch to VLM with `--remote-url` for remote inference via vLLM, Ollama, or LM Studio.

---

## Test Prompt

Use the Docling paper itself as a first smoke test — it exercises headings, tables, figures, and multi-column layout:

```
Convert https://arxiv.org/pdf/2408.09869 to markdown using the docling skill,
then show me the heading tree and how many tables and figures it found.
```

---

## Common Edge Cases

| Situation | Fix |
|---|---|
| Near-empty Markdown output | Enable OCR or switch to VLM pipeline |
| Scanned / image-only PDF | Standard pipeline auto-applies OCR; use VLM for best quality |
| Complex / multi-column layout | Use `vlm` sub-command |
| Password-protected PDF | Pass `--password SECRET` |
| `\ufffd` replacement characters | Try `--ocr-engine tesserocr` or VLM pipeline |
| VLM hallucinating text | Use `force_backend_text=True` in Python API (hybrid mode) |
| Remote VLM call blocked | Pass `--remote-url` — `enable_remote_services` is set automatically |

---

## Links

- [Docling Docs](https://docling-project.github.io/docling/)
- [CLI Reference](https://docling-project.github.io/docling/reference/cli/)
- [Python API Reference](https://docling-project.github.io/docling/reference/document_converter/)
- [Supported Formats](https://docling-project.github.io/docling/usage/supported_formats/)
