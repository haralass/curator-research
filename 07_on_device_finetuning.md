# On-Device Fine-Tuning & Personalization
**Compiled:** May 2026 | **Relevance:** SAMs που μαθαίνουν από user corrections χωρίς cloud

---

## Συμπέρασμα: Ναι, Είναι Feasible

**On-device fine-tuning είναι mainstream ως το 2025-2026 σε Apple Silicon.** Unified memory architecture, MLX framework, και βελτιωμένα μικρά models κάνουν αυτό practical.

Benchmarks:
- M2 MacBook Pro (16GB): Fine-tune Mistral 7B LoRA σε 5,000 examples → ~90 min, ~7GB RAM
- **Για 0.5B–3B models: 2–15 λεπτά** σε οποιοδήποτε M-series chip ≥8GB RAM

---

## Primary Stack: MLX + mlx-lm

**[Apple MLX](https://github.com/ml-explore/mlx)** = definitive framework. Officially preferred από Apple, WWDC 2025 sessions. Ollama 0.19 (March 2026) πρόσθεσε MLX backend.

**[mlx-lm](https://github.com/ml-explore/mlx-lm/blob/main/mlx_lm/LORA.md)** supports:
- LoRA, QLoRA, DoRA
- DPO (Direct Preference Optimization) — για correction-based learning
- **Adapter resumption**: `--resume-adapter-file` — κρίσιμο για incremental updates

```bash
# Initial fine-tuning
mlx_lm.lora --model <base_model> --train --data ./corrections/ --iters 100

# Incremental update
mlx_lm.lora --model <base_model> --resume-adapter-file ./adapters.safetensors \
             --train --data ./new_corrections/ --iters 50

# Fuse adapter
mlx_lm.fuse --model <base_model> --adapter-path ./adapters.safetensors
```

---

## Memory Requirements

| Model | RAM για LoRA | Training Time (M2 Pro / 100 examples) |
|---|---|---|
| Qwen2.5-0.5B | ~1.5–2 GB | < 2 min |
| SmolLM2-1.7B | ~3–4 GB | ~5 min |
| Qwen2.5-1.5B | ~3 GB | ~4 min |
| Llama-3.2-3B | ~5–6 GB | ~8 min |
| Phi-3.5-mini | ~7–8 GB | ~12 min |

**QLoRA** cuts RAM 40–60% — δεν χρειάζεται για <3B models σε modern Mac.

**LoRA hyperparameters για classification:**
```
rank (r): 8–16
alpha: 16–32 (2× rank)
target_modules: q_proj, v_proj (+ k_proj, o_proj για better results)
learning_rate: 1e-4 to 3e-4
```

---

## Continual Learning — Catastrophic Forgetting

**Key finding (IEEE 2025):** Larger models forget MORE. Exception: **Phi-3.5-mini exhibits minimal forgetting** — explicitly identified για continual learning.

### Στρατηγικές (Ranked για Curator)

**Strategy 1 (Recommended): Full Correction History Replay**
Keep growing `corrections.jsonl`. Re-train adapter on ENTIRE history όταν νέες corrections φτάσουν threshold. Simple, robust.

**Strategy 2: Replay Buffer**
Mix 10–20% old examples σε κάθε νέο training batch. Slightly higher memory.

**Strategy 3: Elastic Weight Consolidation (EWC)**
Penalizes changes to important weights. Computationally cheap, no extra data.

**Strategy 4: Separate Adapters per Domain**
One LoRA adapter per SAM (DocumentSAM, CodeSAM, etc.). Eliminates forgetting. Adapters <50MB.

---

## Few-Shot vs Fine-Tuning Decision

| Corrections | Approach |
|---|---|
| 1–10 | Dynamic few-shot στο prompt |
| 11–49 | Few-shot + RAG (retrieve similar past corrections) |
| **50–100** | **Trigger first LoRA fine-tuning run** |
| 100+ | Incremental fine-tuning κάθε 25 νέες corrections |

**Research finding (arXiv:2406.08660):** Fine-tuned small LLMs **significantly outperform** zero-shot large models σε domain-specific classification. Ειδικά για idiosyncratic personal taxonomies — ακριβώς ο Curator use case.

---

## Key Papers

| Paper | Key Insight |
|---|---|
| **On-Device LLM Personalization** — arXiv:2311.12275 | Self-supervised data selection από sparse user annotations — exact problem |
| **Never Start from Scratch** — arXiv:2504.13938 (MobiSys 2025) | Warm-start από task-similar adapter → dramatically cuts examples needed |
| **ASLS** — arXiv:2409.16973 | Continuous learning από user feedback on-device |
| **One PEFT Per User (OPPU)** — arXiv:2402.04401 | Per-user PEFT module: +1.3% parameters, personalizes foundation model |
| **Fine-tuned SLMs vs zero-shot** — arXiv:2406.08660 | Fine-tuned small > zero-shot large για classification |

---

## Apple Foundation Models (macOS 26+)

Apple's Foundation Models Adapter Training toolkit: LoRA adapters (~160MB) για Apple's own ~3B on-device model. Python toolkit, Swift deployment.

**Limitation:** Μόνο Apple's proprietary model, not Ollama. Requires macOS 26 (Tahoe).

**Hybrid path:** Χρησιμοποίησε Ollama/MLX τώρα → προσθεσε Foundation Models support όταν έρθει macOS 26.

---

## Implementation Path για Curator

### Phase 1: RAG-Based Correction (από 1η correction)
```
User corrects: invoice_jan.pdf → "Finance/Invoices" (was "Documents")
↓
Store: {file_hash, metadata, true_label, timestamp} → SQLite
↓
Next inference: retrieve K most similar past corrections → inject as few-shot
```
Zero training overhead, immediate improvement.

### Phase 2: Background LoRA Fine-Tuning (50+ corrections)
```bash
# Format as JSONL
{"prompt": "Classify: invoice_jan2026.pdf [PDF, 45KB]", "completion": "Finance/Invoices"}

# Fine-tune
mlx_lm.lora --model mlx-community/Qwen2.5-1.5B-Instruct-4bit \
             --train --data ./curator_corrections/ \
             --iters 200 --batch-size 4 --lora-layers 8
```

### Phase 3: Personalized SAM (200+ corrections)
```bash
mlx_lm.fuse  # Create personal model
# Replace base model in Ollama with fused personal model
```

---

## Open Problems

1. **Adapter drift** — incremental adapters drift after many rounds. Periodic full re-train από full history
2. **Evaluation without labels** — use "correction rate" ως implicit quality metric
3. **Cold-start** — πρώτες 50 corrections = worst performance when most needed
4. **Category imbalance** — weighted sampling required

---

## Sources
- [mlx-lm LoRA Documentation](https://github.com/ml-explore/mlx-lm/blob/main/mlx_lm/LORA.md)
- [Never Start from Scratch — arXiv 2504.13938](https://arxiv.org/abs/2504.13938)
- [On-Device Personalization — arXiv 2311.12275](https://arxiv.org/html/2311.12275v4)
- [Fine-tuned SLMs — arXiv 2406.08660](https://arxiv.org/html/2406.08660v1)
- [Apple Foundation Models Adapter Training](https://developer.apple.com/apple-intelligence/foundation-models-adapter/)
- [OPPU — arXiv 2402.04401](https://arxiv.org/pdf/2402.04401)
