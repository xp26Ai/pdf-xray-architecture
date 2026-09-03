# pdf-xray-architecture

[العربية](README.md)

**Put in a scanned PDF. Get out the fields, each with its source and confidence. No internet needed.**

> **This system runs in production at my employer, and the code belongs to them. This repository documents the architecture, the engineering decisions, and how it was built.**

A deterministic extraction system for scanned documents. It is 16,543 lines of JavaScript.
One server with 34 endpoints, 41 modules in `lib/`, and 16 smoke-test suites.
Arabic and English OCR run on the device through Tesseract WASM.
The AI layer only adds extra data on top of the result. The result never needs it.

---

## The problem

Invoices, purchase orders, and delivery notes come in as scans. The quality changes from one file to the next. What we need is their exact numbers.

Sending every file to a cloud model has 3 flaws. The result is not reproducible.
The data leaves the server. The service stops when the internet drops, or when the quota runs out.

Plain text matching is also weak. Scan noise changes the characters from one copy to the next.

So I needed a deterministic path. It gives the same answer every time. Each field also carries an honest confidence.

---

## How it works

| Layer | Role |
|---|---|
| Document signature | Finds the issuer from the header wording. It accepts common OCR errors |
| Local OCR | Tesseract WASM runs in two isolated child processes, one Arabic and one English. 24.9 MB of language data sits inside the project |
| Classification | 5 document types, from weighted Arabic and English keywords. Zero external calls |
| Trained templates | You teach label-to-value anchors in a web UI. The server then applies them to every matching document |
| Confidence and provenance | Every field carries two separate confidences `{found, value}`. It also carries its extraction source and a trust class |
| Optional AI enrichment | A daily budget of 450 calls per model. A queue of 14 calls per minute. An on-disk SHA-256 cache |

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

The confidence rules are deterministic. They live in one pure module with no I/O:

- `found`: how sure we are that we found the field. Exact anchor 0.95, fuzzy anchor 0.75, text pattern 0.70.
- `value`: how correct the text is. It starts from the OCR confidence. It has a floor of 0.85 when the expected format matches.
- When the deterministic path and the cloud model agree, `value` goes up by 0.10.
- When they disagree, `value` caps at 0.50. The field then goes to human review.
- Any value below 0.7 gets `review: true`. It moves to the top of the list.

The enrichment layer never throws. The key may be missing, the quota may be spent,
or the call may fail. In all cases the full deterministic result comes back, with a warning attached.

---

## The key design decision

**The problem:** document-to-template matching started as Jaccard similarity over all first-page tokens.
I measured it on 24 real scans from the same issuer. Against one template, the score moved between 0.52 and 0.88.
One real document fell under the 0.50 threshold and was rejected. The similarity was measuring scan noise, not identity.

**The decision:** drop the whole-page similarity. Use a document signature instead. It asks one question only.
Does this page carry the header wording of that issuer?
The comparison first folds common OCR mix-ups (`rn→m`, `0→o`, `1→l`). Then it applies Levenshtein distance.

**Cost and payoff:** each new template needs its header phrase set once, at training time.
In return, the identity stays stable when scan quality drops. And templates from different vendors never collide.

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

The verification gate is one command. It runs a chain of 20 steps in order.
The chain covers every smoke suite, retraining, and schema export. Any step that fails stops the whole chain.

I isolated OCR in a child process after a real crash. A corrupted JPEG aborts WASM
outside any promise chain. The isolation keeps the crash inside the child. The server then respawns it.

---

## Why I built it

I work as an IT supervisor. Matching scanned documents was repeated manual work.
The deterministic path sends 0 files to any outside service. Cloud enrichment is a choice, not a dependency.
