# Local VLMs για Multimodal File Understanding
**Compiled:** May 2026 | **Relevance:** Κατανόηση τι είναι ΜΕΣΑ στα αρχεία, όχι μόνο metadata

---

## Landscape 2026

Ollama's switch σε MLX backend (March 2026) διπλασίασε decode throughput. MLX είναι 30–60% faster από llama.cpp Metal, 3–4x faster σε prompt processing. Αποτέλεσμα: **7B vision model τοπικά σε M3/M4 Mac = genuinely usable** για batch file processing.

---

## Best Local VLMs per Use Case

### Document / PDF Understanding
| Model | Size | Standout | How to Run |
|---|---|---|---|
| **Qwen2.5-VL** | 3B / 7B | Best για structured docs, tables, charts, OCR. 7B rivals GPT-4o στο DocVQA | Ollama `qwen2.5vl`, MLX-VLM |
| **Granite Vision 3.2** | 2B | IBM Research. Matches 10B models στο DocVQA/ChartQA. CPU-capable | Ollama |
| **olmOCR** | 7B | Fine-tuned Qwen2-VL για PDF OCR. Preserves reading order, tables | LM Studio, MLX |
| **FastVLM** (Apple) | 0.5B–7B | Apple CVPR 2025. **85x faster TTFT** vs LLaVA. MLX + macOS app | [github.com/apple/ml-fastvlm](https://github.com/apple/ml-fastvlm) |

### General Image Understanding
| Model | Size | Standout |
|---|---|---|
| **Moondream 2** | 1.8B | Ultra-fast, <4GB RAM, basic image description |
| **Qwen2.5-VL 7B** | 7B | Best quality |
| **Llama 3.2 Vision** | 11B | Solid general visual reasoning |

### Embeddings / Retrieval (Non-Generative)
| Model | Size | Standout |
|---|---|---|
| **Nomic Embed Multimodal** | ~1B | Images + text σε same embedding space. SOTA σε PDF/chart retrieval |
| **ColQwen2.5** | 3B | Visual page-level multi-vector embeddings. Best για document search |
| **ColPali v1.2** | 3B | PaliGemma-based. Apple Silicon via `device_map="mps"` |

---

## Document Understanding Pipeline

### Routing πριν VLM (κρίσιμο για speed)

```
PDF (digital, έχει text) → Docling → text LLM classification
PDF (scanned/image-only) → pdf2image → Qwen2.5-VL 7B
Image file             → Qwen2.5-VL 7B ή Moondream (αν speed)
Code file (.py, .js)   → text LLM (Qwen2.5-Coder 7B) — VLM δεν χρειάζεται
Document (.docx, .pptx) → Docling → text LLM
```

### Docling (Primary Path για PDFs)
[IBM Research / Linux Foundation](https://github.com/docling-project/docling) — 30x faster από raw OCR. Extracts tables, headings, code blocks. macOS arm64. LangChain + LlamaIndex integration.

**OCR vs VLM benchmark:**
| Approach | Accuracy (complex docs) |
|---|---|
| Tesseract OCR | ~42% |
| OCR + LLM correction | ~55% |
| VLM (Qwen2.5-VL) | 63–67% |
| Docling (digital PDFs) | High |

→ **Docling πρώτα, VLM fallback για scanned/complex**.

### PDF Pipeline Code

```python
from pdf2image import convert_from_path
import ollama

def understand_pdf(path):
    # Try Docling first (digital PDF)
    try:
        from docling.document_converter import DocumentConverter
        result = DocumentConverter().convert(path)
        text = result.document.export_to_markdown()
        return classify_text(text)  # text LLM
    except:
        pass
    
    # Fallback: VLM on first 3 pages
    pages = convert_from_path(path, last_page=3)
    for page_img in pages:
        response = ollama.chat(
            model="qwen2.5vl:7b",
            messages=[{
                "role": "user",
                "content": "Classify this document. Output JSON: {type, category, summary}",
                "images": [page_img]
            }]
        )
    return response
```

---

## Image Understanding

**Prompt για Curator:**
```
"Describe this image in 1-2 sentences. Classify as:
 [photo, screenshot, diagram, chart, document, artwork, other].
 Output JSON: {description, category, contains_text: bool}"
```

- **Moondream 2** (1.8B): few seconds/image, batch scanning
- **Qwen2.5-VL**: better accuracy

---

## Code Understanding

**VLMs δεν χρειάζονται για code** — text LLMs είναι καλύτεροι και γρηγορότεροι:

```
Qwen2.5-Coder 7B (Ollama: qwen2.5-coder) — purpose-trained
Prompt: "Summarize what this file does. Identify language + purpose.
         Output JSON: {language, purpose, one_line_summary}"
```

---

## Multimodal Embeddings — Nomic Embed Multimodal

Generates 3584-dim vectors για images, PDFs, charts, AND text **σε ένα shared embedding space**.

→ Text query "tax document" βρίσκει matching PDF page images χωρίς separate pipeline.

**HuggingFace:** `nomic-ai/nomic-embed-multimodal-7b`

---

## ColQwen / ColPali για Document Retrieval

Multi-vector (patch-level) embeddings ανά PDF page. Captures visual layout, tables, figures.

```python
from byaldi import RAGMultiModalModel
model = RAGMultiModalModel.from_pretrained("vidore/colqwen2-v1.0")
model.index(input_path="./my_documents/", index_name="curator_index")
results = model.search("tax return 2024", k=5)
```

Apple Silicon: `device_map="mps"` via transformers.
**Light-ColQwen** για 3–4x smaller storage footprint.

---

## Performance Benchmarks (Apple M-series)

| Chip | Model | Speed |
|---|---|---|
| M3 Max | Qwen2.5-VL 7B | ~15–25 tok/s, first token ~3–5s |
| Any M | Granite Vision 3.2 2B | CPU-capable, reasonable time |
| Any M | Moondream 2 | < 2 sec/image |

**Vision encoder:** +1.5–2 sec per NEW image. Repeated images: 28x speedup με prefix caching (vllm-mlx).

**Practical:** Για batch indexing → 7B feasible. Για real-time → 2–3B models.

---

## Recommended Stack για Curator

```
Orchestration:      Python asyncio (batch processing)
PDF extraction:     Docling (primary), pdf2image (scanned fallback)
VLM inference:      Ollama (qwen2.5vl:7b, granite3.2-vision:2b, moondream)
                    ή mlx-vlm για higher throughput
Embeddings:         nomic-embed-text (text) + nomic-embed-multimodal (images/PDFs)
Vector DB:          LanceDB (native Apple Silicon, no server needed)
Code:               Ollama qwen2.5-coder:7b
Document retrieval: ColQwen2.5 via Byaldi (για rich search)
```

---

## Closest Existing System

**[QiuYannnn/Local-File-Organizer](https://github.com/QiuYannnn/Local-File-Organizer)** — AI file organizer χρησιμοποιώντας Llama3.2 3B + LLaVA v1.6, Nexa SDK, 100% local. Πιο κοντινό existing analogue.

---

## Open Problems

1. **Indexing speed:** 1,000 files × 3–5 sec = ~1 hour. Needs async batching + incremental indexing
2. **Memory pressure:** 7B VLM + macOS = little RAM left. 3B models για background processing
3. **Multi-page PDFs:** Process page 1 + first 3 pages για classification; full pass on demand
4. **Nomic Embed Multimodal:** Δεν είναι ακόμα first-class Ollama model — χρειάζεται HuggingFace
5. **ColPali storage:** 10K documents × 1024 vectors/page = 10M+ vectors. Light-ColQwen για large collections

---

## Sources
- [Docling](https://github.com/docling-project/docling)
- [Qwen2.5-VL Technical Report](https://arxiv.org/pdf/2502.13923)
- [FastVLM — Apple ML Research](https://machinelearning.apple.com/research/fast-vision-language-models)
- [Granite Vision 3.2 — IBM](https://huggingface.co/ibm-granite/granite-vision-3.2-2b)
- [ColPali — arXiv 2407.01449](https://arxiv.org/abs/2407.01449)
- [Nomic Embed Multimodal](https://www.nomic.ai/news/nomic-embed-multimodal)
- [Local File Organizer](https://github.com/QiuYannnn/Local-File-Organizer)
- [olmOCR](https://olmocr.allenai.org/)
- [Byaldi](https://github.com/answerdotai/byaldi)
- [vllm-mlx](https://github.com/waybarrios/vllm-mlx)
