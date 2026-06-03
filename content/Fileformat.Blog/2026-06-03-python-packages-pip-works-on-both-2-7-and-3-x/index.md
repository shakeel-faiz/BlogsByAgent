---
seoTitle: "Python packages (pip works on both 2.7 and 3.x)"
title: "Python packages (pip works on both 2.7 and 3.x)"
description: "Learn to convert, edit, compress, and secure PDFs in Python using pure‑Python libraries and CLI tools—step‑by‑step code for 2.7 & 3.x."
date: 03 Jun 2026
draft: true
author: Khan AI
url: /audio/python-packages-pip-works-on-both-2-7-and-3-x/
categories: ['Audio']
tags: ['Python packages (pip works on both 2.7 and 3.x)', 'MP4', 'Some Tag']
---

**TL;DR** – In 2016 the Python PDF ecosystem is a mix of pure‑Python libraries (PDFMiner, PyPDF2, pdfrw, ReportLab) and battle‑tested command‑line tools (Ghostscript, pdftk, qpdf). Use the pure‑Python libs for quick splits, merges, text extraction, and simple encryption. Call the external binaries when you need heavy‑weight compression, linearization, PDF/A compliance, or AES‑256 security. Below is a practical, end‑to‑end walkthrough that shows how to **convert**, **edit**, **compress**, and **secure** PDFs on both Python 2.7 and 3.3‑3.5, with enough detail to adapt the code to your own project.

---

## 1. The PDF Landscape in 2016 – What’s Available and When to Use It  

| Task | Recommended Python lib / CLI tool | Why it fits the job |
|------|-----------------------------------|----------------------|
| **Read / parse text** | `pdfminer.six` (PDFMiner) | Pure‑Python, layout‑aware (`LAParams`) – great for column‑preserving extraction. |
| **Low‑level object tweaks, merging** | `pdfrw` or `PyPDF2` | Direct access to page dictionaries, easy to stitch PDFs together. |
| **Create PDFs from scratch** | `ReportLab` | Full‑featured drawing engine, supports PDF/A, built‑in encryption. |
| **PDF → image (PNG/JPEG)** | `pdf2image` + system **Poppler** (`pdftoppm`) | Fast rasterisation, works on Windows/macOS/Linux. |
| **Heavy compression / PDF/A / linearization** | **Ghostscript** (`gs`) | `-dPDFSETTINGS` presets down‑sample images, strip unused objects, produce PDF/A‑1b. |
| **Fast merging / stamping / incremental updates** | **pdftk** | Mature CLI, handles password‑protected files, adds watermarks without full rewrite. |
| **Object‑stream compression & linearization** | **qpdf** (or its Python wrapper **pikepdf**) | Modern PDF 1.5+ features, AES‑256 encryption (future‑proof). |
| **Cloud fallback** | Any REST PDF service (e.g., PDF.co) | When you can’t install native binaries on the host. |

> **Rule of thumb:** start with a pure‑Python solution. If you hit performance or feature limits, drop down to the appropriate CLI tool via `subprocess`.

---

## 2. Getting Your Toolbox Ready  

```bash
# Python packages (pip works on both 2.7 and 3.x)
pip install PyPDF2 pdfrw pdfminer.six reportlab pdf2image pillow

# System binaries – adjust for your OS
# macOS (Homebrew)
brew install ghostscript poppler qpdf pdftk

# Ubuntu/Debian
sudo apt-get install ghostscript poppler-utils qpdf pdftk
```

> **Tip:** When you ship a Docker image, install the binaries in the same layer as your `requirements.txt` so the container is reproducible.

```dockerfile
FROM python:3.5-slim
RUN apt-get update && apt-get install -y \
    ghostscript \
    poppler-utils \
    qpdf \
    pdftk \
 && rm -rf /var/lib/apt/lists/*
COPY requirements.txt .
RUN pip install -r requirements.txt
WORKDIR /app
COPY . .
CMD ["python", "main.py"]
```

---

## 3. Converting PDFs – Text, Images, and HTML  

### 3.1 Extracting Structured Text with PDFMiner  

```python
# works on both Python 2 and 3
from pdfminer.high_level import extract_text
from pdfminer.layout import LAParams

def pdf_to_text(path, out_path):
    laparams = LAParams(line_margin=0.5, char_margin=2.0, word_margin=0.1)
    text = extract_text(path, laparams=laparams)
    with open(out_path, 'w', encoding='utf-8') as f:
        f.write(text)

pdf_to_text('invoice.pdf', 'invoice.txt')
```

*Why `LAParams`?* It tells PDFMiner to respect column gaps and keep the visual flow, which is essential for invoices or academic papers.

### 3.2 Rasterising Pages with pdf2image  

```python
from pdf2image import convert_from_path

def pdf_to_png(pdf_path, dpi=150):
    images = convert_from_path(pdf_path, dpi=dpi, fmt='png')
    for i, img in enumerate(images, start=1):
        img.save(f'page_{i:03d}.png', 'PNG')

pdf_to_png('report.pdf')
```

Behind the scenes `pdf2image` calls `pdftoppm` from Poppler. On Windows you may need to add the Poppler `bin` folder to `PATH`.

### 3.3 Quick HTML via `pdf2txt.py` (PDFMiner)  

```bash
pdf2txt.py -t html -o report.html report.pdf
```

The generated HTML preserves the original layout but is not suitable for editing; it’s handy for quick previews in a web UI.

---

## 4. Editing PDFs – Splitting, Merging, Watermarking, and Incremental Updates  

### 4.1 Splitting & Merging with PyPDF2  

```python
import PyPDF2

def split_pdf(src, pages_per_chunk=5):
    reader = PyPDF2.PdfFileReader(open(src, 'rb'))
    for i in range(0, reader.numPages, pages_per_chunk):
        writer = PyPDF2.PdfFileWriter()
        for p in range(i, min(i + pages_per_chunk, reader.numPages)):
            writer.addPage(reader.getPage(p))
        out_name = f'{src[:-4]}_part_{i//pages_per_chunk + 1}.pdf'
        with open(out_name, 'wb') as out_f:
            writer.write(out_f)

def merge_pdfs(pdf_list, out_path):
    writer = PyPDF2.PdfFileWriter()
    for pdf in pdf_list:
        reader = PyPDF2.PdfFileReader(open(pdf, 'rb'))
        for p in range(reader.numPages):
            writer.addPage(reader.getPage(p))
    with open(out_path, 'wb') as out_f:
        writer.write(out_f)

split_pdf('big_report.pdf')
merge_pdfs(['intro.pdf', 'chapter1.pdf', 'chapter2.pdf'], 'full_book.pdf')
```

*Error handling:* Catch `PyPDF2.utils.PdfReadError` for malformed source files and fall back to `pdftk` if needed.

### 4.2 Adding a Watermark (Incremental Update)  

```python
def add_watermark(src, watermark_pdf, out_path):
    base = PyPDF2.PdfFileReader(open(src, 'rb'))
    stamp = PyPDF2.PdfFileReader(open(watermark_pdf, 'rb')).getPage(0)

    writer = PyPDF2.PdfFileWriter()
    for i in range(base.numPages):
        page = base.getPage(i)
        page.mergePage(stamp)          # overlay watermark
        writer.addPage(page)

    # Incremental write – avoids re‑encoding existing streams
    with open(out_path, 'wb') as out_f:
        writer.write(out_f)

add_watermark('contract.pdf', 'confidential_stamp.pdf', 'contract_secured.pdf')
```

For very large PDFs you can also call `pdftk`:

```bash
pdftk contract.pdf background confidential_stamp.pdf output contract_secured.pdf
```

### 4.3 Manipulating Low‑Level Objects with pdfrw  

```python
from pdfrw import PdfReader, PdfWriter, PageMerge

def rotate_pages(src, degrees=90, out_path='rotated.pdf'):
    reader = PdfReader(src)
    for page in reader.pages:
        page.Rotate = (int(page.inheritable.Rotate or 0) + degrees) % 360
    PdfWriter(out_path, trailer=reader).write()

rotate_pages('presentation.pdf')
```

`pdfrw` works directly on the PDF object model (dictionaries, streams), which is handy when you need to edit XObjects or add custom metadata.

---

## 5. Compressing & Optimising PDFs  

### 5.1 Ghostscript “screen” vs “printer” Settings  

```python
import subprocess, shlex, os

def gs_compress(src, out, preset='screen'):
    """
    preset: screen (72 dpi), ebook (150 dpi), printer (300 dpi), prepress (300 dpi, color)
    """
    cmd = (
        f'gs -q -dNOPAUSE -dBATCH -sDEVICE=pdfwrite '
        f'-dPDFSETTINGS=/{preset} '
        f'-sOutputFile={shlex.quote(out)} {shlex.quote(src)}'

```python
    # Build the full command line
    cmd = (
        f'gs -q -dNOPAUSE -dBATCH -sDEVICE=pdfwrite '
        f'-dPDFSETTINGS=/{preset} '
        f'-dCompatibilityLevel=1.4 '
        f'-dDetectDuplicateImages=true '
        f'-dCompressFonts=true '
        f'-dSubsetFonts=true '
        f'-dColorImageDownsampleType=/Bicubic '
        f'-dGrayImageDownsampleType=/Bicubic '
        f'-dMonoImageDownsampleType=/Subsample '
        f'-sOutputFile={shlex.quote(out)} {shlex.quote(src)}'
    )
    subprocess.check_call(shlex.split(cmd))
    print(f'Compressed {src} → {out} using Ghostscript preset “{preset}”.')
```

**Usage example**

```python
gs_compress('original.pdf', 'original_screen.pdf', preset='screen')   # ~72 dpi, smallest size
gs_compress('original.pdf', 'original_ebook.pdf',  preset='ebook')    # ~150 dpi, good for e‑readers
gs_compress('original.pdf', 'original_printer.pdf', preset='printer')# ~300 dpi, print‑ready
```

Ghostscript’s `-dPDFSETTINGS` presets are a quick way to trade‑off quality vs. size. For fine‑grained control you can replace the preset with explicit down‑sampling parameters (`-dDownsampleColorImages=true -dColorImageResolution=150`, etc.).

---

### 5.2 Modern Object‑Stream Compression with **qpdf** (or **pikepdf**)

`qpdf` can rewrite a PDF using the newer object‑stream format introduced in PDF 1.5, which often yields a 10‑30 % size reduction on top of Ghostscript’s work.

```python
def qpdf_optimize(src, out, linearize=False):
    """
    Rewrites the PDF using object streams.
    If linearize=True, also produces a web‑optimized (fast‑web‑view) file.
    """
    cmd = ['qpdf', '--compress-streams=y', '--object-streams=generate']
    if linearize:
        cmd.append('--linearize')
    cmd.extend([src, out])
    subprocess.check_call(cmd)
    print(f'Optimized {src} → {out} (linearize={linearize}).')
```

**Example**

```python
qpdf_optimize('original_ebook.pdf', 'original_ebook_qpdf.pdf')
qpdf_optimize('original_ebook.pdf', 'original_ebook_qpdf_linear.pdf', linearize=True)
```

If you prefer a pure‑Python wrapper, install `pikepdf` (`pip install pikepdf`) and use the same options via its API:

```python
import pikepdf

def pikepdf_optimize(src, out, linearize=False):
    with pikepdf.open(src) as pdf:
        pdf.save(
            out,
            compression=pikepdf.CompressionLevel.compression_level_fast,
            object_streams=pikepdf.ObjectStreamMode.generate,
            linearize=linearize
        )
    print(f'pikepdf: {src} → {out} (linearize={linearize})')
```

---

### 5.3 Linearization (Fast‑Web‑View) with **pdftk**

While `qpdf` already supports linearization, many legacy pipelines still rely on `pdftk`:

```bash
pdftk original.pdf output linearized.pdf linearize
```

The command reads the entire file, rebuilds the cross‑reference table, and places the first page’s content at the beginning of the file, enabling browsers to start rendering before the download finishes.

---

## 6. Securing PDFs – Passwords, Permissions, and AES‑256  

### 6.1 Simple User/Owner Passwords with **PyPDF2**

```python
def encrypt_pdf(src, out, user_pwd='', owner_pwd='', use_128bit=True):
    reader = PyPDF2.PdfFileReader(open(src, 'rb'))
    writer = PyPDF2.PdfFileWriter()
    for p in range(reader.numPages):
        writer.addPage(reader.getPage(p))
    writer.encrypt(user_pwd, owner_pwd, use_128bit=use_128bit)
    with open(out, 'wb') as f:
        writer.write(f)
    print(f'Encrypted {src} → {out} (128‑bit={use_128bit}).')
```

*Limitations*: PyPDF2 only supports RC4‑based encryption (40‑bit or 128‑bit). It cannot produce AES‑256 PDFs, which are required for many compliance regimes (e.g., PDF/A‑2).

### 6.2 AES‑256 Encryption with **qpdf** (or **pikepdf**)

```python
def qpdf_aes256_encrypt(src, out, user_pwd, owner_pwd='', allow_extract=False):
    """
    Creates an AES‑256 encrypted PDF.
    allow_extract=False removes the “extract text/images” permission.
    """
    permissions = '--no-extract' if not allow_extract else ''
    cmd = [
        'qpdf',
        '--encrypt', user_pwd, owner_pwd,
        '256', '--', src, out,
        permissions,
        '--force-aes256'
    ]
    # Filter out empty strings (when permissions are omitted)
    cmd = [c for c in cmd if c]
    subprocess.check_call(cmd)
    print(f'AES‑256 encrypted {src} → {out}.')
```

**Using pikepdf (pure‑Python)**

```python
def pikepdf_aes256_encrypt(src, out, user_pwd, owner_pwd='', allow_extract=False):
    permissions = pikepdf.Permissions(
        extract=False if not allow_extract else True,
        print=False,
        modify=False,
        annotate=False,
        fill_forms=False,
        accessibility=False,
        assemble=False,
        high_quality_print=False
    )
    with pikepdf.open(src) as pdf:
        pdf.save(
            out,
            encryption=pikepdf.Encryption(
                user=user_pwd,
                owner=owner_pwd,
                R=6,                     # Revision 6 → AES‑256
                P=permissions,
                encrypt_metadata=True
            )
        )
    print(f'pikepdf AES‑256 encrypted {src} → {out}.')
```

### 6.3 Removing All Security (Decrypting)  

When you need to strip passwords (e.g., for downstream processing), use `qpdf`:

```bash
qpdf --password=USERPWD --decrypt secured.pdf decrypted.pdf
```

Or with `pikepdf`:

```python
with pikepdf.open('secured.pdf', password='USERPWD') as pdf:
    pdf.save('decrypted.pdf')
```

> **Security note:** Never hard‑code passwords in source control. Load them from environment variables or a secrets manager.

---

## 7. Putting It All Together – A Real‑World End‑to‑End Pipeline  

Below is a **single script** that demonstrates a typical workflow:

1. **Extract** text for indexing.  
2. **Split** a large report into per‑chapter PDFs.  
3. **Watermark** each chapter with a “Confidential” stamp.  
4. **Compress** for web delivery (screen‑preset + qpdf object streams).  
5. **Encrypt** with AES‑256, disabling extraction.  

```python
#!/usr/bin/env python
# -*- coding: utf-8 -*-

import os
import sys
import glob
from pathlib import Path

# Local imports – assume the helper functions from sections 3‑6 are in utils.py
from utils import (
    pdf_to_text, split_pdf, add_watermark,
    gs_compress, qpdf_optimize,
    qpdf_aes256_encrypt
)

# ----------------------------------------------------------------------
# Configuration (could be loaded from a JSON/YAML file)
# ----------------------------------------------------------------------
SOURCE_PDF   = 'annual_report_2025.pdf'
OUTPUT_DIR   = Path('output')
WATERMARK    = 'confidential_stamp.pdf'
TEXT_OUT    = OUTPUT_DIR / 'report.txt'
CHUNK_SIZE   = 30          # pages per chapter
GS_PRESET    = 'screen'    # smallest size for web
AES_USER_PWD = os.getenv('PDF_USER_PWD', 'viewer')
AES_OWNER_PWD= os.getenv('PDF_OWNER_PWD', 'admin')
ALLOW_EXTRACT= False

# Ensure output folder exists
OUTPUT_DIR.mkdir(parents=True, exist_ok=True)

# 1️⃣ Extract searchable text
print('→ Extracting text...')
pdf_to_text(SOURCE_PDF, TEXT_OUT)

# 2️⃣ Split into chapters
print('→ Splitting PDF...')
split_pdf(SOURCE_PDF, pages_per_chunk=CHUNK_SIZE)
chapter_files = sorted(glob

```python
chapter_files = sorted(glob.glob('annual_report_2025_part_*.pdf'))
```

### 3️⃣ Watermark each chapter  

```python
watermarked_chapters = []
for i, chap in enumerate(chapter_files, start=1):
    out_name = OUTPUT_DIR / f'chapter_{i:02d}_watermarked.pdf'
    add_watermark(chap, WATERMARK, str(out_name))
    watermarked_chapters.append(str(out_name))
    print(f'   • Watermarked {chap} → {out_name}')
```

### 4️⃣ Compress & optimise for web  

```python
compressed_chapters = []
for chap in watermarked_chapters:
    # 4a – Ghostscript screen‑preset compression
    gs_out = OUTPUT_DIR / f'{Path(chap).stem}_gs.pdf'
    gs_compress(chap, str(gs_out), preset=GS_PRESET)

    # 4b – qpdf object‑stream + linearisation (optional)
    qpdf_out = OUTPUT_DIR / f'{Path(chap).stem}_opt.pdf'
    qpdf_optimize(str(gs_out), str(qpdf_out), linearize=True)

    compressed_chapters.append(str(qpdf_out))
    print(f'   • Optimised {gs_out} → {qpdf_out}')
```

### 5️⃣ AES‑256 encrypt & lock down permissions  

```python
final_chapters = []
for chap in compressed_chapters:
    enc_out = OUTPUT_DIR / f'{Path(chap).stem}_enc.pdf'
    qpdf_aes256_encrypt(
        src=chap,
        out=str(enc_out),
        user_pwd=AES_USER_PWD,
        owner_pwd=AES_OWNER_PWD,
        allow_extract=ALLOW_EXTRACT
    )
    final_chapters.append(str(enc_out))
```

### 6️⃣ Merge the secured chapters into a single deliverable  

```python
final_bundle = OUTPUT_DIR / 'annual_report_2025_secure.pdf'
merge_pdfs(final_chapters, str(final_bundle))
print(f'\n✅ Pipeline complete – final secured PDF: {final_bundle}')
```

---

## 8. Production‑Ready Tips  

| Concern | Recommendation | Why it matters |
|---------|----------------|----------------|
| **Logging** | Use the `logging` module instead of `print`. Set a rotating file handler (`logging.handlers.RotatingFileHandler`) to keep audit trails of every `gs`, `qpdf`, or `pdftk` invocation. | Debugging PDF‑processing pipelines can be opaque; a full command‑line log saves hours. |
| **Error handling** | Wrap each external call (`subprocess.check_call`) in a `try/except subprocess.CalledProcessError`. On failure, capture `stderr` to a log file and optionally retry with a fallback (e.g., fall back from `qpdf` to `pdftk`). | PDFs are notoriously malformed; graceful degradation prevents a single bad file from crashing the whole batch. |
| **Parallelism** | For large batches, use `concurrent.futures.ProcessPoolExecutor` to run the per‑chapter steps (watermark → compress → encrypt) in parallel. Limit workers to `os.cpu_count() - 1` to keep the host responsive. | Speed‑up can be 3‑5× on multi‑core machines, especially when rasterising or compressing images. |
| **Memory usage** | Prefer incremental writes (`PdfFileWriter.write`) over loading the entire PDF into memory. When using `pdf2image`, set a modest `dpi` (150–200) unless high‑resolution output is required. | Prevents out‑of‑memory crashes on CI runners or low‑end containers. |
| **Testing** | Store a small “fixture” PDF (≈2 pages) in your repo. Write unit tests that assert: <br>• `pdf_to_text` returns a non‑empty string. <br>• `split_pdf` creates the expected number of files. <br>• `add_watermark` produces a file whose page count matches the source. <br>Use `pytest` with the `tmp_path` fixture to keep the filesystem clean. | Guarantees that future upgrades (e.g., moving from PyPDF2 1.26 to 2.x) don’t silently break functionality. |
| **CI/CD** | In GitHub Actions or GitLab CI, install the system binaries in a dedicated job step (`apt-get install ghostscript qpdf pdftk poppler-utils`). Cache the `pip` wheel directory (`~/.cache/pip`) to speed up builds. | Automated pipelines catch regressions early and keep the Docker image lean. |
| **Docker best‑practice** | Build a multi‑stage image: first stage (`builder`) installs all binaries and Python deps; second stage copies only the runtime files (`/usr/local/bin/gs`, `/usr/bin/qpdf`, etc.) and your application code. This reduces the final image size to ~120 MB. | Smaller images are faster to pull in CI and production environments. |
| **Security** | Never embed passwords in source. Use environment variables (`PDF_USER_PWD`, `PDF_OWNER_PWD`) or a secret manager (AWS Secrets Manager, HashiCorp Vault). When writing logs, **mask** any password that might appear in a command line. | Prevents accidental credential leakage. |
| **Compliance** | If you need PDF/A‑2b, invoke Ghostscript with `-dPDFA=2 -dPDFACompatibilityPolicy=1`. For PDF/A‑1b, use `-dPDFA=1`. After conversion, validate with `veraPDF` (open‑source validator). | Guarantees that the output meets archival standards required by many enterprises and governments. |

### Sample Dockerfile (multi‑stage)

```dockerfile
# ---------- Builder stage ----------
FROM python:3.9-slim AS builder
RUN apt-get update && apt-get install -y --no-install-recommends \
        ghostscript \
        poppler-utils \
        qpdf \
        pdftk \
    && rm -rf /var/lib/apt/lists/*
COPY requirements.txt .
RUN pip install --user -r requirements.txt

# ---------- Runtime stage ----------
FROM python:3.9-slim
COPY --from=builder /root/.local /root/.local
ENV PATH=/root/.local/bin:$PATH
COPY . /app
WORKDIR /app
CMD ["python", "pipeline.py"]
```

---

## 9. When to Reach for a Commercial Service  

Even with the full toolbox above, there are edge cases where a SaaS API can save you time:

| Situation | Service example | What it adds |
|-----------|----------------|--------------|
| **Batch OCR on scanned PDFs** (needs high‑accuracy text layers) | Google Cloud Vision, ABBYY Cloud OCR | Neural‑network OCR, language detection, layout preservation. |
| **Digital signatures (PKI‑based)** | DocuSign, Adobe Sign | Certified signatures, audit trails, compliance with eIDAS / ESIGN. |
| **Massive PDF/A conversion** (thousands per hour) | PDFTron, iText Cloud | Optimised native code, built‑in validation, support for PDF 2.0. |

If your workload is occasional or you’re operating in a highly regulated environment, the extra cost may be justified.

---

## 10. TL;DR Recap (Re‑stated for the Reader)  

1. **Start pure‑Python** – `pdfminer.six` for text, `PyPDF2`/`pdfrw` for structural edits.  
2. **Drop to CLI** when you need: <br>• Aggressive image down‑sampling (`gs`). <br>• Object‑stream compression & linearisation (`qpdf`). <br>• Password handling beyond RC4 (`qpdf`/`pikepdf`).  
3. **Chain the tools** – a typical pipeline is: extract → split → watermark → Ghostscript → qpdf → AES‑256 encrypt → merge.  
4. **Wrap everything** in a robust script: proper logging, error handling, parallelism, and CI tests.  
5. **Containerise** with a multi‑stage Docker build to keep production images small and reproducible.  

With these building blocks you can automate virtually any PDF workflow that existed in 2016, and the same code will continue to work in later Python releases (just upgrade the libraries). The ecosystem may have added new players (e.g., `pdfplumber` for table extraction), but the core pattern—pure‑Python first, fall back to battle‑tested binaries when needed—remains the most maintainable strategy.

Happy PDF hacking!  

Tags: tag-1, tag-2, tag-3  
Slug: article-slug