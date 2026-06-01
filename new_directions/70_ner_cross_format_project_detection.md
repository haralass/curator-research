# NER Cross-Format Project Detection
**File:** 70 | **Date:** 2026-06-01 | **Status:** Research Complete | **Priority:** HIGH

## Summary
Named Entity Recognition across heterogeneous file formats (PDFs, DOCX, code, images with text) can extract project names, person names, dates, and org names that serve as cross-format grouping signals stronger than path or embedding similarity alone. Curator should run a lightweight NER pass on file contents during indexing and use shared entities as high-confidence edges in the file graph. This directly addresses the hardcoded-categories problem.

## Key Findings
- Modern NER: BERT-based models (bert-base-NER) reach 95%+ F1 on standard corpora; domain-specific fine-tunes reach 98.5%+
- LLM-based NER adds self-verification to reduce hallucinations on ambiguous entity spans
- Cross-format challenge: OCR required for scanned PDFs/images; visual document understanding (LayoutLM, Donut) handles form-like layouts
- Key entity types for project detection: PROJECT_NAME, PERSON, ORG, DATE, PRODUCT — these five span the vast majority of cross-file linkages on a personal computer
- Multimodal NER (images + text simultaneously) is now accessible via vision-language models — relevant for photos with embedded text or presentation screenshots
- **Apple NaturalLanguage framework covers basic NER natively on macOS without network calls** — `NLTagger` with `.nameType` scheme, zero dependencies

## Relevant Papers / Prior Art
- "A Review of Named Entity Recognition: From Learning Methods to Modelling Paradigms and Tasks" — Springer AI Review 2025
- "Weakly Supervised Cross-Lingual NER via Annotation and Representation Projection" — arxiv 1707.02483
- Encord NER guide — practitioner overview of current SOTA
- Apple NaturalLanguage framework — on-device NER for Swift integration

## Applicability to Curator
Very High. Running Apple's NaturalLanguage `NLTagger` on file text content gives zero-network-call entity extraction natively. Files sharing 2+ named entities (same person name + same project name) get a strong grouping signal. This is additive to embedding similarity and more interpretable: "these 4 files all mention 'Project Helios' and 'Anna Kovacs'."

Implementation sketch:
```swift
let tagger = NLTagger(tagSchemes: [.nameType])
tagger.string = fileText
tagger.enumerateTags(in: fileText.startIndex..<fileText.endIndex,
                     unit: .word, scheme: .nameType) { tag, range in
    if let tag = tag { entities.append((tag, String(fileText[range]))) }
    return true
}
```

For Python sidecar: spaCy `en_core_web_sm` for richer NER with entity linking.

## Open Questions
- How to handle entity disambiguation (common names like "Apple" = company vs fruit)?
- Should entity co-occurrence edges be stored in the SQLite graph or as ChromaDB metadata?
- Performance budget: NER on a 500-file corpus at index time — is it within 10-second startup tolerance?

## Sources
- https://link.springer.com/article/10.1007/s10462-025-11321-8
- https://developer.apple.com/documentation/naturallanguage/nltagger
- https://arxiv.org/pdf/1707.02483
- https://spacy.io/models/en
