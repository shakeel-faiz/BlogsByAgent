---
seoTitle: "AI File Formats in 2020"
title: "AI File Formats in 2020"
description: "Learn the 2020 AI model formats—TensorFlow SavedModel, ONNX, TorchScript, TFLite & more—and choose the right one for portability, reproducibility, and edge deployment."
date: 21 May 2026
draft: true
author: Khan AI
url: /audio/ai-file-formats-in-2020/
categories: ['Audio']
tags: ['AI File Formats in 2020', 'MP4', 'Some Tag']
---

**TL;DR** – By 2020 the AI world had settled on three “go‑to” model containers—TensorFlow SavedModel, ONNX, and TorchScript—while a suite of edge‑focused formats (TensorFlow Lite, Core ML, TensorRT, OpenVINO) handled the ever‑growing demand for on‑device inference. Picking the right format isn’t just a matter of file‑size; it determines portability, reproducibility, and how easily you can push a model from a research notebook to a production service or a smartphone.

---

### Why File Formats Matter (Even in 2020)

- **Portability** – Moving a model from a Jupyter notebook to a cloud‑service, or from a GPU server to a micro‑controller, hinges on a well‑defined serialization format. Without it you’re stuck in a “framework lock‑in” loop.
- **Version control & reproducibility** – A single archive (or a predictable set of files) must capture architecture, learned weights, preprocessing steps, and sometimes even training hyper‑parameters. This makes it possible to replay experiments weeks or months later.
- **Edge & mobile deployment** – Phones, IoT devices, and even tiny micro‑controllers need a lightweight binary that can be loaded in a few milliseconds. Formats like `.tflite` or `.mlmodel` shrink the footprint dramatically.
- **Inter‑operability push** – The industry rallied around open standards (ONNX, SavedModel) to avoid the “my code only runs on my framework” nightmare. The result? A smoother hand‑off between research, cloud, and edge.

---

### The Core Model‑Serialization Formats (2020)

| Format | Ecosystem | Typical Extension(s) | What It Gives You in 2020 |
|--------|-----------|----------------------|---------------------------|
| **TensorFlow SavedModel** | TensorFlow 2.x | `saved_model.pb` + `variables/` folder | Full graph + variables, versioned, ready for TensorFlow Serving or conversion to TFLite. |
| **TensorFlow Lite FlatBuffer** | TensorFlow Lite | `.tflite` | Ultra‑compact, supports post‑training quantization, runs on Android/iOS/embedded. |
| **ONNX (Open Neural Network Exchange)** | Cross‑framework (PyTorch, MXNet, Caffe2…) | `.onnx` | Open, versioned (v1.7, opset‑12 in 2020), works with TensorRT, OpenVINO, and cloud deployment services. |
| **TorchScript (PyTorch Script/Trace)** | PyTorch | `.pt` / `.pth` | Serialized graph (`torch.jit.save`) plus optional `state_dict`; works wherever PyTorch runtime exists. |
| **Keras HDF5** | Keras (TF‑Keras, standalone) | `.h5` / `.keras` | Architecture + weights in a single HDF5 container; still popular for quick prototyping. |
| **Core ML** | Apple ecosystem | `.mlmodel` | Converts from ONNX, TensorFlow, Keras; optimized for iOS/macOS inference. |
| **TensorRT Engine** | NVIDIA GPU inference | `.plan` | Highly‑optimized, hardware‑specific engine (FP16/INT8) generated from ONNX or TensorFlow. |
| **OpenVINO IR** | Intel CPUs/VPU | `.xml` + `.bin` | Intermediate Representation for Intel‑accelerated inference; produced from ONNX/TensorFlow. |
| **Caffe / Caffe2** | Legacy vision pipelines | `.caffemodel`, `.prototxt` | Separate network definition and binary weights; still shows up in older research code. |
| **MXNet Gluon** | MXNet | `.params` + `.json` | Split parameter file and network symbol; useful for hybrid models. |

**Take‑away:** If you’re building a new project in 2020, you’ll most likely end up with a SavedModel, an ONNX file, or a TorchScript archive. Everything else is a specialization for a particular deployment target.

---

### Data & Annotation Formats That Travel With Your Model

Model files are only half the story—training data formats dictate pipeline speed and scalability.

- **TFRecord** – TensorFlow’s binary record format; perfect for sharding massive image or text corpora and streaming them efficiently.
- **HDF5** – General‑purpose hierarchical container; used by Keras and sometimes by PyTorch via `h5py`.
- **Apache Parquet** – Columnar storage for tabular data; increasingly the default for Spark‑based training pipelines.
- **COCO JSON** – De‑facto standard for object‑detection and segmentation annotations (images + bounding boxes + masks).
- **Pascal VOC XML** – Older detection format, still hanging around in legacy datasets.
- **YOLO Darknet `.txt`** – One‑line per image, class + normalized box coordinates; simple and human‑readable.
- **Audio** – Raw `.wav`/`.flac` files for ingestion; pre‑processed features (MFCC, spectrogram) often saved in TFRecord or HDF5.
- **NLP** – Plain‑text corpora (`.txt`, `.jsonl`, `.csv`) or TFRecord of `tf.Example`; Hugging Face tokenizers now ship vocabularies as `.json`.

Choosing the right data format can shave hours off training time, especially when you scale out to multi‑node clusters.

---

### 2020 Trends & Shifts

1. **ONNX Becomes the Lingua Franca**  
   - ONNX 1.7 (June 2020) introduced opset‑12, covering most vision and NLP operators.  
   - Azure, AWS, and GCP all added native ONNX deployment endpoints, making it the easiest “write once, run anywhere” choice.

2. **TensorFlow 2.x & Keras Integration**  
   - `tf.keras` became the default high‑level API.  
   - `tf.saved_model.save` replaced the older `freeze_graph` workflow, delivering a single, version‑aware bundle.

3. **Model Compression & Quantization**  
   - Post‑training 8‑bit quantization landed in TensorFlow Lite and PyTorch’s Quantization Toolkit.  
   - Formats like `.tflite` and TensorRT `.plan` now store quantized weights, cutting file size 4‑10× without major accuracy loss.

4. **Edge‑First Deployments**  
   - Apple’s Core ML and Google’s TensorFlow Lite saw a surge of releases for AR, health, and smart‑home use‑cases.  
   - Conversion pipelines (`tf.lite.TFLiteConverter`, `coremltools`) became a staple in every AI blog of the year.

5. **Versioned Model Registries**  
   - Tools like **MLflow Model Registry**, **TensorFlow Model Garden**, and **Weights & Biases** introduced model versioning, lineage tracking, and metadata storage.  
   - SavedModel’s `signature_def` and ONNX’s model metadata fields started to carry richer provenance information.

6. **Model Cards & Documentation**  
   - Google’s Model Card Toolkit (2020) encouraged a JSON/YAML “model card” to accompany every exported model, describing data provenance, intended use, and performance metrics.

7. **Security & Provenance**  
   - The first high‑profile model‑trojan papers appeared in 2020, prompting discussions about digital signatures for model files (e.g., ONNX signing extensions).

---

### Real‑World Examples & a Quick Cheat‑Sheet

| Example | Format | What It Shows |
|---------|--------|---------------|
| **BERT‑Base (TensorFlow)** | SavedModel (`saved_model.pb`) | Large NLP model ready for TF‑Serving or conversion to TFLite. |
| **ResNet‑50 (PyTorch)** | TorchScript (`model.pt`) | Scripted graph that can be loaded anywhere PyTorch runs; also exportable to ONNX. |
| **YOLOv3 (Darknet → ONNX)** | `.onnx` | Cross‑framework conversion; later compiled to TensorRT `.plan` for real‑time inference. |
| **MobileNetV2 (TensorFlow Lite)** | `.tflite` | 4 MB quantized model for Android/iOS; demonstrates post‑training quantization pipeline. |
| **Mask R‑CNN (Core ML)** | `.mlmodel` | ONNX → Core ML conversion; used in iOS AR apps for instance segmentation. |
| **DeepSpeech (Speech‑to‑Text)** | Custom protobuf (`.pbmm`) | Shows that some research projects still ship proprietary binary formats for offline edge use. |
| **XGBoost 1.0** | `model.json` | Introduced a JSON dump for better language‑agnostic consumption. |
| **StyleGAN2** | Pickle (`.pkl`) | Highlights the reproducibility risk of Python‑specific serialization. |

#### Cheat‑Sheet for Choosing a Format (2020)

- **Maximum cross‑framework compatibility** → Export to **ONNX** (opset‑12).  
- **Staying inside TensorFlow** → Use **SavedModel** for training/serving; convert to **TFLite** for mobile/edge.  
- **PyTorch‑first teams** → Save a **TorchScript** (`torch.jit.save`) for production; keep a **state_dict** (`.pth`) for research reproducibility.  
- **Apple‑centric apps** → Convert to **Core ML** (`coremltools.convert`).  
- **NVIDIA GPU inference** → Convert ONNX → **TensorRT** (`trtexec`) → `.plan`.  
- **Intel CPU/VPU** → Convert to **OpenVINO IR** (`.xml` + `.bin`).  
- **Data pipelines** → Prefer **TFRecord** for image/text, **Parquet** for tabular, **COCO JSON** for detection annotations.

---

### Closing Thoughts

2020 was the year the AI community finally stopped reinventing the wheel for every new project. With **SavedModel**, **ONNX**, and **TorchScript** as the universal lingua‑franca, and a well‑defined set of edge‑focused containers for mobile, IoT, and specialized hardware, engineers could focus more on model innovation and less on format gymnastics. The surrounding ecosystem—versioned registries, model cards, and even early security measures—rounded out a more mature, production‑ready workflow that still underpins most AI deployments today.

---

**Tags:** `ai` `file-formats` `2020`  
**Slug:** `ai-file-formats-2020`