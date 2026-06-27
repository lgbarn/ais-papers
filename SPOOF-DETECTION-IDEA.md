# Spoof Detection from Trading Trend Indicators (VHF / R-Squared)

Idea: reuse trend-strength indicators from technical analysis (Vertical Horizontal
Filter, R-Squared) as a cheap first-stage filter on AIS tracks, then separate
"legitimately straight" from "spoofed-straight" with a residual noise-floor test.
Framed as meta-labeling (AFML).

## Premise and the correction

Naive rule "steady/straight track = spoofing" is backwards: open-ocean merchant
transits (great-circle / rhumb-line) are legitimately straight, straightness
~0.999. That rule flags the whole fleet. The real signal is not *straight*, it is
*too perfect* — residual variance below what GPS jitter + sea state + steering
noise physically allow. Real ship at 0.999 still carries meter-scale GPS jitter
and small heading wander; a spoofed line carries ~zero.

## Trading indicators -> trajectory geometry

Compute over a sliding window of N fixes (the "lookback", like bars).

**VHF (Vertical Horizontal Filter)** = range / sum(|increments|). On a 2-D track:

```
straightness = net_displacement / path_length
             = haversine(start, end) / sum(haversine(p_i, p_i+1))
```

This equals the straightness/efficiency index from movement ecology. 1.0 = dead
straight, ~0 = wandering. Same intuition as VHF (high = trending, low = choppy).

**R-Squared** -> straight-line fit quality of the window's points. 2-D version is
PCA on the (x, y) fixes projected to local meters:

```
R2_path = lambda_1 / (lambda_1 + lambda_2)   # variance fraction on principal axis
```

R2_path ~ 1 => points lie on a line. Also run plain 1-D R^2 / VHF on the
speed-over-ground and heading series directly (existing TA code applies
unchanged) — constant SOG to many decimals is a synthetic tell.

## The two-stage detector (meta-labeling)

- **Primary filter:** VHF / R2_path selects "suspiciously straight/smooth"
  windows. High recall, many false positives (legit transits).
- **Secondary model:** on those flagged windows, test residual RMSE / heading
  entropy against the expected GPS-noise floor. Real-straight -> noisy residuals;
  synthetic-straight -> near-zero. This is the stage that actually separates
  spoof from a normal transit.

Net: VHF and R^2 are the right first stage (cheap, code already exists), but they
decide "straight," not "fake." The discriminator is the residual below the
physical noise floor on the windows they flag.

## Relation to the repo papers

None of the included papers use this. Simula uses message-arrival timing only;
GeoTrackNet learns a spatial density instead of hand formulas; xView3 sidesteps
spoofing via SAR. So the VHF/R^2 + residual-noise-floor detector is a gap in the
current set — candidate to prototype.

## Adjacent physics checks (for completeness, not TA-based)

- Kinematic plausibility: implied speed/accel/ROT from haversine(d)/dt beyond
  hull limits => teleport/jump spoof (the opposite failure mode from straight).
- VHF-radio line-of-sight: AIS is marine VHF (ch 87B/88B, ~162 MHz), LOS range
  ~20-40 nm; claimed position beyond a terrestrial receiver's radio horizon
  `d_km ~= 4.12*(sqrt(h_tx_m)+sqrt(h_rx_m))` is inconsistent => spoof. TDOA /
  multilateration across receivers geolocates the transmitter independently.
- Identity: duplicate MMSI in two places at once, MID/flag mismatch, invalid IMO.

## Next step

Prototype: VHF + R2_path + residual-RMSE over a sliding window, with a `demo()`
self-check separating a synthesized real-ish straight track (GPS jitter) from a
perfect spoof line. Needs a sample AIS CSV (Danish Maritime Authority or US
Marine Cadastre) or synthetic tracks.
