# Personal Knowledge Graph από File Behavior
**Compiled:** May 2026 | **Relevance:** Typed Knowledge Graph του Curator

---

## Κεντρικό Εύρημα

**Κανένα published system δεν συνδυάζει:** file behavioral signals + LLM classification + cluster-to-KG mapping + temporal evolution για personal file organizer. **Curator θα είναι ο πρώτος.**

PKM market: $2.45B (2024), 16.3% CAGR. AI-driven segment: **$11.24B** το 2026.

---

## State of the Art — PKGs

**Canonical definition (Balog & Kenter, ICTIR 2019):** PKG supports 3 functions:
1. **Population** — extracting structured knowledge from personal data
2. **Representation & Management** — maintaining over time
3. **Utilization** — delivering personalized services

**Most comprehensive survey:** ["An Ecosystem for Personal Knowledge Graphs"](https://arxiv.org/abs/2304.09572) (arXiv:2304.09572, 2023) — PKG ως hub connecting files, calendars, emails, sensors → downstream apps.

---

## KG Construction από Documents (2024-2026)

### LLM Pipeline (Dominant Paradigm)

**Stage 1:** LLM extracts `(subject, predicate, object)` triples από document chunks

**Stage 2:** Dynamic schema induction — **AutoSchemaKG** (arXiv:2505.23628):
- Δεν χρειάζεται predefined ontology
- Automatically induces 3 node types: **Entities** (named objects), **Events** (occurrences), **Concepts** (abstract categories)
- Scale: 900M nodes, 5.9B edges

**Stage 3:** Entity resolution — merge "John Smith" + "J. Smith" με embedding similarity

### Key Papers
| Paper | Key Insight |
|---|---|
| **AutoSchemaKG** — arXiv:2505.23628 | Dynamic schema induction, no expert ontology needed |
| **RAKG** — arXiv:2504.09823 | Pre-extracted entities ως RAG queries → better long-doc KG |
| **LightRAG** (EMNLP 2025) | 6000x cheaper than GraphRAG, Ollama support |
| **GraphRAG** (Microsoft) | Reference architecture, Leiden community detection |

### Best Local Tools
- **[LightRAG](https://github.com/hkuds/lightrag)** — Ollama support, 30% faster than naive RAG → **recommended για Curator**
- **[GraphRAG-Local-UI](https://github.com/severian42/GraphRAG-Local-UI)** — Full UI με indexing/visualization/querying
- **[Neo4j LLM Graph Builder](https://github.com/neo4j-labs/llm-graph-builder)** — Upload PDFs → KG, Ollama support

---

## Behavioral Signals → KG Edges

### Co-Access Clusters
Files που ανοίγονται μαζί σε μία session → implicit project/topic boundaries. Same signal as co-citation networks.

**Temporal Knowledge Graphs (ACM WSDM 2024):** Access frequency + recency encoded ως edge weights:
```
score = f(access_count, recency_decay)
```

### FileGram's 3 Memory Channels (adopt this architecture)
| Channel | Captures | Technical Form |
|---|---|---|
| **Procedural** | Behavioral fingerprint (browsing ratios, edit frequency) | 17-dim feature vector |
| **Semantic** | Content meaning (file type distribution, content embeddings) | 1024-dim embeddings |
| **Episodic** | Temporal sequences και recurrent patterns | LLM-segmented episodes |

---

## Cluster → KG Node Mapping

### Neo4j Native HDBSCAN Integration
```cypher
// Cluster nodes
CREATE (c:Cluster {id: $id, stability: $stability, birth_lambda: $birth_lambda})

// File-to-Cluster edges
MATCH (f:File {id: $file_id}), (c:Cluster {id: $cluster_id})
CREATE (f)-[:IN_CLUSTER {membership_prob: $prob, assigned_at: datetime()}]->(c)
```

### Key Design Decision: Cluster ≠ Topic
```
(Cluster:c1 {stability: 0.87}) -[:INSTANTIATES]-> (Topic:n1 {label: "Machine Learning"})
```
- **Cluster** = empirical (embedding geometry)
- **Topic** = semantic (content meaning)
- Drift ανεξάρτητα → store separately

### Bridge Nodes (InfraNodus)
Betweenness centrality → "bridge" nodes = concepts που συνδέουν διαφορετικά topic clusters. **Highest-value nodes** στο KG.

---

## Temporal Knowledge Graph

### Time-Stamped Triples
`(subject, predicate, object, [t_start, t_end])`

Όταν classification αλλάζει → old edge invalidated (δεν διαγράφεται), νέα edge opens. Enables retrospective queries.

**TFCE Framework (2025):** Temporally Embedded Decoder με version history ανά entity. Για Curator: κάθε re-clustering = new version.

---

## KG Retrieval για Local LLMs

**PersonalAI benchmark (arXiv:2506.17001)** — Systematic comparison για 7B models (Ollama tier):

| Strategy | Accuracy | Speed | Best For |
|---|---|---|---|
| **BeamSearch** | Highest | Slowest (6.59 min) | Deep discovery |
| **WaterCircles** | Good | Fastest (0.30 min) | Interactive browsing |

**Storage:** Qdrant > Milvus (6x faster per PersonalAI benchmarks)

**Hybrid retrieval (recommended):**
1. BM25 → exact term matches
2. Vector embeddings → semantic similarity
3. Graph traversal → multi-hop relational

---

## Entity Extraction (Local)

| Tool | What |
|---|---|
| **NuExtract** (Ollama library) | phi-3-mini fine-tuned για structured extraction — best local choice |
| **LangExtract** (Google) | Few-shot, Ollama support, source grounding |
| **LLM Graph Builder** (Neo4j) | PDF/DOC/TXT → Neo4j, Ollama compatible |

---

## Node & Edge Schema για Curator

### Nodes
```
:File       {id, path, name, ext, size, created_at, modified_at, last_accessed_at, access_count}
:Topic      {id, label, confidence, source: ["inferred"|"user"|"llm"]}
:Concept    {id, label, type: ["technology"|"methodology"|"domain"]}
:Person     {id, name, canonical_name}
:Project    {id, name, status: ["active"|"archived"|"inferred"]}
:TimePeriod {id, label, start_date, end_date}
:Cluster    {id, stability, size, birth_lambda, created_at}
```

### Edges
```
(:File)-[:CLASSIFIED_AS {confidence, model, timestamp}]->(:Topic)
(:File)-[:MENTIONS {count, first_seen}]->(:Person)
(:File)-[:BELONGS_TO {confidence}]->(:Project)
(:File)-[:ANCHORED_IN]->(:TimePeriod)
(:File)-[:IN_CLUSTER {membership_prob, assigned_at}]->(:Cluster)
(:Cluster)-[:INSTANTIATES {confidence}]->(:Topic)
(:File)-[:CO_ACCESSED_WITH {count, last_co_access}]->(:File)
(:Topic)-[:RELATED_TO {weight}]->(:Topic)
```

---

## Construction Pipeline (Phased)

**Phase 1 — Structural (no LLM):**
- Index files → `:File` nodes
- HDBSCAN → `:Cluster` nodes + `:IN_CLUSTER` edges
- Date histograms → `:TimePeriod` nodes
- Start tracking `CO_ACCESSED_WITH` από FSEvents

**Phase 2 — Semantic (local LLM):**
- NuExtract → `:Person`, `:Project`, `:Concept`
- Topic classification → `:CLASSIFIED_AS` edges
- AutoSchemaKG-style schema induction → νέα entity types
- Link `:Cluster` → `:Topic` via `:INSTANTIATES`

**Phase 3 — Behavioral (ongoing):**
- Update `access_count` on every open
- Increment `CO_ACCESSED_WITH` (30-min session window)
- Periodic HDBSCAN re-run, cluster drift detection

**Phase 4 — Temporal KG:**
- `[valid_from, valid_to]` σε όλα τα classification edges
- Re-classification → close old edge, open new

---

## Recommended Stack

```
Graph Store:  Kuzu (embedded, in-process, no server — best για desktop app)
              ή Neo4j Community Edition (richer tooling)
Vector Store: Qdrant (local mode, embedded) — 6x faster
LLM Extract:  Ollama llama3.2:3b + NuExtract
Embeddings:   nomic-embed-text
KG Retrieval: LightRAG (Ollama-native, 6000x cheaper than GraphRAG)
```

---

## Useful Cypher Queries

```cypher
// Files related to a person, by recency
MATCH (p:Person {name: "Alice"})<-[:MENTIONS]-(f:File)
RETURN f ORDER BY f.last_accessed_at DESC LIMIT 10

// Bridge topics (connect different clusters)
MATCH (c1:Cluster)-[:INSTANTIATES]->(t:Topic)<-[:INSTANTIATES]-(c2:Cluster)
WHERE c1 <> c2
RETURN t, c1, c2

// Implicit project detection (co-access)
MATCH (f:File {id: $seed})-[r:CO_ACCESSED_WITH]->(neighbor:File)
WHERE r.co_access_count > 3
RETURN neighbor ORDER BY r.co_access_count DESC
```

---

## Open Problems

1. **Δεν υπάρχει evaluation benchmark** για PKG quality από file collections
2. **Temporal drift** — πώς maintain KG edge history όταν topic classification αλλάζει
3. **Cold-start** — bootstrap από content μόνο, μετά enrich με behavioral signals
4. **Schema drift** — καθώς LLM extracts νέα entity types, schema expands

---

## Sources
- [PKG Ecosystem Survey — arXiv 2304.09572](https://arxiv.org/abs/2304.09572)
- [AutoSchemaKG — arXiv 2505.23628](https://arxiv.org/html/2505.23628v1)
- [PersonalAI — arXiv 2506.17001](https://arxiv.org/html/2506.17001v6)
- [LightRAG](https://github.com/hkuds/lightrag)
- [GraphRAG-Local-UI](https://github.com/severian42/GraphRAG-Local-UI)
- [FileGram — arXiv 2604.04901](https://arxiv.org/abs/2604.04901)
- [Neo4j HDBSCAN](https://neo4j.com/docs/graph-data-science/current/algorithms/hdbscan/)
- [NuExtract](https://ollama.com/library/nuextract)
- [OntoFM — arXiv 1307.2018](https://arxiv.org/abs/1307.2018)
- [Kuzu](https://kuzudb.com/)
