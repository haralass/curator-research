## ΜΕΡΟΣ 30: PRACTICAL IMPLEMENTATION — ICLOUD, EMAIL ATTACHMENTS, PROJECTION LAYER, TAURI IPC, AIRDROP, OBSIDIAN
*(Έρευνα 2026-05-22)*

---

### 1. iCloud Drive Files — Αρχεία Stored in iCloud αλλά Όχι Downloaded Locally

#### Findings

Το iCloud Drive χρησιμοποιεί **placeholder files** για να αντιπροσωπεύει αρχεία που βρίσκονται στο cloud αλλά δεν έχουν κατεβεί τοπικά. Το σύστημα λειτουργεί ως εξής:

**Placeholder format:**
- Ένα αρχείο `document.pdf` που δεν είναι downloaded εμφανίζεται ως `.document.pdf.icloud` (dot-prefix + `.icloud` suffix)
- Περιέχει το xattr `com.apple.icloud.itemName` που κρατά το πραγματικό όνομα του αρχείου (`document.pdf`)
- Μόλις κατεβεί, το placeholder διαγράφεται και εμφανίζεται το πραγματικό αρχείο — το xattr εξαφανίζεται

**NSMetadataUbiquitousItemDownloadingStatusKey values:**
- `NSMetadataUbiquitousItemDownloadingStatusCurrent` — downloaded και up-to-date
- `NSMetadataUbiquitousItemDownloadingStatusDownloaded` — downloaded αλλά outdated version
- `NSMetadataUbiquitousItemDownloadingStatusNotDownloaded` — δεν είναι τοπικά διαθέσιμο

**mdls σε placeholder files:**
Το `mdls` μπορεί να τρέξει στο `.icloud` placeholder αρχείο. Επιστρέφει:
- `kMDItemFSName` = `.document.pdf.icloud` (το placeholder όνομα)
- `kMDItemDisplayName` = `document.pdf` (το πραγματικό όνομα, macOS κάνει translation)
- **Δεν** επιστρέφει `kMDItemContentTypeTree`, `kMDItemPixelHeight` κ.λπ. — αυτά απαιτούν local content

**xattr διαθεσιμότητα σε not-downloaded files:**
- `com.apple.icloud.itemName` — ΔΙΑΘΕΣΙΜΟ στο placeholder
- `com.apple.metadata:kMDItemWhereFroms` — ΔΙΑΘΕΣΙΜΟ (αν είχε τεθεί πριν το upload, αλλά iCloud strips πολλά xattrs)
- `com.apple.metadata:_kMDItemUserTags` — ΕΞΑΡΤAΤΑΙ: iCloud διατηρεί tags αλλά μόνο αν υποστηρίζεται από το filesystem
- `kMDItemUseCount`, `kMDItemUsedDates` — ΔΕΝ είναι διαθέσιμα χωρίς local content

**Gotcha:** Το iCloud strips αρκετά xattrs κατά το upload. Τα xattrs που επιβιώνουν περνώντας μέσω iCloud: `com.apple.metadata:_kMDItemUserTags`, `com.apple.FinderInfo`, `com.apple.ResourceFork`. Τα `com.apple.quarantine`, `kMDItemWhereFroms` κ.λπ. συνήθως **δεν** επιβιώνουν.

#### Στρατηγική για Curator

| Option | Πότε | Trade-off |
|--------|------|-----------|
| **Skip** | Default για μη-critical scans | Ασφαλές αλλά χάνουμε αρχεία |
| **Embed metadata only** | Για ταξινόμηση χωρίς download | Χρησιμοποιούμε μόνο filename + path |
| **Trigger download** | Μόνο αν user επιλέξει | `brctl download <path>` ή FileManager API |

Η **προτεινόμενη προσέγγιση** για Curator: detect placeholders, embed με filename-only strategy (χαμηλής ποιότητας embedding), flag τα ως `icloud_undownloaded=True` στο ChromaDB metadata.

#### Code Sketch: Python iCloud Detection

```python
import subprocess
import os
import plistlib
import xattr

def detect_icloud_status(file_path: str) -> dict:
    """
    Ανιχνεύει αν ένα αρχείο είναι iCloud placeholder ή downloaded.
    Returns: dict με status, real_name, και mdls metadata αν διαθέσιμο.
    """
    filename = os.path.basename(file_path)
    
    # Pattern: .filename.ext.icloud
    is_placeholder = (
        filename.startswith('.') and filename.endswith('.icloud')
    )
    
    result = {
        "is_placeholder": is_placeholder,
        "real_name": None,
        "can_embed": False,
        "icloud_undownloaded": False
    }
    
    if is_placeholder:
        # Διαβάζουμε το πραγματικό όνομα από xattr
        try:
            fa = xattr.xattr(file_path)
            real_name_bytes = fa.get('com.apple.icloud.itemName')
            if real_name_bytes:
                result["real_name"] = real_name_bytes.decode('utf-8')
        except Exception:
            # Fallback: αφαιρούμε . prefix και .icloud suffix
            result["real_name"] = filename[1:].removesuffix('.icloud')
        
        result["icloud_undownloaded"] = True
        result["can_embed"] = False  # μόνο filename embedding
        return result
    
    # Αρχείο υπάρχει τοπικά — ελέγχουμε αν είναι up-to-date με mdls
    try:
        mdls_out = subprocess.check_output(
            ["mdls", "-name", "kMDItemFSSize", "-raw", file_path],
            stderr=subprocess.DEVNULL,
            text=True
        ).strip()
        result["can_embed"] = mdls_out not in ["(null)", ""]
    except subprocess.CalledProcessError:
        result["can_embed"] = False
    
    return result


def scan_directory_with_icloud(directory: str) -> list[dict]:
    """Scans directory, handles iCloud placeholders."""
    files = []
    for root, dirs, filenames in os.walk(directory):
        # Skip .git, node_modules κ.λπ.
        dirs[:] = [d for d in dirs if not d.startswith('.') and 
                   d not in ('node_modules', '__pycache__', '.venv')]
        
        for fname in filenames:
            full_path = os.path.join(root, fname)
            status = detect_icloud_status(full_path)
            files.append({"path": full_path, **status})
    
    return files


def trigger_icloud_download(file_path: str) -> bool:
    """Triggers download via brctl (macOS built-in tool)."""
    try:
        # brctl download <path> — ζητά από iCloud daemon να κατεβάσει
        subprocess.run(
            ["brctl", "download", file_path],
            check=True, capture_output=True
        )
        return True
    except (subprocess.CalledProcessError, FileNotFoundError):
        return False
```

**Εκτίμηση γραμμών για full implementation:** ~80-100 γραμμές  
**Gotchas:**
- Το `brctl` δεν είναι documented — μπορεί να αλλάξει σε μελλοντικές macOS εκδόσεις
- Τα placeholder αρχεία δεν εμφανίζονται στο Finder (macOS κάνει virtual mapping), αλλά είναι ορατά με `ls -la` και `os.walk()`
- Στο iCloud Drive folder (`~/Library/Mobile Documents/com~apple~CloudDocs/`), τα πράγματα είναι διαφορετικά από `~/Downloads` — το Curator κυρίως ασχολείται με `~/Downloads` οπότε το πρόβλημα είναι μικρότερο

---

### 2. Email Attachment Detection — Εντοπισμός από Email Attachments

#### Findings

**kMDItemWhereFroms για Mail.app attachments:**
Το `kMDItemWhereFroms` είναι ένα **array** (binary plist) που περιέχει URLs και strings για την προέλευση αρχείου. Για email attachments από Mail.app, το array τυπικά περιέχει:
1. `message:<message-id>` — το RFC 2822 Message-ID του email
2. Sender email address (π.χ. `john.doe@example.com`)
3. Subject string (π.χ. `RE: Project Proposal Q2`)

Παράδειγμα πραγματικής τιμής:
```
["message:<CABc123xyz@mail.gmail.com>", "sender@company.com", "Q2 Budget Report"]
```

**Reliability του signal:**
- **Mail.app**: Πάντα θέτει `kMDItemWhereFroms` με sender + subject — πολύ reliable
- **Chrome**: Θέτει μόνο το download URL (π.χ. `https://mail.google.com/mail/u/0/...`) — partial signal
- **Firefox**: Παρόμοιο με Chrome — URL only
- **Outlook for Mac**: Θέτει `kMDItemWhereFroms` — αλλά format ελαφρώς διαφορετικό
- **Telegram/WhatsApp**: Δεν θέτει email metadata — κανένα signal

**Clustering signal αξία:**
Αρχεία από email → πιθανόν professional/work category. Παραδείγματα:
- PDF invoices, Word documents, Excel spreadsheets
- Presentations από colleagues
- Contracts, NDAs, reports

Ο συνδυασμός `kMDItemWhereFroms` + `kMDItemContentType` (PDF/DOCX/XLSX) + sender domain (company.com vs personal) δίνει πολύ ισχυρό professional signal.

#### Code Sketch: Python Email Attachment Detection

```python
import subprocess
import plistlib
import re
from pathlib import Path

def get_where_froms(file_path: str) -> list[str]:
    """Διαβάζει kMDItemWhereFroms μέσω xattr (binary plist)."""
    try:
        import xattr
        fa = xattr.xattr(file_path)
        raw = fa.get('com.apple.metadata:kMDItemWhereFroms')
        if raw:
            return plistlib.loads(raw)
    except Exception:
        pass
    
    # Fallback: mdls
    try:
        out = subprocess.check_output(
            ["mdls", "-name", "kMDItemWhereFroms", "-raw", file_path],
            text=True, stderr=subprocess.DEVNULL
        ).strip()
        # mdls επιστρέφει: ("value1", "value2", ...)
        if out and out != "(null)":
            # Parse mdls array output
            items = re.findall(r'"([^"]*)"', out)
            return items
    except Exception:
        pass
    
    return []


def classify_email_attachment(file_path: str) -> dict:
    """
    Ανιχνεύει αν ένα αρχείο είναι email attachment και εξάγει metadata.
    """
    where_froms = get_where_froms(file_path)
    
    result = {
        "is_email_attachment": False,
        "sender": None,
        "subject": None,
        "message_id": None,
        "sender_domain": None,
        "category_hint": None
    }
    
    if not where_froms:
        return result
    
    for item in where_froms:
        # Message ID format: "message:<id>"
        if item.startswith("message:"):
            result["message_id"] = item
            result["is_email_attachment"] = True
        # Email address pattern
        elif re.match(r'^[a-zA-Z0-9._%+-]+@[a-zA-Z0-9.-]+\.[a-zA-Z]{2,}$', item):
            result["sender"] = item
            domain = item.split('@')[1]
            result["sender_domain"] = domain
            # Personal vs professional heuristic
            personal_domains = {'gmail.com', 'yahoo.com', 'hotmail.com', 
                                 'outlook.com', 'icloud.com', 'me.com'}
            if domain not in personal_domains:
                result["category_hint"] = "work"
            else:
                result["category_hint"] = "personal"
        # Subject (αν δεν είναι URL και δεν είναι email)
        elif not item.startswith("http") and '@' not in item and len(item) > 3:
            result["subject"] = item
    
    return result


# Παράδειγμα χρήσης:
# info = classify_email_attachment("/Users/user/Downloads/invoice_Q2.pdf")
# if info["is_email_attachment"] and info["category_hint"] == "work":
#     boost_professional_score(file_path)
```

**Εκτίμηση γραμμών για full implementation:** ~100-120 γραμμές  
**Gotchas:**
- Chrome downloads από Gmail δίνουν URL `https://mail.google.com/...` — αυτό ΔΕΝ περιέχει sender/subject αλλά μπορούμε να detect ότι προέρχεται από webmail
- Ελέγχουμε και `com.apple.quarantine` για `LSQuarantineAgentName` — Mail.app sets agent name
- Το `kMDItemWhereFroms` μπορεί να έχει τιμή `[""]` (empty string array) σε μερικά αρχεία — να το χειριζόμαστε

---

### 3. Projection Layer Design — PRISM-Inspired Adaptation Layer πάνω σε Frozen BGE-M3

#### Findings

**Αρχιτεκτονική projection layers (από έρευνα):**

Η βιβλιογραφία δείχνει ότι το βέλτιστο pattern για domain adaptation frozen embeddings είναι:
- **2-layer MLP** με ReLU: `Linear(1024→512) → ReLU → Linear(512→256)` — καλή ισορροπία
- **Linear μόνο**: `Linear(1024→256)` — γρήγορο, λίγοτερο expressive
- **Contrastive objective** (SimCLR / NT-Xent loss): φέρνει κοντά positive pairs, απωθεί negative

**SetFit approach (arXiv:2209.11055):**
Το SetFit χρησιμοποιεί **contrastive Siamese fine-tuning** σε frozen base model + trainable head:
1. Phase 1: Fine-tune embedding layer με contrastive loss σε few-shot pairs
2. Phase 2: Train classification head (logistic regression ή MLP) στα adapted embeddings

Για Curator, η προσαρμογή είναι:
- Phase 1: Projection layer training με user confirmations ως positive pairs
- Phase 2: Clustering με HDBSCAN στο projected space (όχι classification head)

**Output dimensions:**
- 256 dims: Καλό balance speed/quality, αρκετά για HDBSCAN
- 128 dims: Πιο γρήγορο, ελαφρώς χειρότερο quality
- **Προτεινόμενο: 256** — compatible με ChromaDB, γρήγορο UMAP

**Training requirements:**
- Minimum **5-10 positive pairs** για να αρχίσει training
- Κάθε user confirmation (αρχείο A → category X, αρχείο B → category X) = 1 positive pair
- Hard negatives: αρχεία από διαφορετικές confirmed categories = negative pairs
- **Trigger threshold: 10 confirmations** (5 positive pairs αρκετά, 10 δίνει σταθερό training)

#### Complete Design

```
BGE-M3 (frozen, 1024-dim output)
    ↓
ProjectionLayer (trainable)
  Linear(1024 → 512)
  BatchNorm(512)
  ReLU
  Dropout(0.1)
  Linear(512 → 256)
  L2-Normalize
    ↓
256-dim adapted embedding
    ↓
UMAP → HDBSCAN (existing pipeline)
```

#### Code Sketch: PyTorch Projection Layer

```python
import torch
import torch.nn as nn
import torch.nn.functional as F
from torch.optim import AdamW
import numpy as np
from pathlib import Path

class CuratorProjectionLayer(nn.Module):
    """
    Projection layer πάνω σε frozen BGE-M3 embeddings.
    Input: 1024-dim BGE-M3 output
    Output: 256-dim adapted embedding (L2-normalized)
    """
    def __init__(self, input_dim: int = 1024, output_dim: int = 256):
        super().__init__()
        hidden_dim = input_dim // 2  # 512
        
        self.projection = nn.Sequential(
            nn.Linear(input_dim, hidden_dim),
            nn.BatchNorm1d(hidden_dim),
            nn.ReLU(),
            nn.Dropout(0.1),
            nn.Linear(hidden_dim, output_dim)
        )
        # Xavier initialization
        for layer in self.projection:
            if isinstance(layer, nn.Linear):
                nn.init.xavier_uniform_(layer.weight)
                nn.init.zeros_(layer.bias)
    
    def forward(self, x: torch.Tensor) -> torch.Tensor:
        projected = self.projection(x)
        return F.normalize(projected, p=2, dim=-1)  # L2 normalize


class ContrastiveLoss(nn.Module):
    """NT-Xent Loss για contrastive training."""
    def __init__(self, temperature: float = 0.07):
        super().__init__()
        self.temperature = temperature
    
    def forward(self, z1: torch.Tensor, z2: torch.Tensor) -> torch.Tensor:
        # z1, z2: [N, D] — positive pairs (N pairs total)
        N = z1.size(0)
        z = torch.cat([z1, z2], dim=0)  # [2N, D]
        
        sim = torch.mm(z, z.T) / self.temperature  # [2N, 2N]
        # Mask diagonal (self-similarity)
        mask = torch.eye(2 * N, dtype=torch.bool, device=z.device)
        sim.masked_fill_(mask, -float('inf'))
        
        # Positive pairs: (i, i+N) και (i+N, i)
        labels = torch.cat([
            torch.arange(N, 2 * N),
            torch.arange(0, N)
        ]).to(z.device)
        
        return F.cross_entropy(sim, labels)


class ProjectionTrainer:
    """Manages training lifecycle για το projection layer."""
    
    TRIGGER_THRESHOLD = 10  # confirmations πριν αρχίσει training
    
    def __init__(self, model_path: str = None):
        self.model = CuratorProjectionLayer()
        self.optimizer = AdamW(self.model.parameters(), lr=1e-4, weight_decay=1e-5)
        self.loss_fn = ContrastiveLoss(temperature=0.07)
        self.confirmations = []  # list of (embedding, category_label)
        self.trained = False
        
        if model_path and Path(model_path).exists():
            self.model.load_state_dict(torch.load(model_path))
            self.trained = True
    
    def add_confirmation(self, embedding: np.ndarray, category: str):
        """User confirmation → training data."""
        self.confirmations.append((
            torch.tensor(embedding, dtype=torch.float32),
            category
        ))
        
        if len(self.confirmations) >= self.TRIGGER_THRESHOLD:
            self._train_epoch()
    
    def _train_epoch(self, epochs: int = 20):
        """Mini training loop με confirmed pairs."""
        # Group by category → positive pairs
        from collections import defaultdict
        by_cat = defaultdict(list)
        for emb, cat in self.confirmations:
            by_cat[cat].append(emb)
        
        self.model.train()
        for _ in range(epochs):
            for cat, embs in by_cat.items():
                if len(embs) < 2:
                    continue
                # Δημιουργία positive pairs
                z1 = torch.stack(embs[:-1])
                z2 = torch.stack(embs[1:])
                
                p1 = self.model(z1)
                p2 = self.model(z2)
                
                loss = self.loss_fn(p1, p2)
                self.optimizer.zero_grad()
                loss.backward()
                self.optimizer.step()
        
        self.trained = True
    
    def project(self, embedding: np.ndarray) -> np.ndarray:
        """Inference: project BGE-M3 embedding → 256-dim."""
        if not self.trained:
            # Identity-like: truncate to 256 dims (fallback)
            return embedding[:256] / (np.linalg.norm(embedding[:256]) + 1e-8)
        
        self.model.eval()
        with torch.no_grad():
            t = torch.tensor(embedding, dtype=torch.float32).unsqueeze(0)
            out = self.model(t)
            return out.squeeze(0).numpy()
    
    def save(self, path: str):
        torch.save(self.model.state_dict(), path)
```

**Εκτίμηση γραμμών για full implementation:** ~200-250 γραμμές (με persistence, metrics, incremental training)  
**Gotchas:**
- Με λίγα data (10-20 confirmations) το model **overfits** — το Dropout(0.1) + weight_decay βοηθά
- Το `trained=False` fallback (truncate to 256) εξασφαλίζει ότι η pipeline δουλεύει ακόμα και χωρίς training
- Το BatchNorm1d χρειάζεται batch_size > 1 — για single inference χρησιμοποιούμε `model.eval()` (BatchNorm χρησιμοποιεί running stats)
- Μετά από κάθε training, το HDBSCAN clustering θα πρέπει να **re-runs** γιατί τα embeddings αλλάζουν space

---

### 4. Tauri ↔ Python IPC — Rust Backend καλεί Python AI Classifier

#### Findings

**Tauri 2 Sidecar Architecture:**

Το Tauri 2 υποστηρίζει **sidecar binaries** — εξωτερικά executables που bundlάρονται με την app. Για Python, το standard approach είναι PyInstaller για να δημιουργεί single binary.

**Naming convention (critical):**
Το binary πρέπει να ονομαστεί `<name>-<target-triple>` π.χ.:
- `ai_classifier-aarch64-apple-darwin` (Apple Silicon)
- `ai_classifier-x86_64-apple-darwin` (Intel Mac)

**Για dev mode (όχι PyInstaller):**
Χρησιμοποιούμε `beforeDevCommand` script που εκκινεί τον Python process ξεχωριστά, και ο Rust τον βρίσκει μέσω Unix socket.

**IPC Options comparison:**

| Method | Pros | Cons | Best for |
|--------|------|------|----------|
| **stdin/stdout JSON-RPC** | Simple, no ports | Sequential, no concurrency | Short requests |
| **Unix socket** | Fast, concurrent, persistent state | More setup | Long-running AI process |
| **TCP/HTTP (FastAPI)** | Easy debugging, REST API | Localhost port conflicts | Complex APIs |
| **Named pipe** | Similar to Unix socket | Less portable | Simple IPC |

**Για Curator (long-running Python με DenStream state + ChromaDB):**
Η καλύτερη επιλογή είναι **Unix socket + JSON-RPC** — ο Python process ξεκινά μια φορά, κρατά state, και ο Rust επικοινωνεί asynchronously.

#### `tauri.conf.json` snippet

```json
{
  "bundle": {
    "externalBin": [
      "binaries/ai_classifier"
    ]
  },
  "plugins": {
    "shell": {
      "open": false,
      "sidecar": true,
      "scope": [
        {
          "name": "binaries/ai_classifier",
          "sidecar": true,
          "args": ["--socket", "/tmp/curator_ai.sock"]
        }
      ]
    }
  }
}
```

#### `src-tauri/capabilities/default.json` (permissions)

```json
{
  "permissions": [
    "shell:allow-execute",
    "shell:allow-spawn",
    "shell:allow-kill"
  ]
}
```

#### Code Sketch: Rust Side — Spawn + Unix Socket Communication

```rust
// src-tauri/src/ai_client.rs
use std::io::{Read, Write};
use std::os::unix::net::UnixStream;
use serde_json::{json, Value};
use std::time::Duration;

pub struct AiClient {
    socket_path: String,
}

impl AiClient {
    pub fn new(socket_path: &str) -> Self {
        Self { socket_path: socket_path.to_string() }
    }
    
    fn call(&self, method: &str, params: Value) -> Result<Value, String> {
        let mut stream = UnixStream::connect(&self.socket_path)
            .map_err(|e| format!("Socket connect error: {}", e))?;
        stream.set_read_timeout(Some(Duration::from_secs(30))).ok();
        
        let request = json!({
            "jsonrpc": "2.0",
            "method": method,
            "params": params,
            "id": 1
        });
        
        let payload = serde_json::to_string(&request).unwrap() + "\n";
        stream.write_all(payload.as_bytes())
            .map_err(|e| format!("Write error: {}", e))?;
        
        let mut response = String::new();
        stream.read_to_string(&mut response)
            .map_err(|e| format!("Read error: {}", e))?;
        
        serde_json::from_str(&response)
            .map_err(|e| format!("Parse error: {}", e))
    }
    
    pub fn classify_file(&self, file_path: &str, metadata: Value) 
        -> Result<Value, String> 
    {
        self.call("classify", json!({
            "file_path": file_path,
            "metadata": metadata
        }))
    }
    
    pub fn update_cluster(&self, file_path: &str, confirmed_category: &str) 
        -> Result<(), String> 
    {
        self.call("confirm", json!({
            "file_path": file_path,
            "category": confirmed_category
        }))?;
        Ok(())
    }
}

// src-tauri/src/main.rs — Sidecar spawn με tauri-plugin-shell
#[tauri::command]
async fn start_ai_classifier(app: tauri::AppHandle) -> Result<(), String> {
    use tauri_plugin_shell::ShellExt;
    
    let (mut rx, child) = app.shell()
        .sidecar("ai_classifier")
        .map_err(|e| e.to_string())?
        .args(["--socket", "/tmp/curator_ai.sock"])
        .spawn()
        .map_err(|e| e.to_string())?;
    
    // Store child handle για cleanup
    app.manage(std::sync::Mutex::new(Some(child)));
    
    // Wait for ready signal on stdout
    while let Some(event) = rx.recv().await {
        use tauri_plugin_shell::process::CommandEvent;
        if let CommandEvent::Stdout(line) = event {
            if line.contains("READY") {
                break;
            }
        }
    }
    Ok(())
}
```

#### Code Sketch: Python Side — JSON-RPC Unix Socket Server

```python
#!/usr/bin/env /opt/anaconda3/bin/python3
# ai_classifier.py — JSON-RPC server over Unix socket

import sys
import json
import socket
import os
import argparse
from pathlib import Path

# Import Curator AI modules
from classifier_core import CuratorClassifier  # BGE-M3 + HDBSCAN + DenStream

def handle_request(classifier: CuratorClassifier, request: dict) -> dict:
    method = request.get("method")
    params = request.get("params", {})
    req_id = request.get("id", 1)
    
    try:
        if method == "classify":
            result = classifier.classify(
                file_path=params["file_path"],
                metadata=params.get("metadata", {})
            )
            return {"jsonrpc": "2.0", "result": result, "id": req_id}
        
        elif method == "confirm":
            classifier.confirm(
                file_path=params["file_path"],
                category=params["category"]
            )
            return {"jsonrpc": "2.0", "result": "ok", "id": req_id}
        
        else:
            return {"jsonrpc": "2.0", "error": {"code": -32601, 
                    "message": f"Unknown method: {method}"}, "id": req_id}
    
    except Exception as e:
        return {"jsonrpc": "2.0", "error": {"code": -32603, 
                "message": str(e)}, "id": req_id}


def main():
    parser = argparse.ArgumentParser()
    parser.add_argument("--socket", default="/tmp/curator_ai.sock")
    args = parser.parse_args()
    
    # Cleanup old socket
    socket_path = args.socket
    if os.path.exists(socket_path):
        os.unlink(socket_path)
    
    # Initialize AI classifier (loads BGE-M3, ChromaDB, DenStream)
    classifier = CuratorClassifier()
    
    # Signal ready to Rust parent
    print("READY", flush=True)
    
    server = socket.socket(socket.AF_UNIX, socket.SOCK_STREAM)
    server.bind(socket_path)
    server.listen(5)
    
    try:
        while True:
            conn, _ = server.accept()
            with conn:
                data = b""
                while chunk := conn.recv(4096):
                    data += chunk
                    if data.endswith(b"\n"):
                        break
                
                request = json.loads(data.decode('utf-8').strip())
                response = handle_request(classifier, request)
                conn.sendall(json.dumps(response).encode('utf-8'))
    finally:
        server.close()
        if os.path.exists(socket_path):
            os.unlink(socket_path)


if __name__ == "__main__":
    main()
```

**Εκτίμηση γραμμών για full implementation:** ~300-400 γραμμές (Rust client + Python server + error handling + graceful shutdown)  
**Gotchas:**
- Το `brctl` / PyInstaller approach δεν δουλεύει εύκολα με Anaconda virtualenv — για dev mode, χρησιμοποιούμε direct Python path (`/opt/anaconda3/bin/python3`)
- Για production bundle: `pyinstaller --onefile --hidden-import chromadb ai_classifier.py` — το ChromaDB έχει πολλά hidden imports
- Το Unix socket path `/tmp/curator_ai.sock` μπορεί να υπάρχει ήδη αν crash-άρει η app — πάντα κάνουμε cleanup στο startup
- Concurrent requests: το παραπάνω model είναι sequential (1 connection τη φορά) — για concurrency χρησιμοποιούμε threading ή asyncio

---

### 5. AirDrop + Handoff Files — Εντοπισμός Via AirDrop

#### Findings

**kMDItemWhereFroms για AirDrop:**
Για αρχεία που ήρθαν μέσω AirDrop, το `kMDItemWhereFroms` περιέχει **binary plist array** με:
- Display name της συσκευής αποστολής (π.χ. `"Dad's iPhone 12 Pro"` ή `"Haralampos's MacBook Pro"`)
- Μερικές φορές: AirDrop identifier string

Παράδειγμα: `["Dad's iPhone 12 Pro"]` — μόνο device name, ΌΧΙ email/phone.

**com.apple.quarantine για AirDrop:**
Κρίσιμο! Το `com.apple.quarantine` xattr για AirDrop αρχεία περιέχει:
- `LSQuarantineAgentName` = `"sharingd"` — αυτό είναι το **key identifier**
- `LSQuarantineType` = `"AirDrop"` σε νεότερες macOS εκδόσεις
- Format: `0081;<hex_timestamp>;sharingd;<UUID>`

Άρα: detection AirDrop = ελέγχουμε αν `LSQuarantineAgentName == "sharingd"` ΚΑΙ τo `kMDItemWhereFroms` δεν είναι HTTP URL.

**Default AirDrop download location:**
- macOS: `~/Downloads` (default)
- Μπορεί να αλλάξει από το Finder

**Handoff files:**
Δεν υπάρχει ειδικό Handoff metadata σε αρχεία. Το Handoff είναι real-time activity passing — δεν αφήνει αρχεία με metadata. Τα Handoff αρχεία δεν χρειάζονται ξεχωριστή αντιμετώπιση.

**Clustering signal:**
- AirDrop από iPhone → **personal/social** (φωτογραφίες, videos, screenshots, PDFs από φίλους)
- AirDrop από Mac → **personal** ή **work** depending on content
- Device name parsing: αν περιέχει "iPhone" ή "iPad" → πιθανόν personal media
- Content type + AirDrop source = ισχυρό personal signal

#### Code Sketch: Python AirDrop Detection

```python
import subprocess
import plistlib
import re

def get_quarantine_info(file_path: str) -> dict:
    """Διαβάζει com.apple.quarantine xattr."""
    try:
        import xattr
        fa = xattr.xattr(file_path)
        raw = fa.get('com.apple.quarantine')
        if not raw:
            return {}
        
        quarantine_str = raw.decode('utf-8', errors='replace')
        # Format: "0081;<hex_time>;AgentName;<UUID>"
        parts = quarantine_str.split(';')
        if len(parts) >= 3:
            return {
                "flags": parts[0],
                "timestamp_hex": parts[1],
                "agent": parts[2],
                "uuid": parts[3] if len(parts) > 3 else None
            }
    except Exception:
        pass
    return {}


def detect_airdrop(file_path: str) -> dict:
    """
    Ανιχνεύει αν ένα αρχείο ήρθε μέσω AirDrop.
    Returns dict με is_airdrop, sender_device, category_hint.
    """
    result = {
        "is_airdrop": False,
        "sender_device": None,
        "device_type": None,  # "iphone", "ipad", "mac"
        "category_hint": None
    }
    
    # Primary signal: quarantine agent = "sharingd"
    quarantine = get_quarantine_info(file_path)
    agent = quarantine.get("agent", "").lower()
    
    if "sharingd" not in agent:
        return result
    
    result["is_airdrop"] = True
    
    # Secondary signal: device name from kMDItemWhereFroms
    where_froms = get_where_froms(file_path)  # από Topic 2
    for item in where_froms:
        # AirDrop δίνει device name, όχι URL
        if not item.startswith("http") and '@' not in item and len(item) > 2:
            result["sender_device"] = item
            device_lower = item.lower()
            
            if any(x in device_lower for x in ["iphone", "phone"]):
                result["device_type"] = "iphone"
                result["category_hint"] = "personal"
            elif any(x in device_lower for x in ["ipad", "tablet"]):
                result["device_type"] = "ipad"
                result["category_hint"] = "personal"
            elif any(x in device_lower for x in ["macbook", "imac", "mac mini"]):
                result["device_type"] = "mac"
                result["category_hint"] = "personal_or_work"
            else:
                result["category_hint"] = "personal"
            break
    
    return result
```

**Εκτίμηση γραμμών για full implementation:** ~80-100 γραμμές  
**Gotchas:**
- Σε παλιότερες macOS εκδόσεις (< 10.15), το `LSQuarantineType` δεν υπάρχει — rely only στο `agent = "sharingd"`
- Τα AirDrop αρχεία **δεν** έχουν email sender — μόνο device name
- Αν ο user διαγράψει το quarantine flag (`xattr -d com.apple.quarantine`), χάνεται το signal εντελώς
- Το QuarantineEventsV2.db (SQLite στο `~/Library/Preferences/`) κρατά ιστορικό ακόμα και αν τα xattrs διαγραφούν — αλλά αυτό απαιτεί database query (overkill για Curator)

---

### 6. Obsidian Vault Detection — Atomic Unit Handling

#### Findings

**`.obsidian/` directory structure:**
```
.obsidian/
├── app.json              # General app settings
├── appearance.json       # Theme settings (dark/light, CSS)
├── graph.json            # Graph view settings (NOT community data)
├── core-plugins.json     # Enabled core plugins list
├── community-plugins.json # Installed community plugins list
├── hotkeys.json          # Custom keyboard shortcuts
├── workspace.json        # Open panes/tabs state
└── plugins/              # Community plugin data
    └── <plugin-id>/
        ├── data.json
        ├── main.js
        └── manifest.json
```

**Σημαντική διευκρίνιση:** Το `graph.json` περιέχει **graph visualization settings** (colors, node sizes, link distance) — ΌΧΙ community detection data. Τα communities δεν αποθηκεύονται σε αρχείο — υπολογίζονται on-the-fly (Louvain algorithm).

**File Organizer 2000 / Note Companion:**
Το plugin λειτουργεί ως "inbox" pattern:
1. User ρίχνει αρχεία σε ειδικό "Inbox" folder
2. AI (GPT-4/Claude/Ollama) προτείνει: folder placement, tags, title
3. User επιβεβαιώνει ή αποδέχεται auto-organization
Χρησιμοποιεί **cloud AI** (OpenAI/Anthropic) — δεν είναι 100% local. Το Curator είναι local-only, άρα διαφορετικό προσέγγιση.

**Vault ως atomic unit:**
Ένα Obsidian vault είναι σαν git repo — πρέπει να αντιμετωπίζεται ως **ένα object**, όχι χιλιάδες individual notes. Η embedding στρατηγική:
- Embed: `vault_name` + top-level note titles + folder structure + tag cloud
- ΜΗΝ κάνεις scan individual notes ως ξεχωριστά αρχεία

**Άλλα atomic directories:**

| Pattern | Τι είναι | Marker file/dir |
|---------|----------|-----------------|
| `.git/` | Git repository | `.git/` dir |
| `.obsidian/` | Obsidian vault | `.obsidian/` dir |
| `node_modules/` | npm packages | `package.json` + `node_modules/` |
| `.venv/` / `venv/` | Python venv | `pyvenv.cfg` |
| `*.xcodeproj/` | Xcode project | `*.xcodeproj/` |
| `*.xcworkspace/` | Xcode workspace | `*.xcworkspace/` |
| `*.app/` | macOS app bundle | `Contents/Info.plist` |
| `*.framework/` | macOS framework | `Headers/` + binary |
| `__pycache__/` | Python cache | Should be ignored entirely |
| `.cargo/` | Rust registry | `.cargo/` dir |
| `target/` | Rust build | `Cargo.toml` in parent |
| `Pods/` | CocoaPods | `Podfile` in parent |
| `DerivedData/` | Xcode builds | Inside `~/Library/` usually |

#### Code Sketch: Python Vault Detection + Embedding

```python
import os
from pathlib import Path
import json

# Atomic directory markers — (marker, type_name)
ATOMIC_MARKERS = [
    (".git",          "git_repo"),
    (".obsidian",     "obsidian_vault"),
    (".venv",         "python_venv"),
    ("venv",          "python_venv"),
    ("node_modules",  "npm_package"),
    ("__pycache__",   "python_cache"),
    (".cargo",        "rust_cargo"),
    ("Pods",          "cocoapods"),
    ("DerivedData",   "xcode_derived"),
]

ATOMIC_EXTENSIONS = [
    ".app",
    ".framework",
    ".xcodeproj",
    ".xcworkspace",
    ".bundle",
]


def detect_atomic_directory(path: str) -> dict | None:
    """
    Ελέγχει αν ένα directory είναι atomic unit.
    Returns None αν δεν είναι atomic, αλλιώς dict με τύπο.
    """
    p = Path(path)
    if not p.is_dir():
        return None
    
    # Check extension-based atomics
    for ext in ATOMIC_EXTENSIONS:
        if p.suffix == ext:
            return {"type": ext[1:], "path": str(p), "skip_children": True}
    
    # Check marker-based atomics
    for marker, atype in ATOMIC_MARKERS:
        if (p / marker).exists():
            return {"type": atype, "path": str(p), "skip_children": True,
                    "marker": marker}
    
    return None


def embed_obsidian_vault(vault_path: str) -> dict:
    """
    Δημιουργεί embedding metadata για Obsidian vault ως atomic unit.
    ΔΕΝ κάνει scan individual notes.
    """
    p = Path(vault_path)
    vault_name = p.name
    
    # Top-level note titles (μόνο 1 βάθος)
    top_notes = [
        f.stem for f in p.iterdir()
        if f.is_file() and f.suffix == '.md'
        and not f.name.startswith('.')
    ][:20]  # Max 20 titles
    
    # Top-level folders (κατηγορίες)
    top_folders = [
        d.name for d in p.iterdir()
        if d.is_dir() and not d.name.startswith('.')
        and d.name not in ('node_modules', '.trash')
    ]
    
    # Obsidian config: installed plugins (δείχνει τι χρησιμοποιεί ο user)
    plugins = []
    community_plugins_file = p / ".obsidian" / "community-plugins.json"
    if community_plugins_file.exists():
        try:
            plugins = json.loads(community_plugins_file.read_text())[:10]
        except Exception:
            pass
    
    # Count total notes
    total_notes = sum(1 for _ in p.rglob("*.md"))
    
    # Compose text for embedding
    embed_text = f"Obsidian vault: {vault_name}. "
    if top_folders:
        embed_text += f"Sections: {', '.join(top_folders)}. "
    if top_notes:
        embed_text += f"Notes include: {', '.join(top_notes[:10])}. "
    embed_text += f"Total {total_notes} notes."
    
    return {
        "atomic_type": "obsidian_vault",
        "vault_name": vault_name,
        "path": str(p),
        "embed_text": embed_text,
        "top_folders": top_folders,
        "top_notes": top_notes,
        "plugins": plugins,
        "total_notes": total_notes,
        "suggested_category": "knowledge_base"
    }


# Παράδειγμα integration με scanner:
def scan_with_atomic_awareness(root_dir: str) -> list[dict]:
    """
    Scanner που αντιμετωπίζει atomic directories ως ενιαία units.
    """
    results = []
    
    for entry in os.scandir(root_dir):
        if entry.is_dir():
            atomic = detect_atomic_directory(entry.path)
            if atomic:
                if atomic["type"] == "obsidian_vault":
                    vault_data = embed_obsidian_vault(entry.path)
                    results.append(vault_data)
                elif atomic["type"] not in ("python_cache", "npm_package"):
                    # Git repos, Xcode projects κ.λπ. — embed ως unit
                    results.append({
                        "atomic_type": atomic["type"],
                        "path": entry.path,
                        "name": entry.name,
                        "embed_text": f"{atomic['type'].replace('_', ' ')}: {entry.name}"
                    })
                # python_cache, npm_package → skip εντελώς
            else:
                # Recursive scan για κανονικά directories
                results.extend(scan_with_atomic_awareness(entry.path))
        else:
            results.append({"path": entry.path, "atomic_type": None})
    
    return results
```

**Εκτίμηση γραμμών για full implementation:** ~150-200 γραμμές (με πλήρη atomic registry, embedding pipeline integration, edge cases)  
**Gotchas:**
- Το `node_modules/` μπορεί να έχει χιλιάδες αρχεία — πάντα skip, ποτέ scan
- Ένα git repo με `.venv/` μέσα: το `.git/` detection σταματά το recursive scan, άρα το `.venv/` δεν εξετάζεται ξεχωριστά — **σωστή συμπεριφορά**
- Το `*.app` bundle στο `~/Downloads` (downloaded app): embed ως "application installer", suggested_category = "applications"
- Τα `__pycache__/` πρέπει να **αγνοούνται εντελώς** (όχι embedded, όχι scanned)
- Obsidian `.trash/` folder μέσα στο vault: ignore — ήδη έχει γίνει "deleted" από τον user

---

### Σύνοψη ΜΕΡΟΣ 30

| Topic | Key Finding | Lines Estimate |
|-------|------------|----------------|
| iCloud | Placeholders: `.filename.icloud` + `com.apple.icloud.itemName` xattr, `brctl download` για trigger | ~80-100 |
| Email attachments | `kMDItemWhereFroms` array: `["message:<id>", "sender@domain", "Subject"]` — πολύ reliable για Mail.app | ~100-120 |
| Projection layer | 2-layer MLP (1024→512→256) + NT-Xent loss, trigger στα 10 confirmations | ~200-250 |
| Tauri IPC | Unix socket JSON-RPC: Python server prints "READY", Rust connects via socket | ~300-400 |
| AirDrop | `com.apple.quarantine` agent=`"sharingd"` = primary signal, device name από `kMDItemWhereFroms` | ~80-100 |
| Obsidian | `.obsidian/` marker = atomic unit, embed vault_name + folders + note titles | ~150-200 |

**Συνολική εκτίμηση για full implementation όλων των topics:** ~900-1170 γραμμές Python/Rust