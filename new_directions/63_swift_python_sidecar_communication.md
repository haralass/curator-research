# Swift ↔ Python Sidecar Communication
**File:** 63 | **Date:** 2026-06-01 | **Status:** Research Complete

## Summary / Recommendation
**Localhost HTTP (FastAPI/Flask)** is the winning approach for a Python ML sidecar. It offers the best balance of simplicity, debuggability, and ecosystem support. XPC requires Objective-C bridging and a separate bundle; Unix sockets are faster but require more plumbing. For ML inference workloads where latency is measured in hundreds of milliseconds anyway, HTTP overhead is negligible.

## Prior Art & Evidence

### What real apps use:

**Localhost HTTP:**
- **Tauri + Python sidecar** pattern (documented in Tauri ecosystem) — FastAPI spawned as subprocess, Swift/JS calls via URLSession
- **Whisper.cpp apps** — many use a localhost HTTP server for the inference engine
- **Ollama** — serves local LLMs via localhost HTTP; BoltAI communicates with it via URLSession
- **LM Studio** — OpenAI-compatible HTTP API on localhost:1234

**XPC:**
- Used for privileged helper tools (e.g., installer helpers, VPN clients)
- Requires separate .xpc bundle, MachServices in Info.plist, SMJobBless or launchd
- Overkill for a Python ML sidecar; Python can't directly implement XPC service

**Unix socket / Named pipe:**
- Faster than HTTP (~10-50μs vs ~100-500μs round trip) but difference irrelevant when inference takes 100ms+
- More complex framing (need length-prefix or newline-delimited protocol)
- Used by: Docker daemon, PostgreSQL, Redis (optional) for local connections

**NSTask / subprocess stdout:**
- Simple for one-shot calls but poor for sustained bidirectional communication
- Buffering and synchronization issues under load

## Tradeoffs

| Method | Latency | Complexity | Python ease | Debuggability |
|--------|---------|------------|-------------|---------------|
| Localhost HTTP | ~1-5ms | Low | FastAPI trivial | curl, browser, Proxyman |
| Unix socket | ~0.05ms | Medium | socket stdlib | harder |
| XPC | ~0.1ms | Very High | not native | Instruments |
| NSTask stdout | ~1ms | Low | print() | easy but fragile |
| Named pipe | ~0.1ms | Medium | os.mkfifo | harder |

## Decision for Curator
**Use localhost HTTP with FastAPI.** 

Architecture:
```
SwiftUI App
  └── launches Python sidecar via Process()
      └── FastAPI on 127.0.0.1:PORT (random port, passed via env var)
          ├── POST /embed      — generate embeddings
          ├── POST /cluster    — cluster file embeddings  
          ├── POST /classify   — classify file content
          └── GET  /health     — liveness check
SwiftUI calls via URLSession with async/await
```

Port selection: bind to port 0 (OS assigns), read back assigned port, pass to Swift via stdout first line or env var.

## Sources
- https://fastapi.tiangolo.com
- https://developer.apple.com/documentation/foundation/process
- https://tauri.app/v1/guides/features/sidecar
- https://ollama.com (localhost HTTP pattern)
- Unix socket vs TCP: https://lists.freebsd.org/pipermail/freebsd-performance/2005-February/001143.html
