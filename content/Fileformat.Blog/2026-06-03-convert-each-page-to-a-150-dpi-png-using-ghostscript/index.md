---
seoTitle: "Convert each page to a 150‑dpi PNG using Ghostscript"
title: "Convert each page to a 150‑dpi PNG using Ghostscript"
description: "Learn how to generate, merge, split, rotate, watermark, compress & password‑protect PDFs in Python using lightweight libraries and CLI tools."
date: 03 Jun 2026
draft: true
author: Khan AI
url: /audio/convert-each-page-to-a-150-dpi-png-using-ghostscript/
categories: ['Audio']
tags: ['Convert each page to a 150‑dpi PNG using Ghostscript', 'MP4', 'Some Tag']
---

**TL;DR** – In 2014 you can generate, merge, split, rotate, watermark, compress, and password‑protect PDFs entirely from Python using a handful of well‑maintained libraries (PyPDF2, pdfrw, ReportLab, PDFMiner, etc.) plus a couple of command‑line tools (Ghostscript, PDFtk). The code is short, the dependencies are light, and the same patterns work on Django, Flask, or any batch script.

---

## Why PDFs Still Matter (2014)

* **Ubiquity** – Invoices, contracts, e‑books, and regulatory reports are almost always shipped as PDFs.  
* **Server‑side generation** – A Django or Flask app that lets users download a receipt or a report needs to spit out a PDF on the fly.  
* **Automation** – Back‑office pipelines that batch‑process hundreds of contracts, extract data, or archive documents rely on programmatic PDF handling.  
* **Security & compliance** – Enterprises often demand password protection, encryption, or PDF/A archival for legal reasons.

If you’re writing Python code today, you’ll almost certainly hit one of those use‑cases. The good news? The ecosystem in 2014 already offers solid, pure‑Python tools plus a few battle‑tested CLI utilities you can call from `subprocess`.

---

## PDF Basics Every Pythonista Should Know

| Concept | What It Is | Why It Matters in Python |
|---------|------------|--------------------------|
| **Objects & Streams** | PDFs are collections of indirect objects (text, images, fonts) stored in binary *streams*. | Most libraries manipulate objects indirectly (e.g., adding a page) rather than editing raw streams. |
| **Cross‑Reference Table (X‑Ref)** | Index mapping object numbers to byte offsets. | Corrupt the X‑Ref and you’ll get “file is damaged” errors when you write a new PDF. |
| **Page Tree** | Hierarchical structure that defines page order, media boxes, rotation. | Adding/removing pages = editing the page tree. |
| **Encryption (Standard Security Handler)** | 40‑bit or 128‑bit RC4/AES encryption with user/owner passwords. | Used for “secure PDF” features; most libs expose `encrypt(user_pwd, owner_pwd, use_128bit=True)`. |
| **PDF/A & PDF/X** | Sub‑sets of PDF for archival (PDF/A) or printing (PDF/X). | Not natively supported by most 2014 libs, but you can reach them via Ghostscript. |
| **Metadata (XMP)** | XML‑based document information (author, creation date, keywords). | Easy to read/write via `PdfFileWriter.addMetadata()`. |

Understanding these building blocks helps you debug the occasional “invalid X‑Ref” or “cannot rotate page” exception that pops up when you mix libraries.

---

## Pick the Right Library for the Job

| Library | Sweet Spot | Install | Python 2/3 | Known Gaps (2014) |
|---------|------------|---------|------------|-------------------|
| **PyPDF2** | Merge, split, rotate, encrypt, add metadata | `pip install PyPDF2` | 2.6‑2.7, 3.2‑3.3 (experimental) | No text extraction, cannot edit existing content streams, limited PDF/A support |
| **pdfrw** | Low‑level read/write, works nicely with ReportLab | `pip install pdfrw` | 2.6‑2.7, 3.2‑3.3 | No built‑in compression; you’ll need Ghostscript for heavy lifting |
| **ReportLab (open source)** | Create PDFs from scratch (canvas, Platypus) | `pip install reportlab` | 2.6‑2.7, 3.2‑3.3 | Generation only – cannot read existing PDFs |
| **PDFMiner** | Text extraction, layout analysis | `pip install pdfminer` | 2.6‑2.7, 3.2‑3.3 | Slow on large files; does not modify PDFs |
| **PyPDF4** (late‑2014 fork) | Same as PyPDF2 but with bug‑fixes & better Python‑3 support | `pip install PyPDF4` | 2.7, 3.3+ | Still experimental |
| **Ghostscript (CLI)** | Compression, rasterization, PDF → PDF/A conversion | System package (`gs`) | Any | External binary; AGPL licensing considerations |
| **PDFtk (CLI)** | Merge/split, password protect, fill forms | System package (`pdftk`) | Any | Not pure‑Python; Windows binaries required |

A typical workflow looks like this:

1. **Generate** a PDF with ReportLab (or convert HTML with WeasyPrint/xhtml2pdf).  
2. **Manipulate** pages, add watermarks, or merge files with PyPDF2/pdfrw.  
3. **Compress** the result with Ghostscript or PDFtk.  
4. **Secure** it with `encrypt()` from PyPDF2 or a PDFtk command.

---

## Convert & Generate PDFs in a Snap

### 4.1 HTML / Markdown → PDF

*WeasyPrint* (beta in 2014) gives you CSS‑styled PDF output:

```python
from weasyprint import HTML

HTML('report.html').write_pdf('report.pdf')
```

If you prefer a pure‑Python solution that works well with Django templates, *xhtml2pdf* (aka pisa) does the trick:

```python
from xhtml2pdf import pisa

with open('invoice.html') as src, open('invoice.pdf', 'wb') as out:
    pisa.CreatePDF(src.read(), dest=out)
```

### 4.2 Data (CSV, JSON) → PDF with ReportLab

```python
from reportlab.lib.pagesizes import LETTER
from reportlab.pdfgen import canvas
import csv

c = canvas.Canvas('sales_report.pdf', pagesize=LETTER)
c.setFont('Helvetica', 12)
c.drawString(72, 720, "Quarterly Sales Report")

y = 680
with open('sales.csv') as f:
    for row in csv.reader(f):
        c.drawString(72, y, ', '.join(row))
        y -= 20

c.save()
```

### 4.3 PDF → Images or Text (for OCR pipelines)

```python
import subprocess

# Convert each page to a 150‑dpi PNG using Ghostscript
subprocess.call([
    'gs', '-dNOPAUSE', '-sDEVICE=pngalpha',
    '-r150', '-sOutputFile=page-%03d.png',
    '-dBATCH', 'source.pdf'
])
```

```python
from pdfminer.high_level import extract_text

text = extract_text('source.pdf')
print(text[:500])   # preview first 500 characters
```

---

## Edit, Compress, and Secure PDFs

### 5.1 Merging & Concatenating

```python
from PyPDF2 import PdfFileMerger

merger = PdfFileMerger()
for pdf in ['cover.pdf', 'chapter1.pdf', 'chapter2.pdf']:
    merger.append(pdf)

with open('full_book.pdf', 'wb') as out:
    merger.write(out)
```

### 5.2 Splitting & Extracting Pages

```python
from PyPDF2 import PdfFileReader, PdfFileWriter

reader = PdfFileReader(open('big.pdf', 'rb'))
writer = PdfFileWriter()

# Grab pages 6‑10 (0‑based indexing)
for p in range(5, 10):
    writer.addPage(reader.getPage(p))

with open('subset.pdf', 'wb') as out:
    writer.write(out)
```

### 5.3 Rotating, Cropping, and Watermarking

```python
reader = PdfFileReader(open('original.pdf', 'rb'))
writer = PdfFileWriter()

watermark = PdfFileReader(open('wm.pdf', 'rb')).getPage(0)

for i in range(reader.getNumPages()):
    page = reader.getPage(i)

    # Rotate every even page 90° clockwise
    if i % 2 == 0:
        page.rotateClockwise(90)

    # Overlay the same watermark on every page
    page.mergePage(watermark)

    writer.addPage(page)

with open('styled.pdf', 'wb') as out:
    writer.write(out

```python
with open('styled.pdf', 'wb') as out:
    writer.write(out)
```

That’s it – a few dozen lines of code give you a fully‑featured PDF with rotated pages and a translucent logo on every sheet.

---

## 5.4 Compressing PDFs for the Web

Even though PyPDF2 can strip out unused objects, the real size‑reduction comes from a downstream compression step. Ghostscript is the workhorse:

```bash
gs -sDEVICE=pdfwrite -dCompatibilityLevel=1.4 \
   -dPDFSETTINGS=/screen -dNOPAUSE -dBATCH \
   -sOutputFile=compressed.pdf original.pdf
```

| `-dPDFSETTINGS` | Target quality |
|-----------------|----------------|
| `/screen`       | 72 dpi, smallest file (web) |
| `/ebook`        | 150 dpi, decent quality |
| `/printer`      | 300 dpi, print‑ready |
| `/prepress`     | 300 dpi + all fonts embedded |

If you need PDF/A compliance (archival), add `-dPDFA=1` and supply an ICC profile:

```bash
gs -dPDFA=1 -dBATCH -dNOPAUSE -sDEVICE=pdfwrite \
   -sOutputFile=archival.pdf -sPDFACompatibilityPolicy=1 \
   -sColorConversionStrategy=RGB -sProcessColorModel=DeviceRGB \
   -sDefaultRGBProfile=/usr/share/color/icc/AdobeRGB1998.icc \
   original.pdf
```

### 5.5 Password‑Protecting & Encrypting

PyPDF2 (and its fork PyPDF4) expose a simple API:

```python
from PyPDF2 import PdfFileReader, PdfFileWriter

reader = PdfFileReader(open('styled.pdf', 'rb'))
writer = PdfFileWriter()

for p in range(reader.getNumPages()):
    writer.addPage(reader.getPage(p))

# 128‑bit AES encryption
writer.encrypt(user_pwd='readme', owner_pwd='admin', use_128bit=True)

with open('protected.pdf', 'wb') as out:
    writer.write(out)
```

If you need stronger encryption or want to apply permissions (printing, copying), PDFtk is a handy wrapper:

```bash
pdftk styled.pdf output protected.pdf owner_pw admin user_pw readme allow AllFeatures
```

---

## 6. Working With PDF Forms (AcroForms)

Many business processes rely on fillable PDFs – invoices, tax forms, surveys. While the 2014 Python ecosystem doesn’t have a full‑featured form editor, you can still read and populate fields.

### 6.1 Reading Form Fields with PyPDF2

```python
from PyPDF2 import PdfFileReader

reader = PdfFileReader(open('blank_form.pdf', 'rb'))
fields = reader.getFields()
for name, info in fields.items():
    print(f"{name}: {info.get('/V', '(empty)')}")
```

### 6.2 Populating a Form

```python
from pdfrw import PdfReader, PdfWriter, PageMerge

TEMPLATE = 'blank_form.pdf'
OUTPUT   = 'filled_form.pdf'

ANNOT_KEY = '/Annots'
WIDGET_SUBTYPE = '/Widget'
FIELD_KEY = '/T'
VALUE_KEY = '/V'

pdf = PdfReader(TEMPLATE)
for page in pdf.pages:
    annotations = page[ANNOT_KEY]
    if annotations:
        for annotation in annotations:
            if annotation[ANNOT_KEY][0].Subtype == WIDGET_SUBTYPE:
                field_name = annotation[ANNOT_KEY][0][FIELD_KEY][1:-1]  # strip parentheses
                if field_name == 'CustomerName':
                    annotation[ANNOT_KEY][0][VALUE_KEY] = '(John Doe)'
                elif field_name == 'InvoiceDate':
                    annotation[ANNOT_KEY][0][VALUE_KEY] = '(2024‑05‑31)'

PdfWriter().write(OUTPUT, pdf)
```

After filling, you may want to **flatten** the form (make fields non‑editable) – Ghostscript can do that:

```bash
gs -dNOPAUSE -dBATCH -sDEVICE=pdfwrite -dFlattenAllForms=true \
   -sOutputFile=flattened.pdf filled_form.pdf
```

---

## 7. Putting It All Together – A Mini‑Pipeline Example

Below is a self‑contained script that could sit inside a Django view or a Flask route. It:

1. Renders an HTML receipt with Jinja2.  
2. Converts the HTML to PDF via WeasyPrint.  
3. Adds a company logo watermark.  
4. Compresses the result for email attachment.  
5. Password‑protects it with a per‑user password.

```python
import os, subprocess, tempfile
from jinja2 import Environment, FileSystemLoader
from weasyprint import HTML
from PyPDF2 import PdfFileReader, PdfFileWriter

def generate_receipt(data, user_password):
    # 1️⃣ Render HTML template
    env = Environment(loader=FileSystemLoader('templates'))
    tmpl = env.get_template('receipt.html')
    html = tmpl.render(data)

    # 2️⃣ Convert to PDF (WeasyPrint)
    with tempfile.NamedTemporaryFile(delete=False, suffix='.pdf') as tmp_pdf:
        HTML(string=html).write_pdf(tmp_pdf.name)

    # 3️⃣ Watermark (logo.pdf is a single‑page PDF with transparent logo)
    watermark = PdfFileReader(open('logo.pdf', 'rb')).getPage(0)
    source = PdfFileReader(open(tmp_pdf.name, 'rb'))
    writer = PdfFileWriter()
    for i in range(source.getNumPages()):
        page = source.getPage(i)
        page.mergePage(watermark)
        writer.addPage(page)

    # 4️⃣ Write intermediate watermarked PDF
    with tempfile.NamedTemporaryFile(delete=False, suffix='.pdf') as watermarked:
        writer.write(watermarked)

    # 5️⃣ Compress with Ghostscript
    compressed_path = watermarked.name.replace('.pdf', '_compressed.pdf')
    subprocess.check_call([
        'gs', '-sDEVICE=pdfwrite', '-dCompatibilityLevel=1.4',
        '-dPDFSETTINGS=/screen', '-dNOPAUSE', '-dBATCH',
        '-sOutputFile=' + compressed_path,
        watermarked.name
    ])

    # 6️⃣ Encrypt with user password
    final_reader = PdfFileReader(open(compressed_path, 'rb'))
    final_writer = PdfFileWriter()
    for p in range(final_reader.getNumPages()):
        final_writer.addPage(final_reader.getPage(p))
    final_writer.encrypt(user_pwd=user_password, owner_pwd='admin', use_128bit=True)

    final_path = compressed_path.replace('_compressed.pdf', '_final.pdf')
    with open(final_path, 'wb') as out_f:
        final_writer.write(out_f)

    # Clean up temp files
    for f in (tmp_pdf.name, watermarked.name, compressed_path):
        os.remove(f)

    return final_path
```

Drop this function into any WSGI‑compatible framework, call it from a POST handler, and you have a production‑ready PDF generation pipeline that runs entirely on a modest virtual server.

---

## 8. Common Pitfalls & Debugging Tips

| Symptom | Likely Cause | Fix |
|---------|--------------|-----|
| **“File is damaged or corrupted”** when opening the output PDF | X‑Ref table not rebuilt (e.g., you edited the raw stream) | Use a library’s `write()` method; never concatenate binary strings manually. |
| **Missing fonts after merging** | One source PDF embeds fonts, the other does not; the writer drops unused objects | Run Ghostscript with `-dEmbedAllFonts=true` or ensure each PDF embeds its fonts before merging. |
| **Watermark appears behind text** | `mergePage()` respects the existing content order | Use `page.mergeTranslatedPage(watermark, 0, 0, expand=False)` *after* rotating or use `PdfFileWriter.addPage(page)` then overlay a second page containing only the watermark. |
| **Unicode characters become garbled** | ReportLab default font (Helvetica) lacks the glyphs | Register a TTF font with ReportLab (`pdfmetrics.registerFont(TTFont('DejaVu', 'DejaVuSans.ttf'))`) and set it on the canvas. |
| **PDFTk not found on Windows** | Binary not on PATH | Download the Windows installer from the PDFtk website, add its `bin` folder to the system `PATH`, or call it via absolute path. |
| **Performance stalls on > 500‑page PDFs** | Pure‑Python libraries load the whole file into memory | Stream pages using `PdfFileReader(..., strict=False)` and write out incrementally; for massive batches, offload to Ghostscript’s `-dFirstPage`/`-dLastPage` options. |

When in doubt, open the problematic PDF in a viewer that can show the internal structure (e.g., `pdfinfo` from Poppler

When in doubt, open the problematic PDF in a viewer that can show the internal structure (e.g., `pdfinfo` from Poppler, `mutool info` from MuPDF, or Adobe Acrobat’s “Preflight” panel). These tools will surface missing objects, broken X‑Ref entries, or non‑embedded fonts that pure‑Python libraries silently ignore.

### 8.1 Low‑Level Inspection Tools

```bash
# Show basic metadata and page count
pdfinfo suspect.pdf

# List all objects and their offsets (useful for X‑Ref debugging)
pdfobjdump -s suspect.pdf

# Extract every image to see if they were stripped during compression
pdfimages -all suspect.pdf img-
```

If any of these commands report “error parsing object” or “cannot decode stream”, the culprit is almost always a malformed stream that the Python wrapper tried to write back unchanged. The quickest cure is to run the file through Ghostscript once more with `-dPDFSETTINGS=/prepress` – Ghostscript will rebuild the X‑Ref table and re‑encode streams in a known‑good way.

### 8.2 Unit‑Testing Your PDF Pipeline

Because PDFs are binary, a naïve `assert` on file size isn’t enough. A robust test suite should:

1. **Generate a deterministic PDF** (e.g., by fixing timestamps and using a static seed for random IDs).  
2. **Hash the output** (`sha256sum`) and compare against a stored baseline.  
3. **Run a sanity check** with `pdfinfo` to verify page count, dimensions, and encryption status.  
4. **Extract text** with PDFMiner and assert that key strings appear (e.g., invoice numbers).  

```python
def test_receipt_pipeline(tmp_path):
    out_path = generate_receipt(
        data={'order_id': '12345', 'total': '99.99'},
        user_password='test123'
    )
    # 1. deterministic hash
    assert hashlib.sha256(open(out_path, 'rb').read()).hexdigest() == \
           'a1b2c3d4e5f67890deadbeefcafebabe1234567890abcdef1234567890abcdef'

    # 2. pdfinfo sanity
    info = subprocess.check_output(['pdfinfo', out_path]).decode()
    assert 'Pages:' in info and '1' in info
    assert 'Encrypted:' in info and 'yes' in info

    # 3. text extraction
    text = extract_text(out_path)
    assert 'Invoice #12345' in text
```

Run these tests in CI (GitHub Actions, Travis CI, or GitLab CI) to catch regressions when you upgrade PyPDF2 → PyPDF4 or switch Ghostscript versions.

### 8.3 Dockerising the Whole Stack

A reproducible environment eliminates the “works on my laptop” syndrome. Below is a minimal Dockerfile that bundles everything you need for the pipeline described earlier:

```dockerfile
FROM python:3.9-slim

# System dependencies
RUN apt-get update && apt-get install -y \
    ghostscript \
    poppler-utils \
    pdftk \
    && rm -rf /var/lib/apt/lists/*

# Python packages
RUN pip install --no-cache-dir \
    PyPDF2==1.26.0 \
    pdfrw==0.4 \
    reportlab==3.5.34 \
    weasyprint==0.42 \
    pdfminer.six==20201018 \
    Jinja2==3.0.3

# Add your app code
WORKDIR /app
COPY . /app

# Example entrypoint (run a test suite)
CMD ["pytest"]
```

Build and run:

```bash
docker build -t pdf‑pipeline .
docker run --rm pdf‑pipeline
```

Now every developer, CI runner, and production server uses the exact same Ghostscript version (9.53 at the time of writing) and the same Python package set, guaranteeing consistent output.

### 8.4 Packaging Your PDF Utilities

If you find yourself re‑using the same helper functions across projects, consider turning them into a small pip‑installable package:

```text
pdfutils/
├── pdfutils/
│   ├── __init__.py
│   ├── merge.py
│   ├── watermark.py
│   └── compress.py
├── tests/
│   └── test_merge.py
├── setup.cfg
└── pyproject.toml
```

Declare the runtime dependencies (`PyPDF2`, `pdfrw`, `reportlab`) in `setup.cfg` and add an optional `extras_require` entry for the CLI tools:

```ini
[options.extras_require]
cli = ghostscript; pdftk
```

Consumers can then install everything they need with:

```bash
pip install pdfutils[cli]
```

### 9. Looking Ahead – What’s Coming in 2015 and Beyond?

The 2014 landscape is already solid, but a few trends are worth watching:

| Trend | Impact on Python PDF Workflows |
|-------|--------------------------------|
| **Python 3.5+ adoption** | Newer releases of PyPDF2 and PDFMiner are adding proper type hints and async‑friendly APIs. |
| **PDF 2.0 (ISO 32000‑2)** | Adds support for digital signatures and richer metadata. Expect a “PyPDF3” or similar fork to surface in 2016. |
| **WebAssembly PDF renderers** | Projects like PDF.js are moving to the server side via Node‑JS; you may combine a Python front‑end with a JS rendering micro‑service for ultra‑fast previews. |
| **Improved PDF/A tooling** | Ghostscript 9.15+ ships a native PDF/A‑2 validator; you can call it from Python to guarantee archival compliance without a commercial SDK. |

For now, the combination of pure‑Python libraries and battle‑tested CLI tools gives you a production‑grade stack that runs on any Linux, macOS, or Windows box. Keep an eye on the GitHub issue trackers of PyPDF2 and pdfrw – the community is actively fixing bugs (e.g., handling of rotated annotations) and adding features like incremental updates, which will make “edit‑in‑place” workflows possible without a full rewrite.

---

## 10. TL;DR Recap (One‑Liner)

> **Generate → Manipulate → Compress → Secure → Deploy** – with ReportLab (or WeasyPrint) for creation, PyPDF2/pdfrw for page‑level edits, Ghostscript or PDFtk for compression and encryption, and a thin Docker/CI layer for reproducibility, you can handle any 2014‑era PDF requirement in pure Python.

---

### Final Thoughts

PDF handling used to be the domain of heavyweight Java or .NET libraries, but by 2014 the Python ecosystem had matured enough to let you build end‑to‑end pipelines with just a few pip installs and a couple of system binaries. The key is to respect the PDF spec’s low‑level quirks (X‑Ref tables, object streams) and to lean on Ghostscript when you need heavy compression or archival‑grade output. With a solid test suite and containerised builds, you’ll never be surprised by a “file is damaged” error in production again.

Happy coding, and may your PDFs always be lean, secure, and perfectly water‑marked.

Tags: tag-1, tag-2, tag-3
Slug: article-slug