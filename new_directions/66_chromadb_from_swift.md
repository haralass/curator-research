# ChromaDB from Swift — Vector Search Integration
**File:** 66 | **Date:** 2026-06-01 | **Status:** Research Complete

## Summary / Recommendation
There is no native Swift ChromaDB client. The practical approach is to **run ChromaDB inside the Python sidecar** and call it via the sidecar's FastAPI interface — Swift never speaks to ChromaDB directly. The sidecar already exists for ML inference; adding vector search there is zero additional architecture. Consider **FAISS via Python** as a simpler self-contained alternative to ChromaDB.

## Prior Art & Evidence

### ChromaDB HTTP API
- ChromaDB exposes a REST API on localhost (default port 8000) when run as a server
- Any HTTP client can call it — Swift's URLSession works
- But: running ChromaDB as a *separate* server process alongside the Python sidecar adds operational complexity (two processes to manage, port conflicts, startup ordering)

### No Swift ChromaDB client
- GitHub search: no maintained Swift/SwiftPM ChromaDB client as of 2026
- Existing clients: Python (official), JavaScript/TypeScript (official), Rust (community), Go (community)
- Swift community client would need to implement the HTTP API — feasible but unnecessary given sidecar architecture

### Python sidecar as bridge
- Sidecar already runs for embeddings (sentence-transformers, etc.)
- Add `chromadb` Python package to sidecar
- Expose `/vector/search`, `/vector/add`, `/vector/delete` endpoints via FastAPI
- Swift calls these endpoints — clean separation

### FAISS as alternative
- Facebook AI Similarity Search — C++ library with Python bindings
- `faiss-cpu` pip package, no server needed, embedded in process
- For Curator's scale (tens of thousands of files), FAISS flat index fits in RAM (~4MB for 50k files at 384-dim float32)
- No persistence out of the box — serialize index to disk manually (numpy save/load)
- Simpler operationally than ChromaDB

### Qdrant alternative
- Has official REST API + good documentation
- Heavier than FAISS but lighter than a full ChromaDB deployment
- Docker-based or embedded mode (Rust binary)

## Tradeoffs

| Option | Complexity | Persistence | Swift integration | Scale |
|--------|-----------|-------------|-------------------|-------|
| ChromaDB via sidecar | Low | ✅ built-in | Via FastAPI | ✅ |
| FAISS via sidecar | Very Low | Manual | Via FastAPI | ✅ for <1M files |
| ChromaDB as separate process | Medium | ✅ | URLSession direct | ✅ |
| Qdrant embedded | Medium | ✅ | Via FastAPI or direct HTTP | ✅ |
| CoreML + custom ANN | High | Manual | Native | ✅ |

## Decision for Curator
**FAISS via Python sidecar for v1; upgrade to ChromaDB if persistence/filtering needs grow.**

Rationale: Curator's initial scale is personal file collections (10k–100k files). FAISS handles this trivially. Avoid the operational overhead of ChromaDB until needed. The sidecar exposes:

```python
# sidecar/vector_store.py
import faiss, numpy as np

class VectorStore:
    def __init__(self, dim=384):
        self.index = faiss.IndexFlatIP(dim)  # inner product (cosine if normalized)
        self.id_map = {}  # faiss_id -> file_path
    
    def add(self, embeddings, file_paths):
        # normalize for cosine similarity
        faiss.normalize_L2(embeddings)
        start_id = self.index.ntotal
        self.index.add(embeddings)
        for i, path in enumerate(file_paths):
            self.id_map[start_id + i] = path
    
    def search(self, query_embedding, k=10):
        faiss.normalize_L2(query_embedding)
        distances, indices = self.index.search(query_embedding, k)
        return [(self.id_map[i], float(d)) for i, d in zip(indices[0], distances[0]) if i != -1]
```

## Sources
- https://docs.trychroma.com/reference/python-client (HTTP API reference)
- https://github.com/facebookresearch/faiss
- https://python.langchain.com/docs/integrations/vectorstores/faiss
- https://qdrant.tech/documentation/guides/installation/
