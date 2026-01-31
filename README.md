# pypdftotext — Python PDF Text Extraction with OCR

[![PyPI version](https://badge.fury.io/py/pypdftotext.svg)](https://badge.fury.io/py/pypdftotext)
[![Python Support](https://img.shields.io/pypi/pyversions/pypdftotext)](https://pypi.org/project/pypdftotext/)
[![License: MIT](https://img.shields.io/badge/License-MIT-yellow.svg)](https://opensource.org/licenses/MIT)

**Extract text from PDFs in Python using pypdf and Azure Document Intelligence OCR. Handles embedded text and scanned PDFs with automatic OCR fallback, batch processing, and S3 support.**

pypdftotext is a Python library for **PDF text extraction** that uses pypdf's layout mode for embedded text and **Azure Document Intelligence** (Form Recognizer) for **OCR** when pages have little or no embedded text. Use it to **extract text from PDF files**, process **scanned PDFs**, run **batch OCR**, and read PDFs from **AWS S3**.

---

## Table of Contents

- [Features](#features)
- [Installation](#installation)
- [Quick Start](#quick-start)
- [Configuration](#configuration)
- [Batch Processing](#batch-processing)
- [S3 and Optional Features](#s3-and-optional-features)
- [Implementation Details](#implementation-details)
- [Author & Contact](#author--contact)
- [License](#license)
- [Links](#links)
- [Acknowledgments](#acknowledgments)

---

## Features

- **Embedded text extraction** — Fast extraction via pypdf layout mode
- **Automatic OCR fallback** — Azure Document Intelligence when embedded text is missing or sparse
- **Scanned PDF support** — OCR for image-based and scanned PDFs
- **Batch PDF processing** — Process multiple PDFs with parallel OCR
- **Thread-safe API** — Use `PdfExtract` in multi-threaded workflows
- **S3 support** — Read PDFs directly from AWS S3 URIs (`s3://bucket/key.pdf`)
- **Image compression** — Optional preprocessing to reduce file size and improve OCR
- **Handwritten text detection** — Confidence scoring for handwritten content
- **Page splitting & clipping** — Create child PDFs and extract page ranges
- **Flexible configuration** — Env vars, constants, and per-instance config with inheritance

---

## Installation

### Basic Installation

```bash
pip install pypdftotext
```

### Optional Dependencies

```bash
# S3 support (read PDFs from AWS S3)
pip install "pypdftotext[s3]"

# Image compression for scanned PDFs
pip install "pypdftotext[image]"

# All optional features (S3 + image)
pip install "pypdftotext[full]"

# Development (full + type stubs, pytest, coverage)
pip install "pypdftotext[dev]"
```

### Requirements

- **Python** 3.10, 3.11, or 3.12
- **pypdf** 6.0
- **azure-ai-documentintelligence** ≥ 1.0.0
- **tqdm** (progress bars)
- **boto3** (optional, for S3)
- **pillow** (optional, for image compression)

---

## Quick Start

### Enable Azure OCR (optional)

Without Azure OCR configured, pypdftotext returns only **embedded text** from the PDF (via [pypdf layout mode](https://pypdf.readthedocs.io/en/stable/user/extract-text.html)). To support **scanned PDFs** and image-based pages, set up Azure Document Intelligence.

**Prerequisites:**

- [Azure subscription](https://azure.microsoft.com/free/cognitive-services/)
- [Azure Document Intelligence (Form Recognizer) resource](https://portal.azure.com/#create/Microsoft.CognitiveServicesFormRecognizer)

**Configuration:** Set endpoint and key via environment variables or the `constants` module (same pattern applies to AWS credentials for S3).

```bash
export AZURE_DOCINTEL_ENDPOINT="https://your-resource.cognitiveservices.azure.com/"
export AZURE_DOCINTEL_SUBSCRIPTION_KEY="your-subscription-key"
```

```python
from pypdftotext import constants
constants.AZURE_DOCINTEL_ENDPOINT = "https://your-resource.cognitiveservices.azure.com/"
constants.AZURE_DOCINTEL_SUBSCRIPTION_KEY = "your-subscription-key"
```

You can also set these (and other options) on each `PdfExtract` instance via its `config` attribute. See [Configuration](#configuration).

### Basic Usage

**Create an extractor and get text:**

```python
from pypdftotext import PdfExtract

extract = PdfExtract("document.pdf")

# Full text
text = extract.text
print(text)

# Per-page text
for i, page_text in enumerate(extract.text_pages):
    print(f"Page {i + 1}: {page_text[:100]}...")
```

**Optional: customize config per instance**

```python
extract.config.AZURE_DOCINTEL_ENDPOINT = "https://your-resource.cognitiveservices.azure.com/"
extract.config.AZURE_DOCINTEL_SUBSCRIPTION_KEY = "your-subscription-key"
extract.config.PRESERVE_VERTICAL_WHITESPACE = True
```

**Compress images in scanned PDFs** (requires `pypdftotext[image]`). Do this *before* accessing `text` or `text_pages` so OCR uses the compressed PDF.

```python
extract.compress_images(
    white_point=200,       # Remove scanner artifacts (pixels 201–255 → white)
    aspect_tolerance=0.01,
    max_overscale=1.5,
)
```

**Save corrected/compressed PDF:**

```python
from pathlib import Path
Path("compressed_corrected_document.pdf").write_bytes(extract.body)
```

**Split and clip pages:**

```python
# First 10 pages as new PdfExtract (keeps config/metadata)
extract_child = extract.child((0, 9))

# PDF bytes for pages 1, 3, 5 (0-indexed: 0, 2, 4)
clipped_bytes = extract_child.clip_pages([0, 2, 4])
```

---

## Configuration

`PyPdfToTextConfig` and `PyPdfToTextConfigOverrides` control behavior. New configs:

1. Load from **environment variables**, then
2. Inherit from the global **`constants`** (unless disabled),
3. Optionally use a custom **`base`** config,
4. Apply **`overrides`** (overrides win over base/constants).

Disable inheritance from `constants`:

```python
constants.INHERIT_CONSTANTS = False
# or for one config:
from pypdftotext import PyPdfToTextConfig
config = PyPdfToTextConfig(overrides={"INHERIT_CONSTANTS": False})
```

**OCR triggering:** OCR runs when the share of “low-text” pages ≥ `TRIGGER_OCR_PAGE_RATIO` (default 0.99). A page is low-text if it has ≤ `MIN_LINES_OCR_TRIGGER` lines (default 1).

Example — trigger OCR when 50% of pages have fewer than 5 lines:

```python
from pypdftotext import PyPdfToTextConfig

config = PyPdfToTextConfig(
    MIN_LINES_OCR_TRIGGER=5,
    TRIGGER_OCR_PAGE_RATIO=0.5,
)
extract = PdfExtract("document.pdf", config=config)
```

---

## Batch Processing

Process multiple PDFs with parallel OCR:

```python
from pypdftotext.batch import PdfExtractBatch

pdfs = ["file1.pdf", "file2.pdf", "file3.pdf"]
# or by name: {"report": "report.pdf", "invoice": "invoice.pdf"}

batch = PdfExtractBatch(pdfs)
results = batch.extract_all()  # dict[str, PdfExtract]

for name, pdf_extract in results.items():
    print(f"{name}: {len(pdf_extract.text)} characters")
```

Embedded text is extracted first; OCR runs in parallel for PDFs that need it.

---

## S3 and Optional Features

**S3:** Pass an S3 URI as the `pdf` argument (e.g. `s3://bucket/path/file.pdf`). Configure AWS credentials via env vars or programmatically (same style as Azure above). Requires `pypdftotext[s3]`.

**Image compression:** Use `extract.compress_images(...)` before reading text when you need smaller files or better OCR on scanned PDFs. Requires `pypdftotext[image]`.

---

## Implementation Details

- **Page indices** are 0-based.
- **OCR** is triggered by the ratio of low-text pages and line-count threshold (see [Configuration](#configuration)).
- **Corruption detection:** Pages over 25,000 characters are treated as corrupted and return empty text.
- **Progress:** tqdm is used for progress bars; disable or position via config for scripts/logging.

---

## Author & Contact

**KuchikiRenji**

- **GitHub:** [github.com/KuchikiRenji](https://github.com/KuchikiRenji)
- **Email:** [KuchikiRenji@outlook.com](mailto:KuchikiRenji@outlook.com)
- **Discord:** `kuchiki_renji`

---

## License

This project is licensed under the **MIT License** — see the [LICENSE](LICENSE) file for details.

---

## Links

- [GitHub Repository](https://github.com/KuchikiRenji/pypdftotext)
- [Issue Tracker](https://github.com/KuchikiRenji/pypdftotext/issues)
- [PyPI Package](https://pypi.org/project/pypdftotext/)

---

## Acknowledgments

Built on:

- [pypdf](https://github.com/py-pdf/pypdf) — PDF parsing and layout text extraction
- [Azure Document Intelligence](https://azure.microsoft.com/en-us/services/cognitive-services/form-recognizer/) — OCR and document understanding
