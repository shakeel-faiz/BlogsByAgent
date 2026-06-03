---
seoTitle: "hello_reportlab.py – works on Python 2.7 and 3.4"
title: "hello_reportlab.py – works on Python 2.7 and 3.4"
description: "Learn how to create, edit, compress, and secure PDFs in Django/Flask using pure‑Python libraries (ReportLab, PyPDF2, pdfrw) and tools like Ghostscript & QPDF."
date: 03 Jun 2026
draft: true
author: Khan AI
url: /audio/hello-reportlab-py-works-on-python-2-7-and-3-4/
categories: ['Audio']
tags: ['hello_reportlab.py – works on Python 2.7 and 3.4', 'MP4', 'Some Tag']
---

TL;DR  
If you’re writing a Django/Flask service in 2015 and need to **create**, **tweak**, **shrink**, or **lock** PDF files, you can do it with a handful of pure‑Python libraries (ReportLab, PyPDF2, pdfrw) plus a couple of battle‑tested command‑line tools (Ghostscript, QPDF). The code works on both Python 2.7 and Python 3.4, stays under permissive BSD/MIT licenses, and can be run on Windows, macOS, or Linux with only a few extra binaries.

---

## 1. Setting the Stage – Python & PDF Landscape in 2015  

| What mattered in 2015 | Why it matters for you today |
|-----------------------|------------------------------|
| **Python version** – most codebases still on 2.7, 3.4 just released | Write code that runs on both; use `six`/`future` for `bytes` vs `str` handling. |
| **License hygiene** – many commercial SDKs (iText, PDFBox) are GPL/Commercial | Stick to BSD/MIT‑licensed libraries (ReportLab, PyPDF2, pdfrw) unless you’re okay with GPL. |
| **Performance** – pure‑Python is fine for < 10 MB PDFs; larger batches need native tools | Call Ghostscript or QPDF via `subprocess` for heavy‑duty compression or linearization. |
| **Security** – PDFs can embed JavaScript, files, and encryption (AES‑128/256) | Encrypt before sending to S3/Azure, strip JavaScript when you serve PDFs over the web. |
| **OS support** – all major OSes have the same Python ecosystem, but native binaries differ | Install Ghostscript/Poppler binaries per platform; keep them in your CI image. |

Understanding the PDF object model (pages, streams, cross‑reference table) helps when you need low‑level tweaks (e.g., removing a malicious JavaScript action). Also keep an eye on **incremental updates** – many editors simply append changes, which is why you can add a watermark without rewriting the whole file.

---

## 2. Creating PDFs from Scratch (ReportLab)  

ReportLab is the de‑facto standard for programmatic PDF generation. It ships with a permissive BSD license, supports vector graphics, barcodes, and even a commercial add‑on for PDF/A compliance.

```python
# hello_reportlab.py – works on Python 2.7 and 3.4
from __future__ import print_function
from reportlab.lib.pagesizes import A4
from reportlab.pdfgen import canvas

def hello_pdf(path):
    c = canvas.Canvas(path, pagesize=A4)
    c.setFont("Helvetica", 14)
    c.drawString(100, 750, "Hello, PDF world!")
    # draw a simple line chart
    c.line(50, 730, 550, 730)
    c.showPage()          # finish the page
    c.save()

if __name__ == '__main__':
    hello_pdf('hello.pdf')
```

**Tips for 2015‑style projects**

* Use **Platypus** (ReportLab’s high‑level flowable API) for multi‑page invoices or reports.  
* If you need PDF/A for archival, the commercial “PDF/A” add‑on can be activated with `canvas.setPDFVersion('1.4')` and `canvas.setPageCompression(1)`.  
* Keep the canvas thin – heavy graphics should be rasterized beforehand (e.g., with Pillow) to avoid huge file sizes.

---

## 3. Editing & Manipulating Existing PDFs (PyPDF2, pdfrw)  

Most web apps generate a PDF first, then need to **merge**, **split**, **rotate**, or **watermark** it. The two most popular pure‑Python tools are:

| Library | Strength | Weakness |
|---------|----------|----------|
| **PyPDF2** | Easy merging, splitting, encryption | No text extraction, limited compression |
| **pdfrw** | Low‑level access to PDF objects, works nicely with ReportLab overlays | Slightly more verbose API |

### 3.1 Merging PDFs (PyPDF2)

```python
import PyPDF2

def merge_pdfs(pdf_paths, out_path):
    merger = PyPDF2.PdfFileMerger()
    for p in pdf_paths:
        merger.append(p)
    with open(out_path, 'wb') as f:
        merger.write(f)

merge_pdfs(['invoice1.pdf', 'invoice2.pdf', 'terms.pdf'], 'full_package.pdf')
```

### 3.2 Adding a Watermark (pdfrw + ReportLab)

```python
from pdfrw import PdfReader, PdfWriter, PageMerge
from reportlab.pdfgen import canvas
from io import BytesIO

def make_watermark(text):
    packet = BytesIO()
    c = canvas.Canvas(packet, pagesize=(595, 842))  # A4 in points
    c.setFillAlpha(0.15)
    c.setFont('Helvetica-Bold', 72)
    c.saveState()
    c.translate(300, 400)
    c.rotate(45)
    c.drawCentredString(0, 0, text)
    c.restoreState()
    c.save()
    packet.seek(0)
    return PdfReader(packet).pages[0]

def watermark_pdf(src_path, dst_path, watermark_text):
    src = PdfReader(src_path)
    wm = make_watermark(watermark_text)
    for page in src.pages:
        merger = PageMerge(page)
        merger.add(wm, prepend=False).render()
    PdfWriter(dst_path, trailer=src).write()

watermark_pdf('report.pdf', 'report_watermarked.pdf', 'CONFIDENTIAL')
```

### 3.3 Rotating Pages & Updating Metadata

```python
def rotate_and_tag(pdf_path, out_path, angle=90):
    reader = PyPDF2.PdfFileReader(pdf_path)
    writer = PyPDF2.PdfFileWriter()
    for i in range(reader.getNumPages()):
        page = reader.getPage(i)
        page.rotateClockwise(angle)
        writer.addPage(page)
    # add simple metadata
    writer.addMetadata({
        '/Author': 'Acme Corp',
        '/Title': 'Rotated Report',
        '/Subject': 'Internal Use Only'
    })
    with open(out_path, 'wb') as f:
        writer.write(f)

rotate_and_tag('original.pdf', 'rotated.pdf')
```

---

## 4. Converting & Compressing PDFs (HTML → PDF, Ghostscript, QPDF)  

### 4.1 HTML‑to‑PDF with wkhtmltopdf (via pdfkit)

`wkhtmltopdf` wraps WebKit and gives you pixel‑perfect PDFs from HTML/CSS. Install the binary once, then call it from Python:

```python
import pdfkit

options = {
    'page-size': 'A4',
    'margin-top': '0.75in',
    'margin-right': '0.75in',
    'margin-bottom': '0.75in',
    'margin-left': '0.75in',
    'encoding': "UTF-8",
    'no-outline': None
}

pdfkit.from_url('https://example.com/report', 'report.pdf', options=options)
# or from a string:
html = "<h1>Invoice</h1><p>Total: $123.45</p>"
pdfkit.from_string(html, 'invoice.pdf', options=options)
```

*Works on

*Works on Windows, macOS, and Linux as long as the `wkhtmltopdf` binary is on your `PATH`. If you’re using Docker, the official `wickedpdf/wkhtmltopdf` image is a solid base for a headless PDF service.*

### 4.2 Down‑sizing PDFs with Ghostscript  

Ghostscript is the workhorse for PDF rasterisation, colour‑space conversion, and aggressive compression. It’s a native binary, but the command‑line interface is simple enough to wrap in Python.

```python
import subprocess
import shlex
import os

def compress_pdf(src_path, dst_path, quality='ebook'):
    """
    quality: screen (72 dpi), ebook (150 dpi), printer (300 dpi), prepress (300 dpi, colour preserving)
    """
    # Ghostscript command taken from the official docs
    gs_cmd = (
        "gs -sDEVICE=pdfwrite -dCompatibilityLevel=1.4 "
        "-dPDFSETTINGS=/{quality} -dNOPAUSE -dQUIET -dBATCH "
        "-sOutputFile={dst} {src}"
    ).format(quality=quality, src=shlex.quote(src_path), dst=shlex.quote(dst_path))

    # Run the command; raise if something goes wrong
    subprocess.check_call(gs_cmd, shell=True)

# Example usage
compress_pdf('large_report.pdf', 'large_report_compressed.pdf', quality='ebook')
```

**Why Ghostscript?**  

| Use‑case | Ghostscript flag | Typical size reduction |
|----------|------------------|------------------------|
| **Screen‑only PDFs** (e.g., web preview) | `-dPDFSETTINGS=/screen` | 60‑80 % |
| **E‑book quality** (150 dpi, decent images) | `-dPDFSETTINGS=/ebook` | 40‑60 % |
| **Print‑ready** (300 dpi, lossless) | `-dPDFSETTINGS=/printer` | 20‑30 % |
| **Prepress** (preserve colour profiles) | `-dPDFSETTINGS=/prepress` | 10‑20 % |

You can also fine‑tune the compression by adding `-dColorImageDownsampleType=/Bicubic -dColorImageResolution=150` (or any DPI you like). Remember that Ghostscript strips out *most* PDF annotations and form fields unless you explicitly preserve them with `-dPreserveAnnots=true`.

### 4.3 Linearizing (Fast Web View) with QPDF  

If you serve PDFs over HTTP, enabling **Fast Web View** (also called linearization) lets browsers start rendering the first page before the whole file is downloaded. QPDF, a C++‑based tool with a clean CLI, does this in a single pass.

```python
def linearize_pdf(src_path, dst_path):
    """
    QPDF linearization (Fast Web View) – ideal for large PDFs served via CDN.
    """
    cmd = ['qpdf', '--linearize', src_path, dst_path]
    subprocess.check_call(cmd)

linearize_pdf('report.pdf', 'report_linearized.pdf')
```

QPDF also offers powerful encryption, decryption, and object‑stream manipulation, which we’ll explore in the next section.

---

## 5. Security & Encryption – Keeping PDFs Safe  

PDFs can carry embedded JavaScript, external file references, and even encrypted payloads. In a web‑service context you often need to **sanitize** and **protect** PDFs before they leave your perimeter.

### 5.1 Stripping JavaScript and Embedded Files  

Both PyPDF2 and pdfrw let you walk the object tree and delete suspicious entries. The following helper removes any `/JavaScript` or `/EmbeddedFile` streams.

```python
import re
from pdfrw import PdfReader, PdfWriter, PdfDict

JS_RE = re.compile(r'/JavaScript', re.IGNORECASE)
EMBED_RE = re.compile(r'/EmbeddedFile', re.IGNORECASE)

def strip_malicious(pdf_path, out_path):
    pdf = PdfReader(pdf_path)
    for page in pdf.pages:
        # Remove /Annots that contain JavaScript actions
        if page.Annots:
            clean_annots = []
            for annot in page.Annots:
                if not (annot.A and JS_RE.search(str(annot.A))):
                    clean_annots.append(annot)
            page.Annots = clean_annots or None

    # Remove any top‑level /Names entries that point to JavaScript or embedded files
    if pdf.Root.Names:
        for name_type in list(pdf.Root.Names.keys()):
            if JS_RE.search(name_type) or EMBED_RE.search(name_type):
                del pdf.Root.Names[name_type]

    PdfWriter(out_path, trailer=pdf).write()

strip_malicious('incoming.pdf', 'sanitized.pdf')
```

Run this step **before** you accept a PDF from an untrusted source (e.g., user upload). It’s cheap, pure‑Python, and works on both Python 2 and 3.

### 5.2 Encrypting PDFs with AES‑128/256  

PyPDF2 supports standard PDF encryption (RC4‑40, RC4‑128, AES‑128). For stronger protection you can call QPDF, which implements AES‑256.

**Using PyPDF2 (AES‑128):**

```python
def encrypt_pdf(src_path, dst_path, user_pwd, owner_pwd=None):
    reader = PyPDF2.PdfFileReader(src_path)
    writer = PyPDF2.PdfFileWriter()
    for i in range(reader.getNumPages()):
        writer.addPage(reader.getPage(i))
    writer.encrypt(user_pwd, owner_pwd or user_pwd, use_128bit=True)
    with open(dst_path, 'wb') as f:
        writer.write(f)

encrypt_pdf('public.pdf', 'protected.pdf', user_pwd='s3cr3t')
```

**Using QPDF (AES‑256):**

```python
def encrypt_qpdf(src_path, dst_path, password):
    cmd = [
        'qpdf',
        '--encrypt', password, password, '256',
        '--', src_path, dst_path
    ]
    subprocess.check_call(cmd)

encrypt_qpdf('public.pdf', 'protected_aes256.pdf', 'ultrasecure')
```

When you store PDFs in S3 or Azure Blob, encrypt them **twice**: first with the PDF password (so the file is unreadable even if the storage bucket leaks), then rely on the cloud provider’s server‑side encryption for at‑rest protection.

### 5.3 Digital Signatures (Bonus)  

Full‑blown digital signatures are beyond the scope of pure‑Python libraries in 2015, but you can invoke **OpenSSL** or **iText** via a subprocess to embed a PKCS#7 signature. The workflow is:

1. Generate a hash of the PDF (e.g., SHA‑256).  
2. Sign the hash with your private key (`openssl dgst -sha256 -sign key.pem`).  
3. Use `qpdf --add-attachment` to embed the signature as a PDF annotation.

Keep the signing step isolated in a separate microservice if you need to protect the private key.

---

## 6. Deploying a PDF Service – Docker, CI, and Monitoring  

### 6.1 Dockerfile Blueprint  

```Dockerfile
# Base image with Python 3.8 (or 2.7 if you must)
FROM python:3.8-slim

# Install system dependencies
RUN apt-get update && apt-get install -y \
    ghostscript \
    qpdf \
    wkhtmltopdf \
    && rm -rf /var/lib/apt/lists/*

# Create a non‑root user
RUN useradd -ms /bin/bash pdfsvc
USER pdfsvc
WORKDIR /app

# Install Python deps
COPY requirements.txt .
RUN pip install --no-cache-dir -r requirements.txt

# Copy source code
COPY . .

# Expose the Flask API (or Django WSGI) port
EXPOSE 8000
CMD ["gunicorn", "myservice.wsgi:application", "--bind", "0.0.0.0:8000"]
```

**Why this works:**  

* `ghostscript` and `qpdf` are installed from the OS repo, guaranteeing the same binary version across environments.  
* The image stays under 200 MB thanks to `slim`.  
* Running as a non‑root user mitigates container breakout risks.

### 6.2 CI Pipeline (Travis / GitHub Actions)

```yaml
# .github/workflows/pdf.yml
name: PDF Service CI

on: [push, pull_request]

jobs:
  test:
    runs-on: ubuntu-latest
    services:
      redis:
        image: redis:5
    steps:
      - uses: actions/checkout@v2
      - name: Set up Python
        uses: actions/setup

```yaml
      - name: Install dependencies
        run: |
          python -m pip install --upgrade pip
          pip install -r requirements.txt
      - name: Run unit tests
        run: |
          pytest -v tests/
      - name: Lint with flake8
        run: |
          flake8 myservice/ --count --select=E9,F63,F7,F82 --show-source --statistics
      - name: Security scan (bandit)
        run: |
          bandit -r myservice/ -ll
      - name: Build Docker image
        run: |
          docker build -t myorg/pdf-service:${{ github.sha }} .
      - name: Push to registry (if on main)
        if: github.ref == 'refs/heads/main'
        env:
          REGISTRY_USER: ${{ secrets.REGISTRY_USER }}
          REGISTRY_PASS: ${{ secrets.REGISTRY_PASS }}
        run: |
          echo "$REGISTRY_PASS" | docker login -u "$REGISTRY_USER" --password-stdin
          docker push myorg/pdf-service:${{ github.sha }}
```

### 6.3 Runtime Monitoring & Observability  

A PDF‑generation microservice can become a bottleneck under load, especially when you’re invoking heavyweight binaries. A lightweight observability stack keeps you ahead of trouble:

| Concern | Tool | How to integrate |
|---------|------|------------------|
| **Request latency** | **Prometheus** + **Grafana** | Export a `/metrics` endpoint (e.g., using `prometheus_flask_exporter`). Record histogram buckets for `pdf_generation_seconds`. |
| **Error rates** | **Sentry** (or **Rollbar**) | Wrap the PDF‑creation functions in a `try/except` that captures the exception and sends it to Sentry with the PDF name and user ID (redact any PII). |
| **Process health** | **Healthchecks.io** or Kubernetes liveness/readiness probes | Expose `/healthz` that checks that `ghostscript`, `qpdf`, and `wkhtmltopdf` binaries are reachable (`subprocess.run(['gs', '--version'], ...)`). |
| **Resource usage** | **cAdvisor** / **Node Exporter** | Monitor CPU & memory spikes when large PDFs (> 50 MB) are processed. If you see sustained > 80 % CPU, consider off‑loading to a dedicated worker queue (Celery + RabbitMQ). |

**Sample Flask metrics endpoint**

```python
from flask import Flask, jsonify
from prometheus_flask_exporter import PrometheusMetrics

app = Flask(__name__)
metrics = PrometheusMetrics(app)          # automatically registers /metrics

# Example: time the PDF generation route
@metrics.histogram('pdf_generation_seconds',
                   'Time spent generating PDFs',
                   labels={'type': lambda: request.args.get('type', 'unknown')})
@app.route('/api/pdf', methods=['POST'])
def generate():
    # ... your existing PDF logic (ReportLab, wkhtmltopdf, etc.) ...
    return jsonify({'status': 'ok', 'url': pdf_url})
```

### 6.4 Scaling Strategies  

1. **Horizontal scaling** – Deploy multiple replicas behind an API gateway (NGINX, Traefik, or AWS ALB). Because the service is stateless (PDFs are written to a shared object store), any instance can handle any request.  
2. **Queue‑driven workers** – For batch jobs (e.g., nightly invoice generation), push tasks onto a Celery queue. Workers can be sized independently and can run on machines with more CPU cores to accommodate Ghostscript’s parallelism (`-dNumRenderingThreads=4`).  
3. **Cold‑start mitigation** – On serverless platforms (AWS Lambda, Google Cloud Functions) the binary dependencies increase the package size. Use **Lambda Layers** to ship Ghostscript and QPDF once, then keep your Python code thin. Remember the 250 MB unzipped limit; the slim Debian‑based layers stay well under it.  

---

## 7. Real‑World Gotchas & How to Avoid Them  

| Issue | Why it happens | Fix / Work‑around |
|-------|----------------|-------------------|
| **“File not found” for `gs` or `qpdf`** | The binary isn’t on the container’s `PATH` (common on Alpine images). | Use a Debian‑based base or explicitly add `/usr/local/bin` to `PATH`. |
| **Unicode errors when merging PDFs with non‑ASCII metadata** | PyPDF2 expects `bytes` for certain fields; Python 3 strings become Unicode. | Encode metadata with `utf‑8` and pass `PdfFileReader(..., strict=False)`. |
| **Ghostscript stripping out form fields** | By default Ghostscript discards interactive elements. | Add `-dPreserveAnnots=true` to the command line. |
| **Large memory usage when loading a 200 MB PDF with pdfrw** | pdfrw reads the entire file into memory. | Stream pages with `PdfReader(..., decompress=False)` and process one page at a time, or fall back to `qpdf --pages` for pure‑copy operations. |
| **Docker image bloat** | Installing `poppler-utils` for PDF‑to‑image conversion adds ~30 MB. | Only install `poppler-utils` in a separate “builder” stage and copy the needed binaries into the final image. |

---

## 8. TL;DR Recap (for the impatient)

| Goal | Recommended stack (Python 2.7/3.4) |
|------|-----------------------------------|
| **Create PDFs from scratch** | `ReportLab` (vector graphics, Platypus) |
| **Merge / split / rotate / watermark** | `PyPDF2` (high‑level) + `pdfrw` (low‑level overlays) |
| **HTML → PDF** | `wkhtmltopdf` via `pdfkit` |
| **Compress / down‑sample** | Ghostscript (`gs`) |
| **Linearize for web** | QPDF (`--linearize`) |
| **Sanitize / strip JavaScript** | `pdfrw` custom walk‑tree script |
| **Encrypt (AES‑128)** | `PyPDF2.encrypt` |
| **Encrypt (AES‑256)** | QPDF (`--encrypt … 256`) |
| **Deploy** | Docker + Gunicorn + CI (GitHub Actions) + Prometheus/Sentry monitoring |

With these tools you can build a fully featured PDF microservice that runs anywhere from a local dev box to a Kubernetes cluster, stays under permissive licenses, and handles the most common production‑grade PDF tasks without paying for a commercial SDK.

---

## 9. Further Reading & Community Resources  

* **ReportLab User Guide** – https://www.reportlab.com/docs/reportlab-userguide.pdf  
* **PyPDF2 Documentation** – https://pythonhosted.org/PyPDF2/  
* **pdfrw GitHub** – https://github.com/pmaupin/pdfrw  
* **Ghostscript Manual** – https://www.ghostscript.com/doc/current/Use.htm  
* **QPDF Documentation** – https://qpdf.readthedocs.io/en/stable/  
* **Docker Best Practices** – https://docs.docker.com/develop/develop-images/dockerfile_best-practices/  
* **Prometheus Flask Exporter** – https://github.com/rycus86/prometheus_flask_exporter  

If you run into a PDF that refuses to be parsed, try opening it in **Adobe Acrobat** and saving a fresh copy – Acrobat will often “repair” broken cross‑reference tables, after which the Python tools can read it without error.

---

### Closing Thoughts  

PDF handling in 2015 (and today) is a perfect illustration of the “best of both worlds” philosophy: combine **battle‑tested native binaries** for heavy lifting with **pure‑Python glue** for orchestration. This approach gives you the speed and reliability of Ghostscript/QPDF while keeping your application code portable, testable, and easy to maintain. Whether you’re generating invoices, serving e‑books, or sanitizing user‑uploaded contracts, the recipe above will get you from “I have a PDF” to “I have a safe, compressed, and correctly formatted PDF” in a handful of lines of code.

Happy coding, and may your PDFs always be linearized!  

Tags: tag-1, tag-2, tag-3  
Slug: article-slug