# 20 — HippoCamp + FileGram Deep Analysis
_Date: 2026-05-30_

## HippoCamp (arXiv:2604.01221)

**NOT directly usable for Curator evaluation.**
- 581 QA pairs = question answering over personal files, NOT organization tasks
- No "move files to folders" tasks — only read/search/reason
- Dataset (42.4GB) likely gated — project page inaccessible, no public download found
- Can't plug Curator into it without defining our own org-quality ground truth

**Indirect value:** If dataset becomes available, realistic file corpus for clustering pipeline testing.

## FileGram (arXiv:2604.04901)
GitHub: https://github.com/Synvo-ai/FileGram

**NOT local-LLM compatible.** Hardcoded to Claude/Gemini/Cohere. No Ollama support.

**10 tasks (not 32):**
| ID | Name | Relevant to Curator? |
|---|---|---|
| T-05 | **Messy folder cleanup and reorganization** | ✅ YES |
| T-01–T-10 (others) | Understand/create/synthesize/iterate | ❌ No |

**T-05 structure** (our best overlap): messy folder with docs, emails, images, notes, calendar, chat logs → create subdirectories, move+rename files, write rationale. Measures: reading strategy, directory style, naming conventions, version/duplicate handling.

**FileGramBench measures:** user-profiling / behavioral memory (profile reconstruction, trace separation, drift detection). Not organization quality.

## Evaluation Strategy Implications

Neither paper gives us a ready benchmark. We need to build our own:
1. **Synthetic messy folders** with known ground-truth organization (we can generate with FileGramEngine if we have API keys)
2. **T-05 rubric** as human evaluation checklist (directory style, naming, version handling)
3. **20 Newsgroups / RCV1** for quantitative clustering metrics (NMI, ARI, Silhouette)

## 🚪 New Doors
1. **T-05 is adaptable as our evaluation template** — take FileGram's rubric (directory style, naming strategy, duplicate handling) and build our own "Curator Eval Harness" around it. ~2 days work. Could become a dataset contribution.
2. **FileGramEngine synthetic generation** — run with API keys to create realistic messy file snapshots (20 personas × T-05 = 20 messy folder scenarios). Free ground truth for automated evaluation.
3. **"Our own benchmark" is a contribution** — no standard benchmark exists for personal file ORGANIZATION (as distinct from retrieval). Creating one = dataset paper for SIGIR/ECIR short paper track.
4. **FileGram T-05 rubric gaps** — their rubric doesn't mention: uncertainty quantification, conformal prediction sets, per-file memory, or forgetting curves. We can publish a richer evaluation rubric as a research contribution.
