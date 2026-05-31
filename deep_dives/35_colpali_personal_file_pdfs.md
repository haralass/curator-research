# ColPali for Embedding Personal PDF Files in Curator

**arXiv: 2407.01449 | ICLR 2025**
**Date: 2026-05-31**

---

## Overview

ColPali (Collaborative Patch-wise Late Interaction) is a vision-language retrieval model accepted at ICLR 2025 that embeds document pages directly from screenshots, bypassing OCR entirely. Rather than extracting text and then embedding it — the approach Curator currently uses with BGE-M3 — ColPali feeds raw page images into a PaliGemma-3B vision-language model and produces multi-vector patch embeddings that capture both textual and visual content simultaneously. For Curator, this unlocks reliable retrieval over scanned PDFs, academic papers with dense figures, infographics, tables, and any document where text extraction degrades or fails.

---

## 1. Architecture

ColPali is a two-stage system built on top of PaliGemma-3B.

**Vision encoder.** Each document page (rendered as an image, typically 448×448 or higher) is split into a 32×32 grid of patches by SigLIP-So400m, Google's vision transformer. This yields 1,024 visual patch tokens per page.

**Language model contextualisation.** The patch embeddings are linearly projected and passed as "soft" tokens into Gemma 2B, which contextualises them — letting the model attend across patches and produce richer, position-aware embeddings. Six additional tokens come from a short instruction prefix, giving 1,030 vectors per page total.

**Projection to retrieval space.** A final linear projection compresses each token embedding to D=128 dimensions, matching the original ColBERT retrieval space. This dimension was chosen deliberately to align storage costs with text-based dense retrievers.

**Late interaction (MaxSim).** At query time, the query text is tokenised and each token is projected to the same 128-dimensional space. Relevance is computed as:

```
score(q, d) = Σ_i  max_j  (q_i · d_j)
```

For each query token i, find the document patch j with maximum cosine similarity, then sum across all query tokens. This is the ColBERT MaxSim operator applied over image patches instead of text tokens. It allows fine-grained alignment between query concepts and specific visual regions of the page.

**Interpretability.** Because similarity is computed patch-by-patch, you can visualise exactly which regions of a page drove a retrieval result — a significant advantage for debugging and building trust in a personal file organiser.

---

## 2. ViDoRe Benchmark

The authors introduce ViDoRe (Visual Document Retrieval Benchmark) to evaluate page-level retrieval across 10 datasets spanning medical, scientific, and administrative domains in English and French, with 500–1,000 documents per task. The primary metric is NDCG@5.

| System | Avg NDCG@5 |
|---|---|
| ColPali (original paper) | **81.3** |
| Unstructured + BGE-M3 (text pipeline) | 67.0 |
| BM25 | 65.5 |
| SigLIP (no late interaction) | 51.4 |

ColPali's largest gains are on visually complex tasks: 83.9 on TabFQuAD (tables), 81.8 on InfoVQA (infographics), and strong results on ArxivQA (academic figures). Even on text-centric documents it matches or exceeds the text pipeline, because the language model backbone can still "read" the rendered text from the image.

Since the paper, the ecosystem has evolved. ColQwen2-v1.0 (which replaces PaliGemma with Qwen2-VL) achieves **87.3 NDCG@5** on ViDoRe V1. The ViDoRe V2 benchmark (released 2025) raises the bar further; current SOTA models exceed 90. ColQwen2 is the recommended default for new projects due to its Apache 2.0 license and higher accuracy.

---

## 3. Available Models and Memory Requirements

The `colpali-engine` package unifies several ColVision models under a common API:

| Model | Backbone | ViDoRe V1 | License | Notes |
|---|---|---|---|---|
| `vidore/colpali-v1.3` | PaliGemma-3B | ~81 | Gemma | Original paper model |
| `vidore/colqwen2-v1.0` | Qwen2-VL-2B | 87.3 | Apache 2.0 | Recommended default |
| `vidore/colqwen2.5-v0.1` | Qwen2.5-VL-3B | ~88+ | Apache 2.0 | Latest at time of writing |
| ColSmol variants | SmolVLM (~256–500M) | lower | Apache 2.0 | Lightweight, lower accuracy |

**Memory on Apple Silicon.** PaliGemma-3B in bfloat16 requires roughly 6–8 GB of unified memory. ColQwen2-v1.0 (Qwen2-VL-2B backbone) is slightly lighter at approximately 4–6 GB in bfloat16. On an M2 Pro with 16 GB unified memory you have comfortable headroom for ColQwen2. The ColSmol models (under 1B parameters) fit in ~2 GB but lag behind on novel data distributions. For a personal use case with hundreds rather than millions of PDFs, ColQwen2 is the sweet spot.

---

## 4. Local Inference on Apple Silicon

ColPali supports PyTorch MPS (Metal Performance Shaders) natively. The `device_map="mps"` argument works out of the box with bfloat16. There are no MPS-specific patches required in recent versions of colpali-engine.

**Installation:**

```bash
pip install colpali-engine   # requires Python >=3.10, <3.15
```

**Loading the model on MPS:**

```python
import torch
from colpali_engine.models import ColQwen2, ColQwen2Processor

model = ColQwen2.from_pretrained(
    "vidore/colqwen2-v1.0",
    torch_dtype=torch.bfloat16,
    device_map="mps",
).eval()

processor = ColQwen2Processor.from_pretrained("vidore/colqwen2-v1.0")
```

**Embedding a PDF page:**

```python
from pdf2image import convert_from_path

# Render PDF pages as PIL images
pages = convert_from_path("paper.pdf", dpi=150)

# Embed a batch of pages
batch = processor.process_images(pages).to(model.device)
with torch.no_grad():
    page_embeddings = model(**batch)  # shape: (n_pages, 1030, 128)
```

**Querying:**

```python
queries = ["neural network architecture diagram", "experimental results table"]
query_batch = processor.process_queries(queries).to(model.device)

with torch.no_grad():
    query_embeddings = model(**query_batch)  # shape: (n_queries, n_tokens, 128)

scores = processor.score_multi_vector(query_embeddings, page_embeddings)
# scores shape: (n_queries, n_pages), higher = more relevant
```

**Latency estimates on Apple Silicon.** Query encoding (text only) takes approximately 30 ms, comparable to a standard dense retriever. Page image encoding is the bottleneck: for ColQwen2 on an M2 Pro with MPS, expect roughly 1–3 seconds per page depending on resolution and batch size. For Curator's use case — indexing new files in the background rather than real-time — this is acceptable. A 50-page academic PDF indexes in under 3 minutes.

---

## 5. Storage: Embedding Structure and Size

Each page produces 1,030 vectors × 128 dimensions × 2 bytes (bfloat16) = **~257 KB per page**. This is larger than a single BGE-M3 embedding (1,024 floats × 4 bytes = ~4 KB) but the multi-vector representation carries far more retrieval information.

Practical compression options:
- **Token pooling** (built into colpali-engine): reduces the 1,030 vectors by ~67% via clustering, bringing storage to ~85 KB per page with only 2% performance loss (97.8% retention). This is the recommended option for Curator.
- **Binary quantisation**: binarise each 128-dim vector to a 128-bit (16-byte) bitstring, reducing to ~8 KB per page — a 32× reduction at the cost of retrieval quality.

For a personal file collection of 1,000 PDF pages with token pooling: ~85 MB total — entirely manageable on disk.

---

## 6. Integration with ChromaDB

ChromaDB does not natively support multi-vector (ColBERT-style) embeddings in a single collection entry, because it assumes one vector per document. The standard workaround for Curator is to store each page's patch vectors as individual records and aggregate scores at retrieval time, or to use a pre-aggregated pooled single vector.

**Option A: Pooled single vector (simplest, recommended for Curator v1).**

Average the 1,030 patch vectors into one 128-dimensional vector per page and store that in ChromaDB as a standard embedding. You lose the fine-grained late interaction but retain vision-aware semantics.

```python
import chromadb
import numpy as np

client = chromadb.PersistentClient(path="~/Library/Application Support/Curator/chroma")
collection = client.get_or_create_collection(
    name="colpali_pages",
    metadata={"hnsw:space": "cosine"},
)

# page_embeddings: (n_pages, 1030, 128) tensor
pooled = page_embeddings.mean(dim=1).cpu().float().numpy()  # (n_pages, 128)

collection.add(
    ids=[f"file_{file_id}_page_{i}" for i in range(len(pages))],
    embeddings=pooled.tolist(),
    metadatas=[{"file_id": file_id, "page": i, "path": str(path)} for i in range(len(pages))],
)
```

Query:

```python
q_pooled = query_embeddings.mean(dim=1).cpu().float().numpy()
results = collection.query(query_embeddings=q_pooled.tolist(), n_results=10)
```

**Option B: Full multi-vector with re-ranking.**

Store pooled embeddings in ChromaDB for fast ANN retrieval, keep the full patch tensors on disk (as `.pt` files keyed by page ID), retrieve top-K candidates from ChromaDB, then load the full tensors and recompute exact MaxSim scores for final ranking.

```python
# Fast ANN pass
candidates = collection.query(query_embeddings=q_pooled.tolist(), n_results=50)

# Load full patch tensors for candidates
candidate_tensors = [torch.load(f"embeddings/{page_id}.pt") for page_id in candidate_ids]
candidate_stack = torch.stack(candidate_tensors)  # (50, 1030, 128)

# Exact MaxSim re-ranking
exact_scores = processor.score_multi_vector(query_embeddings, candidate_stack)
top_pages = exact_scores.argsort(descending=True)[:10]
```

Option B gives full ColPali accuracy; Option A is a practical starting point with 80–90% of the quality at a fraction of the engineering complexity.

---

## 7. When to Use ColPali vs BGE-M3

Curator should route documents to the appropriate embedding pipeline based on their type.

| Document type | Recommended embedder | Reason |
|---|---|---|
| Scanned PDFs (no text layer) | **ColPali** | Text extraction fails; ColPali reads the rendered image |
| Academic papers with figures/tables | **ColPali** | BGE-M3 ignores visual structure; ColPali encodes it |
| Infographics, slides, receipts | **ColPali** | Inherently visual; OCR is fragile |
| Clean text PDFs (contracts, reports) | **BGE-M3** | Faster, cheaper, equivalent accuracy for pure text |
| Mixed content (partly scanned) | **ColPali** | Safer choice when text extraction is uncertain |
| Photos and images | **llava:7b** (existing) | Captioning pipeline remains appropriate |

**Detection heuristic.** Use `pdfminer` or `pymupdf` to count extractable characters per page. If the page yields fewer than 100 characters, treat it as visually dominant and route to ColPali. This is a cheap pre-check that adds negligible overhead.

```python
import fitz  # pymupdf

def classify_pdf_page(pdf_path: str, page_num: int) -> str:
    doc = fitz.open(pdf_path)
    page = doc[page_num]
    text = page.get_text().strip()
    if len(text) < 100:
        return "colpali"
    return "bge_m3"
```

**Hybrid retrieval.** At search time, query both ChromaDB collections (BGE-M3 and ColPali) and merge results by score. Since both use cosine similarity, a simple union with score normalisation works:

```python
def merged_results(query: str, top_k: int = 10):
    text_results = bge_m3_collection.query(...)
    visual_results = colpali_collection.query(...)
    combined = text_results + visual_results
    combined.sort(key=lambda r: r["score"], reverse=True)
    return combined[:top_k]
```

---

## 8. Implementation Plan for Curator

A practical integration requires the following additions to the Curator codebase:

**Step 1: Add colpali-engine as a dependency.** Pin `colpali-engine>=0.3` and `pdf2image` in `requirements.txt`. `pdf2image` wraps `poppler` for PDF-to-image rendering; install poppler via `brew install poppler`.

**Step 2: Create `CuratorColPaliIndexer` class.** Mirror the existing `BGEIndexer` interface. On first run, download ColQwen2-v1.0 from HuggingFace (~4 GB, cached to `~/.cache/huggingface`). Load the model lazily to avoid memory pressure when ColPali is not needed.

**Step 3: Background indexing pipeline.** When Curator detects a new PDF file (via FSEvents), run the page classifier. For pages routed to ColPali, render them with pdf2image at 150 DPI, embed in batches of 4–8 pages (adjust for available memory), and persist pooled embeddings to the `colpali_pages` ChromaDB collection. Store full patch tensors to `~/Library/Application Support/Curator/colpali_patches/<file_id>/page_<n>.pt` for optional re-ranking later.

**Step 4: Query routing.** At search time, query both collections. If the user's query is visual in nature (contains words like "table", "figure", "graph", "chart", "diagram"), weight ColPali results higher.

**Step 5: Result display.** Since ColPali results are page-level, display the page number alongside the filename in Curator's search results. For confirmed matches with full patch tensors, optionally render the similarity heatmap overlay to show the user which regions matched.

---

## 9. Novelty in the Context of Curator

Curator's current architecture is text-centric: BGE-M3 requires clean text extraction and llava:7b handles images with a captioning pass. ColPali fills a genuine gap — documents where neither pipeline performs well: scanned academic papers, hand-annotated notes, receipts, slides, and mixed-content PDFs.

The key novelty of ColPali from a systems perspective is that it eliminates the preprocessing stack (OCR → layout detection → text chunking) that accounts for most of the engineering fragility in document RAG pipelines. By treating every page as an image, it becomes robust to any PDF structure. For a personal file organiser managing an unpredictable mix of document types, this robustness is more valuable than marginal accuracy improvements on well-formed text documents.

The ColBERT-style late interaction over image patches is also architecturally elegant for Curator's use case: it allows sparse, interpretable matching — the system can show users *where* in a document their query was found, not just that it was found. This aligns with Curator's goal of being a trustworthy, explainable organiser rather than a black-box search engine.

---

## Sources

- [ColPali: Efficient Document Retrieval with Vision Language Models (arXiv:2407.01449)](https://arxiv.org/abs/2407.01449)
- [ColPali HTML paper (arXiv v2)](https://arxiv.org/html/2407.01449v2)
- [ColPali HuggingFace Blog](https://huggingface.co/blog/manu/colpali)
- [colpali-engine GitHub (illuin-tech)](https://github.com/illuin-tech/colpali)
- [vidore/colpali-v1.3 Model Card](https://huggingface.co/vidore/colpali-v1.3)
- [ViDoRe Benchmark Collection](https://huggingface.co/collections/vidore/vidore-benchmark)
- [Scaling ColPali to Billions of PDFs — Vespa Blog](https://blog.vespa.ai/scaling-colpali-to-billions/)
- [ColPali + Qdrant Integration — Qdrant Blog](https://qdrant.tech/blog/qdrant-colpali/)
- [Late Interaction Models Overview — Weaviate](https://weaviate.io/blog/late-interaction-overview)
- [ColPali: Enhanced Document Retrieval — Zilliz Blog](https://zilliz.com/blog/colpali-enhanced-doc-retrieval-with-vision-language-models-and-colbert-strategy)
- [ViDoRe Benchmark V2 Paper](https://arxiv.org/pdf/2505.17166)
