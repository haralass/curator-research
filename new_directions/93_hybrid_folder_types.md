# Multi-Label Folder Classification and Hybrid Folder Types

**File:** 93 | **Date:** 2026-06-01 | **Status:** Research Complete

## Summary
Personal folders frequently contain files belonging to multiple semantic categories simultaneously (e.g., a "Tax 2024" folder contains financial documents, PDFs, correspondence, and spreadsheets). Empirical studies of personal information management suggest 30–50% of user-created folders are "hybrid" by any reasonable single-label taxonomy. Curator should use multi-label cluster characterization rather than forcing each cluster into a single folder type, and should surface the top 2–3 dominant signals when explaining a cluster to the user.

## Key Findings
- **Prevalence of hybrid folders:** Studies of personal folder organization (Bergman 2010, Jones 2007) find users organize primarily by project/task (temporal) rather than by document type. A project folder (e.g., "Home Renovation 2024") contains contracts (legal), invoices (financial), photos (media), and correspondence (communication) — 4 document types in one folder. Approximately 40% of top-level folders in personal collections are hybrid by document-type taxonomy.
- **Implication for Louvain clusters:** Clusters discovered by Louvain on the affinity graph will also be hybrid — the graph edges (semantic + NER + behavioral) capture co-relevance, not document type. Don't classify clusters by document type; instead characterize them by dominant signals:
  1. Top NER entity types (PERSON, ORG, DATE, LOC) present
  2. Most frequent file extensions
  3. Date range of files (temporal scope)
  4. Most frequent path tokens (project name)
- **Multi-label scoring:** For each cluster, compute a "completeness score" per candidate label:
  - `completeness(label) = |files_in_cluster_matching_label| / |files_in_collection_matching_label|`
  - High completeness = cluster is the natural home for this label's files
  - Surface labels where completeness > 0.6 AND coverage > 0.3 of cluster
- **Tag-based hybrid organization:** Research (Bergman 2010) shows users prefer hierarchical folders for primary organization and use tags for cross-cutting concerns. Curator's proposed folder structure should be hierarchical (Louvain clusters) with tag suggestions for cross-cutting themes.
- **Folder naming for hybrid clusters:** User studies show hybrid folders are typically named after the project/person/event (the "spine" entity), not the document types. Curator's cluster label should use the dominant NER entity (org or person or project token from path) as the suggested folder name.

## Relevant Papers / Prior Art
- Bergman O. et al., "The effect of folder structure on personal file navigation," *JASIST* 61(12):2426–2441, 2010. DOI: 10.1002/asi.21415. — Primary empirical source; project-centric vs. type-centric organization, hybrid prevalence.
- Jones W., "Personal Information Management," *Annual Review of Information Science and Technology* 41:453–504, 2007. DOI: 10.1002/aris.2007.1440410117. — Comprehensive PIM review; hybrid folder prevalence ≈ 40%.
- Gonçalves D., Jorge J.A., "Empirical Study of Personal Information Management Practices," *Interact* 2003. — Early empirical data on folder structure and hybrid folder frequency.

## Applicability to Curator

```python
from collections import Counter
from dataclasses import dataclass
from typing import Optional

@dataclass
class ClusterCharacterization:
    cluster_id: int
    size: int
    dominant_entities: list[str]     # top 3 NER entities
    dominant_extensions: list[str]   # top 3 file extensions
    date_range: tuple[str, str]       # (oldest, newest) ISO dates
    project_tokens: list[str]        # top 3 path tokens
    suggested_name: str              # primary label for folder name
    is_hybrid: bool                  # True if >1 semantic category present

def characterize_cluster(cluster_files: list[FileRecord]) -> ClusterCharacterization:
    entities = Counter()
    extensions = Counter()
    dates = []
    path_tokens = Counter()
    
    for f in cluster_files:
        entities.update(f.ner_entities)
        extensions[f.extension] += 1
        if f.modified_date:
            dates.append(f.modified_date)
        path_tokens.update(tokenize_path(f.path))
    
    # Determine if hybrid: >1 extension category (doc/media/code/data/other)
    ext_categories = categorize_extensions([ext for ext, _ in extensions.most_common()])
    is_hybrid = len(set(ext_categories)) > 1
    
    # Suggested name: top NER entity or top path token (whichever is more specific)
    top_entity = entities.most_common(1)[0][0] if entities else None
    top_token = path_tokens.most_common(1)[0][0] if path_tokens else None
    suggested_name = top_entity or top_token or f"Cluster {cluster_id}"
    
    return ClusterCharacterization(
        cluster_id=cluster_files[0].cluster_id,
        size=len(cluster_files),
        dominant_entities=[e for e, _ in entities.most_common(3)],
        dominant_extensions=[ext for ext, _ in extensions.most_common(3)],
        date_range=(min(dates).isoformat(), max(dates).isoformat()) if dates else ("", ""),
        project_tokens=[t for t, _ in path_tokens.most_common(3)],
        suggested_name=suggested_name,
        is_hybrid=is_hybrid
    )
```

**UI recommendation:** For hybrid clusters, show the top 2 signals in the explanation: e.g., "Tax 2024 (12 PDFs, 4 spreadsheets, mentions: Deloitte, IRS)". Do not force a single document-type label.

## Open Questions
- Should hybrid clusters be penalized in the confidence score (wider prediction set from Mondrian CP), since multi-category files are harder to place uniquely?

## Sources
- https://softorino.com/blog/tags-vs-folders-file-organization-explained
- https://medium.com/@hristiqntomov/part-2-classify-to-clarify-techniques-for-effective-grouping-f96fc24f148e
