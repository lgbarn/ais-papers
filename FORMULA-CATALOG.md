# Formula Catalog for AIS Spoof / Anomaly / Dark-Ship Detection

A menu of named, known metrics that can be applied to AIS position tracks to flag
spoofing, anomalies, or dark activity. Each entry: name, what it measures, what it
flags on AIS. Intended as upfront research — point at a family and ask for it; no
implementation here.

The cross-cutting principle: most "is it straight/regular" metrics (families A, D)
are first-stage filters. They need a second stage (residual noise-floor in C/E, or
a physics check in B/F/H) to separate legitimately-regular from fake-regular. That
filter + verifier pairing is the design pattern to ask for.

## A. Path geometry / straightness
- **Straightness index** (= net displacement / path length; also called path
  efficiency or inverse sinuosity) — how direct the path is — too-perfect tracks.
- **Linear-fit R^2 (coefficient of determination), or PCA eigenvalue ratio
  lambda_1/(lambda_1+lambda_2)** — how well points lie on a straight line —
  unnaturally clean linear motion.
- **Hurst exponent** — persistence vs mean-reversion of the series — synthetic
  tracks push it toward 1 (unnaturally smooth); real GPS noise sits lower.
- **Sinuosity / tortuosity and fractal dimension (box-counting)** — path
  wiggliness — synthetic lines have implausibly low fractal dimension.
- **DFA (detrended fluctuation analysis)** — long-range correlation in residuals
  — real GPS has a characteristic exponent; spoof does not.

## B. Kinematics / physics of the hull
- **Implied speed = haversine(distance)/dt vs hull max** — teleport / jump spoof.
- **Acceleration / jerk** — values beyond physical limits = spoof.
- **Curvature kappa = d(heading)/d(arc), centripetal a = v^2/r** — physically
  impossible turns.
- **Cross-track error (XTE)** from an intended great-circle / rhumb route — route
  deviation.
- **Drift angle = heading - course-over-ground** — always exactly zero = synthetic
  (real ships crab in current and wind).

## C. Statistical outlier / filtering
- **Z-score / modified z-score (MAD-based)** — point outliers in speed, accel, or
  report interval.
- **Mahalanobis distance** — multivariate outlier in feature space.
- **Kalman filter innovation / NIS (normalized innovation squared, chi-square
  test)** — predict-next-position residual; large = anomaly, ~zero = too perfect.
- **CUSUM / change-point detection** — behavior regime shifts (e.g. onset of
  spoofing).
- **Isolation Forest / Local Outlier Factor / One-Class SVM / DBSCAN** —
  unsupervised anomaly and density methods (no labels required).

## D. Information theory / complexity
- **Shannon entropy** of the heading or speed distribution — synthetic = low
  entropy.
- **Approximate / Sample / Permutation entropy** — signal regularity — spoof is
  too regular.
- **Lempel-Ziv complexity / compression ratio** — a quantized track that
  compresses too well = synthetic.

## E. Spectral / time-series
- **FFT / power spectral density of position residuals** — real GPS noise has a
  characteristic spectrum; spoof is flat or absent.
- **Residual autocorrelation (Ljung-Box test)** — white noise (real) vs
  structured or zero (fake).
- **Benford's Law** on coordinate / speed digits — fabricated data often violates
  the first-digit distribution.

## F. Geospatial / contextual
- **Point-in-polygon (on land)** — a ship plotting over land = spoof.
- **Draft vs bathymetry** — a deep-draft vessel in shallow water = impossible.
- **KDE / grid-occupancy likelihood** — a position in a near-zero-traffic cell
  (the core of GeoTrackNet).
- **Nearest-neighbor / co-location** — the same MMSI in two places at once =
  duplicate / spoof.

## G. Trajectory similarity (replay spoofing)
- **DTW (dynamic time warping), Frechet distance, Hausdorff distance** — a track
  that exactly retraces another vessel's historical path = replay attack.

## H. RF / signal physics
- **Radio horizon** `d_km ~= 4.12*(sqrt(h_tx_m)+sqrt(h_rx_m))` — a claimed
  position beyond a terrestrial receiver's VHF line-of-sight = spoof.
- **RSSI vs claimed range; TDOA multilateration; Doppler shift** — independently
  geolocate the transmitter and compare to the claimed lat/lon.

## I. Identity / protocol
- **MMSI / MID and IMO check-digit validation** — malformed identity.
- **Reporting-interval conformance** — Class A AIS report rates depend on
  speed and nav status; timing that is too regular or off-spec = synthetic.
