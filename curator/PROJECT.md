# 🦚 Curator — Πλήρης Τεκμηρίωση Έργου

> Αυτό το αρχείο δίνει σε οποιοδήποτε νέο chat ΟΛΟ το context: τι είναι το Curator,
> γιατί υπάρχει, πώς δουλεύει, πού είναι τα αρχεία, τι έχει γίνει, και τι μένει.
>
> Τελευταία ενημέρωση: 2026-05-20

---

## 1. Όραμα — Τι είναι το Curator

Το **Curator** είναι ένα macOS app που **τακτοποιεί αυτόματα τον φάκελο
Downloads** χρησιμοποιώντας **τοπική τεχνητή νοημοσύνη** (Ollama: llava + llama3.2).

**Το πρόβλημα που λύνει:** ο φάκελος Downloads γεμίζει με χάος — PDFs μαθημάτων,
screenshots, εικόνες, εγκαταστάτες, archives μπερδεμένα μεταξύ τους. Το να τα
τακτοποιείς χειροκίνητα είναι ατέρμονο.

**Η ιδέα του Curator:**
1. Παρακολουθεί τον φάκελο Downloads αυτόματα
2. Σε κάθε νέο αρχείο, αποφασίζει σε ποιον φάκελο ανήκει
3. Το μετακινεί εκεί — **χωρίς να σε ρωτήσει**, εκτός αν είναι αβέβαιο
4. **100% τοπικά** — τίποτα δεν φεύγει από τον Mac. Δωρεάν.

**Διαφορά από άλλα tools:**
- Όχι κανόνες-cleanup όπως το Hazel — έχει AI που καταλαβαίνει περιεχόμενο.
- Όχι cloud AI όπως ChatGPT API — όλα τρέχουν τοπικά μέσω Ollama.
- Όχι μόνο file-mover — έχει UI για review, manual rules, statistics.

---

## 2. Αρχιτεκτονική

Το Curator έχει **δύο ανεξάρτητα κομμάτια** που συνεργάζονται:

### Α. Η μηχανή (Python — `~/Library/Scripts/ai_classifier.py`)

Το «μυαλό» που ταξινομεί αρχεία. Καλεί το Ollama για το AI κομμάτι.

**Pipeline ταξινόμησης (από γρήγορο σε αργό):**
1. **Learned rules** — αν έχει μάθει «.png → Εικόνες», απευθείας
2. **Course code** στο όνομα (EPL236, EPL325 κλπ) → αντίστοιχος φάκελος
3. **Extension fast-path** για βίντεο/μουσική/zip/installer → χωρίς AI
4. **Download source** (`kMDItemWhereFroms`) — github→Κώδικας, youtube→Βίντεο
5. **SQLite cache** — αν το ίδιο αρχείο ξανάρθε, χρησιμοποίει την αποθηκευμένη απόφαση
6. **AI** — llava:7b για εικόνες/PDFs, llama3.2:3b για κείμενο
7. **LOW confidence fallback** — αν το AI είναι αβέβαιο, ext-classify (π.χ. .png στις Εικόνες) πριν στείλει σε review

**Έξυπνα features στη μηχανή:**
- Whitelist 15 κατηγοριών στο prompt (anti-«Διάφορα»)
- Confidence calibration (HIGH/MEDIUM/LOW με σαφή ορισμό)
- Timeout στο Ollama (60s) ώστε κολλημένο μοντέλο να μην παγώνει
- File lock (`/tmp/curator_classifier.lock`) — αποτρέπει concurrent runs
- Auto-rename για γνωστά messy patterns (`ChatGPT Image…` → `chatgpt-image-YYYY-MM-DD`)

**Trigger:**
- **launchd agent** (`~/Library/LaunchAgents/com.haralambos.aiclassifier.plist`)
  παρακολουθεί `WatchPaths` και τρέχει τη μηχανή κάθε φορά που πέφτει αρχείο.
- **In-app watcher** στο Tauri app (notify crate) τρέχει επίσης τη μηχανή —
  με τον lock να αποτρέπει διπλό run.

### Β. Η εφαρμογή (Tauri 2 + React + TypeScript)

Το UI / control center. Δεν εκτελεί ταξινόμηση η ίδια — διαχειρίζεται,
δείχνει, και ενορχηστρώνει.

**Stack:**
- **Frontend**: React 19 + TypeScript + Vite + Tailwind v4 + **shadcn/ui**
  (battle-tested component library — όχι hand-CSS)
- **Backend**: Rust (Tauri 2) — εντολές που διαβάζουν/γράφουν files, ξεκινούν
  τον Python classifier, μετακινούν αρχεία
- **Icons**: lucide-react (όχι emoji)
- **Animations**: framer-motion
- **Toasts**: sonner
- **Charts**: recharts

**Plugins (επίσημα Tauri 2):**
- `tauri-plugin-autostart` — εκκίνηση στο login
- `tauri-plugin-notification` — native ειδοποιήσεις
- `tauri-plugin-positioner` — popover κάτω από tray icon
- `tauri-plugin-single-instance` — μόνο ένα παράθυρο τη φορά
- `tauri-plugin-global-shortcut` — ⌘⇧C
- `tauri-plugin-log` — file logging
- `tauri-plugin-updater` — auto-updates (ready, χρειάζεται server για να λειτουργήσει)

---

## 3. Φάκελοι & αρχεία

### Ζωντανός κώδικας (αυτά τρέχουν)

`~/Library/Scripts/` (Python engine + παλιά UI που πια δεν τρέχει):
- **`ai_classifier.py`** — η μηχανή ταξινόμησης (ζωτικό)
- **`classify_pending.py`** — παλιός osascript διάλογος review (πια δεν χρησιμοποιείται από το Tauri, μένει ως backup)
- `curator_window.py`, `curator_menubar.py`, `curator_ui.html` — ΠΑΛΙΑ UI (Python/WKWebView), αντικαταστάθηκε από Tauri, δεν εκτελείται πλέον
- `curator_config.json` — config (inbox, base, models, age_secs, custom_categories)
- `curator_stats.json` — μετρητές (total_sorted, this_month)
- `curator_learned.json` — manual + learned κανόνες
- `curator_undo.log` — JSON-lines ιστορικό για undo
- `curator_cache.db` — SQLite cache αποτελεσμάτων AI

### Tauri app (πηγαίος κώδικας)

`~/Projects/Curator/curator-app/`:
- `src/` — React/TS frontend
  - `App.tsx` — root, state, routing
  - `main.tsx`, `index.css`, `index.html`
  - `lib/curator.ts` — types, helpers (fileKind, relTime, …)
  - `lib/utils.ts` — shadcn `cn` helper
  - `api.ts` — Tauri invoke wrappers + mock data for browser preview
  - `components/`
    - `Sidebar.tsx` — αριστερό nav με logo, items, status footer, theme toggle
    - `Dashboard.tsx` — Overview (stats, chart, recent activity, needs review)
    - `Activity.tsx` — full activity feed με search
    - `Review.tsx` — review cards με thumbnail + Select picker + reveal-file
    - `Rules.tsx` — Learning page με add/remove rules
    - `Folders.tsx` — folders list + New folder + Scan duplicates
    - `Settings.tsx` — folder, AI models, wait threshold
    - `CommandPalette.tsx` — ⌘K Raycast-style
    - `Onboarding.tsx` — first-launch welcome modal
    - `WeekChart.tsx` — recharts bar chart
    - `icons.tsx` — lucide-based file-type icons + nav icons
    - `shared.tsx` — ActivityRow, ActivityFeed, PageHeader
    - `ui/` — shadcn components (button, card, input, select, dialog, command, scroll-area, …)
- `src-tauri/` — Rust backend
  - `src/lib.rs` — όλος ο backend κώδικας
  - `Cargo.toml` — Rust deps (tauri, plugins, notify, sha2, chrono, regex)
  - `tauri.conf.json` — app config (παράθυρο, identifier, updater pubkey)
  - `capabilities/default.json` — permissions
- `package.json` — npm deps
- `vite.config.ts`, `tsconfig.json`

### Δομή Tauri build:
- `target/release/curator-app` — Rust binary
- `target/release/bundle/macos/Curator.app` — bundled .app
- Εγκαθίσταται στο `~/Desktop/Curator.app` (και autostart plist στο `~/Library/LaunchAgents/Curator.plist`)

### Παλαιά / αρχειοθετημένα

- Το παλιό menu bar Python app (`com.haralambos.curator.plist`) απενεργοποιήθηκε.
- Το παλιό `~/Desktop/Curator.app` (Python launcher) μετακινήθηκε στον Κάδο.

---

## 4. Εντολές Rust (Tauri commands)

Όλα στο `src-tauri/src/lib.rs`. Καλούνται από το frontend μέσω `invoke()`.

| Command | Τι κάνει |
|---|---|
| `get_state` | Επιστρέφει AppState (total, pending, month, files, recent, learned, rules, folder) |
| `classify` | Spawns `ai_classifier.py` |
| `sort_file(filename, category)` | Μετακινεί ένα _Review αρχείο σε destination |
| `open_inbox` | `open <inbox folder>` |
| `reveal_file(path)` | `open -R <path>` — επιλέγει το αρχείο στο Finder |
| `undo_last` | Επαναφέρει το τελευταίο move |
| `undo_all` | Επαναφέρει όλα τα recent moves |
| `change_folder` | osascript `choose folder` + ενημερώνει config + launchd plist |
| `get_config` | Διαβάζει `curator_config.json` |
| `save_config(config)` | Γράφει στο `curator_config.json` |
| `list_categories` | Builtin + custom categories |
| `add_category(name)` | Φτιάχνει νέο custom destination |
| `add_rule(pattern, destination)` | Manual κανόνας στο `curator_learned.json` |
| `remove_rule(pattern)` | Διαγραφή κανόνα |
| `import_file(src)` | Drag-drop import: αντιγράφει αρχείο στο inbox |
| `find_duplicates` | SHA-256 σάρωση destinations, groups με 2+ ίδια |

---

## 5. Features (πλήρης λίστα)

### Core
- ✅ Αυτόματη ταξινόμηση (launchd + in-app watcher με lock)
- ✅ AI engine: llava (vision) + llama3.2 (text) μέσω Ollama
- ✅ Fast-path (extension), download-source, learned rules
- ✅ SQLite cache αποτελεσμάτων
- ✅ Timeout στο AI (60s)
- ✅ Auto-rename messy patterns
- ✅ LOW confidence → ext fallback (όχι όλα στο review)
- ✅ Undo (per-action + all)

### Tauri app
- ✅ Menu bar tray με μενού (Open / Organize Now / Quit)
- ✅ Autostart στο login (plugin-autostart)
- ✅ Window-close-to-tray (Accessory policy)
- ✅ Global hotkey ⌘⇧C
- ✅ Single instance
- ✅ Native notifications (batched ανά refresh cycle)
- ✅ Notification → click → app έρχεται μπροστά + auto-navigate σε Review/Activity
- ✅ Light/dark mode toggle (sun/moon στο sidebar footer)

### UI (όλο shadcn)
- ✅ Sidebar με lucide icons (όχι emoji), peacock logo
- ✅ **Overview**: 3 stat cards, «Last 7 days» chart, recent activity, needs review
- ✅ **Activity**: search, grouped by date, hover actions (Undo, Reveal)
- ✅ **Review**: image thumbnails, dropdown με όλες τις κατηγορίες, reveal-file
- ✅ **Learning**: stats, add/remove rules με form
- ✅ **Folders**: list, New folder dialog, Scan duplicates
- ✅ **Settings**: folder, AI models, wait threshold
- ✅ **Command Palette** (⌘K, cmdk-based)
- ✅ **Onboarding** modal first launch
- ✅ Drag & drop αρχείων στο παράθυρο → import + classify
- ✅ Toasts (sonner)
- ✅ Page transitions (framer-motion)
- ✅ Resizable, maximizable window (διορθώθηκε η drag region που τα έκρυβε)
- ✅ Native scroll (κανονικό overflow-y-auto, όχι base-ui ScrollArea)

### Persistence
- `curator_stats.json` — counters
- `curator_learned.json` — rules + corrections
- `curator_undo.log` — JSON-lines, capped στις 500
- `curator_cache.db` — SQLite cache
- `curator_config.json` — folder, models, custom_categories, age_secs, llm_timeout

---

## 6. Πώς το χτίζεις & τρέχεις

**Prerequisites:** Node.js 26+, Rust 1.95+, Xcode CLT, Homebrew, Ollama με τα μοντέλα llava:7b και llama3.2:3b εγκατεστημένα.

```bash
# Frontend deps
cd ~/Projects/Curator/curator-app
npm install

# Type-check + production build (frontend)
npm run build

# Type-check Rust
cd src-tauri && cargo check

# Build .app bundle
cd ..
npm run tauri build -- --bundles app
# Output: src-tauri/target/release/bundle/macos/Curator.app

# Install to Desktop (όπως κάνω εγώ)
cp -R src-tauri/target/release/bundle/macos/Curator.app ~/Desktop/
codesign --force --deep --sign - ~/Desktop/Curator.app

# Refresh icon cache
/System/Library/Frameworks/CoreServices.framework/Versions/A/Frameworks/LaunchServices.framework/Versions/A/Support/lsregister -f ~/Desktop/Curator.app

# Run
open ~/Desktop/Curator.app
```

**Dev server (γρήγορο iteration):**
```bash
cd ~/Projects/Curator/curator-app
npm run tauri dev
```

**Updater signing key:**
- Private: `~/.tauri/curator-signing.key`
- Public: στο `tauri.conf.json` ως `plugins.updater.pubkey`
- Endpoint: placeholder στο `tauri.conf.json` — αντικαθίσταται όταν στηθεί server διανομής

---

## 7. Πώς το δοκιμάζεις

Πλήρες test plan στο `ΗΜΕΡΟΛΟΓΙΟ.md` ή ως follow-up message του chat.
Κύριες περιπτώσεις:
- Drop PDF/image/zip στο inbox → δες πού πάει + notification
- Click notification → app + auto-navigate
- ⌘K → command palette
- ⌘⇧C → άνοιγμα από οπουδήποτε
- Folders → + New folder
- Learning → add rule
- Settings → άλλαξε μοντέλο
- Review → δες thumbnail + Reveal → ανοίγει στο Finder
- Folders → Scan duplicates

---

## 8. Γνωστά όρια / ανοιχτά

- **Auto-updater**: ready αλλά χρειάζεται server με τα updates + Apple Developer cert ($99/χρόνο) για να λειτουργήσει στην πραγματικότητα.
- **Code signing**: ad-hoc μόνο. Για διανομή σε άλλους Mac χρειάζεται Developer ID.
- **Watching folder**: το launchd plist έχει σταθερό WatchPath. Όταν αλλάζεις folder, ξαναγράφεται μέσω `update_watch_plist`. Σε καθαρή εγκατάσταση χωρίς το plist, η αυτόματη ταξινόμηση γίνεται μόνο όσο τρέχει η Tauri app (in-app watcher).
- **Space Quick Look in-app**: δεν υλοποιήθηκε. Ο χρήστης πατάει Reveal → Finder → Space.
- **Multi-folder watching**: μόνο ένας inbox τη φορά.

---

## 9. Φιλοσοφία ανάπτυξης

Από feedback του χρήστη σε αυτή τη σειρά chats:
1. **Όχι emoji στο UI** — μόνο lucide icons + file-type badges. Emoji φαίνονται «AI-generated».
2. **Όχι ψεύτικα κουμπιά / placeholder content** — κάθε στοιχείο πρέπει να κάνει κάτι αληθινό.
3. **Όχι hand-CSS** — χρήση δοκιμασμένου design system (shadcn).
4. **Native-feeling** — Apple style: restraint, λευκός χώρος, χωρίς neon gradients.
5. **Ειλικρίνεια** — όταν κάτι δεν αξίζει ή έχει ρίσκο, λέγεται ευθέως.
6. **Καταγραφή** — κάθε πράξη/απόφαση σημειώνεται στο `ΗΜΕΡΟΛΟΓΙΟ.md`.

---

## 10. Στόχοι (long-term)

- Curator ως **προϊόν που μοιράζεται σε άλλους Mac**:
  - Apple Developer account + signing/notarization
  - Update server (GitHub Release feed)
  - Onboarding που κατεβάζει Ollama + models αυτόματα
- Μελλοντικά features (όχι κρίσιμα):
  - In-app Quick Look (Space για preview)
  - Embedding-based «similar files» αναζήτηση
  - Multi-folder watching
  - Bulk review actions
  - tauri-plugin-dragout (σύρε έξω από το app)

---

## 11. Πώς να συνεχίσεις σε νέο chat

1. **Διάβασε αυτό το αρχείο** + `ΗΜΕΡΟΛΟΓΙΟ.md` πρώτα.
2. Ο φάκελος έργου: `~/Projects/Curator/`
3. Live κώδικας: `~/Library/Scripts/ai_classifier.py` και `~/Projects/Curator/curator-app/`
4. Το `Curator.app` στο Desktop είναι ο τελευταίος καλός build.
5. **Πάντα ενημέρωσε το ΗΜΕΡΟΛΟΓΙΟ** σε κάθε αλλαγή.
6. **Σεβάσου την φιλοσοφία**: shadcn, lucide, no emoji, no fake buttons, ειλικρίνεια.
