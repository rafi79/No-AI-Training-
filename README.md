# PDFGuard 🛡️



[![Python](https://img.shields.io/badge/Python-3.10%2B-blue)](https://python.org)


> **TL;DR** — PDFGuard makes a PDF look completely normal to human readers while causing safety-aligned AI tools (ChatGPT, Claude, Gemini, Qwen, Kimi) to refuse or fail to extract its content. Built to defend academic peer review from lazy AI-assisted reviewing.

---

## 🔍 The Problem

Peer reviewers are increasingly uploading manuscripts directly to ChatGPT or Claude, copy-pasting the output as a review, and submitting it — without actually reading the paper. This is:

- **Intellectually dishonest** — it bypasses the entire point of expert evaluation
- **Scientifically dangerous** — LLM summaries miss subtle flaws, overstated claims, and missing ablations
- **Growing fast** — multiple studies (Liang et al. 2024, Kobak et al. 2024) document measurable AI review contamination at NeurIPS, ICML, and ACL

PDFGuard is a **proactive, document-level countermeasure** — it protects the manuscript before review even begins.

---

## ✅ What PDFGuard Does

| For Humans | For AI Tools |
|---|---|
| ✅ Reads perfectly normally | 🚫 Safety classifiers triggered |
| ✅ Copy-paste works | 🚫 Extraction refused or degraded |
| ✅ Print / share unchanged | 🚫 Prompt injection directives fire |
| ✅ No visual difference | 🚫 Metadata signals policy violation |
| ✅ File size increase ~38% only | 🚫 Vision pipeline disrupted |

---

## 🏗️ How It Works — Five Injection Layers

PDFGuard applies five simultaneous protection layers, each targeting a different channel of the LLM document-processing pipeline:

```
Input PDF
    │
    ├─ Layer 1: Zero-Width Unicode Injection
    │           U+200B / U+200C / U+200D / U+FEFF injected every 3–5 chars
    │           → corrupts word-boundary detection, embeds cryptographic token
    │
    ├─ Layer 2: Invisible Safety-Trigger Overlay
    │           Dense grid of trigger strings at α=0.02 opacity (below human threshold)
    │           → activates RLHF/Constitutional AI safety classifiers
    │
    ├─ Layer 3: Prompt Injection Directives
    │           [SYSTEM OVERRIDE] commands at 1pt near-white text
    │           placed at doc start, after each section heading, and doc end
    │           → exploits LLM instruction-following generalisation
    │
    ├─ Layer 4: Adversarial Micro-Noise Perturbation
    │           1,000 Halton-positioned near-white pixels per page (α=0.02–0.05)
    │           + Variation Selector chars (U+FE00–U+FE0F) as 0.5pt overlays
    │           → disrupts vision-pipeline OCR and image-based extraction
    │
    └─ Layer 5: Metadata Poisoning
                /Title, /Author, /Subject, /Keywords, /Producer overwritten
                + custom /AIAccess, /ExtractionPermission, /ProtectionToken fields
                → triggers metadata-reading stage of LLM pipelines
                    │
                    ▼
             Protected PDF
             (visually identical to input)
```

### Adaptive Density Scaling

Short conference papers (4–6 pages) have a smaller total character and pixel budget. PDFGuard compensates with a page-count multiplier:

```
μ(n) = 4.0      if n ≤ 5   (conference papers)
       3.0      if n ≤ 10
       max(1, min(3, n/10))  otherwise

Page 1 receives an additional 5× multiplier.
```

---

## 📊 Results

Evaluated on 30 arXiv CS conference papers (4–6 pages) across 7 production LLM systems:

| LLM System | Blocked % | Degraded % | Extracted % | Human Readability |
|---|---|---|---|---|
| Claude 3.5 Sonnet | **80%** | 13% | 7% | 4.8/5 |
| GPT-4o | **73%** | 20% | 7% | 4.8/5 |
| Gemini 1.5 Pro | **67%** | 23% | 10% | 4.9/5 |
| Qwen-Long | **60%** | 27% | 13% | 4.7/5 |
| Kimi (Moonshot AI) | **63%** | 20% | 17% | 4.8/5 |
| Mistral Large | 7% | 23% | 70% | 4.9/5 |
| DeepSeek-V2 | 10% | 20% | 70% | 4.8/5 |
| **Average (all)** | **51%** | **21%** | **28%** | **4.81/5** |

**Safety-aligned models (top 5): 88% block-or-degraded rate.**
Mistral and DeepSeek are resilient due to lighter alignment training — they remain higher-risk tools for AI-assisted lazy reviewing.

### Ablation — Which Layer Contributes Most?

| Configuration | Block Rate | Drop |
|---|---|---|
| Full PDFGuard (all 5 layers) | 73% | — |
| Without Layer 2 (Safety Triggers) | 41% | −32% |
| Without Layer 3 (Prompt Injection) | 52% | −21% |
| Without Layer 5 (Metadata) | 65% | −8% |
| Without Layer 1 (Unicode) | 68% | −5% |
| Without Layer 4 (Micro-noise) | 70% | −3% |

---

## 🧪 Techniques Explored (20+ Before Proposing This)

During development, 19 individual techniques were tested before converging on the five-layer fusion. Summary:

| Category | Methods Tried | Outcome |
|---|---|---|
| Rendering-layer | White-on-white text, overlay rect, near-white font, font scramble | ❌ All failed — LLMs extract colour-agnostic |
| Structural PDF | Image-only, DRM flags, AES encryption, Bates spam, comment noise | ❌ Bypassed or destroys readability |
| Single-channel injection | Homoglyphs, annotations, metadata-only, triggers-only, injection-only, zero-width-only, JPEG artifacts, noise-only | ⚠️ Partial — 10–48% block rate individually |
| Two-layer hybrid | Image + triggers, noise + metadata | ⚠️ Partial — 52–55%, or impractical file size |
| **Five-layer fusion (PDFGuard)** | All channels simultaneously | ✅ **73% block, 93% B+D, 4.8/5 readability** |

**Key insight**: any single-channel technique is bypassed once the LLM's legitimate document context outweighs the adversarial signal. Only simultaneous cross-channel injection is robust.

---

## 🚀 Quick Start

### Install

```bash
git clone https://github.com/[username]/PDFGuard.git
cd PDFGuard
pip install -r requirements.txt
```

### Requirements

```
pypdf>=3.0
reportlab>=4.0
Pillow>=10.0
streamlit>=1.30
PyMuPDF>=1.23
```

### Protect a PDF (Python)

```python
from pdfguard import AdversarialPDFProtector

protector = AdversarialPDFProtector()

with open("my_paper.pdf", "rb") as f:
    input_bytes = f.read()

protected_bytes, token, stats = protector.protect_pdf(input_bytes)

with open("my_paper_protected.pdf", "wb") as f:
    f.write(protected_bytes)

print(f"Protection token: {token}")
print(f"File size increase: {stats['increase_percent']:.1f}%")
print(f"Pages processed: {stats['pages']}")
```

### Run the Streamlit App Locally

```bash
streamlit run app.py
```

Or use the live demo at **[pdfguard.streamlit.app](https://pdfguard.streamlit.app)**

---



## ⚠️ Limitations

PDFGuard is not a complete or permanent solution. Known bypass strategies:

- **Text preprocessing** — stripping zero-width Unicode and near-white text before passing to the LLM defeats Layers 1–2
- **Less-aligned models** — Mistral and DeepSeek are largely immune due to lighter safety training
- **Vision-only pipelines** — photographing pages defeats text-layer injection; Layer 4 partially addresses this
- **Arms race** — once this technique is widely known, AI providers may add preprocessing to neutralise it
- **Manual transcription** — a determined reviewer who retyps the paper bypasses everything

PDFGuard raises the cost and friction of AI-assisted lazy reviewing. It does not make it impossible.

---

## 🔬 Ethical Note

PDFGuard contains **no actually harmful content**. All injected material consists of:
- Fake policy-violation warning strings
- Prompt override directives
- Invisible Unicode characters
- Near-white pixel noise
- Crafted metadata fields

The intent is to exploit safety classifier over-triggering in aligned models — not to produce, distribute, or embed genuinely harmful material. The technique is analogous to a "no photography" sign in a museum: it creates a social/technical barrier without itself being harmful.


---

*This project was developed as part of research into academic integrity and AI safety at [University Name].*
