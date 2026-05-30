## ΜΕΡΟΣ 31: FILEGRAM + EMCSTREAM — ΝΕΕΣ ΑΚΑΔΗΜΑΪΚΕΣ ΣΥΝΔΕΣΕΙΣ
*(Έρευνα 2026-05-23)*

---

### ΘΕΜΑ 1: FileGram (arXiv:2604.04901, April 2026)

**Πλήρης τίτλος:** "FileGram: Grounding Agent Personalization in File-System Behavioral Traces"  
**Συγγραφείς:** Shuai Liu, Shulin Tian, Kairui Hu, Yuhao Dong, Zhe Yang, Bo Li, Jingkang Yang, Chen Change Loy, Ziwei Liu  
**Ίδρυμα:** S-Lab, Nanyang Technological University (NTU) + Synvo AI  
**GitHub:** https://github.com/Synvo-ai/FileGram

---

#### 1.1 Τι ακριβώς κάνει το FileGram

Το FileGram δεν είναι file organizer — είναι framework για **AI agent personalization** μέσω file-system behavioral traces. Ο κεντρικός στόχος: ένας AI agent που λειτουργεί στον υπολογιστή του χρήστη να μαθαίνει *πώς* ο χρήστης εργάζεται, παρακολουθώντας atomic filesystem events αντί να βασίζεται αποκλειστικά σε dialogue summaries.

**Τρία core components:**

**FileGramEngine** — Scalable data engine που παράγει συνθετικά workflows: γεννά multimodal action sequences σε μεγάλη κλίμακα, simulate realistic user behavior για training. Βασίζεται σε "persona-driven" γεννήτρια που δημιουργεί 20 διαφορετικά user profiles × 32 tasks = 640 συνθετικά trajectories, με 20,028 atomic actions και ~10K multimodal files.

**FileGramBench** — Diagnostic evaluation suite. Τεστάρει memory systems σε 4 διαστάσεις:
1. *Profile reconstruction* — μπορεί ο agent να ανακατασκευάσει user profile από traces;
2. *Trace disentanglement* — ξεχωρίζει behavioral traces διαφορετικών χρηστών;
3. *Persona drift detection* — ανιχνεύει αλλαγή προτύπων στον χρόνο;
4. *Multimodal grounding* — κατανοεί multimodal file content (εικόνες, κείμενο, κώδικας);

Συνολικά: 4,600 QA pairs + real-world human screen recordings.

**FileGramOS** — Η memory architecture του συστήματος. "Bottom-up" προσέγγιση: εξάγει user profiles απευθείας από **atomic file operations και content deltas** αντί για conversation summaries. Pipeline σε 3 στάδια:

1. **Per-Trajectory Encoding:** Κάθε action sequence επεξεργάζεται σε:
   - Procedural extraction → 17-D fingerprint (action frequencies, workflow patterns)
   - Semantic parsing → VLM-generated descriptors για file content
   - Episode boundary detection → χωρισμός σε logical task segments

2. **Cross-Engram Consolidation:** Aggregation fingerprints σε 3 memory channels:
   - *Procedural*: task workflows και action patterns
   - *Semantic*: meaning/intent πίσω από operations
   - *Episodic*: specific instances με contextual details

3. **Query-Adaptive Retrieval:** Assembly context από "pre-computed clues" και των τριών channels για personalized responses.

---

#### 1.2 Metadata που χρησιμοποιεί

**ΔΕΝ χρησιμοποιεί macOS Spotlight metadata** (κανένα `kMDItemLastUsedDate`, `kMDItemFSCreationDate`, `kMDItemWhereFroms` κ.λπ.).

Βασίζεται **αποκλειστικά σε 12 atomic action types** filesystem events:

```
file_read, file_browse, file_search, file_write, file_edit,
dir_create, file_copy, file_move, file_delete, file_rename,
cross_file_ref, context_switch
```

Επίσης καταγράφει **content deltas**: full snapshots για νεοδημιουργημένα files και patch diffs για edits.

Αξιοσημείωτο: καταγράφει "directory_style", "naming conventions", και organizational depth (1–3+ levels) αλλά ως **παρατηρήσεις** — δεν κάνει active clustering ή categorization.

---

#### 1.3 Forgetting model

**Κανένα forgetting/temporal decay model.** Το σύστημα aggregates statistics across trajectories χωρίς καμία temporal weighting ή fading function. Τα παλιά events έχουν ίση βαρύτητα με τα νέα. Αυτό είναι εμφανές design gap.

---

#### 1.4 Libraries/Architecture

- **LLMs:** Gemini 2.5-Flash, GPT-4.1, Claude Haiku 4.5, Claude Sonnet 4
- **Embeddings:** Cohere embed-english-v3.0 (1024-D)
- **Vision-Language Models:** Unspecified VLM για multimodal file snapshots
- Ανοιχτός κώδικας — Python, χωρίς explicit package list στο paper

---

#### 1.5 Threat model / Use case

Coworking AI agents στο τοπικό filesystem. Ο χρήστης δουλεύει, ο agent παρακολουθεί filesystem events, μαθαίνει behavioral profile, και προσαρμόζει τις απαντήσεις/προτάσεις του. Το paper αναγνωρίζει privacy risks: "file-system traces reveal sensitive patterns—working hours, task priorities, organizational habits."

**Use case:** personalization of AI assistant behavior — όχι organization/categorization.

---

#### 1.6 FileGram vs Curator — Συγκριτική Ανάλυση

| Διάσταση | FileGram | Curator |
|----------|----------|---------|
| **Στόχος** | AI agent personalization | File organization + discovery |
| **Output** | User behavioral profile → personalized agent responses | Κατηγοριοποίηση + cluster labels + HERCULES naming |
| **Embeddings** | Cohere 1024-D text (via API, cloud) | BGE-M3 1024-D (local, offline) |
| **Clustering** | Καμία — μόνο memory channels | UMAP→HDBSCAN + DenStream streaming |
| **macOS metadata** | Καθόλου (μόνο filesystem events) | kMDItemLastUsedDate, kMDItemWhereFroms, kMDItemDownloadedDate κ.λπ. |
| **Forgetting/decay** | Κανένα | DenStream fading factor (λ) |
| **Streaming** | Batch (synthetic trajectories) | Online streaming, real-time |
| **Naming** | Δεν ονομάζει categories | HERCULES LLM naming |
| **Privacy** | Cloud API calls (Gemini, GPT, Cohere) | 100% local, no cloud |
| **Drift handling** | Persona drift *detection* (benchmark task) | Drift-Triggered Refit (automated re-cluster) |

**Τι κάνει το FileGram που ο Curator ΔΕΝ κάνει:**
- Behavioral trace → user profile mapping (3-channel memory)
- Multimodal content understanding με VLM
- Synthetic trajectory generation (FileGramEngine)
- Διαχωρισμός behavioral traces multiple users (trace disentanglement)

**Τι κάνει ο Curator που το FileGram ΔΕΝ κάνει:**
- Actual file organization/categorization με semantic clusters
- 100% offline/local processing (no cloud APIs)
- DenStream fading factor για temporal relevance
- Drift-Triggered Refit (αυτόματο re-clustering)
- HERCULES naming (LLM naming pipeline local)
- macOS Spotlight metadata exploitation
- HDBSCAN density-based clustering (όχι manual categories)

---

#### 1.7 Future Work Angle: Curator ταΐζει FileGram-style memory

Ο Curator παράγει **εξαιρετικά πλούσια behavioral signals** που θα ήταν ideal input για ένα FileGram-style memory system:
- Ποια clusters χρησιμοποιεί ο χρήστης συχνά (access frequency ανά cluster)
- Ποια files τοποθετεί σε νέα clusters (intent signals)
- Πώς αλλάζει η directory structure στον χρόνο (drift signals)
- kMDItemLastUsedDate patterns ανά project type

Αυτό ανοίγει **"Agent API" future work angle** — βλ. Θέμα 4.

**Relevance για Curator paper:** HIGH — Ισχυρό related work που τεκμηριώνει ότι file behavioral traces έχουν ακαδημαϊκό ενδιαφέρον, ενώ ο Curator καλύπτει το organization gap που το FileGram αφήνει ανοιχτό.

---

### ΘΕΜΑ 2: EmCStream (Zubaroğlu & Atalay, 2023)

**Πλήρης τίτλος:** "Online embedding and clustering of evolving data streams"  
**Journal:** Statistical Analysis and Data Mining: The ASA Data Science Journal (Wiley, 2023)  
**DOI:** https://doi.org/10.1002/sam.11590  
**Ίδρυμα:** Middle East Technical University (METU), Τουρκία  
**Κώδικας:** https://gitlab.com/alaettinzubaroglu/emcstream

---

#### 2.1 Τι κάνει ακριβώς

Το EmCStream είναι **online algorithm για clustering evolving (nonstationary) data streams** με ενσωματωμένο concept drift detection και adaptation. Βασικό insight: το UMAP μπορεί να εφαρμοστεί σε stationary streams αλλά δεν μπορεί να προσαρμοστεί σε concept drift — το EmCStream το επεκτείνει.

**Κεντρική ιδέα:** Continuously embed high-dimensional streaming data σε 2D χώρο + cluster σε real-time + ανίχνευσε/προσαρμόσου σε concept drift αυτόματα.

---

#### 2.2 Clustering Algorithm

**K-means** στο embedded 2D space. Το paper σημειώνει ότι μπορεί να αντικατασταθεί από "any distance or partitioning-based clustering algorithm" — k-means επιλέχθηκε ως "most popular and easy-to-implement." Δεν χρησιμοποιεί HDBSCAN ή density-based clustering.

---

#### 2.3 Concept Drift Detection

Το EmCStream εφαρμόζει **continuous drift monitoring** στο embedded space. Από τη βιβλιογραφία γύρω από το paper (METU thesis + Wiley publication), η drift detection βασίζεται σε **στατιστικό τεστ/μετρική διαφοράς κατανομής** στον UMAP embedded space: όταν η κατανομή των embedded points αλλάξει σημαντικά, triggering refit του UMAP. Το paper δεν αναφέρει explicitly ADWIN ή Page-Hinkley — χρησιμοποιεί custom detection βασισμένο στη cluster structure.

---

#### 2.4 Evaluation Datasets και Metrics

**Datasets:** Συνθετικά AND real evolving data streams (από GitLab repo: standard benchmark datasets για stream clustering).

**Metrics:**
- **Adjusted Rand Index (ARI)** — primary metric
- **Purity** — secondary metric

**Baselines:**
- **DenStream** (Cao et al., 2006) — density-based micro-cluster streaming
- **CluStream** (Aggarwal et al., 2003) — micro-cluster + macro-cluster offline

**Αποτέλεσμα:** EmCStream outperforms DenStream AND CluStream σε ARI και purity σε όλα τα synthetic και real datasets.

---

#### 2.5 EmCStream vs Curator's Drift-Triggered Refit

| Διάσταση | EmCStream | Curator |
|----------|-----------|---------|
| **Domain** | Αφηρημένα text/numeric streams (benchmark datasets) | Personal files με macOS metadata |
| **Embedding** | UMAP online (2D output) | BGE-M3 (1024-D) → UMAP (2D για visualization) |
| **Clustering** | K-means (2D UMAP space) | HDBSCAN (density-based, no k assumption) |
| **Streaming** | DenStream-free, direct UMAP online | DenStream fading clusters + periodic HDBSCAN refit |
| **Drift detection** | Custom distribution shift στο embedded space | Fading factor (λ) + configurable Drift-Triggered Refit |
| **Forgetting** | Sliding window ή παρόμοιο (implicit) | Explicit DenStream fading: weight = e^(-λ·Δt) |
| **Metadata** | Καθόλου — pure feature vectors | kMDItemLastUsedDate, kMDItemWhereFroms κ.λπ. |
| **k parameter** | Απαιτεί k (number of clusters) | HDBSCAN: automatic k, handles noise/outliers |
| **Naming** | Καμία cluster naming | HERCULES: αυτόματα LLM-generated cluster names |

**Καλύπτει ή αφήνει gap για Curator;**

Το EmCStream **καλύπτει** το online UMAP+clustering για general streams αλλά **αφήνει σημαντικό gap** για το Curator:

1. **K-means vs HDBSCAN:** Ο Curator δεν απαιτεί k — αυτό είναι κρίσιμο για personal files όπου ο αριθμός κατηγοριών είναι άγνωστος και αλλάζει.
2. **Domain mismatch:** EmCStream εκτιμά generic numeric/text streams — ο Curator εκτιμά personal filesystem με semantic BGE-M3 embeddings.
3. **No macOS metadata:** EmCStream δεν αξιοποιεί καμία metadata πέραν του content.
4. **No naming:** EmCStream δεν δίνει human-readable labels στα clusters.
5. **No forgetting model:** Το implicit window δεν είναι equivalent με το explicit DenStream fading factor.

---

#### 2.6 Πώς να Cite το EmCStream

Στο related work section:

> "EmCStream [Zubaroğlu & Atalay, 2023] combines online UMAP embedding with k-means clustering to handle concept drift in evolving data streams, outperforming DenStream and CluStream on standard benchmarks. However, EmCStream requires a predefined k, targets generic numeric streams, and provides no domain-specific metadata exploitation or cluster naming — limitations that Curator addresses through HDBSCAN (no k assumption), macOS Spotlight metadata integration, DenStream fading weights, and HERCULES automatic naming."

**Baselines που μπορούμε να χρησιμοποιήσουμε:**
- **DenStream** — ήδη στον Curator ως streaming backbone ✓
- **CluStream** — δυνητικό baseline στα experiments αν θέλουμε να δείξουμε ότι ο Curator ξεπερνά traditional stream clustering
- Τα ίδια metrics: ARI και Purity — αν κάνουμε quantitative evaluation

---

### ΘΕΜΑ 3: Συνέπειες για το Publication Plan

#### 3.1 Related Work για ECIR 2027 (Information Retrieval angle)

**Πριν FileGram/EmCStream:** Οι συνδέσεις ήταν κυρίως semantic embedding (BGE-M3), HDBSCAN, DenStream.

**Μετά:**

Προσθέτουμε δύο νέες subsections στο related work:

**"File-System Behavioral Modeling"** — FileGram δείχνει ότι file operations είναι πλούσιο σήμα για user modeling. Ο Curator διαφοροποιείται: δεν κάνει behavioral profiling αλλά semantic organization + macOS metadata exploitation.

**"Online Embedding for Evolving Streams"** — EmCStream justifies τη χρήση UMAP+clustering σε streaming context. Ο Curator προχωρά πέρα: HDBSCAN (no-k), DenStream fading, domain-specific metadata.

**Gap που ο ECIR paper καλύπτει (ενισχυμένος):**
> "Υπάρχουν μελέτες για (α) file behavioral traces για agent memory [FileGram, 2026] και (β) online embedding+clustering για evolving streams [EmCStream, 2023], αλλά κανένα σύστημα δεν συνδυάζει semantic embedding, density-based streaming clustering, macOS metadata exploitation, και automatic naming σε ένα ολοκληρωμένο personal file organization pipeline."

---

#### 3.2 Related Work για CHI 2027 (HCI angle)

Το FileGram προσθέτει ισχυρό HCI justification: αν ακαδημαϊκή εργασία (NTU/Synvo) χτίζει ολόκληρο benchmark γύρω από "how users work with files," τότε **η UX του personal file organization είναι ανοιχτό ερευνητικό πρόβλημα**.

**CHI framing:** "FileGram δείχνει ότι users έχουν distinct behavioral profiles — ο Curator εκθέτει αυτή τη διαφορά οπτικά μέσω UMAP scatter plot, δίνοντας στον χρήστη control (confirm/reject clusters) που το FileGram δεν έχει."

Ο Curator προσθέτει το **human-in-the-loop** dimension που λείπει από FileGram (fully automated) και EmCStream (no UI).

---

#### 3.3 Gap που μένει ανοιχτό μετά FileGram + EmCStream

1. **Temporal-aware personal file organization:** Κανένα paper δεν χρησιμοποιεί macOS `kMDItemLastUsedDate` για time-weighted embeddings.
2. **No-k streaming clustering με naming:** EmCStream έχει k-means, FileGram δεν κάνει clustering — το HDBSCAN + HERCULES combo είναι novel.
3. **Local/offline end-to-end pipeline:** FileGram χρησιμοποιεί cloud APIs. Ο Curator είναι 100% local — privacy-preserving angle.
4. **Provenance-aware clustering:** kMDItemWhereFroms (email, web download, AirDrop) → cluster labeling δεν έχει γίνει.
5. **User confirmation loop για drift:** Κανένα σύστημα δεν ζητά από τον χρήστη να confirm τα new clusters μετά drift.

---

#### 3.4 Ενημερωμένος Πίνακας Related Work

| System | Method | Streaming | Naming | macOS Meta | Privacy | Gap vs Curator |
|--------|--------|-----------|--------|------------|---------|----------------|
| **TagFS** | Tag-based manual | No | Manual | No | Local | No automation |
| **Spotlight** | Inverted index search | No | No | Yes | Local | No clustering |
| **CluStream** [2003] | Micro-cluster + k-means | Yes | No | No | N/A | No naming, needs k |
| **DenStream** [2006] | Density micro-clusters | Yes | No | No | N/A | No naming, no embedding |
| **HDBSCAN** [2013] | Density hierarchy | No | No | No | N/A | No streaming |
| **BGE-M3** [2024] | Multi-lingual embedding | No | No | No | Local | No clustering/naming |
| **EmCStream** [2023] | UMAP + k-means online | Yes | No | No | N/A | Needs k, no macOS meta, no naming |
| **FileGram** [2026] | Filesystem traces → agent memory | Batch | No | No | Cloud | No organization, no local |
| **Curator** [ours] | BGE-M3 + UMAP + HDBSCAN + DenStream + HERCULES | Yes | Yes (LLM) | Yes (full) | 100% local | — |

---

### ΘΕΜΑ 4: "Agent Mode" για Curator — FileGram-Inspired Feature

#### 4.1 Η Ιδέα

Το FileGram δείχνει ότι file behavioral traces μπορούν να τροφοδοτήσουν AI agent memory. Ο Curator ήδη παράγει τέτοια traces — αλλά κρατά τα αποτελέσματα locally για file organization. Τι γίνεται αν εξάγαμε αυτή την πληροφορία σε format κατανοητό από LLM agents;

#### 4.2 "Curator Agent API" — Αξιολόγηση

**Τι θα εξήγαμε:**

```json
{
  "user_file_biography": {
    "generated_at": "2026-05-23T14:30:00",
    "clusters": [
      {
        "cluster_id": 7,
        "hercules_name": "PhD Research — Computer Vision",
        "size": 143,
        "last_accessed": "2026-05-22",
        "access_frequency_7d": 34,
        "dominant_file_types": ["pdf", "py", "ipynb"],
        "provenance": ["email_attachment", "web_download"],
        "semantic_summary": "Deep learning papers and PyTorch experiments",
        "drift_detected": false
      }
    ],
    "recent_activity": {
      "most_accessed_clusters": [7, 3, 12],
      "new_files_last_7d": 23,
      "drift_events_last_30d": 2
    },
    "user_behavioral_fingerprint": {
      "peak_activity_hour": 22,
      "dominant_domains": ["research", "personal_finance", "photography"],
      "organization_depth": 3
    }
  }
}
```

**Format options:**
- **JSON memory object** (παραπάνω) — ideal για direct injection σε Claude/GPT system prompt
- **RAG index** — embed cluster summaries → retrieval για context-aware responses
- **Structured summary** — plain text "biography" για LLM consumption

#### 4.3 Αξίζει ως Feature ή Future Work;

**Verdict: Future Work** — αλλά αξίζει ως explicitly mentioned future work στο paper γιατί:

1. Το FileGram (NTU/Synvo, 2026) legitimizes αυτό το research direction
2. Ο Curator έχει ΗΔΗ τα δεδομένα — η εξαγωγή είναι incremental
3. Privacy-respecting: το JSON δεν περιέχει filenames/content, μόνο cluster metadata
4. Differentiation: FileGram needs behavioral trace recording (invasive) — Curator εξάγει profiles από existing organization (non-invasive)

**Γιατί όχι ως feature τώρα:**
- Χρειάζεται LLM agent integration testing (πέρα του scope)
- Privacy implications χρειάζονται careful design
- Out of scope για ECIR/CHI submission

#### 4.4 Εκτίμηση κώδικα

Για basic JSON export:

```python
# curator/agent_api.py (~80 γραμμές)
import json
from pathlib import Path
from datetime import datetime
from .cluster_store import ClusterStore
from .metadata_cache import MetadataCache

class CuratorAgentAPI:
    """Export Curator state as LLM-agent-consumable memory object."""
    
    def __init__(self, store: ClusterStore, meta: MetadataCache):
        self.store = store
        self.meta = meta
    
    def export_file_biography(self, 
                               top_clusters: int = 20,
                               include_filenames: bool = False) -> dict:
        """
        Export cluster data as FileGram-style memory object.
        Never includes file content — only metadata and cluster statistics.
        """
        clusters = self.store.get_all_clusters()
        
        biography = {
            "generated_at": datetime.utcnow().isoformat(),
            "schema_version": "1.0",
            "clusters": []
        }
        
        for cluster in sorted(clusters, 
                               key=lambda c: c.access_frequency_7d, 
                               reverse=True)[:top_clusters]:
            entry = {
                "cluster_id": cluster.id,
                "hercules_name": cluster.label,
                "size": cluster.size,
                "last_accessed": cluster.last_accessed.isoformat(),
                "access_frequency_7d": cluster.access_frequency_7d,
                "dominant_file_types": cluster.top_extensions[:5],
                "provenance": cluster.provenance_summary,
                "semantic_summary": cluster.hercules_description,
                "drift_detected": cluster.drift_flag
            }
            if include_filenames:
                entry["sample_files"] = cluster.sample_filenames[:3]
            biography["clusters"].append(entry)
        
        biography["behavioral_fingerprint"] = self._compute_fingerprint()
        return biography
    
    def export_as_system_prompt_context(self) -> str:
        """Ready-to-inject text for LLM system prompts."""
        bio = self.export_file_biography(top_clusters=10)
        lines = ["## User's File Organization Context (from Curator)"]
        for c in bio["clusters"]:
            lines.append(
                f"- **{c['hercules_name']}** ({c['size']} files, "
                f"accessed {c['access_frequency_7d']}x/week): "
                f"{c['semantic_summary']}"
            )
        return "\n".join(lines)
    
    def _compute_fingerprint(self) -> dict:
        return {
            "dominant_domains": self.store.top_domains(n=5),
            "organization_depth": self.meta.avg_directory_depth(),
            "drift_events_last_30d": self.store.drift_count(days=30)
        }
```

**Εκτίμηση:** ~80 γραμμές για basic export, ~200 γραμμές για πλήρη API με RAG index generation.

---

### Σύνοψη ΜΕΡΟΣ 31

| Topic | Key Finding | Relevance |
|-------|------------|-----------|
| FileGram (2026) | File behavioral traces → AI agent memory, 3-channel (procedural/semantic/episodic), no macOS metadata, no clustering, cloud APIs | HIGH — related work που ο Curator συμπληρώνει |
| EmCStream (2023) | UMAP + k-means online, drift detection, beats DenStream/CluStream, αλλά k-means (needs k), no naming, generic streams | HIGH — direct comparison baseline |
| ECIR 2027 gap | Κανένα paper δεν συνδυάζει HDBSCAN+DenStream+macOS metadata+HERCULES naming | Ενισχυμένο novelty claim |
| CHI 2027 gap | FileGram = fully automated, Curator = human-in-the-loop confirmation | Νέο HCI differentiation angle |
| Agent API | JSON "file biography" export, ~80 γραμμές, ideal future work | MEDIUM — future work section |

**Επόμενη ενέργεια:** Προσθήκη FileGram + EmCStream στο related work section του ECIR 2027 draft με τις formulations παραπάνω.