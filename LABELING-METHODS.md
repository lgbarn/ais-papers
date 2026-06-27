# AIS Papers — How the Data Gets Labeled

Question this answers: for each method in this folder, do you need a third-party
labeling vendor, or is the supervisory signal generated automatically? Short
answer: for the SSL/unsupervised set you need no labels at all. The only genuine
external dependency is verified ship *identity/type* (registries) and, for
xView3, SAR imagery as a second sensor.

---

## Summary — by how each method gets its training signal

### No labels needed — signal comes free from the AIS stream itself
- **Simula AIS-shutdown** (`2310.15586`) — self-supervised: predicts whether the
  next message *should* have arrived. The signal is derived from message timing;
  no human annotation. (Full setup below.)
- **GeoTrackNet** (`1912.00682`) — unsupervised density model; flags
  low-likelihood tracks. No labels.
- **TrAISformer** (`2109.03958`) — forecasting; the "label" is the next
  position, which the trajectory provides for free. Self-labeled by construction.
- **Conv autoencoder** (`2101.03169`) — unsupervised reconstruction. No labels.
- **UniTE survey** (`2407.12550`) — catalog of masked / contrastive /
  autoregressive pretext tasks; all auto-labeled by definition. Your menu of
  "how to pretrain without labels."

### Auto-labeled by rules/heuristics on AIS (programmatic, no vendor)
- **AIS-gap "going dark"** (Welch 2022) — a gap past a threshold while in
  coverage = a disabling event. A reception model separates intentional shutdown
  from satellite coverage loss, so labels are generated, not hand-annotated.
- **Transshipment** (Miller 2018) — rendezvous rule (two vessels <500m, <2kn,
  >2h) auto-tags events, then ML on top.

### Needs a second data source to label (automated, but not AIS-only)
- **xView3 dark vessels** (`2206.00897`) — labeling trick is cross-modal
  correlation: a SAR detection with no matching AIS track = dark vessel.
  Automated, but requires SAR imagery as the second modality; is_vessel /
  is_fishing / length labels also lean on GFW models + human review. (Full setup
  below.)

### Genuine 3rd-party / external dependency
- **Ship type / identity** (MESTR, the RF classifier) — ground-truth type comes
  from the AIS *static* message (self-reported) or vessel registries. "Auto" in
  that the vessel broadcasts it, but externally provided and spoofable.

### Bottom line
For dark-ship detection you can avoid third-party labeling entirely. Two
label-free routes: (1) pure self-supervised/anomaly on AIS alone (Simula or
GeoTrackNet style — anything anomalous is a candidate), or (2) auto-label by
cross-referencing AIS against a second sensor (SAR/optical/radar) — present in
one, absent in the other = dark. You only need an outside source if you want
verified ship identity/type as ground truth.

---

## Detailed: Simula AIS-shutdown labeling setup (`2310.15586`)

**Fully self-supervised.** The model never learns "shutdown" directly — it learns
a *pretext task* and shutdown is inferred afterward by a separate decision rule.

1. **Pretext task.** Given a window of `w` successive AIS messages (w=25),
   predict one binary: will the next AIS message arrive within time frame
   tau (=10 min)? Output is a single sigmoid (message received yes/no).
2. **Target generation (the "labeling").** Each trajectory is auto-labeled from
   the timestamp gap to the next message: arrives within tau -> "received"
   (green); else -> "not received" (red). 500k trajectories/month extracted
   class-balanced. No operator labels exist or are used.
3. **Shutdown vs ordinary gap.** Not done by the model. At run time a logical
   decision module compares prediction to reality: predicted-message-expected +
   none-received-within-tau = abnormal (candidate intentional shutdown, alert);
   predicted-none-expected + none-received = ordinary coverage gap (no alert).
   Trajectories filtered to >5 km from any port (using ~30,000 GFW ports) to
   exclude legal in-port shutdowns; vessels need >=50 messages/year.
4. **Input / architecture.** Per-message vector `[t, lat, lon, speed, dt, dDV,
   dDH, distance-to-port, second-of-day]`, split into normalized message-history
   and a most-recent-position vector (DMS cyclic encoding). Two-block transformer
   encoder -> 64-dim representation -> dense (100/50/50, ReLU, 10% dropout) ->
   one sigmoid (~470,821 params).
5. **Data / ground truth.** Norwegian Statsat satellite AIS, 2020, ~4.05B
   messages, >60,000 vessels; 12 monthly sets of 500k balanced trajectories.
   Evaluation uses the same auto-derived reception labels (4.5M trajectories
   disappearing >10 min: 99.60% PPV, 99.92% NPV). No true shutdown ground-truth
   set exists; only external validation was qualitative — a discovered cluster of
   ~50 fishing vessels repeatedly killing AIS at the Argentina EEZ, later matched
   to a 2021 Oceana report. Exact loss function not specified.

---

## Detailed: xView3-SAR labeling setup (`2206.00897`)

**Hybrid pipeline:** automated CFAR detection + probabilistic AIS-to-SAR matching
(Global Fishing Watch) fused with expert human annotation. Produces 243,018
verified maritime objects across 991 Sentinel-1 scenes. A "dark vessel" is a SAR
detection with no matching AIS broadcast — identified by the detector plus the
*absence* of an AIS match, not by a positive label.

1. **AIS correlation / matching.** Detections from GFW's CFAR + ConvNet detector
   (160k+ detections from 2020). AIS messages matched to SAR detections by a
   probabilistic model scoring {AIS, SAR-detection} pairs using AIS records
   before/after the image timestamp (Kroodsma et al., ref [14]). Exact matching
   radius/time window: not specified in this paper (deferred to ref [14]).
2. **Label taxonomy & derivation.**
   - *Location/detection* — CFAR auto-detection and/or human boxes (437 images
     hand-labeled, ~176,000 manual detections).
   - *is_vessel vs fixed-structure* — AIS correlation + human review + GFW
     vessel/infrastructure DB.
   - *is_fishing* — GFW ConvNet on vessel movement + registries (~99% acc).
   - *vessel length* — GFW CFAR regression estimate.
3. **Human vs automated + confidence tiers.** Labels are AIS-only (27.5%),
   manual-only (33.4%), or both (39.1%). Confidence: HIGH if GFW + labeler agree;
   MEDIUM if a single source; LOW only for human-flagged uncertain vessels.
   Distribution: 38.9% high / 41.2% medium / 19.9% low.
4. **Sensors / scale.** Sentinel-1 IW-mode GRD (VH+VV, ~20 m res, 10 m pixels);
   AIS from GFW; GEBCO 2020 bathymetry; Sentinel-1 OCN wind. 991 scenes
   (~29,400 x 24,400 px), 243,018 objects, 43.2M km^2, 1,421.81 gigapixels.
5. **Evaluation.** Only medium+high-confidence labels count as ground truth.
   AIS-correlated and dark detections are pooled into one positive set, scored
   jointly via F1 (detection, close-to-shore, vessel, fishing) plus length PE.
