# Spoof Detection from Track-Straightness and Fit-Residual Metrics

Idea: use a cheap straightness/regularity metric as a first-stage filter on AIS
tracks, then separate "legitimately straight" from "spoofed-straight" with a
residual noise-floor test. Two stages: a high-recall filter feeding a precise
verifier.

## Premise and the correction

The naive rule "steady/straight track = spoofing" is backwards. Open-ocean
transits (great-circle / rhumb-line) are legitimately straight, straightness
~0.999, so that rule flags the whole merchant fleet. The real signal is not
*straight*, it is *too perfect* — residual variance below what GPS jitter + sea
state + steering noise physically allow. A real ship at 0.999 still carries
meter-scale GPS jitter and small heading wander; a spoofed line carries ~zero.

## The straightness and fit metrics

Compute over a sliding window of N position fixes.

**Straightness index** (a.k.a. path efficiency, inverse sinuosity):

```
straightness = net_displacement / path_length
             = haversine(start, end) / sum(haversine(p_i, p_i+1))
```

1.0 = dead straight, ~0 = wandering.

**Linear-fit R^2 (coefficient of determination)** — how well the window's points
lie on a straight line. The 2-D version is PCA on the (x, y) fixes projected to
local meters:

```
R2_path = lambda_1 / (lambda_1 + lambda_2)   # variance fraction on principal axis
```

R2_path ~ 1 => points lie on a line. The same R^2 can be run on the
speed-over-ground and heading series directly (1-D) — constant speed-over-ground
to many decimals is a synthetic tell.

## The two-stage detector

- **Stage 1 (filter):** straightness / R2_path selects "suspiciously
  straight/smooth" windows. High recall, many false positives (legit transits).
- **Stage 2 (verifier):** on the flagged windows, test residual RMSE / heading
  entropy against the expected GPS-noise floor. Real-straight -> noisy residuals;
  synthetic-straight -> near-zero. This stage actually separates spoof from a
  normal transit.

Net: straightness and R^2 decide "straight," not "fake." The discriminator is the
residual below the physical noise floor on the windows they flag.

## Relation to the repo papers

None of the included papers use this. Simula uses message-arrival timing only;
GeoTrackNet learns a spatial density instead of hand formulas; xView3 sidesteps
spoofing via SAR. So the straightness + residual-noise-floor detector is a gap in
the current set — candidate to prototype.

## Adjacent physics checks (for completeness)

- Kinematic plausibility: implied speed/accel/turn-rate from haversine(d)/dt
  beyond hull limits => teleport/jump spoof (the opposite failure mode from
  straight).
- Radio line-of-sight: AIS is marine VHF radio (ch 87B/88B, ~162 MHz),
  line-of-sight range ~20-40 nm; a claimed position beyond a terrestrial
  receiver's radio horizon `d_km ~= 4.12*(sqrt(h_tx_m)+sqrt(h_rx_m))` is
  inconsistent => spoof. TDOA / multilateration across receivers geolocates the
  transmitter independently.
- Identity: duplicate MMSI in two places at once, MID/flag mismatch, invalid IMO.

## Next step

Prototype: straightness + R2_path + residual-RMSE over a sliding window, with a
self-check separating a synthesized real-ish straight track (GPS jitter) from a
perfect spoof line. Needs a sample AIS CSV (Danish Maritime Authority or US
Marine Cadastre) or synthetic tracks.
