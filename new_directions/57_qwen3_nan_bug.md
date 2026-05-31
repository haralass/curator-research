# 57 — Qwen3-Embedding NaN Bug on macOS: Root Cause, Workarounds, and Alternatives
_Research date: 2026-05-31_

---

## The Bug: What Happens

**Symptom:** `Qwen/Qwen3-Embedding-0.6B` (and other Qwen3-Embedding variants) produces `nan` values in output embedding vectors when encoded via `sentence-transformers` on macOS. NaN values propagate through any downstream cosine similarity or dot-product computation, causing every comparison to return `nan` or `0.0`.

**Which models are confirmed affected:**
- `Qwen/Qwen3-Embedding-0.6B` — confirmed, primary report
- `Qwen/Qwen3-Embedding-4B` — expected affected (same architecture)
- `Qwen/Qwen3-Embedding-8B` — affected (additional separate NaN bug from FP16 overflow)

**Platform specificity:** macOS only. Confirmed working on Windows and Linux. Affects both CPU and MPS (Apple Silicon GPU). Affects both FP32 and FP16 precision.

**Reproduction steps (minimal):**
```python
from sentence_transformers import SentenceTransformer

model = SentenceTransformer(
    "Qwen/Qwen3-Embedding-0.6B",
    tokenizer_kwargs={"padding_side": "left"},
    model_kwargs={
        "torch_dtype": "float32",
        "device_map": "cpu",
    },
    # No attn_implementation specified → defaults to SDPA → NaN
)

embeddings = model.encode(["hello world"])
# embeddings contains NaN on macOS
```

**Confirmed environment:**
- transformers: 4.55.3 · torch: 2.7.1 · sentence-transformers: 5.1.0
- Hardware: MacBook Pro M2, 32GB RAM · macOS Sequoia 15.6.1

---

## Root Cause: SDPA on macOS

**What SDPA is:** Scaled Dot Product Attention (`torch.nn.functional.scaled_dot_product_attention`) is PyTorch's fused attention kernel, made the default attention backend in `transformers` for `torch >= 2.1.1`. On macOS it dispatches to Apple's Metal Performance Shaders (MPS) or the CPU "math" backend.

**Two layered causes:**

1. **The `-inf` / softmax NaN problem (CPU + MPS):** Qwen3 is a decoder-only LLM used as an embedding model with causal masking. When a row of the attention score matrix is entirely masked (happens with padding tokens or last-token pooling), `softmax([-inf, -inf, ..., -inf])` produces `[nan, nan, ..., nan]`. PyTorch's SDPA "math" backend (used on macOS CPU) does not apply a `_safe_softmax` guard. The eager implementation handles this case correctly. Tracked in pytorch/pytorch #103749, partially addressed by PR #133882.

2. **MPS SDPA regression (PyTorch 2.8.0):** PyTorch 2.8.0 introduced specialized MPS SDPA kernels (PR #152781) that incorrectly assume the query tensor's stride when non-contiguous, producing results ~34× divergent from CPU. Tracked in pytorch/pytorch #163597, targeted for milestone 2.9.0, not yet fixed.

**Why it doesn't fail on Linux/CUDA:** CUDA's Flash Attention and mem-efficient attention backends include proper numerical guards for all-masked rows. The macOS SDPA path lacks these guards.

---

## GitHub Issue Thread Summary

**Primary:** [sentence-transformers #3498 — "Qwen3 produces `nan` embeddings (SDPA + macOS)"](https://github.com/UKPLab/sentence-transformers/issues/3498)
- Opened: August 25, 2025 · Status: **Open** as of 2026-05-31
- Key finding: NaN occurs on both CPU and MPS — it is not purely an MPS issue
- Confirmed fix: `attn_implementation="eager"`

**Related issues:**
- pytorch/pytorch #163597 — MPS SDPA regression in 2.8.0
- pytorch/pytorch #103749 — SDPA NaN with padding mask (all-false rows)
- pytorch/pytorch PR #133882 — `_safe_softmax` for fused kernels (merged, macOS CPU path unclear)
- transformers #33615 — Performance drop with `eager` on Qwen2 (context for cost)
- Qwen3-Embedding-8B discussion #21 — Separate FP16 overflow NaN in 8B (different root cause)

**Current status:** No fix has landed in sentence-transformers, transformers, or PyTorch for the macOS SDPA NaN on Qwen3. `attn_implementation="eager"` remains the only confirmed solution.

---

## Workaround 1: `attn_implementation="eager"` ✅ Recommended

```python
from sentence_transformers import SentenceTransformer

model = SentenceTransformer(
    "Qwen/Qwen3-Embedding-0.6B",
    model_kwargs={
        "attn_implementation": "eager",
        "torch_dtype": "float32",
    },
    tokenizer_kwargs={"padding_side": "left"},
)

embeddings = model.encode(
    ["Your text here"],
    normalize_embeddings=True,
)
```

**Does it work on both CPU and MPS?** Yes — resolved at model load time, overrides SDPA dispatch regardless of device.

**Performance cost:** ~2–4× slower attention computation for typical sequences (256–512 tokens). For Curator's background indexing use case, this is acceptable. The gap is smaller than it sounds: macOS SDPA uses the "math" backend (not Flash Attention), which is already relatively slow.

**Side effects:** None numerically. Higher memory usage for long sequences. Not compatible with `torch.compile()` in some configurations.

---

## Workaround 2: Other Approaches (Less Effective)

**`torch.backends.cuda` flags** — CUDA-only, have NO effect on macOS CPU or MPS. Do not use.

**`PYTORCH_ENABLE_MPS_FALLBACK=1`** — causes unsupported MPS ops to fall back to CPU, but does not change SDPA backend selection. Does NOT fix the NaN issue.

**Version pinning to torch < 2.8.0** — avoids the MPS specialized kernel regression (PR #152781) but does NOT fix the CPU SDPA math NaN (pytorch/pytorch #103749), which exists in all PyTorch 2.x on macOS. Pinning to 2.7.1 reduces severity but does not fully resolve the problem.

**Device-conditional eager injection:**
```python
import platform

is_macos = platform.system() == "Darwin"
model_kwargs = {"torch_dtype": torch.float32}
if is_macos:
    model_kwargs["attn_implementation"] = "eager"
```

---

## Performance Impact

| Scenario | SDPA (when working) | Eager |
|---|---|---|
| Short sequences (< 128 tokens) | Baseline | ~1.5–2× slower |
| Medium sequences (128–512 tokens) | Baseline | ~2–3× slower |
| Long sequences (512–2048 tokens) | Baseline | ~3–5× slower |
| Memory usage | Lower (fused kernel) | Higher |

For Curator: files are embedded as 256–512 token chunks. Estimated per-file embedding time on M2 with eager: 20–80ms per chunk vs 10–40ms with working SDPA. Acceptable for background indexing.

---

## Other Affected Models

**Affected (same decoder architecture + causal masking):**
- All `Qwen3-Embedding-*` variants (0.6B, 4B, 8B)
- `Qwen3-*` base/instruct models if used for embedding
- `gte-Qwen2-7B-instruct` and other Qwen2-based embedding models (similar risk)

**Not affected (encoder-only, no causal masking):**
- `BAAI/bge-m3` — BERT-style bidirectional attention, no causal masking. BGE-M3 NaN reports in Ollama are a separate quantization issue.
- `nomic-ai/nomic-embed-text-v1.5` — modified BERT/RoBERTa architecture, no NaN reports
- `sentence-transformers/all-MiniLM-L6-v2` — BERT-based. General MPS issues exist (issue #1736) but unrelated to this bug.

**Key insight:** The SDPA NaN bug is specific to **decoder-only models with causal attention masks** used in last-token pooling embedding mode. Encoder-only models are not affected.

---

## Recommended Implementation for Curator

```python
import platform
import torch
from sentence_transformers import SentenceTransformer

def load_embedding_model(
    model_name: str = "Qwen/Qwen3-Embedding-0.6B",
    device: str | None = None,
) -> SentenceTransformer:
    """
    Load Qwen3-Embedding with macOS SDPA NaN workaround applied automatically.
    
    On macOS (CPU or MPS), SDPA produces NaN embeddings due to a PyTorch bug
    with causal attention masking. We force attn_implementation='eager' to bypass
    the broken SDPA code path.
    See: https://github.com/UKPLab/sentence-transformers/issues/3498
    """
    is_macos = platform.system() == "Darwin"
    
    model_kwargs: dict = {"torch_dtype": torch.float32}
    
    if is_macos:
        model_kwargs["attn_implementation"] = "eager"
    
    if device is None:
        device = "mps" if (is_macos and torch.backends.mps.is_available()) else "cpu"
    
    return SentenceTransformer(
        model_name,
        model_kwargs=model_kwargs,
        tokenizer_kwargs={"padding_side": "left"},
        device=device,
    )


def embed(
    model: SentenceTransformer,
    texts: list[str],
    is_query: bool = False,
) -> list[list[float]]:
    return model.encode(
        texts,
        prompt_name="query" if is_query else None,
        normalize_embeddings=True,
        batch_size=32,
    ).tolist()
```

**Notes:**
- `padding_side="left"` is required for Qwen3 (decoder model) for correct EOS/last-token pooling
- `normalize_embeddings=True` for unit-norm vectors (cosine similarity)
- `prompt_name="query"` uses Qwen3's built-in query instruction prefix for search queries; omit for document embedding
- `batch_size=32` is a reasonable default on M2 with eager attention; reduce if OOM

---

## Version Matrix

| Component | Affected versions | Fixed in | Notes |
|---|---|---|---|
| sentence-transformers | 5.0.0–5.2.0 (all current) | Not fixed | Issue #3498 open |
| transformers | 4.51.0+ (all Qwen3-supporting) | Not fixed | SDPA default for torch >= 2.1.1 |
| PyTorch (MPS specialized kernel) | 2.8.0 | Targeted 2.9.0 | pytorch #163597 |
| PyTorch (CPU SDPA math NaN) | 2.0+ | Partial (PR #133882) | macOS CPU path unclear |

**Safe combination for Curator:**
```
torch==2.7.1           # avoids MPS 2.8.0 regression
transformers>=4.51.0   # required for Qwen3
sentence-transformers>=5.0.0
# + attn_implementation="eager" unconditionally on macOS
```

---

## Key References

1. sentence-transformers #3498 — Qwen3 produces nan embeddings (SDPA + macOS). https://github.com/UKPLab/sentence-transformers/issues/3498
2. pytorch/pytorch #163597 — SDPA MPS regression on 2.8.0. https://github.com/pytorch/pytorch/issues/163597
3. pytorch/pytorch #103749 — SDPA produces NaN with padding mask. https://github.com/pytorch/pytorch/issues/103749
4. pytorch/pytorch PR #133882 — Update fused kernels with _safe_softmax. https://github.com/pytorch/pytorch/pull/133882
5. Qwen3-Embedding-8B discussion #21 — FP16 overflow NaN in 8B model. https://huggingface.co/Qwen/Qwen3-Embedding-8B/discussions/21
6. transformers #33615 — Qwen2 performance drop with eager attention. https://github.com/huggingface/transformers/issues/33615
7. sentence-transformers #1736 — General MPS issues with BERT models. https://github.com/UKPLab/sentence-transformers/issues/1736
8. Qwen/Qwen3-Embedding-0.6B model card. https://huggingface.co/Qwen/Qwen3-Embedding-0.6B
9. PyTorch SDPA tutorial. https://docs.pytorch.org/tutorials/intermediate/scaled_dot_product_attention_tutorial.html
