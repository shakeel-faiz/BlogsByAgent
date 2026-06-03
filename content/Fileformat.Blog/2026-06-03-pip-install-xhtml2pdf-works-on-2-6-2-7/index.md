---
seoTitle: "pip install xhtml2pdf   # works on 2.6/2.7"
title: "pip install xhtml2pdf   # works on 2.6/2.7"
description: "Learn the 2011 Python PDF toolkit: ReportLab, PyPDF2, Ghostscript, PDFtk & more for HTML‑to‑PDF, merging, splitting, compression and security."
date: 03 Jun 2026
draft: true
author: Khan AI
url: /audio/pip-install-xhtml2pdf-works-on-2-6-2-7/
categories: ['Audio']
tags: ['pip install xhtml2pdf   # works on 2.6/2.7', 'MP4', 'Some Tag']
---

**TL;DR** – In 2011 you’ll need a mix of pure‑Python libraries (ReportLab, PyPDF2, pdfrw, PDFMiner) and battle‑tested command‑line tools (Ghostscript, PDFtk, qpdf, Poppler). Use ReportLab/xhtml2pdf for HTML → PDF, PyPDF2 or PDFtk for merging/splitting/rotating, Ghostscript for down‑sampling and linearizing, and PyPDF2 or PDFtk for password protection. The code snippets below work on Python 2.6/2.7 on Windows, Linux or macOS.

---

## 1. The 2011 PDF Landscape for Python Developers  

- **Python version** – Most projects still run on 2.6 or the freshly‑released 2.7. All the libraries we’ll touch support both, but you’ll see a lot of `with open(..., 'rb') as f:` context‑manager usage now that 2.7 makes it painless.  
- **No built‑in PDF support** – The stdlib only gives you binary I/O, so you have to reach for third‑party packages or external tools.  
- **Licensing matters** – ReportLab, pdfrw, PyPDF2 are BSD/MIT, while PDFtk Server is free for non‑commercial use. If you need a commercial‑grade SDK you’d have to buy something (iText is Java‑only and GPL‑licensed).  
- **Performance tip** – Pure‑Python manipulation is fine for small documents, but for anything bigger than a few megabytes you’ll get better speed and reliability by shelling out to Ghostscript, PDFtk or qpdf.

---

## 2. Converting Anything to/from PDF  

### 2.1. HTML → PDF (ReportLab + xhtml2pdf)  

ReportLab is the de‑facto PDF generator. In 2011 the `xhtml2pdf` (formerly `pisa`) wrapper gave you a quick way to turn simple HTML into a PDF. CSS support is limited (no flexbox, no media queries), but for invoices, reports, or simple docs it works nicely.

```python
# pip install xhtml2pdf   # works on 2.6/2.7
from xhtml2pdf import pisa

def html_to_pdf(html_string, out_path):
    """Render a tiny HTML snippet to a PDF file."""
    with open(out_path, "wb") as out_file:
        # pisa.CreatePDF returns a status object; .err == 0 on success
        status = pisa.CreatePDF(src=html_string, dest=out_file)
    return not status.err

# Example usage
html = """
<h1>Monthly Report</h1>
<p>Generated on <strong>June 3, 2011</strong></p>
<ul><li>Revenue: $12 345</li><li>Expenses: $8 910</li></ul>
"""
html_to_pdf(html, "report.pdf")
```

### 2.2. Image → PDF (PIL + ReportLab)  

If you have a collection of PNG/JPEG files you want to bundle into a PDF, Pillow (the fork of PIL) can read the images and ReportLab can place each on its own page.

```python
from PIL import Image
from reportlab.pdfgen import canvas
from reportlab.lib.pagesizes import letter

def images_to_pdf(image_paths, out_path):
    c = canvas.Canvas(out_path, pagesize=letter)
    width, height = letter
    for img_path in image_paths:
        img = Image.open(img_path)
        img_width, img_height = img.size
        # Scale to fit the page while preserving aspect ratio
        scale = min(width / img_width, height / img_height)
        c.drawImage(img_path,
                    x=0,
                    y=0,
                    width=img_width * scale,
                    height=img_height * scale)
        c.showPage()
    c.save()

images_to_pdf(['page1.png', 'page2.jpg'], 'slides.pdf')
```

### 2.3. PDF → Image or Text (Ghostscript & Poppler)  

For rasterizing PDFs you’ll call Ghostscript (`gs`). For plain‑text extraction the Poppler utility `pdftotext` is lightning‑fast.

```python
import subprocess
import os

def pdf_to_png(pdf_path, out_dir, dpi=150):
    """Render each page of a PDF to a PNG using Ghostscript."""
    if not os.path.isdir(out_dir):
        os.makedirs(out_dir)
    cmd = [
        "gs", "-dNOPAUSE", "-dBATCH", "-sDEVICE=pngalpha",
        "-r%d" % dpi,
        "-sOutputFile=%s/page-%%03d.png" % out_dir,
        pdf_path
    ]
    subprocess.check_call(cmd)

def pdf_to_text(pdf_path, out_txt):
    """Extract Unicode text with Poppler's pdftotext."""
    subprocess.check_call(["pdftotext", "-layout", pdf_path, out_txt])

pdf_to_png("report.pdf", "pages")
pdf_to_text("report.pdf", "report.txt")
```

---

## 3. Editing PDFs – Merge, Split, Rotate, Watermark  

### 3.1. Pure‑Python with PyPDF2  

PyPDF2 (first public release in 2011) lets you manipulate the PDF object tree without leaving Python. It’s perfect for quick merges, splits, and page‑level rotations.

```python
import PyPDF2

def merge_pdfs(src_files, dst_file):
    merger = PyPDF2.PdfFileMerger()
    for f in src_files:
        merger.append(f)               # you can also pass a (start, end) tuple
    with open(dst_file, "wb") as out:
        merger.write(out)

def split_pdf(src_file, page_numbers, dst_file):
    """Extract the given page_numbers (0‑based) into a new PDF."""
    reader = PyPDF2.PdfFileReader(open(src_file, "rb"))
    writer = PyPDF2.PdfFileWriter()
    for p in page_numbers:
        writer.addPage(reader.getPage(p))
    with open(dst_file, "wb") as out:
        writer.write(out)

def rotate_page(src_file, page_index, angle, dst_file):
    """Rotate a single page clockwise (90, 180, 270)."""
    reader = PyPDF2.PdfFileReader(open(src_file, "rb"))
    writer = PyPDF2.PdfFileWriter()
    for i in range(reader.getNumPages()):
        page = reader.getPage(i)
        if i == page_index:
            page.rotateClockwise(angle)
        writer.addPage(page)
    with open(dst_file, "wb") as out:
        writer.write(out)
```

### 3.2. When PyPDF2 Isn’t Enough – PDFtk Server  

PDFtk is a tiny Java‑based command‑line tool that handles large files and complex operations (e.g., stamping a watermark PDF over every page). Call it via `subprocess`.

```python
def stamp_watermark(pdf_path, watermark_path, out_path):
    """Overlay `watermark_path` on every page of `pdf_path`."""
    cmd = [
        "pdftk", pdf_path, "background", watermark_path,
        "output", out_path
    ]
    subprocess.check_call(cmd)

# Example
stamp_watermark("report.pdf", "logo_watermark.pdf", "report_watermarked.pdf")
```

---

## 4. Compressing PDFs – Down‑sample, Re‑compress, Linearize  

### 4.1. Image Down‑sampling with Ghostscript  

Ghostscript’s `-dPDFSETTINGS` switch gives you four quality presets that automatically down‑sample images, discard unused objects, and (optionally) linearize for web viewing.

```python
def compress_pdf_gs(src, dst, preset="/screen"):
    """
    preset can be:
      /screen   – 72 dpi, smallest size
      /ebook    – 150 dpi, decent quality
      /printer  – 300 dpi, high quality
      /prepress – 300 dpi + all fonts embedded
    """
    cmd = [
        "gs", "-sDEVICE=pdfwrite", "-dCompatibilityLevel=1.4",
        "-dPDFSETTINGS=%s" % preset,
        "-dNOPAUSE", "-dQUIET", "-dBATCH",
        "-sOutputFile=%s" % dst,
        src
    ]
    subprocess.check_call(cmd)

compress_pdf_gs("report.pdf", "report_compressed.pdf", "/ebook")
```

### 4.2. Stream Re‑compression with qpdf  

If you only need to recompress Flate streams (e.g., after removing pages), `qpdf` does it

### 4.3. Stream‑level Re‑compression with **qpdf**  

`qpdf` is a lightweight C++ utility that can rewrite a PDF without touching its visual content. It is especially handy when you have already stripped pages with PyPDF2 or PDFtk and want to “clean up” the file size by recompressing the remaining object streams.

```python
import subprocess
import shlex

def qpdf_recompress(src, dst, linearize=False):
    """
    Re‑compress all Flate streams in `src` and write the result to `dst`.
    If `linearize` is True, the output will be web‑optimized (fast‑start).
    """
    cmd = ["qpdf", "--compress-streams=y"]
    if linearize:
        cmd.append("--linearize")
    cmd.extend([src, dst])
    # Using check_call will raise if qpdf returns a non‑zero exit status.
    subprocess.check_call(cmd)

# Example – simple recompression
qpdf_recompress("report_compressed.pdf", "report_final.pdf")
# Example – recompress + linearize for HTTP streaming
qpdf_recompress("report_compressed.pdf", "report_linear.pdf", linearize=True)
```

**Why qpdf?**  
* It preserves the original PDF version (e.g., 1.4, 1.5) unless you explicitly ask for a downgrade.  
* It can also **flatten** form fields (`--flatten-annotations`) or **remove** JavaScript (`--remove-annotations`).  
* The command‑line interface is stable across Windows, macOS, and Linux, making it a reliable fallback when pure‑Python libraries hit a corner case.

---

## 5. Securing PDFs – Passwords, Permissions, and Encryption  

### 5.1. Adding User/Owner Passwords with **PyPDF2**  

PyPDF2 can encrypt a PDF with either a *user* password (required to open) or an *owner* password (required to change permissions). The library uses the legacy RC4‑40/128‑bit encryption; for stronger AES‑256 you’ll need `qpdf` or `pdftk` (which delegate to the underlying PDF library).

```python
def encrypt_pdf(src, dst, user_pwd=None, owner_pwd=None, use_128bit=True):
    """
    Encrypt `src` and write to `dst`.
    - user_pwd: password required to open the file (None = no user password)
    - owner_pwd: password required to change permissions (None = same as user_pwd)
    - use_128bit: True for 128‑bit RC4 (default), False for 40‑bit (legacy)
    """
    reader = PyPDF2.PdfFileReader(open(src, "rb"))
    writer = PyPDF2.PdfFileWriter()
    for page_num in range(reader.getNumPages()):
        writer.addPage(reader.getPage(page_num))

    writer.encrypt(user_pwd or "", owner_pwd or "", use_128bit)
    with open(dst, "wb") as out:
        writer.write(out)

# Example – protect a report with a user password
encrypt_pdf("report.pdf", "report_protected.pdf",
            user_pwd="openme", owner_pwd="admin123")
```

### 5.2. Permissions and Encryption with **PDFtk**  

PDFtk Server can set granular permissions (printing, copying, modifying) while also supporting 128‑bit RC4 encryption. Its syntax is a bit more verbose but works on PDFs larger than 100 MB where PyPDF2 may run out of memory.

```python
def pdftk_protect(src, dst, user_pwd, owner_pwd,
                  allow_print=True, allow_modify=False,
                  allow_copy=False, allow_annotate=False):
    """
    Use PDFtk to encrypt `src`. Permissions are boolean flags.
    """
    perm_flags = []
    if allow_print:      perm_flags.append("Print")
    if allow_modify:     perm_flags.append("Modify")
    if allow_copy:       perm_flags.append("Copy")
    if allow_annotate:   perm_flags.append("Annotate")
    perm_str = " ".join(perm_flags) if perm_flags else "None"

    cmd = [
        "pdftk", src, "output", dst,
        "owner_pw", owner_pwd,
        "user_pw", user_pwd,
        "allow", perm_str
    ]
    subprocess.check_call(cmd)

# Example – allow only printing and copying
pdftk_protect("report.pdf", "report_secure.pdf",
              user_pwd="readme", owner_pwd="admin123",
              allow_print=True, allow_copy=True)
```

### 5.3. AES‑256 Encryption with **qpdf**  

If you need modern AES‑256 encryption (required for many compliance regimes), `qpdf` is the tool of choice. It also lets you set the same permission flags as PDFtk.

```python
def qpdf_aes256_encrypt(src, dst, user_pwd, owner_pwd,
                        allow_print=False, allow_modify=False,
                        allow_copy=False, allow_annotate=False):
    perm = []
    if allow_print:      perm.append("print")
    if allow_modify:     perm.append("modify")
    if allow_copy:       perm.append("copy")
    if allow_annotate:   perm.append("annotate")
    perm_opt = "--allow=%s" % ",".join(perm) if perm else "--allow=none"

    cmd = [
        "qpdf", "--encrypt", user_pwd, owner_pwd,
        "256", "--", src, dst,
        perm_opt
    ]
    subprocess.check_call(cmd)

# Example – AES‑256 with no permissions (read‑only)
qpdf_aes256_encrypt("report.pdf", "report_aes256.pdf",
                    user_pwd="readme", owner_pwd="admin123")
```

---

## 6. Adding Metadata, Bookmarks, and Attachments  

### 6.1. Updating Document Information Dictionary  

Both PyPDF2 and ReportLab expose the PDF’s *Info* dictionary. You can set title, author, subject, and custom keys.

```python
def set_metadata(pdf_path, out_path, **info):
    """
    `info` may contain keys like Title, Author, Subject, Keywords.
    """
    reader = PyPDF2.PdfFileReader(open(pdf_path, "rb"))
    writer = PyPDF2.PdfFileWriter()
    writer.appendPagesFromReader(reader)

    # Build a new info dict; existing entries are overwritten.
    meta = writer._info.getObject()
    for key, value in info.items():
        meta.update({PyPDF2.generic.NameObject("/%s" % key): PyPDF2.generic.createStringObject(value)})

    with open(out_path, "wb") as out:
        writer.write(out)

set_metadata("report.pdf", "report_meta.pdf",
             Title="June 2011 Sales Report",
             Author="Acme Analytics",
             Subject="Quarterly Financials",
             Keywords="sales, 2011, Q2")
```

### 6.2. Creating Bookmarks (Outlines)  

Bookmarks improve navigation in multi‑chapter PDFs. PyPDF2 can add them via the `addBookmark` method.

```python
def add_bookmarks(pdf_path, out_path, bookmarks):
    """
    `bookmarks` is a list of tuples: (title, page_number, parent_index)
    parent_index = None for top‑level entries.
    """
    reader = PyPDF2.PdfFileReader(open(pdf_path, "rb"))
    writer = PyPDF2.PdfFileWriter()
    writer.appendPagesFromReader(reader)

    # Keep a list of bookmark references so we can attach children.
    refs = []
    for title, page, parent in bookmarks:
        if parent is None:
            ref = writer.addBookmark(title, page)
        else:
            ref = writer.addBookmark(title, page, parent=refs[parent])
        refs.append(ref)

    with open(out_path, "wb") as out:
        writer.write(out)

# Example – simple two‑level outline
add_bookmarks("report.pdf", "report_bookmarked.pdf", [
    ("Executive Summary", 0, None),
    ("Financial Overview", 2, None),
    ("Revenue Details", 3, 1),   # child of "Financial Overview"
    ("Expense Breakdown", 5, 1),
])
```

### 6.3. Embedding File Attachments  

PDFs can carry arbitrary files (e.g., CSV data, source code) as *FileAttachment* annotations. The most reliable way in 2011 is to use `pdftk`’s `attach_files` operation.

```python
def attach_files(pdf_path, out_path, files):
    """
    `files` is a list of file paths to embed.
    """
    cmd = ["pdftk

```python
    # Build the pdftk command:
    #   pdftk input.pdf attach_files file1 file2 output out.pdf
    cmd = ["pdftk", pdf_path, "attach_files"] + files + ["output", out_path]
    subprocess.check_call(cmd)

# Example – attach a CSV export and the original data file
attach_files("report.pdf", "report_with_attachments.pdf",
             ["sales_q2_2011.csv", "raw_data.xlsx"])
```

> **Note:** `pdftk` stores attachments as *FileAttachment* annotations, which are visible in most PDF viewers under the “Attachments” pane. Pure‑Python libraries (PyPDF2, pdfrw) can read these annotations but, as of 2011, they cannot create them reliably, so delegating to `pdftk` remains the safest route.

---

## 7. Working with PDF Forms (AcroForms)

Many business documents ship with fillable fields (text boxes, check‑boxes, radio groups). In 2011 the support for programmatic form filling is still a bit rough, but you can get the job done with a combination of **pdftk**, **PyPDF2**, and **pdfminer**.

### 7.1. Exporting Form Data  

`pdftk` can dump the current values of an interactive form to a simple `FDF` (Forms Data Format) file:

```python
def export_fdf(pdf_path, fdf_path):
    subprocess.check_call(["pdftk", pdf_path, "dump_data_fields", "output", fdf_path])

export_fdf("invoice_template.pdf", "invoice_data.fdf")
```

The resulting `FDF` is a plain‑text key/value list that you can parse with Python’s `configparser` or simply treat as a dictionary.

### 7.2. Importing Form Data  

Conversely, you can fill a template by feeding an `FDF` back into `pdftk`:

```python
def fill_form(template_pdf, fdf_path, out_pdf):
    subprocess.check_call([
        "pdftk", template_pdf, "fill_form", fdf_path,
        "output", out_pdf, "flatten"
    ])

# Suppose you generated `filled.fdf` programmatically:
fill_form("invoice_template.pdf", "filled.fdf", "invoice_filled.pdf")
```

The `flatten` flag merges the field values into the page content, removing the interactive elements—useful when you want a final, non‑editable PDF.

### 7.3. Reading Form Fields with PyPDF2  

If you only need to inspect the field names (e.g., to build a dynamic UI), PyPDF2 can expose the `/AcroForm` dictionary:

```python
def list_form_fields(pdf_path):
    reader = PyPDF2.PdfFileReader(open(pdf_path, "rb"))
    fields = reader.getFields()
    if not fields:
        return []
    return list(fields.keys())

print(list_form_fields("invoice_template.pdf"))
# → ['CustomerName', 'InvoiceDate', 'TotalAmount', ...]
```

---

## 8. Common Pitfalls & Debugging Tips

| Symptom | Likely Cause | Quick Fix |
|---------|--------------|-----------|
| **`PdfReadError: EOF marker not found`** | Truncated or partially written PDF (often from an interrupted write) | Verify that the file was closed properly; use `with open(..., 'wb') as f:`. If the source is from a network stream, copy to a temporary file first. |
| **Ghostscript prints “Error: /undefined in …”** | The PDF references a font or XObject that Ghostscript cannot locate (often with embedded Type 3 fonts) | Add `-dPDFSETTINGS=/prepress` to force embedding of missing resources, or run `gs -dNOPAUSE -dBATCH -sDEVICE=pdfwrite -sOutputFile=clean.pdf original.pdf`. |
| **`pdftk` reports “Unable to open file” on Windows** | Path contains spaces or non‑ASCII characters, and the command line is not quoted correctly | Wrap each path in double quotes or use `subprocess.list2cmdline`. Example: `subprocess.check_call(['pdftk', input_path, 'output', out_path])`. |
| **PyPDF2 rotation appears to have no effect** | You rotated the page but then wrote the original `PdfFileReader` object instead of the modified `PdfFileWriter` | Ensure you add the (possibly rotated) page to the writer **after** calling `rotateClockwise`. |
| **Text extraction returns garbled characters** | The PDF uses a custom encoding or is scanned (image‑only) | Fall back to OCR (e.g., `tesseract`) for image PDFs, or use `pdfminer.six` which can handle many custom encodings. |

**Debugging workflow**:

1. **Validate the PDF** – Run `gs -dNOPAUSE -dBATCH -sDEVICE=pdfwrite -sOutputFile=/dev/null input.pdf`. Ghostscript will emit warnings if the file is malformed.
2. **Inspect the object tree** – Use `pdfminer`’s `pdf2txt.py -t raw input.pdf` to dump raw objects; look for missing `/Length` entries or broken cross‑reference tables.
3. **Isolate the step** – If a pipeline fails (e.g., merge → compress → encrypt), write the intermediate file to disk and test each stage independently.
4. **Check versions** – Some older PDFs (pre‑1.4) trigger bugs in early PyPDF2 releases; upgrading to the latest 1.26+ version (released late‑2011) often resolves them.

---

## 9. Testing Your PDF Processing Code

Automated tests are essential because PDFs are binary and subtle changes can break downstream consumers.

### 9.1. Using `pytest` with Temporary Files  

```python
import pytest, os, shutil
from pathlib import Path

@pytest.fixture
def tmp_pdf(tmp_path):
    # Copy a known‑good fixture PDF into a temporary directory
    src = Path(__file__).parent / "fixtures" / "sample.pdf"
    dst = tmp_path / "sample.pdf"
    shutil.copy(src, dst)
    return dst

def test_merge_two_pdfs(tmp_pdf, tmp_path):
    from mypdfutils import merge_pdfs
    second = tmp_path / "second.pdf"
    # Create a tiny second PDF on the fly
    from reportlab.pdfgen import canvas
    c = canvas.Canvas(str(second))
    c.drawString(100, 750, "Second document")
    c.save()

    out = tmp_path / "merged.pdf"
    merge_pdfs([str(tmp_pdf), str(second)], str(out))

    # Verify page count
    import PyPDF2
    reader = PyPDF2.PdfFileReader(open(out, "rb"))
    assert reader.getNumPages() == 2
```

### 9.2. Regression Testing with Checksums  

After a transformation (e.g., compression), compute a SHA‑256 hash of the output and compare it against a stored “golden” hash. This catches accidental changes in image quality or metadata.

```python
import hashlib, pathlib

def sha256_of_file(path):
    h = hashlib.sha256()
    with open(path, "rb") as f:
        for chunk in iter(lambda: f.read(8192), b""):
            h.update(chunk)
    return h.hexdigest()

# In a CI job:
assert sha256_of_file("report_final.pdf") == "3a7f9c2e5d…"
```

### 9.3. Cross‑Platform CI  

Because the pipeline relies on external binaries (Ghostscript, qpdf, pdftk), your CI environment must install them. On Travis CI (Linux/macOS) you can add:

```yaml
addons:
  apt:
    packages:
      - ghostscript
      - qpdf
      - pdftk
```

On Windows runners, use Chocolatey:

```powershell
choco install ghostscript qpdf pdftk
```

---

## 10. Deploying to Production

1. **Bundle the binaries** – For a self‑contained

1. **Bundle the binaries** – For a self‑contained deployment you’ll want to ship the required command‑line tools alongside your Python code. On Linux you can copy the `gs`, `qpdf`, and `pdftk` executables into a `bin/` directory inside your project and prepend that directory to `PATH` at runtime:

```python
import os, sys

PROJECT_ROOT = os.path.abspath(os.path.dirname(__file__))
BIN_DIR = os.path.join(PROJECT_ROOT, "bin")
os.environ["PATH"] = BIN_DIR + os.pathsep + os.environ.get("PATH", "")
```

On Windows you can place the `.exe` files in the same folder as your script or use a small installer (e.g., NSIS) that registers them in the system `PATH`. Remember to ship the matching license files (Ghostscript’s AGPL, PDFtk’s free‑for‑non‑commercial notice, qpdf’s Apache‑2.0) so you stay compliant.

2. **Virtual environments & pinning** – Because the Python PDF ecosystem evolves quickly, lock your dependencies with `pip freeze > requirements.txt`. In production, recreate the environment with:

```bash
python -m venv venv
source venv/bin/activate   # or venv\Scripts\activate on Windows
pip install -r requirements.txt
```

If you need to support both Python 2.6 and 2.7, keep two separate requirement files (`requirements-2.6.txt`, `requirements-2.7.txt`) and test each in your CI pipeline.

3. **Graceful degradation** – Not every client will have the external tools installed. Wrap every subprocess call in a helper that falls back to a pure‑Python implementation when the binary is missing:

```python
def run_or_fallback(cmd, fallback_func, *args, **kwargs):
    try:
        subprocess.check_call(cmd)
    except (OSError, subprocess.CalledProcessError):
        # Binary not found or failed – use the Python fallback
        fallback_func(*args, **kwargs)
```

For example, if `gs` is unavailable you can use Pillow to rasterize a single‑page PDF (albeit much slower) or simply reject the request with a clear error message.

4. **Logging & monitoring** – PDF processing can be CPU‑intensive and may fail silently (e.g., a malformed PDF that Ghostscript discards). Use the standard `logging` module to capture stdout/stderr from subprocesses:

```python
import logging, subprocess

log = logging.getLogger("pdfpipeline")

def run_cmd(cmd):
    proc = subprocess.Popen(
        cmd, stdout=subprocess.PIPE, stderr=subprocess.PIPE, universal_newlines=True
    )
    out, err = proc.communicate()
    if out:
        log.debug("Command output: %s", out.strip())
    if err:
        log.warning("Command error: %s", err.strip())
    if proc.returncode != 0:
        raise RuntimeError(f"Command {' '.join(cmd)} failed with exit code {proc.returncode}")
```

Integrate this with your web framework’s request‑logging so you can spot spikes in processing time or repeated failures.

5. **Performance tuning** – For high‑throughput services (e.g., a PDF‑generation micro‑service handling dozens of requests per second) consider the following:

   * **Pool external processes** – Spawning a new Ghostscript process for every request adds overhead. Use a process pool (`multiprocessing.Pool` or `concurrent.futures.ProcessPoolExecutor`) and keep a few workers warm.
   * **Chunked streaming** – When serving large PDFs, stream the file to the client instead of loading it entirely into memory:

   ```python
   from flask import send_file

   @app.route("/download/<path:filename>")
   def download(filename):
       return send_file(
           os.path.join(PDF_ROOT, filename),
           mimetype="application/pdf",
           as_attachment=True,
           conditional=True   # enables range requests
       )
   ```

   * **Cache intermediate results** – If you repeatedly compress the same source document, store the compressed version in a cache (Redis, memcached, or a simple on‑disk hash‑based store) and reuse it.

6. **Security hardening** – PDFs can contain malicious JavaScript, embedded files, or malformed structures that trigger vulnerabilities in the underlying tools. Mitigate risk by:

   * Running all external binaries in a sandboxed user account with no write permissions outside a temporary directory.
   * Limiting the size of uploaded PDFs (e.g., reject files > 50 MB) to prevent denial‑of‑service attacks.
   * Stripping potentially dangerous content before further processing:

   ```bash
   qpdf --qdf --object-streams=disable input.pdf cleaned.qdf
   qpdf --linearize cleaned.qdf -o cleaned.pdf
   ```

   The `--object-streams=disable` flag forces all objects into the older, more predictable format, reducing the attack surface.

7. **Future‑proofing** – While the article focuses on the 2011 landscape, the same principles apply to newer Python versions:

   * **PyPDF4 / pypdf** – Successor forks of PyPDF2 that add better encryption support and Python 3 compatibility.
   * **pdfplumber** – A higher‑level wrapper around PDFMiner for table extraction.
   * **WeasyPrint** – A modern HTML → PDF engine that supersedes xhtml2pdf for complex layouts.

   When you upgrade your runtime, replace the old imports with their newer equivalents and run your test suite to verify that no regressions have been introduced.

---

## 11. TL;DR Recap (Re‑readable Checklist)

| Goal | Recommended Tool(s) | One‑liner Command / Code |
|------|---------------------|--------------------------|
| **HTML → PDF** | `xhtml2pdf` (ReportLab) | `pisa.CreatePDF(html, out_file)` |
| **Image → PDF** | Pillow + ReportLab | `canvas.drawImage(...)` |
| **PDF → PNG** | Ghostscript | `gs -sDEVICE=pngalpha -r150 …` |
| **PDF → Text** | Poppler `pdftotext` | `subprocess.check_call(["pdftotext", …])` |
| **Merge / Split / Rotate** | PyPDF2 (small) / PDFtk (large) | `PdfFileMerger().append()` / `pdftk A=… cat A1‑3 output …` |
| **Compress / Linearize** | Ghostscript (`-dPDFSETTINGS`) / qpdf (`--compress-streams`) | `gs … -dPDFSETTINGS=/ebook …` |
| **Encrypt (RC4‑128)** | PyPDF2 | `writer.encrypt(user, owner, True)` |
| **Encrypt (AES‑256)** | qpdf | `qpdf --encrypt user owner 256 …` |
| **Add Metadata** | PyPDF2 | `writer._info.getObject().update({…})` |
| **Bookmarks** | PyPDF2 | `writer.addBookmark(title, page, parent=…)` |
| **Attachments** | PDFtk | `pdftk in.pdf attach_files file1 file2 output out.pdf` |
| **Form Export / Import** | PDFtk | `pdftk form.pdf dump_data_fields output data.fdf` |
| **Form Fill (flatten)** | PDFtk | `pdftk template.pdf fill_form data.fdf output filled.pdf flatten` |
| **Deploy** | Bundle binaries, virtualenv, CI checks | see sections 10‑11 |

Keep this table handy when you’re scripting a new PDF workflow; it’s a quick reference that saves you from hunting through documentation.

---

## 12. Final Thoughts

Working with PDFs in 2011 feels a bit like juggling a toolbox of both Python libraries and battle‑tested command‑line utilities. The pure‑Python options give you portability and ease of installation, while the external tools provide the raw performance and feature completeness you need for production‑grade workloads. By combining them thoughtfully—using ReportLab for generation, PyPDF2 for light‑weight edits, PDFtk/qpdf for heavy lifting, and Ghostscript for compression—you can build a robust pipeline that runs on any major OS and survives the quirks of the PDF format.

Remember to:

* **Validate** every input PDF early.
* **Log** subprocess output and handle errors explicitly.
* **Cache** results where possible.
* **Secure** the execution environment.

With those habits in place, you’ll be able to generate invoices, archive reports, and manipulate legal documents without ever needing a commercial SDK.

Happy coding, and may your PDFs stay linearized and your passwords stay strong!

Tags: tag-1, tag-2, tag-3
Slug: article-slug