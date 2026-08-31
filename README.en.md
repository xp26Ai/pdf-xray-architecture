# pdf-xray-architecture

[العربية](README.md)

**A scanned PDF in → extracted fields out, each with its source and confidence — no internet required.**

> **This system runs in production at my employer, and the code belongs to them. This repository documents the architecture, the engineering decisions, and how it was built.**

A deterministic extraction system for scanned documents, in 16,543 lines of JavaScript.
One server with 34 endpoints, 41 modules in `lib/`, and 16 smoke-test suites.
Arabic/English OCR runs fully on-device through Tesseract WASM.
The AI layer is optional enrichment on top of the result, never a requirement for it.

---

## The problem

Invoices, purchase orders, and delivery notes arrive as scans of varying quality. What matters is their exact numbers.

Sending every file to a cloud model has 3 flaws. The result is not reproducible.
The data leaves the server. And the service stops with the internet, or when the quota runs out.

Literal text matching alone is fragile too: scan noise changes the characters themselves between copies.

What was needed: a deterministic path that returns the same answer every time, with an honest per-field confidence.

---

## How it works

| Layer | Role |
|---|---|
| Document signature | Identifies the issuer from its header wording, tolerant of common OCR errors |
| Local OCR | Tesseract WASM in two isolated child processes (Arabic and English); 24.9 MB of language data ships inside the project |
| Classification | 5 document types via weighted Arabic/English keywords, with zero external calls |
| Trained templates | Label-to-value anchors taught through a web UI, then applied deterministically to every matching document |
| Confidence and provenance | Every field carries two separate confidences `{found, value}`, plus its extraction source and a trust class |
| Optional AI enrichment | A daily budget of 450 calls per model, a 14-calls-per-minute queue, and an on-disk SHA-256 cache |

From file to fields:

```
PDF ──> document signature ──> trained template ──> deterministic extraction (anchors + patterns)
             │                                                │
             ▼                                                ▼
     isolated local OCR                     fields: value + {found, value} + source
                                                              │
                                             (optional) cloud model enrichment
                                                              │
                                  cross-check ──> reproducible / verified / ai_only
```

The confidence rules are deterministic and live in one pure module with no I/O:

- `found`: how firmly the field was located. Exact anchor 0.95, fuzzy anchor 0.75, text pattern 0.70.
- `value`: how correct the text is. It starts from OCR confidence and is floored at 0.85 when the expected format matches.
- Agreement between the deterministic path and the cloud model raises `value` by 0.10.
- Disagreement caps it at 0.50, which pushes the field into human review.
- Any value below 0.7 is flagged `review: true` and sorted to the top of the list.

And the enrichment layer never throws. When the key is missing, the quota is spent,
or the call fails, the full deterministic result comes back with only a warning attached.

---

## The key design decision

**The problem:** document-to-template matching started as Jaccard similarity over all first-page tokens.
Measured on 24 real scans from the same issuer, the score wobbled between 0.52 and 0.88 against one template.
A genuine document even fell below the 0.50 threshold and was rejected — the similarity was measuring scan noise, not identity.

**The decision:** replace whole-page similarity with a document signature that asks only the identity question:
does this page carry the issuer's distinctive header wording?
The comparison folds recurring OCR confusions (`rn→m`, `0→o`, `1→l`) and then applies Levenshtein distance.

**Cost and payoff:** every new template needs its header phrase defined once, at training time.
In return: identity stays stable as scan quality degrades, and templates of different vendors never collide.

---

## What is in this repository

The code belongs to my employer, so it is not here. This repository documents the architecture and the build decisions.

The production layout:

```
server.js            1,233 lines — 34 HTTP endpoints
lib/                 41 modules, 9,273 lines — signature, OCR, templates, confidence, budget
smoke-*.js           16 suites, 232 assertions — all run without internet
scripts/             batch training and schema export
public/              upload, review, and training UIs
vendor/              both Tesseract language packs — no download at runtime
```

The verification gate is one command that runs a chain of 20 sequential steps:
every smoke suite, retraining, and schema export. Any failing step stops the whole chain.

Isolating OCR in a child process came from a real crash: a corrupted JPEG aborts WASM
outside any promise chain. Isolation confines the crash to the child, which is then respawned.

---

## Why I built it

I work as an IT supervisor, and matching scanned documents was repetitive manual work.
The deterministic path sends 0 files to any external service; cloud enrichment is a choice, not a dependency.
