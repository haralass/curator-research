# Energy Measurement για Edge AI Inference — macOS Apple Silicon
**Compiled:** May 2026 | **Relevance:** Curator per-classification energy cost measurement

---

## Το Πρόβλημα

Το Apple Silicon δεν έχει Intel-style RAPL registers. Αντ' αυτού έχει δύο proprietary interfaces:

| Interface | Sudo; | Granularity | Notes |
|---|---|---|---|
| **IOReport** (private IOKit) | Όχι | < 1 ms | Model-based estimate; 1 mJ resolution per channel |
| **powermetrics** (CLI) | Ναι | Fixed intervals (e.g. 500ms) | Άβολο programmatically |

**Κρίσιμη παρατήρηση**: IOReport values είναι model-based estimates, όχι direct sensor readings. Useful για relative comparisons εντός του ίδιου device — όχι absolute cross-device.

---

## Primary Tool: zeus-apple-silicon

**Repo**: [https://github.com/ml-energy/zeus-apple-silicon](https://github.com/ml-energy/zeus-apple-silicon)
**Blog**: [ML.ENERGY — Profiling LLMs on Macs](https://ml.energy/blog/energy/measurement/profiling-llm-energy-consumption-on-macs/)

Αυτό είναι το **definitive tool** για per-inference energy measurement. C++17/Python library που wraps IOReport:

```python
from zeus_apple_silicon import AppleEnergyMonitor

monitor = AppleEnergyMonitor()
monitor.begin()
# run Ollama classification here
result = monitor.end()
# result: joules per subsystem — cpu, gpu, ane, dram
```

Δίνει energy delta για **CPU clusters, GPU, ANE, DRAM** ξεχωριστά. Sub-millisecond granularity.

### Supporting Tools

| Tool | Sudo; | Output | Χρήση |
|---|---|---|---|
| [macmon](https://github.com/vladkens/macmon) | Όχι | JSON (watts) | Sidecar JSON power stream, Prometheus integration |
| [fluidtop](https://github.com/FluidInference/fluidtop) | Ναι | TUI | AI-focused, ANE tracking |
| [asitop](https://github.com/tlkh/asitop) | Ναι | TUI | Visual monitoring κατά inference |
| [socpowerbud](https://github.com/dehydratedpotato/socpowerbud) | Όχι | Frequency/voltage | Sudoless alternative |

---

## Ollama-Specific Measurement

Ollama's API επιστρέφει: `eval_count` (output tokens), `eval_duration` (ns). **Δεν έχει energy data.**

**Practical strategy:**

```
Curator File Organizer
    │
    ├── zeus-apple-silicon (Python)   ← energy measurement
    │       └── IOReport → CPU/GPU/ANE/DRAM joules
    │
    ├── Ollama HTTP /api/generate
    │       └── eval_count, eval_duration
    │
    └── Per-classification log:
            {model, quant, J_total, J_gpu, J_ane, ms, J/decision, J/token, result}
```

---

## Key Papers

| Paper | arXiv | Key Finding |
|---|---|---|
| **Small is Sufficient** | 2510.01889 | Specialized models **65.8% more energy-efficient**; 27.8% global AI energy savings από model selection |
| **Intelligence per Watt** (Stanford) | 2511.07885 | IPW metric; **77.1% energy savings** vs cloud για tasks <20B params; Apple M4 Max competitive |
| **Scaling Laws for Energy Efficiency** | 2512.16531 | Energy scales predictably με parameter count — can predict cost before running |
| **Where Do the Joules Go?** | 2601.22076 | Task type alone: **25× energy variation**; GPU utilization: 3–5× variation |
| **TokenPowerBench** | 2512.03024 | MoE models: **2–3× lower token energy** vs dense; Q8→Q4: ~30% cut |
| **Efficient MoE on Apple NPUs** | 2604.18788 | NPUMoE: **1.81–7.37× energy reduction** on M-series με <1.1% accuracy loss |
| **Characterizing SLMs on Edges** | 2511.11624 | Llama 3.2 = best balance accuracy + efficiency; TinyLlama για ultra-low-power |
| **Towards Green AI** | 2602.05712 | Energy footprint LLM inference σε dev tools |

---

## Metrics

| Metric | Τύπος | Χρήση |
|---|---|---|
| **J/decision** | Total joules per classification | Primary metric για Curator |
| **J/token** | Joules / output token count | Universal standard |
| **ms/decision** | Latency | Paired με J/decision |
| **EDP** | J × ms (Energy-Delay Product) | Combined efficiency metric |
| **IPW** | Accuracy / average watts | "Intelligence per Watt" — Stanford standard |
| **Accuracy/J** | F1 / J_per_decision | Curator-specific version of IPW |
| **Tokens/watt** | tok/s / W | Throughput-normalized |

**Σύσταση για Curator:** J/decision + ms/decision + Accuracy/J

---

## Apple Silicon Subsystem

- **ANE (Apple Neural Engine)**: ~2W peak vs ~20W GPU — 80× more efficient per FLOP αλλά **Ollama/llama.cpp δεν χρησιμοποιεί ANE** για autoregressive decoding
- **GPU**: Primary compute για LLM decode. Memory bandwidth bottleneck (400–546 GB/s)
- **Unified Memory**: No PCIe overhead — major efficiency advantage

### Throughput Comparison (M2 Ultra reference)
| Framework | Tokens/s |
|---|---|
| MLX | ~230 |
| MLC-LLM | ~190 |
| llama.cpp | ~150 |
| **Ollama** | **20–40** (HTTP overhead) |

Ollama's lower throughput = longer inference windows = more joules per request.

---

## Open Gaps

1. **No native Ollama energy API** — πρέπει να instrumented at OS level
2. **IOReport accuracy unknown** — Apple hasn't published calibration data
3. **ANE measurements near-zero** για Ollama/llama.cpp (ANE unused)
4. **No published Ollama-specific energy benchmarks** — gap στη βιβλιογραφία
5. **Thermal throttling** — μετά 5–15 min sustained inference, SoC down-clocks non-linearly

---

## Experimental Design για Curator (Specialized vs Generalist)

### Setup
```
Per file, run N times (warm-started model):
├── Generalist: Llama 3.2 3B ή Qwen2.5 7B
└── Specialized: fine-tuned/distilled 0.5–1B per domain

Measure: J/decision, ms/decision, accuracy vs ground truth
Compute: Accuracy/J_decision (Intelligence-per-Joule)
```

### Warm-up Protocol
1. Load model once (exclude `load_duration`)
2. Discard first 3–5 inference calls
3. Run 20–50 classifications per model per file type
4. Control thermal state via macmon

### Key Efficiency Levers για να δοκιμάσεις
- **Quantization**: Q4_K_M vs FP16 → ~30–50% energy reduction
- **Model size**: Energy super-linear με parameters (1B→70B = 7.3× energy, not 70×)
- **Specialization**: 65.8% more efficient βάσει "Small is Sufficient"

---

## Academic Validation

Το paper "Small is Sufficient" (arXiv 2510.01889) παρέχει direct academic backing:
> Switching to the most energy-efficient model that meets task utility saves 27.8% of global AI inference energy. Classification tasks with mature label sets show **up to 98% energy savings**.

Αυτό είναι το theoretical grounding για το research question του Curator.

---

## Sources
- [zeus-apple-silicon](https://github.com/ml-energy/zeus-apple-silicon)
- [ML.ENERGY Blog](https://ml.energy/blog/energy/measurement/profiling-llm-energy-consumption-on-macs/)
- [Small is Sufficient — arXiv 2510.01889](https://arxiv.org/abs/2510.01889)
- [Intelligence per Watt — arXiv 2511.07885](https://arxiv.org/abs/2511.07885)
- [Scaling Laws for Energy — arXiv 2512.16531](https://arxiv.org/abs/2512.16531)
- [Where Do the Joules Go — arXiv 2601.22076](https://arxiv.org/abs/2601.22076)
- [TokenPowerBench — arXiv 2512.03024](https://arxiv.org/abs/2512.03024)
- [NPUMoE — arXiv 2604.18788](https://arxiv.org/abs/2604.18788)
- [macmon](https://github.com/vladkens/macmon)
- [fluidtop](https://github.com/FluidInference/fluidtop)
- [asitop](https://github.com/tlkh/asitop)
- [Stanford IPW](https://hazyresearch.stanford.edu/blog/2025-11-11-ipw)
