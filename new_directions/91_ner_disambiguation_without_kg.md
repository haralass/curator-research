# NER Disambiguation Without a Knowledge Graph

**File:** 91 | **Date:** 2026-06-01 | **Status:** Research Complete

## Summary
For Curator's personal-file NER pipeline (Apple NaturalLanguage in Swift + spaCy `en_core_web_sm` in Python), entity disambiguation without a knowledge graph is achievable through three practical techniques: (1) frequency filtering to treat low-count entities as noise, (2) contextual clustering of entity surface forms to merge aliases, and (3) per-user entity learning from user feedback signals. A knowledge graph (Wikidata, etc.) would overfit to public entities and misidentify private proper names (e.g., a person named "John" in files is not necessarily John Lennon).

## Key Findings
- **Apple NaturalLanguage disambiguation:** Apple's `NLTagger` with `.personalName`, `.organizationName`, `.placeName` types provides entity detection but no disambiguation or entity linking. No built-in KB linkage. Disambiguation must be done in application code.
- **spaCy EntityLinker:** spaCy's `EntityLinker` component requires a `KnowledgeBase` object. Building a KB from Wikidata or custom data is possible but heavyweight for a local Mac app. For personal files, a Wikidata KB would produce false positives (linking "Amazon" in a personal receipt to Amazon.com rather than treating it as a merchant name in user context).
- **Frequency filtering (recommended for Curator):** Count entity surface form occurrences across all files. Entities appearing fewer than 3 times are likely noise or hapax legomena; entities appearing in > 50% of files are stop-entities (too common to be discriminative). Filter both ends.
  - Threshold: keep entities with 3 ≤ count ≤ 0.5 · n_files
- **Alias clustering (recommended):** Cluster entity surface forms by string similarity (edit distance ≤ 2 or shared 3-gram Jaccard > 0.7) to merge aliases. Example: "J. Smith", "John Smith", "J Smith" → same entity cluster.
- **spaCyFishing:** spaCy wrapper for the entity-fishing tool that links to Wikidata. Overkill for personal files; adds 500 MB model download. Not recommended.
- **NICE algorithm (arXiv:2210.06164):** NER-enhanced iterative entity disambiguation using coherence and semantic similarity. Applicable if Curator gains a graph of files — entity coherence across co-clustered files can serve as disambiguation signal.
- **User-specific entity learning:** The most effective approach for personal files. After user approves a folder assignment, extract entities from that folder's files and add them to a per-user entity frequency table. Over time, high-frequency entities in a folder become strong cluster features.
- **Privacy:** All entity processing must remain on-device. No entity data should be sent to remote APIs.

## Relevant Papers / Prior Art
- arXiv:2210.06164, "Focusing on Context is NICE: Improving Overshadowed Entity Disambiguation." — Iterative coherence-based disambiguation; applicable to co-clustered files.
- spaCy EntityLinker API documentation: https://spacy.io/api/entitylinker. — Official reference.
- Mihalcea R., Csomai A., "Wikify!: Linking documents to encyclopedic knowledge," *CIKM* 2007. — Classic entity linking; establishes that prior probability of entity surface form is the strongest single disambiguation feature (usable without a KB via frequency counts).

## Applicability to Curator

```python
# Python sidecar: entity processing pipeline
from collections import Counter
import re
from difflib import SequenceMatcher

class EntityDisambiguator:
    def __init__(self, min_count=3, max_freq_ratio=0.5):
        self.min_count = min_count
        self.max_freq_ratio = max_freq_ratio
        self.entity_counts: Counter = Counter()
        self.alias_map: dict[str, str] = {}  # surface_form → canonical_form

    def update(self, entities: list[str], total_files: int):
        """Update entity counts from a batch of extracted entities."""
        self.entity_counts.update(entities)
        self._rebuild_alias_map()

    def _rebuild_alias_map(self):
        """Cluster entity surface forms by edit distance."""
        forms = list(self.entity_counts.keys())
        canonical = {}
        for form in sorted(forms, key=lambda x: -self.entity_counts[x]):
            # Check if this form is an alias of an existing canonical
            found = False
            for canon in canonical:
                sim = SequenceMatcher(None, form.lower(), canon.lower()).ratio()
                if sim > 0.85:
                    canonical[canon].append(form)
                    self.alias_map[form] = canon
                    found = True
                    break
            if not found:
                canonical[form] = [form]
                self.alias_map[form] = form

    def filter_entities(self, entities: list[str], total_files: int) -> list[str]:
        """Return discriminative entities only."""
        result = []
        for e in entities:
            canon = self.alias_map.get(e, e)
            count = self.entity_counts[canon]
            if self.min_count <= count <= self.max_freq_ratio * total_files:
                result.append(canon)
        return result

    def entity_cooccurrence_matrix(self, file_entities: list[list[str]], n_files: int) -> np.ndarray:
        """Build NER co-occurrence affinity matrix for SNF."""
        from sklearn.preprocessing import normalize
        from sklearn.feature_extraction.text import CountVectorizer
        
        docs = [' '.join(self.filter_entities(ents, n_files)) for ents in file_entities]
        vec = CountVectorizer(binary=True)
        X = vec.fit_transform(docs).toarray().astype(np.float32)
        # Jaccard similarity
        intersection = X @ X.T
        row_sums = X.sum(axis=1, keepdims=True)
        union = row_sums + row_sums.T - intersection
        S_ner = np.where(union > 0, intersection / union, 0.0)
        return S_ner
```

**Integration:** Swift NLTagger extracts raw entity strings → sent to Python sidecar → `EntityDisambiguator.filter_entities()` → `entity_cooccurrence_matrix()` → S_ner fed to SNF.

## Open Questions
- Should entity types (PERSON, ORG, DATE, LOC) be kept separate as distinct NER modalities in SNF, rather than merged into a single co-occurrence matrix? (Likely yes for DATE entities, which are temporal context signals distinct from social graph entities.)

## Sources
- https://spacy.io/api/entitylinker
- https://arxiv.org/pdf/2210.06164
- https://github.com/Lucaterre/spacyfishing
- https://kristinelpetrosyan.medium.com/ner-and-ned-with-spacy-dd847800b7d9
