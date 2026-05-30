## ΜΕΡΟΣ 29: NOVEL DIRECTIONS — KNOWLEDGE GRAPH, FORGETTING CURVE, SEMANTIC COMPRESSION, LIFE DETECTION, PUBLICATION
*(Έρευνα 2026-05-22)*

---

### Topic 1: Personal Knowledge Graph — Αρχεία ως Nodes, Typed Edges, Louvain Community Detection

#### Σύνοψη ιδέας
Αντί να βλέπουμε τα αρχεία ως μεμονωμένα αντικείμενα σε clusters, τα οργανώνουμε σε **γράφο σχέσεων**: κάθε αρχείο είναι node, και οι σχέσεις μεταξύ τους είναι typed edges. Στη συνέχεια εφαρμόζουμε Louvain community detection για να βρούμε φυσικές κοινότητες (π.χ. "Thesis Project Cluster", "Job Applications Cluster"). Είναι εξελιγμένο clustering με semantic memory.

#### Papers που βρέθηκαν

**Κεντρικό paper:**
- **"An ecosystem for personal knowledge graphs: A survey and research roadmap"** — *ScienceDirect / Journal of Web Semantics*, 2024. Μοναδικό comprehensive survey για Personal KGs. Ορίζει Personal KG ως "γράφο που αναπαριστά αντικείμενα και συμβάντα από τη ζωή ενός ατόμου". Δεν εστιάζει σε desktop files αλλά σε emails/social media/contacts. **Κενό: αρχεία ως nodes δεν καλύπτεται.**
  
- **"PersonalAI: A Systematic Comparison of Knowledge Graph Storage and Retrieval Approaches for Personalized LLM agents"** — arXiv:2506.17001, 2026. Hybrid graph design με standard edges + hyper-edges για semantic & temporal representations. Εστιάζει σε LLM personalization, όχι file organization.

- **"Towards Self-organizing Personal Knowledge Assistants in Evolving Corporate Memories"** — arXiv:2308.01732. Semantic Web / Desktop approaches για PIM με KG construction.

- **"The EpisTwin: A Knowledge Graph-Grounded Neuro-Symbolic Architecture for Personal AI"** — arXiv:2603.06290, 2026. Personal KGs με strict data sovereignty — εξαιρετικά relevant γιατί επισημαίνει ότι personal KGs χρειάζονται continuous evolution και subjective context.

**GitHub repo που βρέθηκε:**
- **`obra/knowledge-graph`** (https://github.com/obra/knowledge-graph) — Obsidian vault ως knowledge graph. Files = nodes, wiki links = edges. Υλοποιεί Louvain community detection μέσω του `graphology` library (JavaScript). Χρησιμοποιεί SQLite + sqlite-vec για vector storage, 384-dim embeddings από τίτλο + tags + πρώτη παράγραφο. Έχει PageRank, betweenness centrality, community detection. **Αυτό είναι το πιο κοντινό existing project στην ιδέα του Curator.**

**Σημαντική διαφορά Curator vs obra/knowledge-graph:**
- obra: μόνο untyped edges (wiki links)
- Curator: typed edges (semantic cosine, NER co-occurrence, temporal co-modification, same source domain)
- obra: Obsidian notes
- Curator: οποιοδήποτε αρχείο (PDF, DOCX, images, code)

#### Τύποι edges για Curator

| Edge Type | Περιγραφή | Υπολογισμός |
|-----------|-----------|-------------|
| `semantic_similarity` | BGE-M3 cosine ≥ 0.65 | existing embeddings |
| `ner_cooccurrence` | Κοινά named entities (άτομα, οργανισμοί) | spaCy NER |
| `temporal_proximity` | Δημιουργήθηκαν/τροποποιήθηκαν εντός 48h | kMDItemFSCreationDate |
| `source_domain` | Ίδιο source domain (π.χ. university portal) | kMDItemWhereFroms |
| `citation_link` | Ένα αρχείο αναφέρει filename άλλου | regex scan |
| `workflow_sequence` | Ανοίχτηκαν διαδοχικά εντός 5min | kMDItemUsedDates |

#### Αξιολόγηση για Curator

**Προτεραιότητα: HIGH**

Λόγοι:
1. Τα υπάρχοντα HDBSCAN clusters δεν έχουν inter-cluster relationships — ο KG τα συνδέει
2. Community detection → φυσικές ομαδοποιήσεις που διασχίζουν cluster boundaries
3. Typed edges = εξήγηση γιατί δύο αρχεία σχετίζονται (σπουδαίο για UX)
4. Louvain τρέχει σε O(n log n) — αρκετά γρήγορο για 10k αρχεία

**Τι χρειάζεται:**
```
pip install networkx python-louvain spacy
python -m spacy download en_core_web_sm
```

**Εκτιμώμενος κώδικας:** ~150-200 γραμμές Python

#### Code Sketch

```python
# curator_knowledge_graph.py
import networkx as nx
import community as community_louvain  # python-louvain
import spacy
from sentence_transformers import SentenceTransformer
import numpy as np
from itertools import combinations
from datetime import timedelta

nlp = spacy.load("en_core_web_sm")

def build_file_knowledge_graph(file_records: list[dict]) -> nx.Graph:
    """
    file_records: list of dicts με keys:
      - path, text_content, embedding (np.array),
        created_at, modified_at, used_dates, where_froms
    """
    G = nx.Graph()
    
    # Nodes
    for rec in file_records:
        G.add_node(rec["path"], **{
            "cluster_id": rec.get("cluster_id"),
            "created": rec["created_at"],
        })
    
    # Edges: typed
    for a, b in combinations(file_records, 2):
        edges_added = []
        
        # 1. Semantic similarity
        cos_sim = np.dot(a["embedding"], b["embedding"])
        if cos_sim >= 0.65:
            edges_added.append(("semantic", float(cos_sim)))
        
        # 2. NER co-occurrence
        ents_a = {e.text.lower() for e in nlp(a["text_content"][:2000]).ents
                  if e.label_ in ("PERSON", "ORG", "GPE")}
        ents_b = {e.text.lower() for e in nlp(b["text_content"][:2000]).ents
                  if e.label_ in ("PERSON", "ORG", "GPE")}
        overlap = len(ents_a & ents_b)
        if overlap >= 2:
            edges_added.append(("ner", overlap))
        
        # 3. Temporal proximity (created within 48h)
        dt = abs((a["created_at"] - b["created_at"]).total_seconds())
        if dt < 172800:  # 48 hours
            edges_added.append(("temporal", 1.0 - dt/172800))
        
        # 4. Same source domain
        if a.get("where_froms") and b.get("where_froms"):
            from urllib.parse import urlparse
            dom_a = {urlparse(u).netloc for u in a["where_froms"]}
            dom_b = {urlparse(u).netloc for u in b["where_froms"]}
            if dom_a & dom_b:
                edges_added.append(("source_domain", 1.0))
        
        if edges_added:
            weight = sum(w for _, w in edges_added) / len(edges_added)
            edge_types = [t for t, _ in edges_added]
            G.add_edge(a["path"], b["path"],
                       weight=weight, types=edge_types)
    
    return G

def detect_communities(G: nx.Graph) -> dict:
    partition = community_louvain.best_partition(G, weight="weight")
    return partition  # {filepath: community_id}
```

#### Gaps (Novel Contribution)
- **Zero existing work** χρησιμοποιεί typed edges για personal file graphs
- obra/knowledge-graph: untyped (wiki links μόνο)
- AIOS-LSFS: semantic retrieval μόνο, χωρίς graph structure
- **Curator gap:** combining kMDItemWhereFroms + kMDItemUsedDates + NER + semantic σε unified typed graph — **δεν υπάρχει πουθενά**

---

### Topic 2: Forgetting Curve για Αρχεία — Proactive Resurface πριν το Spotlight

#### Σύνοψη ιδέας
Ο χρήστης ξεχνάει **ΠΟΥ έβαλε** ένα αρχείο. Εφαρμόζουμε το Ebbinghaus forgetting curve model: κάθε αρχείο έχει "memory strength" που μειώνεται εκθετικά. Όταν η strength πέσει κάτω από threshold, το Curator εμφανίζει notification: "Πριν 3 εβδομάδες δούλευες σε: thesis_chapter3.pdf — θυμάσαι που το έχεις;"

#### Θεωρητικό Υπόβαθρο

**Ebbinghaus Forgetting Curve:**
```
R(t) = e^(-t/S)
```
όπου:
- `R` = retention (0-1, πιθανότητα ο χρήστης να θυμάται location)
- `t` = χρόνος από τελευταία χρήση (σε ημέρες)
- `S` = stability (αυξάνεται με κάθε χρήση, spaced repetition effect)

**Για αρχεία:**
- `t` = `datetime.now() - kMDItemLastUsedDate`
- `S` = `1 + 0.5 * kMDItemUseCount` (περισσότερες χρήσεις → πιο "σταθερή" μνήμη)
- Threshold resurface: R < 0.3 (έχει ξεχαστεί 70%+)

#### Papers που βρέθηκαν

**PIM / Re-finding:**
- **"Keeping Found Things Found: The Study and Practice of Personal Information Management"** — William Jones, 2007. Κλασικό βιβλίο PIM. Ορίζει το πρόβλημα re-finding ως κεντρικό challenge. Δεν έχει computational forgetting model.
  
- **"A Personal Knowledge Graph for Researchers"** — FIRE 2023, ACM DL (dl.acm.org/doi/10.1145/3632754.3632773). Εστιάζει σε research papers, όχι arbitrary files.

- **FileGram** (arXiv:2604.04901, 2026) — Closest related work. Εντοπίζει "persona drift" αλλά για AI agent personalization. Χρησιμοποιεί file-system behavioral traces για memory architecture με procedural, semantic, episodic channels. **Δεν έχει forgetting model.**

- **"Spaced Repetition and Mnemonics Enable Recall of Multiple Strong Passwords"** — arXiv:1410.1490. Δείχνει ότι spaced repetition λειτουργεί για memory. Εφαρμογή σε passwords, όχι file locations.

**Κενό:** Δεν υπάρχει **κανένα paper** που να εφαρμόζει Ebbinghaus model σε file location memory. Αυτό είναι genuine novel contribution.

#### macOS Metadata για Forgetting Curve

Από την έρευνα του Spotlight metadata:
- `kMDItemLastUsedDate`: timestamp τελευταίας χρήσης (LaunchServices — μόνο από double-click ή LaunchServices open)
- `kMDItemUsedDates`: array ημερομηνιών χρήσης (history)
- `kMDItemUseCount`: αριθμός φορών που άνοιξε το αρχείο

Αυτά τα 3 attributes αρκούν για πλήρες forgetting model.

```python
import subprocess, json
from datetime import datetime, date
import math

def get_file_memory_metadata(filepath: str) -> dict:
    """Ανακτά kMDItem* attributes μέσω mdls."""
    result = subprocess.run(
        ["mdls", "-name", "kMDItemLastUsedDate",
         "-name", "kMDItemUsedDates",
         "-name", "kMDItemUseCount", filepath],
        capture_output=True, text=True
    )
    # Parse mdls output (simplified)
    lines = result.stdout.strip().split("\n")
    meta = {}
    for line in lines:
        if "kMDItemUseCount" in line:
            try:
                meta["use_count"] = int(line.split("=")[-1].strip())
            except ValueError:
                meta["use_count"] = 0
        elif "kMDItemLastUsedDate" in line and "(" in line:
            # Parse date string from mdls
            date_str = line.split("=")[-1].strip().strip('"')
            try:
                meta["last_used"] = datetime.strptime(date_str[:19], "%Y-%m-%d %H:%M:%S")
            except Exception:
                meta["last_used"] = None
    return meta

def ebbinghaus_retention(filepath: str) -> float:
    """
    Επιστρέφει R ∈ [0,1]: πιθανότητα χρήστης να θυμάται location.
    R < 0.3 → trigger resurface notification.
    """
    meta = get_file_memory_metadata(filepath)
    last_used = meta.get("last_used")
    use_count = meta.get("use_count", 1) or 1
    
    if last_used is None:
        return 0.0  # Άγνωστο — resurface
    
    t_days = (datetime.now() - last_used).total_seconds() / 86400
    S = 1.0 + 0.5 * use_count  # stability αυξάνεται με χρήση
    R = math.exp(-t_days / S)
    return R

def get_files_to_resurface(file_paths: list[str],
                           threshold: float = 0.3) -> list[str]:
    """Επιστρέφει αρχεία που πρέπει να resurfaced."""
    return [fp for fp in file_paths
            if ebbinghaus_retention(fp) < threshold]
```

#### Αξιολόγηση για Curator

**Προτεραιότητα: HIGH (+ Novel Contribution)**

Λόγοι:
1. **Zero papers** για file location forgetting — πλήρες research gap
2. macOS metadata (kMDItemLastUsedDate, kMDItemUseCount) είναι ήδη διαθέσιμο
3. Curator ήδη χρησιμοποιεί αυτά τα metadata → minimal additional implementation
4. UX value: proactive notification πριν ο χρήστης ψάξει στο Spotlight
5. Εξαιρετικό για CHI paper (human memory + file systems = interdisciplinary)

**Τι χρειάζεται:**
- Κανένα νέο pip package (μόνο `subprocess` + `math`)
- ~50 γραμμές Python για forgetting curve engine
- Tauri notification system για resurface UI

**Gaps (Novel Contribution):**
- Δεν υπάρχει κανένα existing system που να μοντελοποιεί **file location memory** με Ebbinghaus
- FileGram: file behavioral traces αλλά χωρίς forgetting model
- Spotlight: reactive (ψάχνει όταν ζητηθεί) — Curator: **proactive** (resurface πριν χρειαστεί)
- **Novel term:** "File Location Retention Score" (FLRS) = R(t) based on kMDItemUsedDates

---

### Topic 3: Semantic Compression — Synthesis Note από Παρόμοια Αρχεία

#### Σύνοψη ιδέας
Όταν πολλά αρχεία έχουν overlapping content (cosine ≥ 0.80), αντί να τα αφήνουμε ως ξεχωριστά αρχεία, το Curator τα "συμπιέζει" σε ένα synthesized summary note. Δεν είναι απλό dedup (delete duplicates) — είναι **synthesis**: κρατάμε τα μοναδικά elements από κάθε αρχείο και δημιουργούμε ένα νέο "master document".

#### Papers που βρέθηκαν

**Multi-Document Summarization / Synthesis:**
- **"Do Multi-Document Summarization Models Synthesize?"** — DeYoung et al., TACL 2024 (ACL 2024, Bangkok). Κεντρικό εύρημα: ακόμα και GPT-4 αποτυγχάνει στη σύνθεση πολλαπλών εγγράφων — over-sensitive στο input ordering, under-sensitive στις αλλαγές composition. Προτείνει diverse candidate generation + selection. **Σχετικό αλλά εστιάζει σε opinion/evidence synthesis.**

- **"Information fusion in the context of multi-document summarization"** — Academia.edu. Word graph alignment με SBERT embeddings για cluster-then-synthesize approach.

- **"Advanced multiple document summarization via iterative recursive transformer networks"** — PMC 2025. Iterative refinement για multi-doc synthesis.

**Text Deduplication:**
- **RETSim: Resilient and Efficient Text Similarity** — arXiv:2311.17264, ICLR. Google's RETSim model — state-of-art για near-duplicate detection. Cosine threshold 0.10-0.15 για near-duplicates, 0.80+ για partial overlap.
- **UniSim** (github.com/google/unisim) — Πακέτο για efficient similarity + fuzzy matching + clustering. Χρησιμοποιεί RETSim internally.
- **text-dedup** (github.com/ChenghaoMou/text-dedup) — All-in-one text deduplication: MinHash, SimHash, SemDeDup, BF.

#### Σχεδιασμός για Curator

**Pipeline:**
1. BGE-M3 embeddings (ήδη υπάρχουν στο Curator)
2. Cosine similarity matrix μεταξύ αρχείων στο ίδιο cluster
3. Groups με pairwise cosine ≥ 0.80 → "semantic duplicates group"
4. Local Ollama LLM (qwen2.5:7b ή llama3.2) → synthesize group σε 1 summary note
5. Αποθήκευση summary + references στα original files

**Γιατί qwen2.5:7b;**
- Τρέχει τοπικά σε 8-16GB RAM
- Context window: 128K tokens (αρκετό για 5-10 small files)
- Multilinguality: Greek + English
- Inference: ~3-8 δευτερόλεπτα ανά σύνθεση σε Apple Silicon

```python
# curator_semantic_compression.py
import numpy as np
from sklearn.metrics.pairwise import cosine_similarity
import ollama  # pip install ollama
from pathlib import Path
from itertools import combinations

SYNTHESIS_THRESHOLD = 0.80  # cosine ≥ 0.80 → synthesize candidates
SYNTHESIS_PROMPT = """You are a knowledge synthesis assistant.
The following {n} documents have highly overlapping content.
Create ONE comprehensive synthesis note that:
1. Captures ALL unique information from each document
2. Removes redundancy  
3. Preserves important details, dates, and specific facts
4. Is written in the SAME language as the documents

Documents:
{documents}

Output: A single synthesized document that replaces all {n} originals."""

def find_semantic_duplicates(
    embeddings: dict[str, np.ndarray],
    threshold: float = SYNTHESIS_THRESHOLD
) -> list[list[str]]:
    """Βρίσκει groups αρχείων με overlapping content."""
    paths = list(embeddings.keys())
    emb_matrix = np.stack([embeddings[p] for p in paths])
    sim_matrix = cosine_similarity(emb_matrix)
    
    # Union-Find για grouping
    parent = {p: p for p in paths}
    def find(x):
        while parent[x] != x:
            parent[x] = parent[parent[x]]
            x = parent[x]
        return x
    def union(x, y):
        parent[find(x)] = find(y)
    
    for i, j in combinations(range(len(paths)), 2):
        if sim_matrix[i, j] >= threshold:
            union(paths[i], paths[j])
    
    groups: dict[str, list[str]] = {}
    for p in paths:
        root = find(p)
        groups.setdefault(root, []).append(p)
    
    return [g for g in groups.values() if len(g) >= 2]

def synthesize_group(file_paths: list[str],
                     model: str = "qwen2.5:7b") -> str:
    """Χρησιμοποιεί Ollama για synthesis των n αρχείων."""
    documents = []
    for i, fp in enumerate(file_paths, 1):
        try:
            content = Path(fp).read_text(errors="ignore")[:3000]
            documents.append(f"--- Document {i}: {Path(fp).name} ---\n{content}")
        except Exception:
            continue
    
    prompt = SYNTHESIS_PROMPT.format(
        n=len(documents),
        documents="\n\n".join(documents)
    )
    
    response = ollama.generate(model=model, prompt=prompt)
    return response["response"]

def run_semantic_compression(
    cluster_files: dict[str, list[str]],  # cluster_id → [file_paths]
    embeddings: dict[str, np.ndarray],
    output_dir: str = "~/Downloads/.curator_synthesis/"
) -> list[dict]:
    """Main pipeline για semantic compression."""
    results = []
    for cluster_id, files in cluster_files.items():
        cluster_embs = {f: embeddings[f] for f in files if f in embeddings}
        groups = find_semantic_duplicates(cluster_embs)
        
        for group in groups:
            synthesis = synthesize_group(group)
            out_path = Path(output_dir).expanduser() / f"synthesis_{cluster_id}_{len(group)}files.md"
            out_path.parent.mkdir(exist_ok=True)
            out_path.write_text(synthesis)
            results.append({"group": group, "synthesis_path": str(out_path)})
    return results
```

#### Performance Εκτίμηση

| Αριθμός αρχείων ανά group | qwen2.5:7b (Apple M2 Pro) | llama3.2:3b |
|---------------------------|---------------------------|-------------|
| 2 αρχεία × 1500 words     | ~4-6 δευτ.                | ~2-3 δευτ.  |
| 5 αρχεία × 1000 words     | ~8-12 δευτ.               | ~4-6 δευτ.  |
| 10 αρχεία × 500 words     | ~10-15 δευτ.              | ~5-8 δευτ.  |

#### Αξιολόγηση για Curator

**Προτεραιότητα: MEDIUM-HIGH**

Λόγοι υπέρ:
1. Μετατρέπει τον Curator από file organizer σε **knowledge manager**
2. Χρησιμοποιεί ήδη υπάρχοντα BGE-M3 embeddings
3. Ollama ήδη local — δεν χρειάζεται cloud
4. Genuine UX value: 30 παρόμοια lecture PDFs → 1 master note

Λόγοι κατά:
1. LLM synthesis: περιστασιακά hallucinations (perils of 7B model)
2. User trust: πρέπει να φαίνεται "αρχεία πηγές" στο synthesis
3. Non-text files (images, executables): δεν υποστηρίζεται
4. Χρόνος: για 100+ αρχεία σε ένα cluster → μπορεί να πάρει λεπτά

**Pip installs:**
```
pip install ollama scikit-learn
```

**Gaps:**
- UniSim/RETSim: dedup μόνο — **δεν synthesize**
- text-dedup: dedup + remove — **δεν synthesize**  
- "Do Multi-Document Summarization Models Synthesize?": desktop files δεν αναφέρονται
- **Curator gap:** desktop file synthesis με local LLM triggered by embedding threshold — **novel UX paradigm**
- Novel term: **"Semantic Compression"** (αντί για dedup/summarization)

---

### Topic 4: Zero-Shot New Life Category Detection — Νέος Τύπος Ζωής vs Νέο Cluster

#### Σύνοψη ιδέας
Στο Curator, νέα αρχεία μπορούν να είναι:
- **(α) Νέο cluster** εντός γνωστού domain (π.χ. νέο course εντός academic domain)
- **(β) Εντελώς νέος τύπος ζωής** (π.χ. πρώτα internship αρχεία — ο χρήστης μπήκε σε νέα ζωή)

Η διάκριση είναι κρίσιμη: (α) = quiet reorganization, (β) = major notification "Φαίνεται ότι ξεκινάς κάτι νέο — να δημιουργήσω νέα κατηγορία 'Internship';".

#### Papers που βρέθηκαν

**Concept Drift & Novelty Detection:**
- **"Learning Unbiased Cluster Descriptors for Interpretable Imbalanced Concept Drift Detection"** (ICD3) — arXiv:2603.06757, 2025. Imbalanced data concept drift detection. Addresses detection σε streaming unlabeled data.

- **"Adaptive Anomaly Detection in the Presence of Concept Drift"** — arXiv:2506.15831. Time-series anomaly detection όταν το "normal" αλλάζει.

- **"Exploring Zero-Shot Data Drift Detection with Large Language Models"** — Springer, 2025. LLM-based zero-shot drift detection χωρίς labeled instances ή predefined metrics.

- **"Unsupervised Parameter-free Outlier Detection using HDBSCAN* Outlier Profiles"** — arXiv:2411.08867, Nov 2024. Βελτίωση GLOSH: αυτόματο selection του minpts και threshold για inlier/outlier classification. **Άμεσα εφαρμόσιμο στον Curator.**

- **"A Novel Incremental Clustering Technique with Concept Drift Detection"** — arXiv:2003.13225. Incremental clustering με integrated drift detection.

- **EmCStream** (Zubaroğlu, 2023, Statistical Analysis and Data Mining): UMAP + concept drift detection για evolving streams. Αυτόματη ανίχνευση drift και re-embedding. **Πολύ relevant για Curator's UMAP refit mechanism.**

- **"Online Clustering for Novelty Detection and Concept Drift in Data Streams"** — ResearchGate. Centroids + radius threshold: νέο point εκτός όλων των radii → novel cluster candidate.

**OOD Detection:**
- **"Zero-Shot Out-of-Distribution Detection with Outlier Label Exposure"** — arXiv:2406.01170. CLIP-based zero-shot OOD detection. Δεν εφαρμόζεται άμεσα σε files.

- **"ClusterMine: Robust Label-Free Visual Out-Of-Distribution Detection"** — arXiv:2511.07068. CLIP ως zero-shot OOD detector.

#### Αλγόριθμος για Curator

**Διάκριση (α) vs (β) — 3-tier test:**

```
Tier 1: DenStream outlier flag
  → Αν νέο αρχείο δεν εντάχθηκε σε κανένα cluster: OUTLIER

Tier 2: GLOSH score + centroid distance
  → GLOSH score > 0.8 AND min cosine distance από κάθε centroid > 0.45
  → "New cluster candidate" (πιθανώς νέος τύπος ζωής)

Tier 3: LLM semantic check (optional)
  → Αν embedding cluster των νέων outliers έχει coherence > 0.7
  AND δεν overlap με καμία γνωστή κατηγορία:
  → "Life Event Signal" → user notification
```

```python
# curator_life_event_detector.py
import numpy as np
import hdbscan
from sklearn.metrics.pairwise import cosine_similarity

def detect_new_life_category(
    new_file_embeddings: list[np.ndarray],
    existing_centroids: dict[str, np.ndarray],  # cluster_name → centroid
    glosh_threshold: float = 0.75,
    centroid_dist_threshold: float = 0.45,
    coherence_threshold: float = 0.65
) -> dict:
    """
    Αποφασίζει αν τα νέα αρχεία αντιπροσωπεύουν:
    - 'known_extension': νέο cluster εντός γνωστού domain
    - 'new_life_category': εντελώς νέος τύπος ζωής
    - 'noise': τυχαία αρχεία
    """
    if len(new_file_embeddings) < 3:
        return {"verdict": "noise", "confidence": 0.5}
    
    emb_matrix = np.stack(new_file_embeddings)
    
    # Step 1: GLOSH outlier scores
    clusterer = hdbscan.HDBSCAN(min_cluster_size=3,
                                  prediction_data=True)
    clusterer.fit(emb_matrix)
    outlier_scores = hdbscan.validity.GLOSH(clusterer, k=5)
    high_glosh_frac = np.mean(outlier_scores > glosh_threshold)
    
    # Step 2: Centroid distances
    centroid_matrix = np.stack(list(existing_centroids.values()))
    similarities = cosine_similarity(emb_matrix, centroid_matrix)
    max_sim_per_file = similarities.max(axis=1)
    far_from_known = np.mean(1 - max_sim_per_file > centroid_dist_threshold)
    
    # Step 3: Internal coherence of new files
    internal_sim = cosine_similarity(emb_matrix)
    np.fill_diagonal(internal_sim, 0)
    coherence = internal_sim.sum() / (len(emb_matrix) * (len(emb_matrix)-1))
    
    # Decision logic
    if far_from_known > 0.7 and high_glosh_frac > 0.6 and coherence > coherence_threshold:
        return {
            "verdict": "new_life_category",
            "confidence": (far_from_known + high_glosh_frac + coherence) / 3,
            "suggested_name": None  # LLM call here
        }
    elif far_from_known > 0.4 and coherence > 0.5:
        return {"verdict": "known_extension", "confidence": 0.6}
    else:
        return {"verdict": "noise", "confidence": 0.5}
```

#### Practical Examples

| Scenario | GLOSH | Centroid distance | Coherence | Verdict |
|----------|-------|-------------------|-----------|---------|
| Νέα courses ίδιου university | 0.4 | 0.2 (academic cluster) | 0.8 | known_extension |
| Πρώτα internship αρχεία | 0.85 | 0.52 (all clusters) | 0.78 | **new_life_category** |
| Random downloads | 0.7 | 0.6 | 0.2 | noise |
| Side project (coding) | 0.75 | 0.48 | 0.72 | new_life_category |

#### Αξιολόγηση για Curator

**Προτεραιότητα: HIGH (+ Novel Contribution)**

Λόγοι:
1. Καμία existing system δεν κάνει **life event detection** από file patterns
2. DenStream ήδη στον Curator → GLOSH scores διαθέσιμα από HDBSCAN
3. Extremely powerful UX: "Φαίνεται ότι ξεκίνησες Internship — να δημιουργήσω φάκελο;"
4. Zero papers για αυτό ακριβώς το πρόβλημα (personal file life stage transitions)

**Pip installs:**
```
pip install hdbscan  # ήδη υπάρχει
```

**Gaps (Novel Contribution):**
- Concept drift detection papers: business processes / data streams — **όχι personal file life stages**
- GLOSH: outlier detection — **δεν ερμηνεύει σε "life event"**
- DenStream: stream clustering — **δεν φτιάχνει life event notifications**
- **Curator gap:** φιλοσοφική/UX καινοτομία: file system ως **autobiography mirror** — αρχεία αντικατοπτρίζουν life transitions → novel PIM paradigm

---

### Topic 5: Ακαδημαϊκή Δημοσίευση — Curator's Novel Contributions

#### Τα 3 Novel Contributions του Curator

**Contribution 1: File Biography**
- `kMDItemUseCount` + `kMDItemUsedDates` ως **organization signal** (όχι απλά metadata)
- Hypothesis: αρχεία που χρησιμοποιούνται συχνά ανήκουν σε "hot" clusters → αυτόματα προωθούνται σε πιο accessible θέση
- **Zero papers** που χρησιμοποιούν macOS behavioral metadata ως clustering/organization feature

**Contribution 2: Drift-Triggered UMAP Refit**
- DenStream εντοπίζει concept drift → trigger automatic UMAP refit
- EmCStream (2023) κάνει κάτι παρόμοιο αλλά με k-means, όχι HDBSCAN
- **Novel:** DenStream → UMAP → HDBSCAN pipeline με drift-triggered refit — δεν υπάρχει σε κανένα paper

**Contribution 3: FLACON + kMDItemUseCount Fusion**
- FLACON: information-theoretic flags (entropy-based)
- Fusion με behavioral signal (kMDItemUseCount)
- **Novel:** combining information-theoretic flagging με behavioral frequency — δεν υπάρχει

#### Σχετικά Papers (Related Work για σύγκριση)

| System | Venue | Method | Limitation vs Curator |
|--------|-------|--------|----------------------|
| **llama-fs** (iyaja, 2024) | GitHub (10K+ stars) | Llama3 + tree structure | Cloud API, no streaming, no embeddings |
| **AIOS-LSFS** (arXiv:2410.11843) | ICLR 2025 | LLM semantic file system | Server-based, no local embeddings, no drift |
| **FileGram** (arXiv:2604.04901) | arXiv 2026 | File behavioral traces + memory | Agent-focused, no organization, no clustering |
| **obra/knowledge-graph** | GitHub 2026 | Obsidian + Louvain | Notes only, untyped edges, no drift |
| **HERCULES** (naming) | Internal/prior art | Hierarchical naming | Naming only, no organization |

#### Conference Deadlines & Venues

**CHI 2027:**
- Full paper deadline: **10 Σεπτεμβρίου 2026** (AoE)
- Venue: ACM CHI — Human Factors in Computing Systems
- Format: 10+ pages ACM format
- Topics: PIM, file management, proactive systems, notification design
- **Fit για Curator:** HIGH — ειδικά για Forgetting Curve + Life Event Detection (strong HCI angle)

**UMAP 2026:**
- Full paper deadline: **29 Ιανουαρίου 2026** (abstract: 22 Ιανουαρίου)
- Venue: User Modeling, Adaptation and Personalization, Gothenburg Sweden, June 8-11 2026
- Format: 8 pages (full), 4 pages (short)
- Topics: knowledge graphs, user modeling, personalization, LLM-based systems
- **Fit για Curator:** MEDIUM-HIGH — ειδικά για Knowledge Graph + Forgetting Curve angle
- **ΣΗΜΕΙΩΣΗ:** Deadline 29 Ιανουαρίου 2026 έχει παρέλθει (τώρα Μάη 2026) → **UMAP 2027**

**ECIR 2026:**
- Full paper deadline: **2 Οκτωβρίου 2025** (παρήλθε)
- Short paper: **14 Οκτωβρίου 2025** (παρήλθε)
- Conference: Delft, Netherlands, 29 Μαρτίου – 2 Απριλίου 2026 (έχει γίνει)
- **ECIR 2027:** deadline ~Οκτώβριος 2026 — **αυτό είναι το target**
- Format: 12 pages full papers
- Topics: information retrieval, personal information management, semantic search
- **Fit για Curator:** HIGH — IR angle (FLACON + embeddings + retrieval)

**Συγκεντρωτικός πίνακας targets:**

| Venue | Deadline target | Format | Best Contribution |
|-------|----------------|--------|-------------------|
| CHI 2027 | Sept 10 2026 | Full paper (10p) | Forgetting Curve + Life Event |
| ECIR 2027 | ~Oct 2026 | Full paper (12p) | FLACON + Drift-Triggered Refit |
| UMAP 2027 | ~Jan 2027 | Full/Short paper | Knowledge Graph + Forgetting |
| IUI 2027 | ~Oct 2026 | Full paper (8p) | All 3 contributions combined |

**IUI (Intelligent User Interfaces):** Εξαιρετικό fit — combining intelligent systems με user interface, ακριβώς αυτό κάνει ο Curator.

#### Τι χρειάζεται για Submission

**Baselines για comparison:**
1. **llama-fs** (iyaja/llama-fs) — baseline για automatic file organization
2. **AIOS-LSFS** — baseline για semantic file retrieval
3. **Manual organization** (user study) — baseline για user behavior
4. **Simple HDBSCAN without drift** — ablation για drift-triggered refit
5. **Random naming** — baseline για HERCULES naming quality

**Evaluation metrics:**
- Organization quality: Silhouette score, Davies-Bouldin index (clustering quality)
- Re-finding time: user study (seconds to locate a file)
- Naming quality: BERTScore vs manual labels
- Drift detection F1: synthetic drift injection test
- User satisfaction: SUS (System Usability Scale)

**Dataset:**
- Synthetic: 10K files με known categories + injected drift events
- Real: ~/Downloads από 5-10 users (IRB required for CHI)
- macOS only (kMDItem* metadata)

**Code sketch για evaluation:**
```python
# Silhouette score για clustering quality
from sklearn.metrics import silhouette_score
import numpy as np

def evaluate_clustering(embeddings: np.ndarray,
                        labels: np.ndarray) -> dict:
    """Baseline vs Curator clustering quality."""
    sil = silhouette_score(embeddings, labels, metric="cosine")
    from sklearn.metrics import davies_bouldin_score
    db = davies_bouldin_score(embeddings, labels)
    return {
        "silhouette": sil,       # higher = better (max 1.0)
        "davies_bouldin": db,    # lower = better
    }

# BERTScore για naming quality
from bert_score import score as bert_score

def evaluate_naming(generated_names: list[str],
                    reference_names: list[str]) -> float:
    P, R, F1 = bert_score(generated_names, reference_names,
                           lang="en", model_type="microsoft/deberta-xlarge-mnli")
    return F1.mean().item()
```

#### Αξιολόγηση

**Αρκεί για CHI/UMAP/ECIR paper;**

**CHI 2027:** YES, αν:
- User study με ≥12 participants (re-finding time + SUS)
- Forgetting Curve + Life Event Detection = strong HCI narrative
- "File Biography" framing = compelling story

**ECIR 2027:** YES, αν:
- Drift-Triggered Refit: quantitative comparison vs static UMAP
- FLACON: comparison vs TF-IDF flags
- 10K+ files dataset

**UMAP 2027:** YES (shorter paper, 8 pages):
- Knowledge Graph + Forgetting Curve = user modeling angle
- kMDItemUsedDates ως user behavior signal

**Recommendation:** Target **CHI 2027** (Sept 2026 deadline) για full paper. Παράλληλα **ECIR 2027** (Oct 2026) ως second venue. Το contribution pattern ("file as behavioral entity" + "system that learns user's file life") είναι compelling για CHI narrative.

**Κυρίαρχο gap:** Κανένα existing system δεν αντιμετωπίζει αρχεία ως **"autobiographical objects"** — κάθε αρχείο έχει biography (πότε δημιουργήθηκε, πόσες φορές χρησιμοποιήθηκε, πότε ξεχάστηκε, ποια life stage αντιπροσωπεύει). Αυτή η conceptual framing είναι η novelty του Curator.

---

## Συνοπτικός Πίνακας Priorities

| Topic | Προτεραιότητα | Novel Gap | Υλοποίηση | Pip installs |
|-------|-------------|-----------|-----------|-------------|
| Knowledge Graph | HIGH | Typed edges για personal files | ~200 γρ. | networkx, python-louvain, spacy |
| Forgetting Curve | HIGH ★ | Zero papers για file location memory | ~80 γρ. | (μόνο stdlib) |
| Semantic Compression | MEDIUM-HIGH | Desktop file synthesis + local LLM | ~120 γρ. | ollama, scikit-learn |
| Life Category Detection | HIGH ★ | Life stage transitions σε files | ~100 γρ. | hdbscan (ήδη) |
| Publication | HIGH ★★ | CHI 2027 target | user study + eval | bert-score |

**★★ = Άμεση ενέργεια συνιστάται**

---

## Βιβλιογραφία / Sources

1. DeYoung et al., "Do Multi-Document Summarization Models Synthesize?", TACL/ACL 2024 — https://arxiv.org/abs/2301.13844
2. Ghosh et al., "Unsupervised Parameter-free Outlier Detection using HDBSCAN* Outlier Profiles", arXiv:2411.08867, Nov 2024
3. Liu et al., "FileGram: Grounding Agent Personalization in File-System Behavioral Traces", arXiv:2604.04901, Apr 2026
4. Menschikov et al., "PersonalAI: Systematic Comparison of KG Storage Approaches", arXiv:2506.17001
5. obra/knowledge-graph: https://github.com/obra/knowledge-graph
6. taynaud/python-louvain: https://github.com/taynaud/python-louvain
7. google/unisim (RETSim): https://github.com/google/unisim — arXiv:2311.17264
8. ChenghaoMou/text-dedup: https://github.com/ChenghaoMou/text-dedup
9. iyaja/llama-fs: https://github.com/iyaja/llama-fs
10. AIOS-LSFS: https://github.com/agiresearch/AIOS-LSFS — arXiv:2410.11843, ICLR 2025
11. Zubaroğlu, "Online embedding and clustering of evolving data streams" (EmCStream), 2023 — https://onlinelibrary.wiley.com/doi/full/10.1002/sam.11590
12. Jones, "Keeping Found Things Found", 2007 — https://dl.acm.org/doi/10.5555/2155696
13. Apple Developer, kMDItemLastUsedDate — https://developer.apple.com/documentation/coreservices/kmditemlastuseddate
14. ACM UMAP 2026 CFP — https://www.um.org/umap2026/call-for-full-short-papers/
15. CHI 2027 Papers — https://chi2027.acm.org/authors/papers/
16. ECIR 2026 — https://ecir2026.eu/