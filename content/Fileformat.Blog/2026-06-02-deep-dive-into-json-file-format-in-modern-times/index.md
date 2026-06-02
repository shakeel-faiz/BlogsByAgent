---
seoTitle: "Deep Dive into JSON file format in Modern Times"
title: "Deep Dive into JSON file format in Modern Times"
description: "Discover JSON’s journey from a JavaScript sketch to the universal API format, plus modern parsing tricks, schemas, and security tips for confident use."
date: 02 Jun 2026
draft: true
author: Khan AI
url: /audio/deep-dive-into-json-file-format-in-modern-times/
categories: ['Audio']
tags: ['Deep Dive into JSON file format in Modern Times', 'MP4', 'Some Tag']
---

**TL;DR** – JSON (JavaScript Object Notation) started as a lightweight way for JavaScript to talk to itself, but today it’s the lingua franca of web APIs, config files, and even edge‑computing pipelines. It’s strict, UTF‑8‑only, and supported everywhere, yet the ecosystem has sprouted schemas, streaming formats, binary cousins, and security‑hardening parsers. This post unpacks the “why” and “how” of modern JSON so you can use it confidently—whether you’re writing a one‑line config or a high‑throughput micro‑service.

---

## 1. From Crockford’s Sketch to an Official Standard  

*Created by Douglas Crockford in 2001* – JSON was originally a **tiny subset of JavaScript literal syntax** meant to be easy for humans to read and for browsers to parse. The format stayed informal for a decade, then got its first formal definition in **IETF RFC 8259 (2017)** and finally a **stand‑alone ECMA‑404 (2023)** spec.  

Why does that matter? Because the spec is now **the only “official” JSON definition**. All major languages (JavaScript, Python, Go, Rust, Java, C#, PHP, etc.) ship built‑in parsers (`JSON.parse`, `json.loads`, `encoding/json`, `serde_json`) that claim RFC 8259 compliance. The result? **> 90 % of public web APIs** (REST, GraphQL‑over‑HTTP, OpenAPI) speak JSON by default, and countless tools—npm, VS Code, ESLint, Docker Compose, Terraform—store their configuration in plain‑text JSON files.

Performance is also impressive. Modern JavaScript engines (V8, SpiderMonkey, JavaScriptCore) can **parse more than 200 MB/s on a single core**, and specialized libraries like **simdjson** push that to **> 4 GB/s** on a 3.5 GHz CPU. In practice, a typical API payload of a few kilobytes is parsed in microseconds.

---

## 2. Core Concepts You Must Know  

| Concept | What It Means | Why It Matters |
|---------|---------------|----------------|
| **Data Types** | `string`, `number`, `object`, `array`, `true`, `false`, `null` | The spec deliberately limits primitives—no dates, binary blobs, or comments. |
| **Unicode** | JSON text is UTF‑8 (or UTF‑16/UTF‑32) – UTF‑8 is de‑facto. | Guarantees that a file written on macOS reads the same on Windows, Linux, or an embedded device. |
| **Escaping** | `\"`, `\\`, `\/`, `\b`, `\f`, `\n`, `\r`, `\t`, `\uXXXX` | Required for control characters and non‑ASCII code points; misuse leads to parsing errors. |
| **Schema** | **JSON Schema** (draft‑2020‑12) describes structure, constraints, defaults. | Enables validation, auto‑generated docs, and type‑safe code generation. |
| **Streaming** | **NDJSON** (JSON Lines) – one JSON object per line; streaming parsers (`JSONStream`, reviver). | Lets you process gigabytes of data without loading the whole file into memory. |
| **Binary Variants** | **BSON**, **MessagePack**, **CBOR**, **UBJSON** – binary encodings of JSON‑compatible data. | Reduce size and parsing time for high‑throughput services (MongoDB, Kafka, IoT). |
| **Extended Syntaxes** | **JSON5**, **JSONC**, **Hjson**, **Relaxed JSON** – allow comments, trailing commas, unquoted keys. | Great for human‑editable config files, but not interchangeable with strict JSON. |
| **Linked Data** | **JSON‑LD** adds `@context`, `@id`, `@type`. | Powers knowledge graphs, schema.org rich snippets, and hypermedia APIs. |
| **Patch & Merge** | **JSON Patch** (RFC 6902) & **JSON Merge Patch** (RFC 7386). | Efficient incremental updates (e.g., HTTP PATCH). |
| **Security** | Risks: JSON injection, prototype pollution, deep‑nesting DoS. | Must be mitigated with strict parsers and input sanitisation. |

### Minimal Valid JSON (RFC 8259)

```json
{
  "name": "Alice",
  "age": 30,
  "isAdmin": false,
  "tags": ["dev", "json"],
  "profile": null
}
```

Notice the lack of comments, trailing commas, or single‑quoted strings—those are **illegal** in strict JSON.

---

## 3. Schemas, Validation, and Code Generation  

If you’ve ever spent an afternoon debugging a missing field in an API response, you’ll appreciate **JSON Schema**. A concise schema for the example above looks like this:

```json
{
  "$schema": "https://json-schema.org/draft/2020-12/schema",
  "type": "object",
  "required": ["name", "age"],
  "properties": {
    "name": { "type": "string", "minLength": 1 },
    "age":  { "type": "integer", "minimum": 0 },
    "isAdmin": { "type": "boolean" },
    "tags": {
      "type": "array",
      "items": { "type": "string" }
    },
    "profile": { "type": ["object", "null"] }
  },
  "additionalProperties": false
}
```

Tools like **OpenAPI**, **GraphQL Code Generator**, **Quicktype**, and **json-schema-to-typescript** can ingest this schema and spit out **strongly‑typed client libraries** (TypeScript interfaces, Go structs, Rust enums). The workflow has shifted from “write code, then test” to **schema‑first development**, where the contract lives in a JSON Schema file and the code is generated automatically.

---

## 4. Streaming, NDJSON, and Event‑Driven Pipelines  

When you need to process logs, click‑streams, or massive CSV‑like datasets, loading the whole JSON array into memory is a non‑starter. **NDJSON (Newline‑Delimited JSON)** solves this by putting one complete JSON object on each line:

```
{"ts":"2026-06-01T12:00:00Z","level":"info","msg":"service started"}
{"ts":"2026-06-01T12:00:05Z","level":"error","msg":"db connection failed","code":503}
{"ts":"2026-06-01T12:00:10Z","level":"info","msg":"retrying connection"}
```

Node’s `JSONStream`, Python’s `ijson`, and Rust’s `serde_json::Deserializer` can read these line‑by‑line, yielding a **constant‑memory** pipeline. In Kafka‑Connect, the **JSON converter** works natively with NDJSON, making it the default for log aggregation and real‑time dashboards.

If you need to describe a *change* rather than a whole document, **JSON Patch** (RFC 6902) is the go‑to format:

```json
[
  { "op": "replace", "path": "/age", "value": 31 },
  { "op": "add", "path": "/tags/2", "value": "senior" }
]
```

A server can apply this patch atomically, saving bandwidth and avoiding race conditions.

---

## 5. Binary Cousins – When Speed Beats Readability  

Strict JSON is human‑readable, but sometimes you need **smaller payloads** and **faster parsing**. Enter the binary family:

| Format | Typical Use‑Case | Notable Feature |
|--------|------------------|-----------------|
| **BSON** | MongoDB storage & wire protocol | Supports additional types (Date, ObjectId). |
| **MessagePack** | Micro‑service RPC, embedded devices | 30‑50 % size reduction vs. JSON, still schema‑agnostic. |
| **CBOR** | IoT, constrained networks | Designed for small code footprints and deterministic encoding. |
| **UBJSON** | General purpose binary JSON | Simple spec, easy to implement in low‑level languages. |

A quick hex dump of a MessagePack‑encoded object (the same data as our minimal JSON) looks like:

```
82                # map of 2 entries
a4 6e 61 6d 65    # key "name"
a5 41 6c