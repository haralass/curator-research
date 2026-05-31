# DataMapPlot as a Visual File Map Feature in Curator
*Research memo — 2026-05-31*

---

## 1. Executive Summary

DataMapPlot (TutteInstitute / Leland McInnes) is a mature Python library that wraps DeckGL to produce interactive 2D scatter plots from UMAP + HDBSCAN outputs. Its `offline_mode=True` flag produces a single, fully self-contained HTML file. Tauri 2 can load that file in a second webview window, inject a thin initialization script to bridge DeckGL click events into Tauri's IPC, and fire `revealItemInDir()` via the official Opener plugin — revealing the clicked file in macOS Finder. The Python computation can run as a PyInstaller sidecar, with progress streamed over stdout. The concept is not entirely novel (Liquid 2D Scatter Space, 2005; Galaxies document maps; semantic treemaps), but a semantic UMAP-based 2D landscape of a personal file system in a polished native desktop app has not been shipped as a product. The feature has high user value and good technical feasibility for a Tauri 2 + React project.

---

## 2. DataMapPlot — Technical Capabilities

### 2.1 Library Overview

DataMapPlot (version 0.7+, still active as of SciPy 2025 talk by Leland McInnes) is designed specifically to make UMAP data maps compelling to non-technical users through annotation labels, cluster markers, spatial color palettes, and a rich widget system. It targets the "communication and exploration" use case, not just internal data science tooling.

The library sits on top of:
- **DeckGL** (vis.gl) for GPU-accelerated WebGL rendering
- **UMAP** for dimensionality reduction
- **HDBSCAN** for clustering
- A custom label placement algorithm for readable cluster annotations

### 2.2 Interactive HTML Output

Calling `create_interactive_plot(...)` returns an HTML object that can be:
- Written to a `.html` file (`.save("map.html")`)
- Displayed inline in Jupyter

With `offline_mode=True`, the resulting file is **fully self-contained**: all JavaScript (DeckGL, deck.gl layers, fonts) is base64-encoded and inlined. No network calls are required at render time. The tradeoff is file size — for 5 000 points plus embedded JS bundles, expect 5–15 MB depending on metadata richness and DeckGL bundle size. This is acceptable for a local app.

The HTML runs in any modern browser or WebKit webview without a web server.

### 2.3 Hover Tooltips and Per-Point Metadata

The `hover_text` parameter accepts a list/array of strings — one per data point — that are shown in a tooltip when the cursor hovers over a point. This is the primary metadata channel.

Beyond plain text, the `extra_point_data` parameter accepts a pandas DataFrame with arbitrary columns (file name, file size, creation date, tags, MIME type, folder path, file extension). These columns are stored in the plot's `datamap.metaData` JavaScript object, keyed by column name. Custom `on_click` JavaScript can read any column from `extra_point_data` for the clicked point.

Concretely for Curator, `hover_text` would show a formatted card:
```
curator_notes.md
Modified: 2025-03-14
~/Documents/Curator/notes
```
and `extra_point_data` would include the absolute file path for click-to-reveal.

### 2.4 Click Interactivity

`create_interactive_plot` accepts an `on_click` parameter that is a **JavaScript string template** executed when a user clicks a point. The template can reference `{hover_text}` and any column from `extra_point_data` by name, e.g., `{filepath}`.

Default example from the docs opens a Google search:
```python
on_click = "window.open('https://google.com/search?q=' + encodeURIComponent('{hover_text}'))"
```

For Curator, the click handler would instead need to call into the Tauri IPC bridge. That cannot be done purely within the `on_click` string because the Tauri `__TAURI__` API is only available if Tauri has injected its initialization script first. The solution is covered in Section 4.2 (Tauri injection bridge).

A practical `on_click` for Curator:
```javascript
window.__tauriOpenFile__('{filepath}')
```
where `__tauriOpenFile__` is a function injected by Tauri's initialization script that calls `invoke('reveal_in_finder', { path: filepath })`.

### 2.5 Cluster Coloring, Labeling, and Navigation

- **Color**: Each HDBSCAN cluster gets a distinct color from a spatial palette derived from the UMAP layout positions (similar clusters get similar hues). Custom color maps are also supported (`label_color_map`).
- **Labels**: Cluster centroids are annotated with text labels. These are auto-generated from the cluster's content (e.g., the most representative n-gram or topic) and placed with collision avoidance.
- **Label layers**: Multiple `label_layers` can be passed for hierarchical annotation (e.g., broad topic at low zoom, fine-grained label at high zoom).
- **Navigation**: The DeckGL canvas supports pan and zoom natively. A minimap widget shows the global layout while the user zooms into a region.
- **Cluster boundary polygons**: Optional alpha-blended convex-hull-style polygons can be drawn around each cluster (`cluster_boundary_polygons=True`).

### 2.6 Search and Filter

As of version 0.7, the widget system includes:
- **`enable_search=True`**: adds a search bar that highlights matching points (text search over `hover_text` or `extra_point_data` columns).
- **Topic browser widget**: lets users navigate the cluster hierarchy.
- **Minimap widget**: global overview.
- **Layer visibility toggles**: show/hide clusters or metadata overlays.
- **Histogram widget**: filter by a numeric metadata field (e.g., file size, modification date).
- **Colormap switcher**: switch between coloring by cluster, by file type, by date, etc.

The widget system (`custom_html`, `custom_js`, `custom_css` params) also allows injecting arbitrary HTML/JS into the output, which is the escape hatch for the Tauri IPC bridge.

### 2.7 Performance with 5 000 Points

DeckGL renders via WebGL, which handles hundreds of thousands of points at 60fps on modern hardware. 5 000 points is well within the smooth-performance tier. There is no client-side performance concern for the visualization itself.

The bottleneck is Python preprocessing:
- **Embedding generation** (e.g., with `sentence-transformers`): ~30–90 seconds for 5 000 files on CPU, depending on model.
- **UMAP**: ~10–40 seconds for 5 000 points on CPU.
- **HDBSCAN**: seconds.
- **DataMapPlot HTML generation**: seconds.

Total pipeline: realistically **2–5 minutes** on a modern Mac with CPU-only inference. This is acceptable for a one-time or on-demand "generate map" action, but warrants a progress UI.

---

## 3. Tauri 2 Integration Architecture

### 3.1 Loading the Self-Contained HTML File

Tauri 2 provides `WebviewWindowBuilder` with a `WebviewUrl` argument. A local HTML file can be loaded via a file URL or via Tauri's asset protocol. The recommended approach for an arbitrary generated file:

```rust
use tauri::webview::{WebviewWindowBuilder, WebviewUrl};

let html_path = app.path().app_data_dir()?.join("file_map.html");
WebviewWindowBuilder::new(
    app,
    "file-map",
    WebviewUrl::App(html_path.to_string_lossy().to_string().into())
)
.title("File Landscape")
.inner_size(1200.0, 800.0)
.build()?;
```

Alternatively, navigate an existing webview:
```rust
webview_window.navigate(tauri::Url::from_file_path(&html_path).unwrap());
```

On macOS, local file URLs load fine in WebKit. CSP restrictions within the generated HTML may need to be relaxed by injecting a permissive meta tag or stripping the existing CSP header via Tauri's `on_page_load` hook.

### 3.2 IPC Bridge: Injecting the Tauri API into the DataMapPlot Page

The critical issue: the Tauri JavaScript IPC (`window.__TAURI__`) is only available in pages that are served from Tauri's own protocol (`tauri://` or `https://tauri.localhost`). A self-contained HTML file loaded from a local path starts in a different origin context.

**Solution: `initialization_script`**

`WebviewWindowBuilder` exposes `.initialization_script(script: &str)` which injects JavaScript **before the page parses its own scripts**. This script runs in the same JS context as the page and can define global functions.

```rust
const BRIDGE_SCRIPT: &str = r#"
    window.__tauriOpenFile__ = function(filePath) {
        window.__TAURI_INTERNALS__.invoke('reveal_in_finder', { path: filePath });
    };
"#;

WebviewWindowBuilder::new(app, "file-map", WebviewUrl::App(...))
    .initialization_script(BRIDGE_SCRIPT)
    .build()?;
```

On the Rust side:
```rust
#[tauri::command]
async fn reveal_in_finder(path: String, app: tauri::AppHandle) -> Result<(), String> {
    tauri_plugin_opener::reveal_item_in_dir(&path)
        .map_err(|e| e.to_string())
}
```

Then in DataMapPlot's Python call:
```python
datamapplot.create_interactive_plot(
    umap_coords,
    labels,
    hover_text=formatted_hover_strings,
    extra_point_data=df_with_filepaths,
    on_click="window.__tauriOpenFile__('{filepath}')",
    enable_search=True,
    offline_mode=True,
)
```

**Alternative: postMessage bridge**

If the `initialization_script` approach has CSP friction, an alternative is to use `window.postMessage` from the DataMapPlot page and listen for it in the parent Tauri window:
```javascript
// In DataMapPlot's on_click
window.parent.postMessage({ type: 'open_file', path: '{filepath}' }, '*');
```
Then in Tauri's main webview JavaScript:
```javascript
window.addEventListener('message', (event) => {
  if (event.data.type === 'open_file') {
    invoke('reveal_in_finder', { path: event.data.path });
  }
});
```
This works if the file map is loaded in an iframe within the Tauri app's main page rather than a separate WebviewWindow. Either approach is viable.

### 3.3 Opening Files in Finder

Tauri 2's official `tauri-plugin-opener` provides:

JavaScript API:
```typescript
import { revealItemInDir } from '@tauri-apps/plugin-opener';
await revealItemInDir('/Users/hp/Documents/notes.md');
```

Rust API (for use in commands):
```rust
use tauri_plugin_opener::OpenerExt;
app.opener().reveal_item_in_dir("/Users/hp/Documents/notes.md")?;
```

On macOS, this calls `NSWorkspace.shared.activateFileViewerSelecting([url])`, which opens Finder with the file selected. This is the cleanest UX for "click a dot → reveal file."

### 3.4 Permissions Configuration

In `tauri.conf.json`:
```json
{
  "plugins": {
    "opener": {
      "all": true
    },
    "shell": {
      "all": false
    }
  }
}
```

In `capabilities/*.json`:
```json
{
  "permissions": [
    "opener:allow-reveal-item-in-dir"
  ]
}
```

---

## 4. Python Sidecar Architecture

### 4.1 Tauri 2 Sidecar Mechanism

Tauri's sidecar system allows bundling an arbitrary binary alongside the app. The binary must be named `<name>-<target-triple>` (e.g., `curator-pipeline-aarch64-apple-darwin`) and declared in `tauri.conf.json`:

```json
{
  "bundle": {
    "externalBin": ["binaries/curator-pipeline"]
  }
}
```

Tauri copies the correct binary for the build target. On macOS arm64, it selects `curator-pipeline-aarch64-apple-darwin`.

### 4.2 Python Packaging with PyInstaller

The pipeline Python script:
```python
# curator_pipeline.py
import sys, json
import numpy as np
from sentence_transformers import SentenceTransformer
import umap, hdbscan
import datamapplot

def main(file_list_json: str, output_html_path: str):
    files = json.loads(file_list_json)
    # ... embed, reduce, cluster, plot
    print(json.dumps({"status": "embedding", "progress": 0}), flush=True)
    # ...
    print(json.dumps({"status": "done", "html_path": output_html_path}), flush=True)

if __name__ == "__main__":
    main(sys.argv[1], sys.argv[2])
```

Bundling:
```bash
pyinstaller --onefile \
    --hidden-import=umap \
    --hidden-import=hdbscan \
    --hidden-import=datamapplot \
    --hidden-import=sentence_transformers \
    curator_pipeline.py
```

The resulting binary is ~200–400 MB due to NumPy/SciPy/PyTorch dependencies. This is large but standard for ML desktop apps (similar to the example `dieharders/example-tauri-v2-python-server-sidecar` on GitHub which does the same for a FastAPI + LLM stack).

### 4.3 Spawning and Progress Streaming

```rust
use tauri_plugin_shell::ShellExt;

#[tauri::command]
async fn generate_file_map(
    file_list: Vec<String>,
    app: tauri::AppHandle,
) -> Result<String, String> {
    let output_path = app.path().app_data_dir()
        .unwrap().join("file_map.html");
    let file_list_json = serde_json::to_string(&file_list).unwrap();

    let (mut rx, _child) = app.shell()
        .sidecar("curator-pipeline")
        .unwrap()
        .args([&file_list_json, output_path.to_str().unwrap()])
        .spawn()
        .map_err(|e| e.to_string())?;

    while let Some(event) = rx.recv().await {
        match event {
            CommandEvent::Stdout(line) => {
                // Parse JSON progress messages, emit to frontend
                let _ = app.emit("pipeline-progress", &line);
            }
            CommandEvent::Terminated(_) => break,
            _ => {}
        }
    }
    Ok(output_path.to_string_lossy().to_string())
}
```

Frontend subscribes to `pipeline-progress` events and renders a progress bar. On completion, the Rust command returns the HTML path and the frontend opens the file map window.

### 4.4 Handling the 2–5 Minute Runtime

- Run via `spawn()`, not `execute()`, so the UI remains responsive.
- Stream structured JSON progress lines from Python (`{"status": "embedding", "progress": 0.3, "message": "Embedding 1500/5000 files"}`).
- Show a dedicated progress modal in React with stage labels: Indexing → Embedding → Reducing → Clustering → Generating Map.
- Cache the generated HTML alongside a hash of the file list; if unchanged, skip recomputation.
- Optional: background generation triggered automatically when Curator indexes files, so the map is ready by the time the user requests it.

**Known PyInstaller sidecar issue**: `process.kill()` kills only the PyInstaller bootloader PID, not the Python child. Handle graceful shutdown by having the Python script listen for a SIGTERM or a special stdin message (`"STOP\n"`).

---

## 5. Prior Work on Visual File System Maps

### 5.1 Hierarchical / Structural Visualizations (1990s–2000s)

**Treemaps** (Shneiderman, 1992) represent file system hierarchy as nested rectangles sized by file size. They preserve structure but not semantic similarity — two unrelated files in the same folder appear adjacent. Standard macOS and Windows disk usage apps (DaisyDisk, WinDirStat) use this approach.

**SeeSoft** (Eick et al., 1992) maps source code lines to colored pixels, showing structural relationships in code files. Not applicable to general personal file systems.

**SpaceTree** (Grosjean et al., 2002) is a focus+context hierarchical tree browser — still structure-based, not semantic.

**Cone Tree / Perspective Wall** (Robertson et al., 1991) — 3D hierarchical browsers. Interesting but hierarchy-bound.

### 5.2 Scatter / 2D Semantic Approaches

**Liquid 2D Scatter Space (L2DSS)** (IEEE Information Visualisation, 2005) is the closest precedent. It visualizes file system metadata (size, date, type) as axes in a 2D scatter plot with "liquid browsing" interaction to handle overplotting. Key difference from Curator's proposal: L2DSS uses *explicit metadata dimensions* (file size vs. modification date) as axes — it is not semantic. Files are placed by their properties, not by the meaning of their content. The mobile variant (ML2DSS, 2004) extended it to mobile devices.

**Galaxies** (Rennison, 1994) reduces high-dimensional document representations to a 2D scatter of "docupoints" where proximity means semantic similarity. This is the closest conceptual ancestor — but it was a 1990s research prototype never productized for personal file systems.

**Document Landscape** (Skupin & Buttenfield, 1996) maps MDS-reduced term-frequency vectors of newspaper articles onto a 2D terrain landscape (height = cluster density). Again: research prototype, documents not files, no personal context.

**Semantic Treemaps** (Feng et al., ~2002) arranges bookmarks in a treemap where spatially adjacent items are semantically similar, using 2D Euclidean distance to approximate cosine similarity. Closer to the idea, but still a treemap, not a free scatter, and applied to browser bookmarks.

**SemLens** provides visual analysis of semantic data with scatter plots and semantic lenses — but targets structured RDF/OWL data, not file systems.

**Patent US5625767** (1997): "Method and system for two-dimensional visualization of an information taxonomy and of text documents based on topical content of the documents." Describes generating a 2D semantic space map where position encodes topic. Foundational but never a consumer product.

**Patent US8812507**: "Method and apparatus for maintaining and navigating a non-hierarchical personal spatial file system." Describes a spatial, non-hierarchical PIM where items have persistent 2D positions. Interesting — but position is user-assigned, not computed from semantic similarity.

### 5.3 Modern ML-Based Approaches

**Atlas** (Nomic AI, 2023–) provides UMAP-based 2D maps for text datasets, with search, zoom, and cluster labeling. It is a cloud SaaS product for data scientists, not a local file system explorer. The UX inspiration is directly relevant.

**Conceptually similar exploration tools**: tools like Obsidian's graph view, DEVONthink's semantic links, and Eagle (image organizer) provide semantic clustering but not as a 2D navigable landscape.

No shipping product as of 2026 provides: UMAP-based 2D semantic landscape of a user's personal file system, local/private, with point-click file reveal, in a polished native desktop app.

### 5.4 Novelty Assessment

The combination that is novel:
1. Full local pipeline (no cloud, no telemetry)
2. UMAP + HDBSCAN on *general file content embeddings* (not just code or documents of one type)
3. Interactive navigable landscape with cluster labels
4. Click-to-reveal file in Finder
5. Integrated into a file management app (Curator) with existing indexing infrastructure

Each piece individually exists in prior work. The integration into a usable, private, native desktop file manager is new as a shipped product. The closest ship-ready inspiration is Nomic Atlas, but it is cloud-based and for curated datasets, not personal file systems.

---

## 6. User Value Analysis

### 6.1 Tasks Enabled by a Visual File Map

| Task | Folder Navigation | Visual Map |
|------|------------------|------------|
| Find a specific file by name | Fast (search) | Slow (dot scan) |
| Discover thematically related files across folders | Very hard | Immediate (proximity) |
| Identify topic clusters you didn't know you had | Impossible | Automatic |
| Spot orphaned / misclassified files | Tedious | Visual outliers |
| Understand overall collection shape and coverage | Impossible | At a glance |
| Rediscover forgotten files | Relies on memory | Serendipitous browsing |
| Find duplicates or near-duplicates | Impossible without tooling | Near points in map |
| Answer "what have I been working on lately?" | Calendar + file dates | Recent-highlighted map |

### 6.2 The Rediscovery Problem

Hierarchical file systems suffer from the "where did I put that?" problem and the "I forgot this existed" problem. A semantic map surfaces latent structure: files you created years apart but on the same topic appear near each other, regardless of folder. This is a qualitatively different mode of personal information retrieval — browsing by meaning rather than location.

### 6.3 The Topography Metaphor

A good UMAP map (especially with DataMapPlot's cluster labels and spatial color palette) creates a "mental map" — users quickly learn that "my biology notes are in the top-left cluster" and can navigate by geographic memory, which is cognitively natural. Nomic Atlas user research supports this: users describe their data maps as landscapes they "explore."

---

## 7. Implementation Plan

### Phase 1: Python Pipeline Script (1–2 days)

```python
# curator_pipeline.py

import sys, json, os
import numpy as np
import pandas as pd
from pathlib import Path

def embed_files(file_paths):
    from sentence_transformers import SentenceTransformer
    model = SentenceTransformer('all-MiniLM-L6-v2')  # 80MB, fast on CPU
    texts = []
    valid_paths = []
    for p in file_paths:
        try:
            text = extract_text(p)  # use existing Curator extractor
            texts.append(text[:1000])  # truncate for speed
            valid_paths.append(p)
        except:
            pass
    embeddings = model.encode(texts, show_progress_bar=False, batch_size=64)
    return embeddings, valid_paths

def run_pipeline(file_list, output_path):
    import umap, hdbscan, datamapplot

    progress("embedding", 0, "Extracting file content")
    embeddings, paths = embed_files(file_list)

    progress("reducing", 0.4, "Running UMAP")
    reducer = umap.UMAP(n_components=2, metric='cosine', random_state=42)
    coords = reducer.fit_transform(embeddings)

    progress("clustering", 0.7, "Clustering with HDBSCAN")
    clusterer = hdbscan.HDBSCAN(min_cluster_size=5, gen_min_span_tree=True)
    labels = clusterer.fit_predict(coords)

    progress("generating", 0.85, "Generating interactive map")
    df = pd.DataFrame({
        'filepath': paths,
        'filename': [Path(p).name for p in paths],
        'extension': [Path(p).suffix for p in paths],
    })
    hover = [f"{Path(p).name}\n{p}" for p in paths]

    fig = datamapplot.create_interactive_plot(
        coords,
        labels,
        hover_text=hover,
        extra_point_data=df,
        on_click="window.__tauriOpenFile__('{filepath}')",
        enable_search=True,
        offline_mode=True,
        cluster_boundary_polygons=True,
        darkmode=True,
        title="Your File Landscape",
    )
    fig.save(output_path)
    progress("done", 1.0, output_path)
```

### Phase 2: Tauri Command and Sidecar Config (1 day)

- Configure `tauri.conf.json` with `externalBin`.
- Write the `generate_file_map` command (see Section 4.3).
- Write the `reveal_in_finder` command wrapping `tauri-plugin-opener`.
- Add `pipeline-progress` event emission.

### Phase 3: React UI (1–2 days)

- "View File Landscape" button in Curator sidebar.
- Progress modal with stage labels and animated bar.
- On completion: open file map in a new window via `WebviewWindowBuilder`.
- Inject the IPC bridge via `initialization_script`.

### Phase 4: PyInstaller Build Integration (0.5 days)

- Add PyInstaller step to the CI/CD build.
- Rename output binary per Tauri target triple convention.
- Handle macOS code signing for the sidecar binary.

### Phase 5: Caching and Incremental Updates (1 day)

- Hash the file list + file modification times.
- Store hash alongside the generated HTML in `app_data_dir`.
- Skip pipeline if hash matches; regenerate only when files change.
- Option: incrementally embed new files only, re-run UMAP on the full set.

---

## 8. Technical Risks and Mitigations

| Risk | Likelihood | Mitigation |
|------|-----------|------------|
| PyInstaller bundle too large (400MB+) | Medium | Use a smaller embedding model; explore `onnxruntime` instead of PyTorch |
| CSP blocks Tauri initialization script in DML HTML | Medium | Strip CSP meta tags or use postMessage iframe approach |
| UMAP quality poor on heterogeneous file types | Low-Medium | Embed file content + metadata; filter out binary files |
| `process.kill()` leaves zombie PyInstaller process | Low-Medium | Implement stdin "STOP" protocol; use `kill_on_drop` in Rust |
| File map stale when files change | Low | Hash-based cache invalidation |
| DataMapPlot `on_click` JS escaping issues in filepath strings | Low | URL-encode filepath; decode in bridge function |
| 5 000 file embedding takes > 5 min on older Mac | Low | Use `all-MiniLM-L6-v2` (fast); add progress feedback; consider background pre-generation |

---

## 9. Relevant Links

- DataMapPlot docs: https://datamapplot.readthedocs.io/en/latest/
- DataMapPlot interactive intro: https://datamapplot.readthedocs.io/en/latest/interactive_intro.html
- DataMapPlot selection/filtering: https://datamapplot.readthedocs.io/en/latest/selection_and_filtering.html
- DataMapPlot offline mode: https://datamapplot.readthedocs.io/en/latest/offline_mode.html
- DataMapPlot SciPy 2025 talk: https://cfp.scipy.org/scipy2025/talk/DM3QPX/
- Tauri 2 sidecar docs: https://v2.tauri.app/develop/sidecar/
- Tauri 2 Opener plugin: https://v2.tauri.app/plugin/opener/
- Tauri 2 calling Rust: https://v2.tauri.app/develop/calling-rust/
- Tauri 2 IPC deepwiki: https://deepwiki.com/tauri-apps/tauri/4-ipc-and-frontend-backend-communication
- Tauri Python sidecar example: https://github.com/dieharders/example-tauri-v2-python-server-sidecar
- Liquid 2D Scatter Space (2005 precedent): https://ieeexplore.ieee.org/document/1509115/
- Nomic Atlas (commercial inspiration): https://atlas.nomic.ai

---

## 10. Verdict

**Build it.** The feature is technically feasible with the existing Tauri 2 + React stack. The main engineering unknowns are (a) PyInstaller bundle size with ML deps, and (b) the CSP/IPC bridge into the DataMapPlot HTML — both are solvable. The user value is high and qualitatively different from anything else in Curator's current UI. The feature is novel as a shipped personal-file-system product despite clear academic precedents.

Start with a proof-of-concept: hardcode 500 files, run the pipeline in a local Python script, generate the HTML, load it in a Tauri webview, inject the bridge script manually, and test the click-to-reveal flow. That PoC can be done in a single weekend and will validate the architecture before committing to PyInstaller bundling.
