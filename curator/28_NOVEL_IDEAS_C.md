## ΜΕΡΟΣ 28: ΝΕΕΣ ΠΡΩΤΟΠΟΡΙΑΚΕΣ ΙΔΕΕΣ — ΑΞΙΟΛΟΓΗΣΗ
*(Έρευνα 2026-05-22 — 18 searches, 40+ sources)*

---

### Ιδέα 1: Smart Rename Suggestions

**Περιγραφή:** Για αρχεία με αναφορικά ονόματα (document.pdf, IMG_4523.jpg, untitled.py) → προτείνουμε καλύτερο όνομα βασισμένο στο περιεχόμενο μέσω LLM.

#### (α) Υπάρχει παρόμοιο;

Ναι — και μάλιστα αρκετά παρόμοια εργαλεία:

- **[ai-renamer (ozgrozer)](https://github.com/ozgrozer/ai-renamer)** — Node.js CLI που χρησιμοποιεί Ollama/LM Studio (Llava, Gemma, Llama) για rename based on content. Υποστηρίζει εικόνες και PDF. **Τρέχει locally.**
- **[ai-rename (brooksc)](https://github.com/brooksc/ai-rename)** — Python, local LLM, γενικό rename.
- **[ollama-rename-files (octrow)](https://github.com/octrow/ollama-rename-files)** — Python + Ollama, content-aware rename, zero API cost.
- **[wunderkind2k1/ai-pdf-renamer](https://github.com/wunderkind2k1/ai-pdf-renamer)** — Batch PDF renamer, fully offline, LLM για descriptive filenames.
- **[yousefebrahimi0/Offline-AI-File-Organizer](https://github.com/yousefebrahimi0/Offline-AI-File-Organizer)** — Combines rename + organize, 100% local.
- **RenameClick** — Commercial macOS/Windows app, local-first, Ollama support, preview πριν apply.
- **NameQuick** — macOS native, BYOK + Ollama support.

**Η έρευνα στο DEV Community** λέει: "LLMs solve intent recognition and content understanding simultaneously for file renaming." Η προσέγγιση: αν το filename δεν είναι readable English → feed στο LLM → generate descriptive name.

**Συμπέρασμα:** Το concept **υπάρχει** στο open-source ecosystem. Το κάνουν standalone εργαλεία. Δεν το έχει κανείς **ενσωματωμένο** σε organizer workflow με clustering.

#### (β) Αξίζει να υλοποιηθεί;

**Ναι** — αλλά η αξία είναι στην **integration**, όχι στο concept per se. Τα standalone εργαλεία απαιτούν manual invocation. Στο Curator, το rename suggestion εμφανίζεται **inline** στο approval UI, ακριβώς όταν ο χρήστης ήδη κοιτάζει το αρχείο. Αυτή η contextual παρουσίαση είναι η πραγματική καινοτομία.

**Επιπλέον πλεονέκτημα:** Ήδη έχουμε MarkItDown για content extraction + Ollama. Η προσθήκη είναι minimal.

#### (γ) Τι απαιτείται τεχνικά;

1. **Trigger condition:** `len(filename_stem) < 6` OR `filename matches r'^(document|untitled|img_|image|photo|scan|file|download)\d*$'` (case-insensitive)
2. **Content extraction:** MarkItDown (ήδη έχουμε) → πρώτα 500 tokens
3. **LLM prompt:** `"Given this file content snippet, suggest a concise descriptive filename (max 5 words, lowercase, underscores). No extension. Content: {snippet}"`
4. **FSEvents / avoid re-scan loop:** Χρησιμοποιούμε το `notify` crate (Rust) με debouncing. Κατά το rename, το σύστημα κρατάει ένα `HashSet<PathBuf>` με "pending renames" — αν το event path είναι εκεί, το skip. Η [Apple FSEvents documentation](https://developer.apple.com/library/archive/documentation/Darwin/Conceptual/FSEvents_ProgGuide/) και το [notify-debouncer-mini](https://docs.rs/notify) υποστηρίζουν αυτό.
5. **UX:** Inline suggestion στο approval UI — `[document.pdf → lecture_notes_epl326.pdf] ✓ Accept / ✎ Edit / ✗ Skip`
6. **Persist:** Μόνο αν ο χρήστης accept → `os.rename()` + update SQLite catalog

```python
import re

UNINFORMATIVE_PATTERN = re.compile(
    r'^(document|untitled|img_?|image|photo|scan|file|download|screenshot|'
    r'attachment|temp|tmp|new|copy|draft)\d*$',
    re.IGNORECASE
)

def needs_rename_suggestion(path: str) -> bool:
    stem = Path(path).stem
    return bool(UNINFORMATIVE_PATTERN.match(stem)) or len(stem) <= 5

def suggest_rename(path: str, ollama_client) -> str:
    content = extract_content(path)[:500]  # MarkItDown
    prompt = f"""File content snippet:
{content}

Suggest a concise filename (max 5 words, lowercase_with_underscores, no extension):"""
    return ollama_client.generate(model="qwen2.5:7b", prompt=prompt).strip()
```

#### (δ) Πολυπλοκότητα

| Κομμάτι | Γραμμές κώδικα |
|---|---|
| Uninformative filename detection | ~15 γραμμές Python |
| Rename suggestion (LLM call) | ~20 γραμμές Python |
| Avoid FSEvents re-scan loop (Rust) | ~30 γραμμές Rust |
| Approval UI inline suggestion | ~40 γραμμές React/TSX |
| SQLite catalog update on accept | ~10 γραμμές Python |
| **Σύνολο** | **~115 γραμμές** |

**Βαθμός δυσκολίας:** Χαμηλός-Μέτριος. Ο μόνος tricky τομέας είναι το FSEvents loop prevention.

---

### Ιδέα 2: Smart Archive (Semester-End Detection)

**Περιγραφή:** Αυτόματη ανίχνευση cluster που δεν έχει αγγιχτεί 3+ μήνες ΚΑΙ έχει >20 αρχεία → πρόταση αρχειοθέτησης.

#### (α) Υπάρχει παρόμοιο;

Στον enterprise χώρο, υπάρχει:
- **Jira:** Archive inactive projects (>180 days no activity) — [Atlassian docs](https://confluence.atlassian.com/enterprise/archive-inactive-projects-1688929025.html)
- **Microsoft Teams:** Lifecycle management, inactivity detection, owner notification — [ShareGate](https://sharegate.com/blog/build-a-microsoft-teams-lifecycle-management-plan-archiving-and-deleting-inactive-teams)
- **Smartcat:** Automatic project archiving — [Smartcat help](https://help.smartcat.com/1539811-automatic-project-archiving/)
- **Egnyte:** Content lifecycle policies — [Egnyte](https://helpdesk.egnyte.com/hc/en-us/articles/360034403532-Content-Lifecycle-Policies)

**Για personal files / local file systems:** Δεν υπάρχει κάτι ανάλογο. Η έρευνα PIM (Personal Information Management) αναγνωρίζει ότι "people find it difficult to evaluate the worth of accumulated materials" — [arxiv:2402.06421 "What's in People's Digital File Collections?"](https://arxiv.org/pdf/2402.06421) — αλλά κανένα tool δεν αυτοματοποιεί την αρχειοθέτηση.

**macOS mtime concept:** Το directory mtime ενημερώνεται μόνο όταν αλλάζουν τα άμεσα contents (προσθήκη/αφαίρεση αρχείων), ΟΧΙ αναδρομικά. Άρα χρειαζόμαστε `max(kMDItemLastUsedDate)` across all files in cluster — αυτό είναι στο SQLite catalog μας ήδη.

#### (β) Αξίζει;

**Ναι** — και με υψηλή χρησιμότητα για φοιτητή. Τα semester clusters (EPL326-Fall2025) έχουν φυσικό lifecycle. Μετά τις εξετάσεις, δεν ξανανοίγονται. Η αυτόματη πρόταση αρχειοθέτησης εξοικονομεί mental load.

**Novelty:** Η combination "cluster-aware + academic calendar heuristic + local-only" είναι genuinely new. Τα enterprise tools λειτουργούν σε folder ιεραρχίες, όχι σε dynamically-discovered semantic clusters.

#### (γ) Τι απαιτείται τεχνικά;

1. **Signal:** `SELECT cluster_id, COUNT(*), MAX(last_used_date) FROM files GROUP BY cluster_id HAVING COUNT(*) > 20 AND MAX(last_used_date) < datetime('now', '-90 days')`
2. **Semester heuristic (optional):** Αν cluster name contains "Fall", "Spring", "2024", "2025" → lower threshold (60 days αντί 90)
3. **UX:** macOS notification (Tauri `tauri-plugin-notification`) + in-app banner
4. **Destination:** `~/Documents/Archive/{YYYY}-{Season}/{ClusterName}/`
5. **Action:** Move files → update SQLite catalog → remove cluster from active view

```python
import sqlite3
from datetime import datetime, timedelta

def find_archivable_clusters(db_path: str, min_files: int = 20,
                              dormant_days: int = 90) -> list[dict]:
    cutoff = datetime.now() - timedelta(days=dormant_days)
    conn = sqlite3.connect(db_path)
    rows = conn.execute("""
        SELECT cluster_id, cluster_name, COUNT(*) as file_count,
               MAX(last_used_date) as last_active
        FROM files
        GROUP BY cluster_id
        HAVING file_count > ? AND last_active < ?
        ORDER BY last_active ASC
    """, (min_files, cutoff.isoformat())).fetchall()
    return [{"id": r[0], "name": r[1], "count": r[2], "last_active": r[3]}
            for r in rows]
```

#### (δ) Πολυπλοκότητα

| Κομμάτι | Γραμμές κώδικα |
|---|---|
| SQL query για dormant clusters | ~15 γραμμές Python |
| Semester heuristic | ~10 γραμμές Python |
| Move files + update catalog | ~30 γραμμές Python |
| Tauri notification trigger | ~20 γραμμές Rust/TS |
| UI banner + confirmation dialog | ~50 γραμμές React/TSX |
| **Σύνολο** | **~125 γραμμές** |

**Βαθμός δυσκολίας:** Χαμηλός. Απλή SQL query + file operations.

---

### Ιδέα 3: Project Completeness Score

**Περιγραφή:** Για κάθε cluster, εκτιμούμε αν "ολοκληρώθηκε": έχει όλους τους expected τύπους αρχείων; Λέκτρά (PDF) + assignments (DOCX) + κώδικας (Python/C) = ολοκληρωμένο course cluster.

#### (α) Υπάρχει παρόμοιο;

Μερικώς — σε εντελώς διαφορετικά contexts:
- **Heron Data** — "Document Completeness Score" για financial document submission: "evaluates submissions against a predefined checklist" — [herondata.io](https://www.herondata.io/glossary/document-completeness-score). Αλλά χρησιμοποιεί **predefined** requirements, όχι inferred.
- **e-Discovery** — Patent US8140494 για "collection transparency" — εξετάζει αν μια document collection είναι complete για legal discovery. Πάλι predefined rules.
- **arXiv content-based file type detection** — [arXiv:2101.08508](https://arxiv.org/abs/2101.08508) — κατηγοριοποιεί file types from content, αλλά δεν αξιολογεί completeness.

**Για personal file clusters:** Δεν βρέθηκε κανένα tool ή paper. Η ιδέα να **infer** τα expected file types από το cluster content ίδιο (χωρίς predefined rules) είναι genuinely novel.

#### (β) Αξίζει;

**Μέτρια** — ενδιαφέρον concept αλλά με αδύναμο signal. Το πρόβλημα: πώς ορίζουμε "complete" χωρίς domain knowledge; Μια λύση: χρησιμοποιούμε το LLM για να infer τι θα έπρεπε να έχει ένα cluster.

**Χρησιμότητα για φοιτητή:** Υψηλή — "EPL326 δεν έχει source code — μήπως ξέχασες να αποθηκεύσεις;" είναι genuine value.

#### (γ) Τι απαιτείται τεχνικά;

1. **File type distribution per cluster:** `SELECT extension, COUNT(*) FROM files WHERE cluster_id=? GROUP BY extension`
2. **LLM inference για expected types:**

```python
def infer_expected_types(cluster_name: str, file_types: list[str],
                          sample_content: str, ollama_client) -> list[str]:
    prompt = f"""Cluster name: {cluster_name}
Current file types: {file_types}
Sample content: {sample_content[:200]}

What file types would you expect in a complete collection for this project/course?
Reply with a JSON list of extensions, e.g. [".pdf", ".py", ".docx"]"""
    response = ollama_client.generate(model="qwen2.5:7b", prompt=prompt)
    return json.loads(response)

def completeness_score(present: set, expected: set) -> float:
    if not expected:
        return 1.0
    return len(present & expected) / len(expected)
```

3. **UI:** Μικρός indicator δίπλα στο cluster name: `EPL326 ●●●○ 75% complete (missing: .py files)`

#### (δ) Πολυπλοκότητα

| Κομμάτι | Γραμμές κώδικα |
|---|---|
| File type distribution query | ~10 γραμμές Python |
| LLM inference για expected types | ~25 γραμμές Python |
| Completeness score calculation | ~10 γραμμές Python |
| UI indicator component | ~30 γραμμές React/TSX |
| **Σύνολο** | **~75 γραμμές** |

**Βαθμός δυσκολίας:** Χαμηλός — αλλά accuracy εξαρτάται από LLM quality. Ο LLM μπορεί να κάνει λάθος για obscure clusters.

---

### Ιδέα 4: File Velocity

**Περιγραφή:** Track πόσο γρήγορα μεγαλώνει κάθε cluster (files/week). Rapidly growing = active project → υψηλότερη UI priority. Dormant = candidate for archive.

#### (α) Υπάρχει παρόμοιο;

Στον enterprise χώρο:
- **File server monitoring tools** (SolarWinds, Dynatrace) παρακολουθούν filesystem growth rate — [networkmanagementsoftware.com](https://www.networkmanagementsoftware.com/file-server-monitoring-tools/) — αλλά σε server/IT context, όχι personal files.
- **NGS research clusters:** Μελέτη σε HPC clusters έδειξε ότι τα NGS projects έχουν ρυθμό ανάπτυξης 9.8 αρχεία/μήνα vs 1.3 για μη-NGS — [PMC5928410](https://www.ncbi.nlm.nih.gov/pmc/articles/PMC5928410/). Αυτό δείχνει ότι το velocity είναι χρήσιμο signal για project activity detection.
- **Filesystem growth patent** (US6832236) — monitoring για production UNIX systems.

**Για personal file organizer:** Κανένα tool δεν κάνει velocity-based UI prioritization.

#### (β) Αξίζει;

**Ναι** — και είναι **πολύ εύκολο να υλοποιηθεί**. Το SQLite catalog ήδη έχει `scan_timestamp` για κάθε αρχείο. Η velocity είναι απλή SQL query. Το UI benefit είναι πραγματικό: τα πιο active projects εμφανίζονται πρώτα.

**Novelty:** Στο context του Curator (semantic clusters + velocity sorting) είναι novel. Σε isolation δεν είναι καινούριο concept.

#### (γ) Τι απαιτείται τεχνικά;

```python
def compute_cluster_velocity(db_path: str, days: int = 7) -> dict[str, float]:
    """Returns files/week per cluster."""
    conn = sqlite3.connect(db_path)
    cutoff = (datetime.now() - timedelta(days=days)).isoformat()
    rows = conn.execute("""
        SELECT cluster_id, COUNT(*) as new_files
        FROM files
        WHERE scan_timestamp > ?
        GROUP BY cluster_id
    """, (cutoff,)).fetchall()
    return {r[0]: r[1] * (7.0 / days) for r in rows}

# Sort clusters by velocity (descending) in approval UI
clusters_sorted = sorted(clusters, key=lambda c: velocity.get(c.id, 0), reverse=True)
```

**UI:** Badge `🔥 +12 files this week` δίπλα σε active clusters. Γκρι badge `😴 No activity 3mo` για dormant.

#### (δ) Πολυπλοκότητα

| Κομμάτι | Γραμμές κώδικα |
|---|---|
| SQL velocity query | ~15 γραμμές Python |
| Cluster sorting by velocity | ~5 γραμμές Python |
| UI badge component | ~20 γραμμές React/TSX |
| Integration με approval UI sort | ~10 γραμμές |
| **Σύνολο** | **~50 γραμμές** |

**Βαθμός δυσκολίας:** Πολύ χαμηλός. Αξία/κόστος ratio εξαιρετικό.

---

### Ιδέα 5: Seasonal Patterns (Temporal Priors)

**Περιγραφή:** Ανίχνευση ότι κάθε Σεπτέμβριο εμφανίζονται EPL course files, κάθε Μάιο exam files → χρήση ως prior για routing νέων αρχείων.

#### (α) Υπάρχει παρόμοιο;

Στη βιβλιογραφία:
- **Calendar-based periodicity detection** — [arXiv:1202.2926](https://arxiv.org/pdf/1202.2926) — ανιχνεύει annual/monthly/daily periodicities σε temporal patterns. Θεωρητική βάση υπάρχει.
- **Deep Temporal Graph Clustering (ICLR 2024)** — [arXiv:2305.10738](https://arxiv.org/pdf/2305.10738) — temporal relationships σε clustering tasks.
- **Cold start σε seasonal forecasting** — [arXiv:1710.08473](https://arxiv.org/pdf/1710.08473) — "Key to solving cold-start is leveraging repeated patterns over fixed periods and metadata." Αυτό είναι ακριβώς το πρόβλημά μας.
- **File PIM research:** [arXiv:2402.06421](https://arxiv.org/pdf/2402.06421) "What's in People's Digital File Collections?" — αναγνωρίζει temporal patterns στη χρήση files αλλά δεν τα εκμεταλλεύεται για routing.

**Για file organizer με academic calendar priors:** Δεν βρέθηκε κανένα ανάλογο σύστημα.

#### (β) Αξίζει;

**Μέτρια — με σημαντικό cold start πρόβλημα.**

**Πλεονεκτήματα:**
- Μετά από 1+ χρόνο data: "Σεπτέμβριος + filename mentions 'EPL' → πιθανό νέο course" είναι ισχυρό signal
- Χωρίς extra LLM calls — pure statistical prior

**Μειονεκτήματα:**
- Χρόνος 1 → δεν υπάρχουν patterns
- Ο χρήστης αλλάζει courses κάθε εξάμηνο → seasonal patterns υπάρχουν αλλά οι cluster names αλλάζουν
- Complexity/benefit ratio αμφίβολο για πρώτο sprint

**Σύσταση:** Υλοποίηση σε μεταγενέστερο sprint, μετά από 1+ χρόνο real usage data.

#### (γ) Τι απαιτείται τεχνικά;

```python
from collections import defaultdict

def build_seasonal_prior(db_path: str) -> dict[int, dict[str, float]]:
    """Returns month -> {cluster_pattern -> probability}."""
    conn = sqlite3.connect(db_path)
    rows = conn.execute("""
        SELECT strftime('%m', scan_timestamp) as month,
               cluster_name, COUNT(*) as count
        FROM files
        GROUP BY month, cluster_name
    """).fetchall()
    
    month_totals = defaultdict(int)
    month_clusters = defaultdict(lambda: defaultdict(int))
    for month, cluster, count in rows:
        month_totals[int(month)] += count
        month_clusters[int(month)][cluster] += count
    
    priors = {}
    for month, clusters in month_clusters.items():
        total = month_totals[month]
        priors[month] = {c: n/total for c, n in clusters.items()}
    return priors

def seasonal_routing_boost(filename: str, current_month: int,
                            priors: dict) -> dict[str, float]:
    """Returns cluster_name -> boost_score based on seasonal prior."""
    month_prior = priors.get(current_month, {})
    return month_prior  # Use as additive score to cosine similarity
```

#### (δ) Πολυπλοκότητα

| Κομμάτι | Γραμμές κώδικα |
|---|---|
| Seasonal prior computation | ~30 γραμμές Python |
| Prior integration σε routing | ~15 γραμμές Python |
| Cold start handling | ~10 γραμμές Python |
| **Σύνολο** | **~55 γραμμές** |

**Βαθμός δυσκολίας:** Χαμηλός τεχνικά, αλλά low ROI στο πρώτο χρόνο.

---

### Ιδέα 6: Semantic Neighborhood Alerts

**Περιγραφή:** Όταν φτάνει νέο αρχείο παρόμοιο με 3+ αρχεία που ο χρήστης δεν έχει ανοίξει ποτέ → alert "Αυτό μοιάζει με αρχεία που δεν έχεις χρησιμοποιήσει ποτέ. Τα χρειάζεσαι;"

#### (α) Υπάρχει παρόμοιο;

- **Proactive Information Retrieval via Screen Surveillance** (SIGIR 2017) — [dl.acm.org](https://dl.acm.org/doi/10.1145/3077136.3084151) — χτίζει model χρήσης για proactive retrieval, αλλά με screen surveillance (ανησυχητικό για privacy).
- **Six modes of proactive resource management** (CHI/NordiCHI 2004) — [dl.acm.org](https://dl.acm.org/doi/10.1145/1028014.1028022) — τυπολογία proactive behaviors: preparation, optimization, advising. Η "Semantic Neighborhood Alert" εμπίπτει στην κατηγορία "advising".
- **Patent US9910968** — "Automatic notifications for inadvertent file events" — notifications όταν διαγραφές αρχείων υπερβαίνουν threshold.
- **Patent US9208153** — "Filtering relevant event notifications in file sharing" — quantifies user interest + file similarity για notifications.

Η **ακριβής** ιδέα (cosine similarity + zero-use count → batch alert) δεν βρέθηκε.

#### (β) Αξίζει;

**Ναι** — με προσοχή στο UX. Η κύρια ανησυχία είναι notification fatigue. Η λύση: **batching** (μία φορά την εβδομάδα, όχι per-file).

**Τεχνική βάση:** Ήδη έχουμε:
- DenStream cosine similarities
- `kMDItemLastUsedDate` + `kMDItemUseCount` μέσω osxmetadata
- SQLite catalog

**Novelty:** Η combination "embedding similarity + zero-use signal + proactive declutter suggestion" σε local file organizer είναι genuinely novel.

#### (γ) Τι απαιτείται τεχνικά;

```python
def find_semantic_neighborhood_alerts(
    db_path: str,
    new_file_embedding: list[float],
    similarity_threshold: float = 0.7,
    min_similar_unused: int = 3,
    days_unused: int = 180
) -> list[dict]:
    """Find clusters with many unused files similar to new file."""
    conn = sqlite3.connect(db_path)
    
    # Get clusters where mean use count is 0 and files are old
    dormant_clusters = conn.execute("""
        SELECT cluster_id, COUNT(*) as unused_count
        FROM files
        WHERE use_count = 0 
          AND last_used_date < datetime('now', '-? days')
        GROUP BY cluster_id
        HAVING unused_count >= ?
    """, (days_unused, min_similar_unused)).fetchall()
    
    alerts = []
    for cluster_id, count in dormant_clusters:
        # Check cosine similarity with cluster centroid
        centroid = get_cluster_centroid(db_path, cluster_id)
        similarity = cosine_similarity(new_file_embedding, centroid)
        if similarity >= similarity_threshold:
            alerts.append({
                "cluster_id": cluster_id,
                "unused_count": count,
                "similarity": similarity
            })
    return alerts
```

**Batching:** Τρέχει 1x/εβδομάδα, όχι real-time. Αποθηκεύουμε pending alerts σε SQLite, εμφανίζουμε grouped notification.

#### (δ) Πολυπλοκότητα

| Κομμάτι | Γραμμές κώδικα |
|---|---|
| Neighborhood similarity query | ~35 γραμμές Python |
| Alert batching + SQLite storage | ~20 γραμμές Python |
| Tauri notification (grouped) | ~25 γραμμές Rust/TS |
| UI "Declutter suggestion" panel | ~60 γραμμές React/TSX |
| **Σύνολο** | **~140 γραμμές** |

**Βαθμός δυσκολίας:** Μέτριος — το UX design (non-intrusive) είναι πιο δύσκολο από το code.

---

### Ιδέα 7: Project Timeline Reconstruction

**Περιγραφή:** Από file metadata, ανακατασκευάζουμε το lifecycle ενός project: πότε ξεκίνησε, peak activity, τρέχουσα κατάσταση. Εμφάνιση ως sparkline/timeline.

#### (α) Υπάρχει παρόμοιο;

**Στο digital forensics domain:**
- **Automated timeline reconstruction** — [ScienceDirect](https://www.sciencedirect.com/science/article/pii/S174228761200031X) — αναλύει MAC timestamps (Modified/Accessed/Created) για forensic investigations. Χρησιμοποιεί exactly τα ίδια metadata που έχουμε.
- **SoK: Timeline event reconstruction** (2025) — [ScienceDirect](https://www.sciencedirect.com/science/article/pii/S266628172500071X) — comprehensive survey.
- **LLM-based forensic timeline analysis** — [arXiv:2505.03100](https://arxiv.org/pdf/2505.03100) — 2025, χρησιμοποιεί LLM για timeline analysis.

**Για UI:** Tools όπως Plaso, EnCase, Sleuth Kit κάνουν timeline visualization αλλά σε forensic context, όχι personal file organizer.

**React sparkline:** [react-sparklines](https://github.com/borisyankov/react-sparklines) — ελαφρύ, npm package, αξιόπιστο.

**Για personal file organizer context:** Δεν βρέθηκε ανάλογο. Η εφαρμογή της forensic timeline τεχνικής για project lifecycle visualization σε personal organizer είναι novel.

#### (β) Αξίζει;

**Ναι** — χαμηλό κόστος, υψηλή αισθητική αξία. Το sparkline per cluster στο Deep Reorganize UI δίνει άμεσα context χωρίς να απαιτεί user action.

**Χρήση:** "EPL326 — active Jan-May 2025, peak March (exam period), dormant since June" → άμεση κατανόηση.

#### (γ) Τι απαιτείται τεχνικά;

```python
def compute_cluster_timeline(
    db_path: str, cluster_id: str, bucket: str = 'week'
) -> list[dict]:
    """Returns time-bucketed file activity for sparkline."""
    conn = sqlite3.connect(db_path)
    
    bucket_format = {
        'day': '%Y-%m-%d',
        'week': '%Y-%W', 
        'month': '%Y-%m'
    }[bucket]
    
    rows = conn.execute(f"""
        SELECT strftime('{bucket_format}', creation_date) as period,
               COUNT(*) as file_count,
               SUM(CASE WHEN modification_date > creation_date 
                        THEN 1 ELSE 0 END) as modified_count
        FROM files
        WHERE cluster_id = ?
        GROUP BY period
        ORDER BY period
    """, (cluster_id,)).fetchall()
    
    return [{"period": r[0], "files": r[1], "edits": r[2]} for r in rows]
```

**React component:**
```tsx
import { Sparklines, SparklinesBars } from 'react-sparklines';

const ClusterTimeline = ({ clusterData }) => (
  <div className="cluster-timeline">
    <Sparklines data={clusterData.map(d => d.files)} width={100} height={20}>
      <SparklinesBars style={{ fill: "#4a9eff", opacity: 0.8 }} />
    </Sparklines>
    <span className="timeline-label">{clusterData[0]?.period} → {clusterData[clusterData.length-1]?.period}</span>
  </div>
);
```

#### (δ) Πολυπλοκότητα

| Κομμάτι | Γραμμές κώδικα |
|---|---|
| SQL timeline aggregation | ~25 γραμμές Python |
| React sparkline component | ~30 γραμμές TSX |
| Integration σε Deep Reorganize UI | ~20 γραμμές TSX |
| **Σύνολο** | **~75 γραμμές** |

**Βαθμός δυσκολίας:** Πολύ χαμηλός. Εξαιρετικό αξία/κόστος.

---

### Ιδέα 8: Learning from Rejection

**Περιγραφή:** Όταν ο χρήστης απορρίπτει proposed move (unchecks file στο approval UI), το καταγράφουμε ως negative signal — "αυτό το αρχείο ΔΕΝ ανήκει στο cluster X."

#### (α) Υπάρχει παρόμοιο;

**Στη βιβλιογραφία υπάρχει σχετική έρευνα:**

- **COBRAS: Interactive Clustering with Pairwise Queries** — [Springer](https://link.springer.com/chapter/10.1007/978-3-030-01768-2_29) — "answering no → cannot-link constraint." Exactly αυτό θέλουμε. COBRAS ενσωματώνει must-link/cannot-link constraints σε iterative clustering.
- **IPBC: Human-in-the-Loop Semi-Supervised Clustering** — [arXiv:2601.18828](https://arxiv.org/pdf/2601.18828) — feedback loop με "simple constraints such as must-link or cannot-link relationships." Δείχνει ότι "only a small number of interactive refinement steps can substantially improve cluster quality."
- **TINDER: Clustering with Reject Option** — [arXiv:1606.05896](https://arxiv.org/pdf/1606.05896) — analyst rejects a clustering → system returns different clustering "as different as possible from previous while still fitting data."
- **Learning from Negative User Feedback (Sequential Recommenders)** — [arXiv:2308.12256](https://arxiv.org/abs/2308.12256) — "important lever of user control... reduce similar recommendations."
- **SetFit contrastive learning** — [arXiv:2209.11055](https://arxiv.org/abs/2209.11055) — in-class = positive pairs (score 1), out-class = negative pairs (score 0). Το rejection signal μπορεί να τροφοδοτεί negative pairs.
- **Contrastive learning για clustering** — [arXiv:2203.12230](https://arxiv.org/pdf/2203.12230) — "Negative Selection by Clustering for Contrastive Learning."

**Συμπέρασμα:** Η θεωρητική βάση **υπάρχει** (cannot-link constraints, negative feedback clustering). Η **εφαρμογή** σε personal file organizer approval UI είναι novel.

#### (β) Αξίζει;

**Ναι** — είναι η πιο academically interesting ιδέα από τις 9. Transformative long-term: μετά από 50+ rejections, το σύστημα κατανοεί τις personal preferences του χρήστη.

**Δύο επίπεδα implementation:**

**Level 1 (Απλό):** Statistical: track rejections per (file_type, cluster) pair → αν >3 rejections, raise cosine threshold για αυτό το cluster.

**Level 2 (Advanced):** Cannot-link constraints: αύξηση cosine distance μεταξύ file embedding και rejected cluster centroid → update DenStream micro-cluster weights.

#### (γ) Τι απαιτείται τεχνικά;

```python
# SQLite table για rejection log
CREATE TABLE IF NOT EXISTS rejection_log (
    id INTEGER PRIMARY KEY,
    file_id TEXT,
    rejected_cluster_id TEXT,
    timestamp DATETIME DEFAULT CURRENT_TIMESTAMP,
    file_embedding BLOB,  -- store για future use
    rejection_reason TEXT  -- null or user-provided
);

# Level 1: Statistical threshold adjustment
def get_cluster_threshold_adjustment(
    db_path: str, cluster_id: str, base_threshold: float = 0.7
) -> float:
    conn = sqlite3.connect(db_path)
    rejection_count = conn.execute("""
        SELECT COUNT(*) FROM rejection_log
        WHERE rejected_cluster_id = ?
        AND timestamp > datetime('now', '-30 days')
    """, (cluster_id,)).fetchone()[0]
    
    # Raise threshold by 0.05 per 5 rejections, max +0.20
    adjustment = min(0.20, (rejection_count // 5) * 0.05)
    return base_threshold + adjustment

# Level 2: Embedding space adjustment (cannot-link)
def apply_cannot_link(
    file_embedding: np.ndarray,
    cluster_centroid: np.ndarray,
    push_factor: float = 0.1
) -> np.ndarray:
    """Push file embedding away from rejected cluster centroid."""
    direction = file_embedding - cluster_centroid
    direction = direction / (np.linalg.norm(direction) + 1e-8)
    return file_embedding + push_factor * direction
```

**SetFit integration:** Τα rejections γίνονται negative pairs στο contrastive training. Κάθε 20 νέα rejections → re-fine-tune SetFit classifier.

#### (δ) Πολυπλοκότητα

| Κομμάτι | Γραμμές κώδικα |
|---|---|
| Rejection log table + recording | ~20 γραμμές Python |
| Level 1: Threshold adjustment | ~20 γραμμές Python |
| Level 2: Cannot-link embedding | ~30 γραμμές Python |
| SetFit re-training pipeline | ~50 γραμμές Python |
| UI: track unchecked files | ~15 γραμμές React/TSX |
| **Σύνολο (Level 1)** | **~55 γραμμές** |
| **Σύνολο (Level 1 + 2)** | **~135 γραμμές** |

**Βαθμός δυσκολίας:** Level 1 = Χαμηλός, Level 2 = Μέτριος-Υψηλός.

---

### Ιδέα 9: Progressive Disclosure Approval UI

**Περιγραφή:** Στο approval UI, εμφανίζουμε πρώτα τα "surprising" moves (αρχεία που μετακινήθηκαν σημαντικά σημασιολογικά) → έπειτα τα routine. Τώρα sort by confidence. Το semantic distance είναι καλύτερο signal.

#### (α) Υπάρχει παρόμοιο;

**Progressive Disclosure σε UX:**
- Η έννοια είναι established — [Nielsen Norman Group](https://www.nngroup.com/videos/progressive-disclosure/), [IxDF](https://ixdf.org/literature/topics/progressive-disclosure) — αλλά αφορά hiding/showing UI complexity, όχι ordering review items.
- [arXiv:1811.02164](https://arxiv.org/pdf/1811.02164) — "Progressive Disclosure: Designing for Effective Transparency" — theoretical framework.

**Surprise-based ordering:**
- **Recommender systems:** "Limits to Surprise in Recommender Systems" — [arXiv:1807.03905](https://arxiv.org/pdf/1807.03905) — "normalised surprise" metric. Δείχνει ότι surprise είναι measurable και actionable quantity.
- **Anomaly-first review:** Δεν βρέθηκε ακριβώς — αλλά το concept είναι logical extension του anomaly detection.
- **Attention management survey** — [arXiv:1806.06771](https://arxiv.org/pdf/1806.06771) — "cognitive theories and ML techniques for managing interruptions." Supports prioritizing high-attention items first.

**Για file organization approval UI:** Δεν βρέθηκε ανάλογο. Η χρήση cosine distance (current location centroid → proposed location centroid) ως "surprise score" για review ordering είναι novel.

#### (β) Αξίζει;

**Ναι** — και είναι εξαιρετικά **εύκολο** να υλοποιηθεί. Η κύρια παρατήρηση: high-surprise moves (file που πήγε από "Personal" σε "EPL436-Algorithms") απαιτούν περισσότερη προσοχή από low-surprise moves (file που πήγε από "EPL326" σε "EPL326/Assignments"). Το sort by surprise ευθυγραμμίζεται με αυτό.

**Bonus:** Δίνει στον χρήστη multi-sort options: Confidence | Surprise | Cluster | Type — minimal code, significant UX improvement.

#### (γ) Τι απαιτείται τεχνικά;

```python
import numpy as np

def compute_surprise_score(
    file_current_cluster_centroid: np.ndarray,
    proposed_cluster_centroid: np.ndarray
) -> float:
    """Higher = more surprising move."""
    dot = np.dot(file_current_cluster_centroid, proposed_cluster_centroid)
    norm = (np.linalg.norm(file_current_cluster_centroid) *
            np.linalg.norm(proposed_cluster_centroid))
    cosine_sim = dot / (norm + 1e-8)
    return 1.0 - cosine_sim  # Distance = surprise

def sort_proposals_by_surprise(proposals: list[dict]) -> list[dict]:
    return sorted(proposals, key=lambda p: p['surprise_score'], reverse=True)
```

**React UI sort toggle:**
```tsx
type SortMode = 'confidence' | 'surprise' | 'cluster' | 'type';

const SortToggle = ({ mode, onChange }: { mode: SortMode; onChange: (m: SortMode) => void }) => (
  <div className="sort-toggle">
    <span>Sort by:</span>
    {(['confidence', 'surprise', 'cluster', 'type'] as SortMode[]).map(m => (
      <button key={m} className={mode === m ? 'active' : ''} onClick={() => onChange(m)}>
        {m.charAt(0).toUpperCase() + m.slice(1)}
      </button>
    ))}
  </div>
);
```

**Backend:** Ήδη έχουμε cosine distances από το clustering pipeline. Απλώς χρειαζόμαστε `current_cluster_centroid` per file (trivial — είναι στο DenStream state).

#### (δ) Πολυπλοκότητα

| Κομμάτι | Γραμμές κώδικα |
|---|---|
| Surprise score computation | ~15 γραμμές Python |
| Sort proposals by surprise | ~5 γραμμές Python |
| React sort toggle UI | ~25 γραμμές TSX |
| Integration σε approval pipeline | ~10 γραμμές |
| **Σύνολο** | **~55 γραμμές** |

**Βαθμός δυσκολίας:** Πολύ χαμηλός. Η καλύτερη αξία/κόστος ιδέα από τις 9.

---

## Πίνακας Πηγών

| Ιδέα | Βασικές Πηγές |
|---|---|
| Smart Rename | [ai-renamer](https://github.com/ozgrozer/ai-renamer) · [ollama-rename-files](https://github.com/octrow/ollama-rename-files) · [RenameClick](https://rename.click/) · [DEV Community](https://dev.to/hetianhe/why-file-renaming-is-still-a-hard-problem-and-how-ai-changes-it-57n8) · [notify-rs](https://docs.rs/notify) |
| Smart Archive | [Atlassian archive](https://confluence.atlassian.com/enterprise/archive-inactive-projects-1688929025.html) · [arXiv:2402.06421](https://arxiv.org/pdf/2402.06421) · [ShareGate lifecycle](https://sharegate.com/blog/build-a-microsoft-teams-lifecycle-management-plan-archiving-and-deleting-inactive-teams) |
| Completeness Score | [Heron Data](https://www.herondata.io/glossary/document-completeness-score) · [arXiv:2101.08508](https://arxiv.org/abs/2101.08508) · US8140494 patent |
| File Velocity | [PMC5928410](https://www.ncbi.nlm.nih.gov/pmc/articles/PMC5928410/) · [networkmanagementsoftware](https://www.networkmanagementsoftware.com/file-server-monitoring-tools/) |
| Seasonal Patterns | [arXiv:1202.2926](https://arxiv.org/pdf/1202.2926) · [arXiv:2305.10738](https://arxiv.org/pdf/2305.10738) · [arXiv:1710.08473](https://arxiv.org/pdf/1710.08473) · [arXiv:2402.06421](https://arxiv.org/pdf/2402.06421) |
| Semantic Alerts | [CHI/NordiCHI proactive](https://dl.acm.org/doi/10.1145/1028014.1028022) · [SIGIR proactive IR](https://dl.acm.org/doi/10.1145/3077136.3084151) · US9910968 patent |
| Timeline | [ScienceDirect forensic timeline](https://www.sciencedirect.com/science/article/pii/S174228761200031X) · [arXiv:2505.03100](https://arxiv.org/pdf/2505.03100) · [react-sparklines](https://github.com/borisyankov/react-sparklines) |
| Learning from Rejection | [COBRAS](https://link.springer.com/chapter/10.1007/978-3-030-01768-2_29) · [arXiv:2601.18828](https://arxiv.org/pdf/2601.18828) · [arXiv:1606.05896](https://arxiv.org/pdf/1606.05896) · [arXiv:2308.12256](https://arxiv.org/abs/2308.12256) · [arXiv:2209.11055](https://arxiv.org/abs/2209.11055) |
| Progressive Disclosure | [arXiv:1807.03905](https://arxiv.org/pdf/1807.03905) · [arXiv:1806.06771](https://arxiv.org/pdf/1806.06771) · [arXiv:1811.02164](https://arxiv.org/pdf/1811.02164) · [IxDF PD](https://ixdf.org/literature/topics/progressive-disclosure) |

---

## Τελικός Πίνακας Αξιολόγησης

| # | Ιδέα | Υπάρχει παρόμοιο; | Genuinely Novel στο Curator context; | Αξίζει; | Πολυπλοκότητα (γραμμές) | Sprint |
|---|---|---|---|---|---|---|
| 9 | Progressive Disclosure Approval UI | Μερικώς (surprise σε recommenders) | Ναι — cosine distance ως review ordering | ★★★★★ | ~55 | Sprint 1 |
| 4 | File Velocity | Ναι (enterprise monitoring) | Μέτρια — novel στο semantic cluster context | ★★★★★ | ~50 | Sprint 1 |
| 7 | Project Timeline Reconstruction | Ναι (forensics — διαφορετικό context) | Ναι — personal organizer application | ★★★★☆ | ~75 | Sprint 1 |
| 1 | Smart Rename Suggestions | Ναι (standalone tools) | Ναι — integration σε organizer workflow | ★★★★☆ | ~115 | Sprint 1 |
| 8 | Learning from Rejection | Ναι (COBRAS, cannot-link) | Ναι — εφαρμογή σε approval UI | ★★★★☆ | ~55–135 | Sprint 2 |
| 2 | Smart Archive | Ναι (enterprise, διαφορετικό) | Ναι — cluster-aware + academic calendar | ★★★★☆ | ~125 | Sprint 2 |
| 6 | Semantic Neighborhood Alerts | Μερικώς (patents, proactive computing) | Ναι — similarity + zero-use + local | ★★★☆☆ | ~140 | Sprint 2 |
| 3 | Project Completeness Score | Μερικώς (legal e-discovery) | Ναι — inferred (όχι predefined) requirements | ★★★☆☆ | ~75 | Sprint 3 |
| 5 | Seasonal Patterns | Ναι (χρονοσειρές, θεωρητικά) | Ναι — file routing με annual priors | ★★☆☆☆ | ~55 | Sprint 4+ |

### Σύνοψη Προτεραιοτήτων

**Sprint 1 (quick wins — υψηλό αξία/κόστος):**
- Ιδέα 9 (Progressive Disclosure) — ~55 γραμμές, immediate UX win
- Ιδέα 4 (File Velocity) — ~50 γραμμές, instant cluster prioritization
- Ιδέα 7 (Timeline) — ~75 γραμμές, visual delight χωρίς complexity
- Ιδέα 1 (Smart Rename) — ~115 γραμμές, tangible daily value

**Sprint 2 (medium effort, high academic/practical value):**
- Ιδέα 8 (Learning from Rejection) — ακαδημαϊκά ισχυρό, Level 1 είναι απλό
- Ιδέα 2 (Smart Archive) — σημαντικό για φοιτητή με semester cycles

**Sprint 3:**
- Ιδέα 6 (Semantic Alerts) — σωστό UX design είναι το δύσκολο κομμάτι
- Ιδέα 3 (Completeness Score) — useful αλλά LLM accuracy αβέβαιη

**Sprint 4+:**
- Ιδέα 5 (Seasonal Patterns) — απαιτεί 1+ χρόνο real data, low ROI αρχικά