# Covariate Shift Detection and Recalibration Triggers for Conformal Prediction

**File:** 98 | **Date:** 2026-06-01 | **Status:** Research Complete

## Summary
Conformal prediction coverage guarantees are valid only when the calibration distribution matches the test distribution (exchangeability assumption). For Curator, distribution shift occurs when the user adds a new domain of files (new job, new project type, language change). The recommended approach is: detect shift using Page-Hinkley test on prediction set sizes (proxy for coverage degradation); trigger recalibration when the test fires; use ADWIN for gradual drift and Page-Hinkley for abrupt change. Recalibration should be triggered automatically but should not block the UI.

## Key Findings
- **Exchangeability assumption:** Mondrian CP guarantees class-conditional 1−ε coverage only when calibration and test distributions are exchangeable. For personal files, this assumption breaks when: (a) user starts a new project domain (e.g., switches jobs, starts home renovation), (b) file naming conventions change (e.g., language change), (c) the Louvain clustering produces new cluster IDs after re-clustering (stale calibration labels).
- **Page-Hinkley test:** Sequential change detection for abrupt shifts in the mean of a stream. Applied to prediction set size stream: if average set size grows from 1.2 to 2.5, it means coverage is degrading. PH threshold: δ=0.01 (sensitivity), λ=5 (alarm threshold). Available in `river` Python library: `river.drift.PageHinkley`.
- **ADWIN (Adaptive Windowing):** Maintains a sliding window; detects gradual drift in prediction set size. Available in `river`: `river.drift.ADWIN`. Use ADWIN for gradual drift (slow accumulation of new file types) and Page-Hinkley for abrupt shift.
- **MMD (Maximum Mean Discrepancy):** Kernel-based two-sample test for distribution shift in the embedding space. More powerful than size-based heuristics but requires storing calibration embeddings. For Curator: compute MMD between stored calibration embeddings and new-file embeddings batch. Threshold: p-value < 0.05 from permutation test. Computationally expensive; run weekly rather than per-file.
- **EnbPI (Ensemble Batch Prediction Intervals):** Online CP variant for regression; not directly applicable to Curator's classification task.
- **Online CP with drift detection:** arXiv:2602.16537 and arXiv:2511.04275 propose online CP algorithms that adaptively update calibration sets using drift detection signals. These achieve minimax-optimal regret under unknown drift rates.
- **Practical recalibration trigger for Curator:**
  1. **Real-time (cheap):** Monitor rolling average of prediction set sizes over last 50 files. If average > 2.0 (target: 1.2 for ε=0.10), trigger recalibration.
  2. **Batch (weekly):** Compute MMD between new files' embeddings and calibration set embeddings. If MMD p-value < 0.05, trigger recalibration.
  3. **Always recalibrate after:** Re-clustering (Louvain re-run produces new cluster IDs → all calibration labels are stale).
- **Recalibration cost:** With MAPIE or crepes, recalibration takes < 1 second for n < 5,000. Run in background; no UI block.

## Relevant Papers / Prior Art
- arXiv:2602.16537, "Optimal training-conditional regret for online conformal prediction," 2026. — Minimax-optimal online CP under drift.
- arXiv:2511.04275, "Online Conformal Inference with Retrospective Adjustment for Faster Adaptation to Distribution Shift," 2025. — Retrospective adjustment for faster drift adaptation.
- arXiv:2602.19790, "Drift Localization using Conformal Predictions," 2026. — Using CP itself as a drift detector.
- Bifet A., Gavalda R., "Learning from Time-Changing Data with Adaptive Windowing," *SDM* 2007. — ADWIN algorithm; canonical reference.
- Page E.S., "Continuous Inspection Schemes," *Biometrika* 41(1/2):100–115, 1954. — Page-Hinkley test; DOI: 10.2307/2333009.

## Applicability to Curator

```python
from river import drift
import numpy as np

class RecalibrationMonitor:
    def __init__(self, set_size_threshold=2.0, window=50):
        self.ph = drift.PageHinkley(threshold=5, min_instances=window)
        self.adwin = drift.ADWIN()
        self.set_size_history = []
        self.threshold = set_size_threshold

    def update(self, prediction_set_size: int) -> bool:
        """Returns True if recalibration should be triggered."""
        self.set_size_history.append(prediction_set_size)
        self.ph.update(prediction_set_size)
        self.adwin.update(prediction_set_size)
        
        if self.ph.drift_detected or self.adwin.drift_detected:
            return True
        
        # Rolling average check
        if len(self.set_size_history) >= 50:
            recent_avg = np.mean(self.set_size_history[-50:])
            if recent_avg > self.threshold:
                return True
        
        return False

    def reset(self):
        self.ph = drift.PageHinkley(threshold=5, min_instances=50)
        self.adwin = drift.ADWIN()
        self.set_size_history = []

# Integration in Curator's prediction pipeline:
monitor = RecalibrationMonitor()

for file in new_files:
    pred_set = cc.predict_set(file.features, significance=0.10)
    if monitor.update(len(pred_set)):
        # Trigger background recalibration
        schedule_recalibration()
        monitor.reset()
```

**Always-recalibrate trigger:**
```python
# After any Louvain re-clustering:
def on_reclustering_complete(new_labels):
    # Calibration set labels are now stale
    cc = ConformalClassifier()  # fresh instance
    # Re-fit and re-calibrate with new cluster labels
    recalibrate_with_new_labels(new_labels)
    monitor.reset()
```

## Open Questions
- Should Curator store the calibration set embeddings in GRDB for MMD computation? This adds ~200KB per 1,000 calibration files (384-dim float32 = 1,536 bytes/embedding). At 1,000 files: 1.5 MB — acceptable.

## Sources
- https://arxiv.org/pdf/2602.16537
- https://arxiv.org/pdf/2511.04275
- https://arxiv.org/pdf/2602.19790
- https://www.sciencedirect.com/science/article/abs/pii/S0959152425000022
- https://www.sciencedirect.com/science/article/pii/S2214212625001577
