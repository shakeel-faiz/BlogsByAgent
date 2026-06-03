---
seoTitle: "Example usage"
title: "Example usage"
description: "Learn how to read, split, merge, rotate, watermark, compress, and password‑protect PDFs using Python 2.7 libraries and command‑line tools in 2012."
date: 03 Jun 2026
draft: true
author: Khan AI
url: /audio/example-usage/
categories: ['Audio']
tags: ['Example usage', 'MP4', 'Some Tag']
---

**TL;DR** – In 2012 the Python ecosystem (mostly Python 2.7) gives you everything you need to **read, split, merge, rotate, watermark, compress, and password‑protect** PDFs without leaving the comfort of your script. Pure‑Python libraries (PyPDF2, pdfrw, ReportLab, PDFMiner, xhtml2pdf) handle the “happy path”, while Ghostscript, PDFtk, and poppler‑utils fill the gaps for high‑quality compression and image extraction. Below is a hands‑on, end‑to‑end guide that shows you how to stitch these tools together, explains the underlying PDF concepts you’ll bump into, and points out the 2012‑specific quirks you’ll need to watch for.

---

## 1. Why a 2012‑centred guide?

* **Python version reality** – In mid‑2012 the vast majority of production code runs on **Python 2.7**. Python 3.2 has just landed, but most PDF wheels still ship Python‑2‑only binaries. All the snippets below therefore target 2.7 (with a few notes on the emerging 3.x support).  
* **PDF spec status** – PDF‑1.7 (ISO 32000‑1) is the de‑facto standard; PDF‑2.0 won’t appear until 2017. That means the encryption model is limited to RC4‑40/128‑bit, and most libraries expose exactly those options.  
* **Tooling culture** – Desktop‑oriented scripts, occasional web‑service hooks, and heavy reliance on command‑line utilities (Ghostscript, PDFtk, poppler) are the norm. Wrapping those tools with `subprocess` is considered idiomatic.  
* **Developer pain points** – File‑size matters (mobile/tablet consumption), security is a must (owner/user passwords), and converting from HTML/Markdown to PDF is a frequent requirement for invoices and reports.  

Understanding these constraints helps you pick the right library today and avoid “future‑proofing” dead‑ends that weren’t realistic back then.

---

## 2. Core Python PDF libraries (the 2012 toolbox)

| Library | What it does best | License | Python 2/3 support (2012) |
|---------|-------------------|---------|---------------------------|
| **PyPDF2** (fork of PyPDF) | Merge/split/rotate pages, add metadata, simple password protection | BSD | 2.7 (3.x experimental) |
| **pdfrw** | Low‑level read/write, great when you need to feed a ReportLab canvas into an existing PDF | MIT | 2.7 / 3.x (experimental) |
| **ReportLab (open‑source edition)** | Generate PDFs from scratch – canvas, flowables, barcodes, QR codes, built‑in Flate compression | BSD | 2.7 / 3.x |
| **PDFMiner** | Text extraction with layout awareness (PDF → text/HTML) | MIT | 2.7 |
| **xhtml2pdf / pisa** | Convert simple HTML (and thus Markdown) to PDF using ReportLab under the hood | LGPL | 2.7 |
| **External wrappers** (Ghostscript, PDFtk, qpdf, poppler‑utils) | High‑quality compression, linearization, advanced encryption, image rasterisation | Various | 2.7 (via `subprocess`) |

> **Takeaway:** Pure‑Python tools cover *most* day‑to‑day editing tasks. When you need **real compression**, **digital signatures**, or **high‑fidelity image extraction**, you still call out to an external binary.

---

## 3. PDF fundamentals you’ll bump into

### 3.1 Object model & incremental updates  
A PDF is a collection of **objects** (dictionaries, streams, arrays). The **cross‑reference table** maps each object number to its byte offset, and the **trailer** points to the root catalog and the optional info dictionary. Most editors (including PyPDF2) perform **incremental updates** – they append a new xref table and trailer instead of rewriting the whole file. This is why you can “add a password” without touching the original content stream.

### 3.2 Page‑level vs document‑level operations  
*Page‑level* – rotate, crop, add watermarks, extract images.  
*Document‑level* – merge, split, set metadata, encrypt/decrypt.  
Understanding the distinction helps you decide whether you need a **page‑oriented** library (pdfrw, PyPDF2) or a **document‑oriented** generator (ReportLab).

### 3.3 Compression in 2012  
* **Flate (zlib)** – default for text and vector graphics; ReportLab uses it automatically.  
* **JPEG** – for raster images; you can down‑sample before embedding to shrink size.  
* **Ghostscript presets** – `-dPDFSETTINGS=/screen|/ebook|/printer|/prepress` let you trade quality for size in a single command.  

### 3.4 Security basics  
PDF‑1.7 only defines the **Standard Security Handler** (RC4 40‑bit and 128‑bit). Two passwords exist:

| Password | What it controls |
|----------|-------------------|
| **User** | Opening the document (view‑only). |
| **Owner** | Changing permissions, removing the user password. |

Permissions bits let you disable printing, copying, or modifying. Pure‑Python libs can set these flags, but they **cannot create PKI‑based digital signatures** – you need an external tool like `openssl` + `pdfsig`.

---

## 4. Typical workflows (code snippets)

Below are ready‑to‑run examples for the most common tasks. All snippets assume **Python 2.7**; for Python 3 just replace `print` statements and open files with `encoding='utf-8'` where appropriate.

### 4.1 Merging PDFs (PyPDF2)

```python
import PyPDF2

def merge_pdfs(pdf_paths, output_path):
    writer = PyPDF2.PdfFileWriter()
    for path in pdf_paths:
        reader = PyPDF2.PdfFileReader(open(path, 'rb'))
        for page_num in range(reader.getNumPages()):
            writer.addPage(reader.getPage(page_num))
    with open(output_path, 'wb') as out_f:
        writer.write(out_f)

# Example usage
merge_pdfs(['invoice1.pdf', 'invoice2.pdf', 'terms.pdf'], 'full_package.pdf')
```

### 4.2 Adding a watermark (pdfrw + ReportLab)

```python
from pdfrw import PdfReader, PdfWriter, PageMerge
from reportlab.pdfgen import canvas
from io import BytesIO

def create_watermark(text):
    packet = BytesIO()
    c = canvas.Canvas(packet, pagesize=(595, 842))   # A4 size in points
    c.setFont("Helvetica", 72)
    c.setFillColorRGB(0.6, 0.6, 0.6, alpha=0.3)      # Light gray, semi‑transparent
    c.saveState()
    c.translate(300, 400)
    c.rotate(45)
    c.drawCentredString(0, 0, text)
    c.restoreState()
    c.save()
    packet.seek(0)
    return PdfReader(packet)

def stamp(input_pdf, output_pdf, watermark_text):
    watermark = create_watermark(watermark_text)
    reader = PdfReader(input_pdf)
    writer = PdfWriter()
    for page in reader.pages:
        merger = PageMerge(page)
        merger.add(watermark.pages[0]).render()
        writer.addpage(page)
    writer.write(output_pdf)

# Apply a "CONFIDENTIAL" stamp
stamp('draft_report.pdf', 'draft_report_watermarked.pdf', 'CONFIDENTIAL')
```

### 4.3 Compressing a PDF with Ghostscript (subprocess)

```python
import subprocess
import os

def compress_pdf(src_path, dst_path, preset='/screen'):
    """
    preset can be one of:
        /screen  – low‑res, smallest size
        /ebook   – medium‑res, good for e‑readers
        /printer – high‑res, suitable for printing
        /prepress – max quality, minimal compression
    """
    gs_cmd = [
        'gs',
        '-sDEVICE=pdfwrite',
        '-dCompatibilityLevel=1.4',
        '-dPDFSETTINGS={}'.format(preset),
        '-dNOPAUSE',
        '-dQUIET',
        '-dBATCH',
        '-sOutputFile={}'.format(dst_path),
        src_path

```python
    ]
    # Run Ghostscript; capture any error output for debugging
    try:
        subprocess.check_output(gs_cmd, stderr=subprocess.STDOUT)
        print("Compressed {} → {} (preset={})".format(src_path, dst_path, preset))
    except subprocess.CalledProcessError as e:
        print("Ghostscript failed:")
        print(e.output.decode('utf-8'))
        raise

# Example – shrink a 5 MB report to under 1 MB for mobile download
compress_pdf('full_report.pdf', 'full_report_small.pdf', preset='/screen')
```

> **Why Ghostscript?**  
> Pure‑Python libraries only apply *Flate* compression to streams they create. Existing PDFs often contain uncompressed image streams (e.g., scanned TIFFs). Ghostscript re‑encodes those images using JPEG or down‑samples them, delivering the dramatic size reductions that mobile‑first teams demand.

---

### 4.4 Splitting a PDF into single‑page files (PyPDF2)

```python
def split_pdf(source_path, output_dir):
    reader = PyPDF2.PdfFileReader(open(source_path, 'rb'))
    for i in range(reader.getNumPages()):
        writer = PyPDF2.PdfFileWriter()
        writer.addPage(reader.getPage(i))
        out_name = os.path.join(output_dir, 'page_{:03d}.pdf'.format(i + 1))
        with open(out_name, 'wb') as out_f:
            writer.write(out_f)
        print("Wrote:", out_name)

# Split a 30‑page contract into individual pages
split_pdf('contract.pdf', '/tmp/contract_pages')
```

*Tip:* If you need to preserve the original document’s metadata (author, creation date, etc.) copy the `Info` dictionary from the source `PdfFileReader` to each new `PdfFileWriter` before writing.

---

### 4.5 Rotating pages (PyPDF2)

```python
def rotate_pdf(source_path, output_path, rotation=90):
    """
    rotation – degrees clockwise (must be 90, 180, or 270)
    """
    reader = PyPDF2.PdfFileReader(open(source_path, 'rb'))
    writer = PyPDF2.PdfFileWriter()
    for page in reader.pages:
        page.rotateClockwise(rotation)
        writer.addPage(page)
    with open(output_path, 'wb') as out_f:
        writer.write(out_f)
    print("Rotated {}° and saved to {}".format(rotation, output_path))

rotate_pdf('portrait_scans.pdf', 'portrait_scans_landscape.pdf', rotation=90)
```

Because the rotation is stored as a **page‑level transformation matrix**, the underlying content stream stays untouched – a fast, loss‑less operation.

---

### 4.6 Extracting text with PDFMiner (layout‑aware)

```python
from pdfminer.high_level import extract_text

def pdf_to_text(pdf_path, txt_path):
    text = extract_text(pdf_path)
    with open(txt_path, 'w') as out_f:
        out_f.write(text)
    print("Extracted text to", txt_path)

pdf_to_text('annual_report.pdf', 'annual_report.txt')
```

*What you’ll see:* PDFMiner returns the logical flow of characters, preserving columns and simple tables. It does **not** handle OCR – scanned PDFs still need an external OCR engine (e.g., `tesseract`) before you can extract meaningful text.

---

### 4.7 Converting HTML/Markdown to PDF (xhtml2pdf)

```python
import markdown
from xhtml2pdf import pisa

def md_to_pdf(md_file, pdf_file):
    # Convert Markdown → HTML
    with open(md_file, 'r') as f:
        html = markdown.markdown(f.read(), extensions=['tables', 'fenced_code'])
    # Wrap HTML in a minimal <body> tag for xhtml2pdf
    html = '<html><body>{}</body></html>'.format(html)

    # Render PDF
    with open(pdf_file, 'wb') as out_f:
        pisa_status = pisa.CreatePDF(src=html, dest=out_f)
    if pisa_status.err:
        raise Exception("xhtml2pdf failed to render {}".format(md_file))
    print("Generated PDF:", pdf_file)

md_to_pdf('invoice.md', 'invoice.pdf')
```

*Why not ReportLab directly?* ReportLab’s low‑level canvas is great for programmatic graphics, but for **rich text** (tables, headings, inline CSS) the HTML‑to‑PDF pipeline of `xhtml2pdf` (a thin wrapper around ReportLab) saves hours of manual layout work.

---

### 4.8 Password‑protecting a PDF (PyPDF2)

```python
def encrypt_pdf(source_path, output_path, user_pwd='', owner_pwd='s3cr3t', 
                allow_printing=False, allow_modifying=False):
    reader = PyPDF2.PdfFileReader(open(source_path, 'rb'))
    writer = PyPDF2.PdfFileWriter()
    for page in reader.pages:
        writer.addPage(page)

    # Permission bits – see PyPDF2 docs for the full mask
    perms = 0
    if allow_printing:
        perms |= 0b100
    if allow_modifying:
        perms |= 0b010

    writer.encrypt(user_pwd, owner_pwd, use_128bit=True, permissions=perms)
    with open(output_path, 'wb') as out_f:
        writer.write(out_f)
    print("Encrypted {} → {} (owner pwd hidden)".format(source_path, output_path))

encrypt_pdf('confidential.pdf', 'confidential_protected.pdf',
            user_pwd='readme', owner_pwd='admin123', allow_printing=False)
```

**Caveat (2012):** PyPDF2 only supports the *Standard Security Handler* (RC4). If you need AES‑256 encryption (PDF‑2.0) you must call out to `qpdf` or `PDFtk` with the appropriate flags.

---

### 4.9 Advanced encryption & linearization with PDFtk

```python
import subprocess

def pdftk_encrypt(src, dst, user_pwd='', owner_pwd='owner', 
                  allow='Printing'):
    """
    allow – a comma‑separated list of permissions:
        Printing, DegradedPrinting, ModifyContents,
        CopyContents, ModifyAnnotations, FillForms,
        ScreenReaders, Assembly, HighQualityPrint
    """
    cmd = [
        'pdftk', src, 'output', dst,
        'owner_pw', owner_pwd,
        'user_pw', user_pwd,
        'allow', allow
    ]
    subprocess.check_call(cmd)
    print("PDFtk encrypted {} → {} (allow={})".format(src, dst, allow))

pdftk_encrypt('report.pdf', 'report_locked.pdf',
              user_pwd='viewonly', owner_pwd='supersecret',
              allow='Printing,CopyContents')
```

PDFtk also offers **linearization** (`pdftk in.pdf cat output out.pdf linearize`) which makes the first page render instantly in a web viewer – a nice touch for large PDFs served over HTTP.

---

### 4.10 Extracting embedded images (poppler‑utils)

```python
def extract_images(pdf_path, out_dir):
    """
    Uses `pdfimages` from the poppler suite.
    """
    if not os.path.isdir(out_dir):
        os.makedirs(out_dir)
    cmd = ['pdfimages', '-all', pdf_path, os.path.join(out_dir, 'img')]
    subprocess.check_call(cmd)
    print("Extracted images from {} into {}".format(pdf_path, out_dir))

extract_images('presentation.pdf', '/tmp/presentation_imgs')
```

`pdfimages` writes each raster image in its native format (JPEG, JPX, CCITT). This is far more reliable than trying to parse the PDF stream manually with `pdfrw`, especially when the source PDF was generated by Adobe Acrobat.

---

## 5. Common pitfalls in the 2012 ecosystem (and how to dodge them)

| Symptom | Typical cause | Fix / Work‑around |
|---------|---------------|-------------------|
| **“Incorrect page count after merge”** | Some source PDFs are *linearized* (web‑optimized). PyPDF2 reads the linearized xref but discards the secondary xref, leading to missing pages. | Run `pdftk in.pdf cat output out.pdf` first to flatten the file, then merge. |

| Symptom | Typical cause | Fix / Work‑around |
|---------|---------------|-------------------|
| **“UnicodeDecodeError when reading a PDF”** | The PDF contains text streams encoded with UTF‑16BE or UTF‑32, but `PdfFileReader` is fed a file opened in text mode or the default `latin‑1` codec. | Always open PDFs in binary mode (`'rb'`). If you need to inspect raw strings, decode with `utf‑16` and ignore errors: `text = raw.decode('utf‑16', errors='ignore')`. |
| **“Watermark appears behind the page content”** | `PageMerge` adds the watermark *after* the existing content matrix, but some PDFs have a **/Resources** dictionary that forces a different drawing order. | Use `PageMerge(page, prepend=True).add(watermark_page).render()` to insert the watermark *before* the existing content. |
| **“Ghostscript throws ‘Error: /undefined in …’”** | The input PDF uses a custom font subset that Ghostscript cannot locate, often because the font files are embedded as Type 3 without proper `FontDescriptor`. | Run `gs -dNOPAUSE -dBATCH -sDEVICE=pdfwrite -dPDFSETTINGS=/prepress -sOutputFile=clean.pdf original.pdf` first; Ghostscript will substitute missing fonts with a generic fallback, then re‑run your compression step. |
| **“PDFtk reports ‘Permission denied’ on Windows”** | Windows blocks execution of binaries that are not in the system `PATH` or are flagged as unsafe by SmartScreen. | Place `pdftk.exe` in a directory that is added to the `PATH` (e.g., `C:\Program Files\PDFtk\bin`) and run the script with elevated privileges, or use the `subprocess` `shell=True` flag sparingly. |
| **“PDFMiner returns garbled characters (e.g., �) for accented text”** | The PDF stores text in a custom encoding (e.g., WinAnsi) but does not provide a proper `ToUnicode` CMap. PDFMiner falls back to the default Latin‑1 mapping. | Pre‑process the PDF with `mutool clean -c` (from MuPDF) to rebuild the CMap, or fall back to OCR (`tesseract`) for those pages. |
| **“ReportLab generated PDFs are larger than expected”** | By default ReportLab embeds the full Helvetica, Times‑Roman, and Courier families even though they are standard 14 fonts that viewers already have. | Pass `pdfmetrics.registerStandardFonts()` and set `canvas.setPageCompression(1)`; also avoid embedding images at full resolution – down‑sample with Pillow before calling `drawImage`. |

---

## 6. Testing & debugging PDFs in 2012

### 6.1 Unit‑testing your PDF pipeline  
Because PDFs are binary blobs, a naïve `assertEqual(open(a).read(), open(b).read())` will often fail due to incremental updates or timestamp differences. A more robust strategy is:

```python
import hashlib, unittest, os
from mypdfmodule import process_pdf   # your wrapper that calls the chain above

class TestPDFPipeline(unittest.TestCase):
    def setUp(self):
        self.src = 'tests/fixtures/sample.pdf'
        self.tmp = 'tmp/out.pdf'

    def tearDown(self):
        if os.path.exists(self.tmp):
            os.remove(self.tmp)

    def test_merge_and_compress(self):
        # Run the full pipeline (merge → watermark → compress)
        process_pdf(self.src, self.tmp,
                    watermark='CONFIDENTIAL',
                    gs_preset='/screen')
        # Compare SHA‑256 digests rather than raw bytes
        with open(self.tmp, 'rb') as f:
            out_hash = hashlib.sha256(f.read()).hexdigest()
        # The expected hash was generated on a clean CI machine
        self.assertEqual(out_hash,
                         'e3b0c44298fc1c149afbf4c8996fb92427ae41e4649b934ca495991b7852b855')
```

*Tip:* Store the expected hash in a separate file (`expected.sha256`) so you can update it when the pipeline legitimately changes (e.g., you switch from `/screen` to `/ebook`).

### 6.2 Visual diff tools  
For layout‑sensitive bugs (watermarks misplaced, rotation wrong) use **diff‑pdf** (a small Qt utility) or the command‑line `compare` tool from ImageMagick:

```bash
convert -density 150 original.pdf[0] -quality 90 original.png
convert -density 150 processed.pdf[0] -quality 90 processed.png
compare -metric AE original.png processed.png diff.png
```

A non‑zero pixel count (`AE`) indicates a visual difference; the generated `diff.png` highlights exactly where the rendering diverges.

### 6.3 Logging external binaries  
When you wrap Ghostscript, PDFtk, or `pdfimages` with `subprocess`, capture both `stdout` and `stderr`:

```python
proc = subprocess.Popen(gs_cmd, stdout=subprocess.PIPE, stderr=subprocess.PIPE)
out, err = proc.communicate()
if proc.returncode != 0:
    logger.error("Ghostscript failed: %s", err.decode('utf-8'))
    raise RuntimeError(err)
else:
    logger.debug("Ghostscript output: %s", out.decode('utf-8'))
```

Having the full console dump makes it trivial to spot missing fonts, permission errors, or malformed PDF structures that would otherwise surface as cryptic `PdfReadError` exceptions.

---

## 7. Packaging your PDF‑toolchain for production

1. **Virtualenv (or `venv` on Python 3)** – Isolate the pure‑Python dependencies (`PyPDF2`, `pdfrw`, `ReportLab`, `PDFMiner`, `xhtml2pdf`).  
   ```bash
   virtualenv -p /usr/bin/python2.7 venv
   source venv/bin/activate
   pip install -r requirements.txt
   ```

2. **System‑level binaries** – Ghostscript, PDFtk, and the Poppler utilities are *not* installable via `pip`. On Debian/Ubuntu you can pin them in a `Dockerfile` or an Ansible playbook:
   ```Dockerfile
   FROM python:2.7-slim
   RUN apt-get update && apt-get install -y \
       ghostscript \
       pdftk \
       poppler-utils \
       && rm -rf /var/lib/apt/lists/*
   COPY . /app
   WORKDIR /app
   RUN pip install -r requirements.txt
   CMD ["python", "run_my_pipeline.py"]
   ```

3. **Entry‑point script** – Provide a thin CLI wrapper that parses arguments with `argparse` and dispatches to the appropriate helper functions. Example skeleton:

   ```python
   import argparse
   from mypdf import (merge_pdfs, stamp, compress_pdf,
                      encrypt_pdf, extract_images)

   def main():
       parser = argparse.ArgumentParser(description='2012‑style PDF toolkit')
       sub = parser.add_subparsers(dest='cmd')

       # merge
       p_merge = sub.add_parser('merge')
       p_merge.add_argument('inputs', nargs='+')
       p_merge.add_argument('-o', '--output', required=True)

       # watermark
       p_stamp = sub.add_parser('watermark')
       p_stamp.add_argument('src')
       p_stamp.add_argument('dst')
       p_stamp.add_argument('-t', '--text', required=True)

       # ... add other sub‑commands similarly ...

       args = parser.parse_args()
       if args.cmd == 'merge':
           merge_pdfs(args.inputs, args.output)
       elif args.cmd == 'watermark':
           stamp(args.src, args.dst, args.text)
       # etc.
   if __name__ == '__main__':
       main()
   ```

4. **Continuous Integration** – Even in 2012, many teams were already using Jenkins or Travis CI. Add a job that runs the unit‑test suite from §6 and also verifies that the external binaries are present (`which gs && which pdftk`). Fail the build early if any binary is missing.

---

## 8. Looking beyond 2012 (a brief note)

While this guide is anchored in the 2012 landscape, the concepts remain valid:

| 2012 tool | Modern equivalent (2024) |
|-----------|--------------------------|
| PyPDF2 (fork) | **pypdf** (actively maintained, supports Python 3.8+) |
| pdfrw | **pdfplumber** (for extraction) + **pdfium** bindings

| pdfrw | **pdfminer.six** (still works for low‑level manipulation, Python 3‑only) |
| ReportLab (open‑source) | **WeasyPrint** (HTML → PDF with CSS3 support) |
| PDFMiner | **pdfplumber** (high‑level text & table extraction) |
| xhtml2pdf / pisa | **WeasyPrint** or **wkhtmltopdf** (better CSS, faster) |
| Ghostscript | **MuPDF (fitz)** – lightweight rendering & compression library |
| PDFtk | **qpdf** – modern, supports AES‑256 encryption and linearization |
| poppler‑utils (pdfimages, pdftotext) | **pdf2image** (Python wrapper around Poppler) |

### 8.1 When to upgrade

* **Python 3 migration** – If you’re starting a new project, prefer `pypdf` (the successor to PyPDF2) and `pdfminer.six`. Both have clean 3‑only codebases and receive security patches.  
* **AES‑256 encryption** – Switch to `qpdf` or `PyMuPDF` (the Python bindings for MuPDF) for modern security requirements.  
* **High‑quality image handling** – `pdf2image` + Pillow give you DPI‑aware rasterisation without the Ghostscript overhead.  
* **HTML → PDF with CSS3** – WeasyPrint renders modern CSS (flexbox, grid) that `xhtml2pdf` cannot handle.  

Even if you stay on a 2012 stack for legacy reasons, keeping an eye on these newer tools makes the eventual migration smoother.

---

## 9. Full‑stack example: “Invoice generator” in 2012 style

Below is a compact script that ties together the most common steps a small business might need each month:

```python
#!/usr/bin/env python
# -*- coding: utf-8 -*-

import os, sys, hashlib, subprocess, argparse
from datetime import datetime
import markdown
from xhtml2pdf import pisa
import PyPDF2

# ----------------------------------------------------------------------
# Helper functions (re‑used from earlier sections)
# ----------------------------------------------------------------------
def md_to_pdf(md_path, pdf_path):
    with open(md_path, 'r') as f:
        html = markdown.markdown(f.read(),
                                 extensions=['tables', 'fenced_code'])
    html = '<html><body>{}</body></html>'.format(html)
    with open(pdf_path, 'wb') as out_f:
        if pisa.CreatePDF(src=html, dest=out_f).err:
            raise RuntimeError('Failed to render {}'.format(md_path))

def merge_pdfs(inputs, output):
    writer = PyPDF2.PdfFileWriter()
    for p in inputs:
        r = PyPDF2.PdfFileReader(open(p, 'rb'))
        for i in range(r.getNumPages()):
            writer.addPage(r.getPage(i))
    with open(output, 'wb') as f:
        writer.write(f)

def compress_pdf(src, dst, preset='/ebook'):
    gs_cmd = [
        'gs', '-sDEVICE=pdfwrite',
        '-dCompatibilityLevel=1.4',
        '-dPDFSETTINGS={}'.format(preset),
        '-dNOPAUSE', '-dQUIET', '-dBATCH',
        '-sOutputFile={}'.format(dst), src
    ]
    subprocess.check_call(gs_cmd)

def encrypt_pdf(src, dst, user_pwd, owner_pwd):
    r = PyPDF2.PdfFileReader(open(src, 'rb'))
    w = PyPDF2.PdfFileWriter()
    for p in r.pages:
        w.addPage(p)
    w.encrypt(user_pwd, owner_pwd, use_128bit=True)
    with open(dst, 'wb') as f:
        w.write(f)

# ----------------------------------------------------------------------
# Main pipeline
# ----------------------------------------------------------------------
def build_invoice(md_dir, logo_path, out_dir,
                  user_pwd='', owner_pwd=''):
    # 1. Convert each Markdown invoice to PDF
    pdfs = []
    for md_file in sorted(os.listdir(md_dir)):
        if not md_file.lower().endswith('.md'):
            continue
        base = os.path.splitext(md_file)[0]
        pdf_path = os.path.join(out_dir, base + '.pdf')
        md_to_pdf(os.path.join(md_dir, md_file), pdf_path)

        # 2. Stamp company logo as a watermark (simple overlay)
        #    Here we reuse the earlier stamp() function but with a logo image.
        #    For brevity we call an external ImageMagick command.
        watermarked = os.path.join(out_dir, base + '_wm.pdf')
        cmd = [
            'convert', pdf_path,
            '-gravity', 'center',
            '-draw', "image over 0,0 0,0 '{}'".format(logo_path),
            watermarked
        ]
        subprocess.check_call(cmd)
        pdfs.append(watermarked)

    # 3. Merge all invoices into a single PDF for the accountant
    merged = os.path.join(out_dir, 'all_invoices_merged.pdf')
    merge_pdfs(pdfs, merged)

    # 4. Compress for email attachment
    compressed = os.path.join(out_dir, 'all_invoices_compressed.pdf')
    compress_pdf(merged, compressed, preset='/screen')

    # 5. Encrypt with a user‑password (client) and owner‑password (admin)
    final = os.path.join(out_dir, 'all_invoices_final.pdf')
    encrypt_pdf(compressed, final, user_pwd, owner_pwd)

    # 6. Print a SHA‑256 checksum for integrity verification
    with open(final, 'rb') as f:
        digest = hashlib.sha256(f.read()).hexdigest()
    print('✅ Finished! Final invoice PDF: {}'.format(final))
    print('   SHA‑256: {}'.format(digest))

# ----------------------------------------------------------------------
if __name__ == '__main__':
    parser = argparse.ArgumentParser(description='2012‑style invoice pipeline')
    parser.add_argument('--md-dir', required=True, help='Directory with .md invoices')
    parser.add_argument('--logo', required=True, help='Path to PNG logo for watermark')
    parser.add_argument('--out-dir', default='out', help='Where to write intermediate PDFs')
    parser.add_argument('--user-pwd', default='', help='Password required to open the PDF')
    parser.add_argument('--owner-pwd', default='admin', help='Owner password for permissions')
    args = parser.parse_args()

    os.makedirs(args.out_dir, exist_ok=True)
    build_invoice(args.md_dir, args.logo, args.out_dir,
                  user_pwd=args.user_pwd, owner_pwd=args.owner_pwd)
```

**What this script demonstrates**

1. **Pure‑Python conversion** (`markdown → PDF`) via `xhtml2pdf`.  
2. **Image‑based watermarking** using the ubiquitous `convert` (ImageMagick) – a common 2012 shortcut when you need a logo overlay without fiddling with `pdfrw`.  
3. **Merging & compression** with `PyPDF2` and Ghostscript.  
4. **Password protection** (owner vs. user) for compliance with client‑side confidentiality policies.  
5. **Checksum generation** – a simple integrity check that can be emailed alongside the PDF.

All external binaries (`gs`, `convert`) are invoked via `subprocess`, mirroring the “wrap‑the‑CLI” idiom that dominated Python PDF work in 2012.

---

## 10. TL;DR Recap (for the impatient)

| Goal | Pure‑Python library | External tool (if needed) |
|------|---------------------|---------------------------|
| Merge / split / rotate | **PyPDF2 / pdfrw** | – |
| Add watermarks / logos | **pdfrw + ReportLab** | **ImageMagick** for image overlays |
| High‑quality size reduction | – | **Ghostscript** (`-dPDFSETTINGS`) |
| Password protection (RC4) | **PyPDF2** | **PDFtk / qpdf** for AES‑256 |
| Text extraction (layout) | **PDFMiner** | **tesseract** for OCR on scans |
| HTML/Markdown → PDF | **xhtml2pdf** | **WeasyPrint / wkhtmltopdf** (modern) |
| Image extraction | – | **pdfimages** (poppler) |
| Linearization / fast web view | – | **PDFtk** (`linearize`) or **qpdf** |

With these building blocks you can script virtually any PDF workflow that a 2012‑era Python developer would need, while keeping the door open for a painless upgrade to the 2020s toolchain.

---

## 11. Frequently asked questions (2012 edition)

| Question | Short answer | Where to dig deeper |
|----------|--------------|---------------------|
| **Can I edit the PDF metadata (author, creation date) without touching the pages?** | Yes – open the file with `PdfFileReader`, copy the `Info` dictionary to a new `PdfFileWriter`, modify the fields, then write the file. | `reader.getDocumentInfo()` → `writer.addMetadata(info_dict)` |
| **Is there a way to “flatten” form fields so the values become part of the content stream?** | Not directly in PyPDF2. The usual workaround is to print the PDF to a new PDF with Ghostscript (`gs -dNOPAUSE -dBATCH -sDEVICE=pdfwrite -sOutputFile=flat.pdf filled.pdf`). This rasterises the form fields into the page content. | Ghostscript `-dPDFSETTINGS=/prepress` for best quality. |
| **My PDF contains JavaScript actions (e.g., auto‑print). Will the libraries preserve them?** | Most pure‑Python tools treat JavaScript as just another PDF object and will preserve it during merge/split, but they do not execute or validate it. If you need to strip JavaScript for security, use `pdftk` with the `cat` command and the `output` option, which discards interactive elements. | `pdftk in.pdf cat output out.pdf` |
| **How do I handle PDFs that use the newer “X‑Ref stream” introduced in PDF‑1.5?** | PyPDF2 and pdfrw in their 2012 releases can read X‑Ref streams, but they sometimes mis‑interpret the object offsets, leading to “PdfReadError”. The safest route is to run `qpdf --linearize` or `pdftk` to rewrite the file into a classic cross‑reference table before processing. | `qpdf --qdf in.pdf out.qdf` – the QDF format is human‑readable and can be inspected with a text editor. |
| **Can I add a digital signature (PKI) using only Python?** | Not with the 2012‑era pure‑Python libraries. You need an external tool such as `openssl` + `pdfsig` (part of the Poppler suite) or a commercial SDK (e.g., iText). | `pdfsig --sign --cert mycert.pem --key mykey.pem input.pdf output.pdf` |

---

## 12. Where to find help and keep the stack alive

* **Mailing lists & forums** – The `pdf-tools` Google Group (still active in 2012) is the primary place for PyPDF2 and pdfrw questions.  
* **Stack Overflow tags** – Use `pypdf2`, `pdfrw`, `reportlab`, `ghostscript` to surface existing answers.  
* **GitHub forks** – The original PyPDF2 repository is no longer maintained; many developers have forked it (e.g., `pypdf/pypdf2`). When you encounter a bug, check the fork’s issue tracker before filing a new one.  
* **Linux distro packages** – On Debian/Ubuntu, the `python-pypdf2`, `python-pdfrw`, `python-reportlab`, `ghostscript`, `poppler-utils`, and `pdftk` packages are kept reasonably up‑to‑date. Installing via `apt-get` ensures binary compatibility with the system’s `glibc`.  
* **Documentation snapshots** – Because many of the original project sites have gone offline, the Internet Archive’s Wayback Machine is a surprisingly reliable source for the 2012 README files and API docs.

---

## 13. Performance tips for large‑scale batch jobs

1. **Avoid repeated opening/closing** – When you need to process hundreds of PDFs, keep a single `PdfFileReader` instance alive for the duration of the operation and reuse its underlying file object.  
2. **Stream‑based merging** – Instead of loading every page into memory, use `PdfFileWriter.appendPagesFromReader(reader)` which streams pages directly to the output file.  
3. **Parallelise at the file level** – The Python `multiprocessing` module works well because each worker can invoke its own Ghostscript or PDFtk subprocess without contention.  
4. **Cache external binary calls** – Ghostscript’s startup overhead is non‑trivial. For massive batches, consider using the `-dNOPAUSE -dBATCH` flags together with a single long‑running process that reads from `stdin` (e.g., via a named pipe).  
5. **Monitor file descriptors** – On Unix, each `subprocess.Popen` call opens a pipe; forgetting to close them can quickly exhaust the limit on a busy CI server. Use `with subprocess.Popen(...) as proc:` or explicitly call `proc.communicate()`.

---

## 14. Final thoughts

Even though the Python ecosystem has evolved dramatically since 2012, the core ideas presented here—*incremental updates*, *external‑binary orchestration*, and *layered abstraction*—remain the backbone of any robust PDF automation pipeline. By mastering the tools listed in this guide you’ll be able to:

* Build end‑to‑end invoice, reporting, or archival workflows that run on modest hardware.  
* Keep your codebase portable across Linux, macOS, and Windows (the only truly cross‑platform requirement is a working Ghostscript/PDFtk install).  
* Future‑proof your scripts: swapping out `PyPDF2` for `pypdf` or `pdftk` for `qpdf` is a matter of changing a few import lines and command‑line flags, not a complete rewrite.

So roll up your sleeves, fire up a virtualenv, and let Python do the heavy lifting on PDFs—just as developers did back in 2012, only now with a clearer roadmap for the years ahead.

---

Tags: tag-1, tag-2, tag-3
Slug: article-slug