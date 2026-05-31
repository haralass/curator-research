# SAM Orchestration — Model Routing & Specialized Models
**Compiled:** May 2026 | **Relevance:** Curator per-file-domain routing architecture

---

## Κεντρική Ιδέα

Αντί για ένα μεγάλο generalist model, **αναλύεις κάθε query και το στέλνεις στο μικρότερο model που μπορεί να το χειριστεί σωστά.** Αυτό είναι το model routing / SAM orchestration pattern.

HiPEAC Vision 2025 (σελ. 35-36) αναφέρει orchestration algorithms για SAMs ως **explicitly open research problem**.

---

## Taxonomy Routing Approaches

| Παράδειγμα | Πώς λειτουργεί | Καλύτερο για |
|---|---|---|
| **Difficulty-aware** | Simple query → small model, hard → large | Cost reduction |
| **Domain/intent-based** | Κατηγορεί το domain, routes σε specialist | Εξειδίκευση |
| **Clustering-based** | Semantic grouping → best model per cluster | Unsupervised |
| **Uncertainty-based** | Routes βάσει confidence του router | Quality-critical |
| **RL-based** | Policy από feedback (PPO) | Adaptive, online |
| **Cascading** | Try small first, escalate on failure | Latency vs quality |

---

## Key Papers

### Surveys
| Paper | Venue | Key Insight |
|---|---|---|
| **Dynamic Model Routing Survey** — arXiv 2603.04445 | 2026 | Most comprehensive — covers all 6 paradigms, open problems |
| **Doing More with Less Survey** — arXiv 2502.00409 | 2025 | Formalizes 4 implementation families; energy/environmental cost under-studied |

### Routing Frameworks
| Paper | Venue | Key Insight |
|---|---|---|
| **MoMA** — arXiv 2509.07571 | 2025 | Πρώτο unified LLM + agent routing framework; MoE-based router; TOPSIS algorithm |
| **HierRouter** — arXiv 2511.09873 | 2025 | PPO-based RL routing; **2.4x quality improvement** over best individual model |
| **UniRoute** — arXiv 2502.08773 | 2025 | Routes σε models δεν είδε κατά training; 30+ unseen LLMs |
| **xRouter** — arXiv 2510.08439 | 2025 | Cost-aware RL orchestration; financial cost ως reward signal |

### Domain-Specific (Πιο Σχετικό)
| Paper | Venue | Key Insight |
|---|---|---|
| **Multimodal Routing** — arXiv 2511.06441 | 2025 | **Πιο κοντινό στον Curator** — routes text/image/audio/video/docs σε specialists. 92.3% accuracy, 70% cost reduction, 18% latency improvement |
| **SOLVE-Med** — arXiv 2511.03542 | 2025 | **Structurally identical** — 10×1B specialist models + router + orchestrator για medical. Outperforms standalone 14B. Proof SAM pattern works. |

### Specialized vs Generalist
| Paper | Key Insight |
|---|---|
| **"Comparing Specialised Small and General LLMs"** — arXiv 2402.12819 | Μόνο **100 labelled samples** αρκούν για specialized να ξεπεράσει generalist |
| **Playing DOOM with 1.3M params** — arXiv 2604.07385 | Small specialized models: 10–30x cost savings σε narrow tasks |

### Important Caution
**"The Myth of Expert Specialization in MoEs"** — arXiv 2604.09780:
> Internal MoE routing reflects hidden-state geometry, NOT semantic domains.
> Δεν μπορείς να βασιστείς σε off-the-shelf MoE για domain specialization — χρειάζεσαι explicit routing.

---

## GitHub Repos & Tools

| Repo | Stars | Τι κάνει |
|---|---|---|
| [lm-sys/RouteLLM](https://github.com/lm-sys/routellm) | High | 4 routers: Matrix Factorization, BERT, Causal LLM. **85% cost reduction** at 95% GPT-4 quality |
| [ulab-uiuc/LLMRouter](https://github.com/ulab-uiuc/LLMRouter) | Active | 16+ routing models; single/multi-round/agentic |
| [withmartian/routerbench](https://github.com/withmartian/routerbench) | ICML 2024 | Benchmark για multi-LLM routing — 405k+ inference outcomes |
| [Not-Diamond/awesome-ai-model-routing](https://github.com/Not-Diamond/awesome-ai-model-routing) | — | Best curated index of all routing papers + tools |
| [NVIDIA-AI-Blueprints/llm-router](https://github.com/NVIDIA-AI-Blueprints/llm-router) | Production | Intent-based (Qwen 1.7B) + auto-routing (CLIP embeddings) |

### Commercial Tools
- **Not Diamond**: Learns meta-model από query-response pairs → trains custom router
- **OpenRouter Auto Router**: Χρησιμοποιεί Not Diamond tech, routes among 19 models
- **Martian Router**: Real-time dynamic routing, cost + quality + latency

---

## Non-Functional Requirements στο Model Selection

| NFR | State of the Art |
|---|---|
| Latency | Well-studied — route to fastest model that meets quality bar |
| Cost (financial) | Very well-studied — core use case RouteLLM/Not Diamond |
| Accuracy/Quality | Well-studied |
| Privacy | **Mostly unaddressed** — PPRoute, PRISM early work |
| Energy/Carbon | **Gap** — surveys explicitly call out as ignored |
| Offline capability | **Gap** — not addressed anywhere |

---

## Open Gaps (2025–2026)

1. **Routing σε unseen models** — UniRoute (2025) first attempt, not production-tested
2. **Routing χωρίς text query** — όλοι οι routers βλέπουν text. Routing από file metadata = unaddressed
3. **Multimodal input routing** — explicitly open problem στα surveys
4. **Standardized benchmarks για domain routing** — RouterBench covers difficulty only; δεν υπάρχει για file domains
5. **Adaptive routing χωρίς retraining** — δεν μπορείς να προσθέσεις νέο specialist χωρίς full retraining
6. **Fully offline personal routing** — PRISM/PPRoute assume cloud exists; fully local = unaddressed
7. **Orchestrator cold-start** — πώς profiling νέων specialist models χωρίς extensive benchmarking

---

## Τι Είναι Novel για τον Curator

Το per-file-domain routing του Curator είναι genuinely novel σε 5 σημεία:

### 1. Το Signal Δεν Είναι Text Query
Όλοι οι routers βλέπουν natural language query. Ο Curator routes βάσει **file extension / MIME type / content structure** — αρχιτεκτονικά διαφορετικό από όλα τα published systems.

### 2. Fully Offline, No Cloud
Κάθε commercial router (Not Diamond, OpenRouter, Martian) assume API calls. Per-file local routing = **no cloud at all** — δεν υπάρχει στη βιβλιογραφία.

### 3. Classification Vocabulary ≠ Response Quality
Τρέχων routing: "ποιο model θα δώσει την καλύτερη απάντηση;"
Curator routing: "ποιο model έχει το **σωστό classification vocabulary** για αυτό το domain;"
Διαφορετικό success metric.

### 4. Routing Decision ΠΡΙΝ Inference
Cascading systems invoke first, then escalate. File domain determined from extension/MIME **before any model runs** — near-zero-cost deterministic pre-routing. Αντιστρέφει το standard cascading model.

### 5. Non-Overlapping Domains
Query routing: τεράστιο overlap problem. File domains: **.mp4 = media, .py = code, .docx = document** — mostly deterministic. Structurally easier, αλλά πουθενά δεν formalized.

### SOLVE-Med = Πιο Κοντινό Analogue
10 × 1B medical specialists + router = structurally identical architecture. Κανείς δεν το εφάρμοσε σε file classification.

---

## Προτεινόμενη Αρχιτεκτονική Curator

```
File Input
    │
    ├── Extension/MIME Router (deterministic, ~0ms)
    │   ├── .pdf / .docx / .txt → DocumentSAM
    │   ├── .mp4 / .jpg / .mp3 → MediaSAM
    │   ├── .py / .js / .rs    → CodeSAM
    │   ├── .csv / .xlsx        → DataSAM
    │   └── .zip / .dmg        → ArchiveSAM
    │
    ├── [Ambiguous files → Orchestrator LLM decides]
    │
    └── Each SAM: small specialized model (0.5–1B) fine-tuned on domain
```

---

## Sources
- [Dynamic Model Routing Survey — arXiv 2603.04445](https://arxiv.org/abs/2603.04445)
- [MoMA — arXiv 2509.07571](https://arxiv.org/abs/2509.07571)
- [HierRouter — arXiv 2511.09873](https://arxiv.org/abs/2511.09873)
- [Multimodal Routing — arXiv 2511.06441](https://arxiv.org/abs/2511.06441)
- [SOLVE-Med — arXiv 2511.03542](https://arxiv.org/abs/2511.03542)
- [UniRoute — arXiv 2502.08773](https://arxiv.org/abs/2502.08773)
- [Comparing Specialised vs General LLMs — arXiv 2402.12819](https://arxiv.org/abs/2402.12819)
- [MoE Myth — arXiv 2604.09780](https://arxiv.org/abs/2604.09780)
- [RouteLLM](https://github.com/lm-sys/routellm)
- [awesome-ai-model-routing](https://github.com/Not-Diamond/awesome-ai-model-routing)
- [RouterBench](https://github.com/withmartian/routerbench)
- [NVIDIA LLM Router Blueprint](https://github.com/NVIDIA-AI-Blueprints/llm-router)
- [Privacy-Preserving LLM Routing — arXiv 2604.15728](https://arxiv.org/abs/2604.15728)
- [PRISM — arXiv 2511.22788](https://arxiv.org/abs/2511.22788)
- [OptiRoute — arXiv 2502.16696](https://arxiv.org/abs/2502.16696)
