---
name: project-curator
description: "Curator app — what it is, current state, open questions, pending work"
metadata: 
  node_type: memory
  type: project
  originSessionId: 7dedeaac-40c6-4614-a5cf-e5a203916c6a
---

macOS file organizer app. Watches ~/Downloads/Νέες_Λήψεις, classifies files using local AI, moves them to organized folders.

**Stack:** Tauri 2 + React 19 + TypeScript (frontend) · Rust (backend commands) · Python (AI classifier at ~/Library/Scripts/ai_classifier.py) · Ollama (llava:7b for images, llama3.2:3b for text) · ChromaDB (semantic embeddings) · instructor library (structured LLM output)

**Python:** /opt/anaconda3/bin/python3 — has all packages (chromadb, markitdown, instructor, openai, fastembed). Homebrew Python does NOT have them.

**launchd plist:** ~/Library/LaunchAgents/com.haralambos.aiclassifier.plist — WatchPaths = ~/Downloads/Νέες_Λήψεις

**Config:** ~/Library/Scripts/curator_config.json

**Current classifier approach (KNOWN TO BE WRONG):** Hardcoded DESTINATIONS dict with predefined folder names. LLM is prompted with a fixed list of categories. This creates fundamental bias — LLM must pick from the list even when nothing fits. User identified this as the core flaw.

**New direction agreed (2026-05-21):** Dynamic discovery — no predefined categories. Files get embedded, clustered dynamically, LLM names the clusters. Keep old system running in parallel, build new one separately.

**New architecture decided:**
- Phase 1 (0–15 files): embed + LLM summary, hold flat, don't move files
- Phase 2 (15–50 files): agglomerative clustering → k-LLMmeans naming → propose folders to user
- Phase 3 (50+ files): DBSTREAM online clustering (river library), incremental updates
- Cold start: LLM generates pseudo-constraints for pairs (from "LLMs Enable Few-Shot Clustering" paper)
- Split/merge detection every 10 new files via silhouette score

**Key research findings (2026-05-21):**
- messy-folder-reorganizer-ai (GitHub): closest existing project, Rust+Ollama+agglomerative+LLM naming
- FLACON paper (MDPI 2025): 7.8× improvement, 6-dimensional flag system, nobody applied to personal files yet
- k-LLMmeans (arXiv 2502.09667): summaries as centroids — LLM-generated text IS the folder name, evolves over time
- DBSTREAM (river Python lib): online clustering, creates new micro-clusters as files arrive
- Generative clustering (AAAI 2025): KL divergence on LLM distributions — unexplored for files, most innovative

**Pending fixes in CURRENT system (still need doing):**
- sort_file in Rust (Review page manual sort) does NOT save tags to undo log — so manually sorted files show no tags in PreviewPanel

**What was done this session (2026-05-21):**
- Switched ollama-instructor → instructor library (better retry/validation)
- Added garbled Unicode sanitizer in reasoning text
- Lowered CONF_AUTO 0.82→0.78, CONF_MEDIUM 0.60→0.50
- Fixed reembed_all: images no longer re-run vision LLM
- Settings: added embed_model, conf_auto, conf_medium fields
- Added Deep Scan page (SHA-256 duplicate detection across Downloads)
- Added scan_deep + delete_file Rust commands
- Removed dead onUndo2 code from App.tsx
