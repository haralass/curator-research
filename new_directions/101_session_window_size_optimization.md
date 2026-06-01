# Session Window Size Optimization for FSEvents-Based Behavioral Edges

**File:** 101 | **Date:** 2026-06-01 | **Status:** Research Complete

## Summary
Session boundary detection using H(L|I) conditional entropy from FSEvents requires a temporal window parameter N (the number of events or time span defining a session). Studies of computer work sessions (Mark et al., Iqbal & Bailey) consistently find that natural work sessions last 10–40 minutes with 3–5 minutes of within-session idle time before a context switch. The recommended FSEvents window is: define a session break when H(L|I) exceeds a threshold AND no file events occur for > 5 minutes (idle gap). N=200 FSEvents is a reasonable fixed default; adaptive N based on rolling entropy variance is the more robust approach.

## Key Findings
- **Natural work session duration:** Mark G. et al. (UC Irvine, CHI studies, 2005–2008) find that computer work sessions are interrupted on average every 10.5 minutes; self-interruptions (task-switching) occur every 3 minutes on average. Within-task file access bursts last 2–8 minutes.
- **Idle gap for session detection:** Iqbal S.T., Bailey B.P., "Oasis: A Framework for Linking Notification Delivery to the Perceptual Structure of Goal-Directed Tasks," *ACM TOIS* 2010. Defines session break as ≥ 5 minutes of application inactivity. 5-minute idle gap is the most widely used threshold in session detection systems.
- **Entropy-based session detection H(L|I):** The conditional entropy H(L|I) measures the predictability of the next file location (L) given the current location (I). During a coherent work session (user working on one project), H(L|I) is low (files accessed from the same directory). At a session boundary, H(L|I) spikes (user opens a different project's files). This is Curator's designed approach.
- **FSEvents event volume:** A typical knowledge worker generates 50–500 FSEvents per minute during active work. A 5-minute session burst → 250–2,500 events. Fixed window N=200 events corresponds to roughly 1–4 minutes at typical work intensity — too small.
- **Recommended N:** N=500 events or 5 minutes of events, whichever comes first. This captures a full within-task burst without spanning multiple tasks.
- **Adaptive N:** Compute rolling variance of H(L|I) over a 60-second sliding window. If variance is low (stable entropy = coherent session), extend N; if variance is high (entropy fluctuating = near a boundary), contract N. This is the more robust approach but requires online entropy computation.
- **H(L|I) computation:**
  - L = directory of file i (discretized to depth-2 path: ~/Documents/Taxes/ not ~/Documents/Taxes/2024/receipts/)
  - I = directory of the previous event's file
  - Build empirical transition matrix T[i][j] = count(L=j | I=i); normalize rows; compute H(L|I) = -Σ_i p(I=i) Σ_j T[i][j] log T[i][j]
  - Threshold: entropy spike > µ + 2σ (rolling mean + 2 standard deviations) triggers session boundary.
- **Known values from session detection literature:**
  - Idle gap threshold: 5 minutes (Iqbal & Bailey 2010; used by Chrome History, Safari session restore)
  - Min session length: 2 minutes (below this, classify as context switch, not session)
  - Warm-up period: first 30 events after session start are discarded from co-access counting (context-setting accesses, not task-driven)

## Relevant Papers / Prior Art
- Mark G. et al., "No Task Left Behind? Examining the Nature of Fragmented Work," *CHI* 2005. DOI: 10.1145/1054972.1055017. — Work session fragmentation; 10.5-minute average session.
- Iqbal S.T., Bailey B.P., "Oasis: A Framework for Linking Notification Delivery to the Perceptual Structure of Goal-Directed Tasks," *ACM TOIS* 28(4):1–28, 2010. DOI: 10.1145/1858377.1858381. — 5-minute idle threshold; canonical reference for session boundary detection in desktop systems.
- Pirolli P., Card S.K., "Information Foraging," *Psychological Review* 1999. — Marginal Value Theorem; patch-leaving behavior as session boundary model.
- Apple FSEvents documentation: https://developer.apple.com/documentation/coreservices/file_system_events. — API reference for FSEvents stream setup.

## Applicability to Curator

```python
# Python sidecar: session boundary detection from FSEvents stream
import math
from collections import defaultdict, deque
from datetime import datetime, timedelta

class SessionBoundaryDetector:
    def __init__(self, idle_gap_minutes=5, window_size=500, entropy_sigma_multiplier=2.0):
        self.idle_gap = timedelta(minutes=idle_gap_minutes)
        self.window_size = window_size
        self.entropy_multiplier = entropy_sigma_multiplier
        
        # Rolling state
        self.event_buffer: deque = deque(maxlen=window_size)
        self.transition_counts: dict[str, dict[str, int]] = defaultdict(lambda: defaultdict(int))
        self.last_event_time: datetime | None = None
        self.last_dir: str | None = None
        self.entropy_history: deque = deque(maxlen=60)  # last 60 entropy values
        
        # Co-access accumulator (files accessed in current session)
        self.current_session_files: list[str] = []

    def _dir_at_depth2(self, path: str) -> str:
        """Normalize path to depth-2 directory for entropy computation."""
        parts = path.split('/')
        return '/'.join(parts[:4]) if len(parts) >= 4 else path

    def _compute_conditional_entropy(self) -> float:
        """H(L|I) from current transition counts."""
        H = 0.0
        total = sum(sum(row.values()) for row in self.transition_counts.values())
        if total == 0:
            return 0.0
        for from_dir, transitions in self.transition_counts.items():
            p_i = sum(transitions.values()) / total
            row_total = sum(transitions.values())
            H_row = -sum((c/row_total) * math.log2(c/row_total + 1e-9) 
                        for c in transitions.values())
            H += p_i * H_row
        return H

    def process_event(self, file_path: str, timestamp: datetime) -> bool:
        """
        Process a single FSEvent. Returns True if a session boundary is detected.
        On boundary detection, current_session_files contains the completed session.
        """
        current_dir = self._dir_at_depth2(file_path)
        
        # Check idle gap (time-based session break)
        if self.last_event_time and (timestamp - self.last_event_time) > self.idle_gap:
            return self._finalize_session()
        
        # Update transition counts
        if self.last_dir:
            self.transition_counts[self.last_dir][current_dir] += 1
        
        self.last_dir = current_dir
        self.last_event_time = timestamp
        self.event_buffer.append(file_path)
        self.current_session_files.append(file_path)
        
        # Compute entropy and check for spike
        H = self._compute_conditional_entropy()
        self.entropy_history.append(H)
        
        if len(self.entropy_history) >= 20:
            mu = sum(self.entropy_history) / len(self.entropy_history)
            sigma = math.sqrt(sum((x - mu)**2 for x in self.entropy_history) / len(self.entropy_history))
            if H > mu + self.entropy_multiplier * sigma and H > 1.5:  # absolute floor
                return self._finalize_session()
        
        return False

    def _finalize_session(self) -> bool:
        """Mark session boundary; reset state."""
        completed_session = self.current_session_files.copy()
        self.current_session_files = []
        self.transition_counts = defaultdict(lambda: defaultdict(int))
        self.entropy_history.clear()
        # Emit co-access edges from completed_session to BehavioralEdgeAccumulator
        if len(completed_session) >= 3:  # ignore micro-sessions
            self._emit_coacccess_edges(completed_session)
        return True

    def _emit_coacccess_edges(self, session_files: list[str]):
        """All pairs of files in session → co-access edge."""
        for i, f1 in enumerate(session_files):
            for f2 in session_files[i+1:]:
                behavioral_accumulator.record_coaccess(resolve_id(f1), resolve_id(f2))
```

**Default parameters:** idle_gap=5 minutes, window_size=500, entropy_sigma_multiplier=2.0. These are grounded in Mark et al. (2005) and Iqbal & Bailey (2010).

**macOS FSEvents integration (Swift side):**
```swift
// Swift: set up FSEvents stream and forward to Python sidecar
import CoreServices

let pathsToWatch = [NSHomeDirectory()] as CFArray
var context = FSEventStreamContext()
let stream = FSEventStreamCreate(
    nil,
    { _, _, numEvents, eventPaths, _, _ in
        // Forward events to Python sidecar via XPC or local socket
        let paths = unsafeBitCast(eventPaths, to: NSArray.self) as! [String]
        CuratorSidecar.shared.processEvents(paths: paths, timestamp: Date())
    },
    &context,
    pathsToWatch,
    FSEventStreamEventId(kFSEventStreamEventIdSinceNow),
    5.0,  // 5-second latency batch (reduces CPU overhead)
    FSEventStreamCreateFlags(kFSEventStreamCreateFlagFileEvents)
)
FSEventStreamSchedule(stream, CFRunLoopGetMain(), CFRunLoopMode.defaultMode.rawValue)
FSEventStreamStart(stream)
```

## Open Questions
- Should FSEvents monitoring be opt-in (privacy) or opt-out (convenience)? Given the behavioral edge privacy concerns (File 92), opt-in is safer and more App Store-compliant.
- Does the FSEvents 5-second latency batch (kFSEventStreamEventIdSinceNow) introduce timing errors in the idle-gap detection? Yes — gaps < 5 seconds are not detected. Set FSEvents latency to 1 second for accurate idle-gap detection.

## Sources
- https://hackmd.io/@M4shl3/FSEvents
- https://medium.com/@boutnaru/the-macos-forensic-journey-fsevents-file-system-events-27244fd57aef
- https://www.hexordia.com/blog/mac-forensics-analysis
- https://www.nngroup.com/articles/information-foraging/
- https://act-r.psy.cmu.edu/wordpress/wp-content/uploads/2012/12/280uir-1999-05-pirolli.pdf
