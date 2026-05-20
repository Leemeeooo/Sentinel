# Sentinel
Probabilistic physical context signal that uses the physics of signal propagation to establish individual probabilistic location without requiring GPS API, IP location servers.

*Built inside Google Apps Script by treating platform limitations as architectural constraints.*

---

## The Problem

GPS can be spoofed with $500 hardware. IP geolocation databases are trivially defeated by residential VPNs. Timezone checks are bypassed by VPN clients that sync system clocks. Behavioral biometrics require invasive baselining.

None of these methods exploit the one thing that cannot be faked: **the finite speed of light in optical fiber.**

A signal from Brussels to Johannesburg cannot arrive faster than ~45ms one-way, regardless of software optimization, because the path length is fixed and the speed of light is a physical constant.

Sentinel measures this.

---
## Scope and Limitations

**The latency values used in this repository are simulated.** This is a proof of concept, not a field measurement. 

The goal was to see if the math pipeline, from ECEF multilateration to geodetic output, using only vanilla linear algebra and no external libraries, survives the constraints of Google Apps Script. Whether it holds against live RTT data is a question I'm leaving open for someone with the infrastructure to test.

**What this means:**
- The physics is sound
- The solver works within GAS constraints
- The beacon network architecture is designed for real deployment
- But the numbers in the worked examples are synthetic

**What this does not mean:**
- The system is untested in principle, the math pipeline is validated
- The constraints are insurmountable, they are documented and designed around
- The project is incomplete, it is a feasibility study with a clear path to production.

---

## What This Is (And Isn't)

**Sentinel is a signal, not a proof.**

It does not produce an unspoofable GPS. It produces a confidence score that answers:

> *"How consistent are these network measurements with the previously established baseline context?"*

The output is a threat score from 0.0 (perfectly consistent) to 1.0 (critically anomalous). Administrators configure thresholds:

| Score | Status | Action |
|-------|--------|--------|
| 0.00–0.39 | HEALTHY | UNLOCK |
| 0.40–0.69 | MEDIUM ANOMALY | CHALLENGE (2FA required) |
| 0.70–1.00 | HIGH ANOMALY | LOCK (manual review) |

A signal can be noisy, that is, it can have false positives, and can be one input among many. A proof implies cryptographic certainty. 
Sentinel is explicitly the former.

---

## The Architecture: Three Tiers, One Constraint Philosophy

Sentinel is built on Google Apps Script (GAS), not because GAS is the ideal platform for network multilateration, but because GAS is where the protected assets live. The protection layer must exist inside the same trust boundary.

Please Note: This is **constraint-driven design**, not technology-driven design.

### Browser (The Sensor)
- RTT measurement via `performance.now()` + `fetch()`
- Parallel beacon scanning
- Physics filter isolating minimum-queuing samples

### GAS (The Glue)
- User interface and identity (Workspace SSO)
- Sheet protection and access control
- Audit logging to Google Sheets
- Baseline storage in PropertiesService
- Threat scoring arithmetic (fast, stateless)

### GCP (The Muscle)
- Beacon servers with static IPs and predictable routing
- Minimal HTTP heartbeat endpoints (no computation, no logging)

**The insight:** Push computation to the browser, push persistence to the sheet, keep the server lean and stateless. Every GAS limitation becomes a design boundary that shapes the solution.

---

## The Physics

Network packets travel through optical fiber at approximately **2.0 × 10⁸ m/s**,  not the vacuum speed of light. Fiber paths follow railway rights-of-way and undersea trenches, typically 1.2–1.5× the great-circle distance.

Using the vacuum speed of light would overestimate distances by ~50%, producing systematic errors of hundreds of kilometers. This is the single most important constant in the system.

A measured RTT includes:
- **Propagation** (50–80% of RTT for long-haul)
- **Transmission** (negligible for small packets)
- **Processing** (~0.1–0.2ms per router hop)
- **Queuing** (highly variable - filtered via repeated sampling)

The system absorbs asymmetric routing errors (10–30% of RTT) into its confidence metric rather than modeling them explicitly.

---

## Mathematics: Spherical Multilateration

Given N beacons at known positions and measured ranges, the receiver position satisfies:

```
(x - xᵢ)² + (y - yᵢ)² + (z - zᵢ)² = rᵢ²
```

Linearizing via reference subtraction yields a 3×3 system solved via Cramer's rule. All positions are computed in ECEF coordinates before conversion back to geodetic (WGS-84).

Post-solve residual analysis validates consistency:
- **RMSE < 50 km:** Excellent
- **RMSE 50–300 km:** Moderate (routing asymmetry expected)
- **RMSE 300–1000 km:** Suspicious (possible VPN/anycast)
- **RMSE > 1000 km:** Anomaly (physically inconsistent)

---

## What It Can Detect

- **Cross-continent VPN:** Massive geospatial drift + high RMSE
- **Satellite internet:** Low drift but very high RMSE (solved near ground station)
- **CDN/anycast fronting:** Clustered RTTs across diverse beacons
- **Selective beacon blocking:** Detected via beacon loss signal
- **Fabricated RTTs:** Impossible geometry producing extreme RMSE

## What It Cannot Detect (Fundamental Limitations)

- **Same-city VPN:** Propagation difference < 1ms, below measurement resolution
- **Residential proxies:** Local network paths produce consistent measurements
- **Compromised legitimate devices:** All measurements remain valid
- **Advanced timing attacks:** Nation-state capability required

These are **features of the threat model, not bugs to fix.**

---

## The Platform: Why GAS and What It Costs

GAS has hard limitations that fundamentally shape the design:

| Limitation | Impact | Design Response |
|------------|--------|-----------------|
| 6-minute execution | No long-running analysis | Heavy computation in browser |
| 500 KB PropertiesService | ~500-800 user baselines max | Hybrid: PropertiesService for speed, Sheets for queries |
| No WebSockets | No real-time monitoring | Pull-based architecture |
| No persistent background processes | No continuous watchdog | Time-driven triggers for periodic checks |
| CORS/iframe sandboxing | No external JS libraries | Inline all code; accept simpler UI |
| Limited crypto APIs | No AES/RSA/ECDSA | HMAC-SHA256 for beacon signatures |
| Cold start latency (1–3s) | Problematic for time-sensitive flows | Acceptable for scan-based use case |

**Every limitation is a decision boundary.** Crossing it means leaving GAS. Staying within it means accepting tradeoffs and building something that works well enough, where "well enough" is defined by the threat model.

---

## Security Analysis

### The Client-Side Trust Problem

Measurements are performed in the browser and transmitted to the server. A malicious client can fabricate arbitrary payloads. The server has no cryptographic proof that reported RTTs were actually measured.

**Acknowledged limitation.** Sentinel is a signal, not proof.

### Attack Surfaces

| Surface | Mitigation |
|---------|------------|
| Beacon compromise | Topology signature hashing |
| Man-in-the-middle | HTTPS with certificate pinning |
| Client code modification | CSP/SRI (limited in GAS) |
| Denial of service | Rate limiting at infrastructure level |

---

## The Manuscript

The full technical manuscript is available in this repository:
- **Physics of propagation** (speed of light in fiber, latency budgets)
- **Mathematical foundation** (ECEF conversions, linearization, Cramer's rule)
- **Security analysis** (detection capabilities, fundamental limitations)
- **Platform architecture** (constraint-driven design philosophy)

**Note:** Operational details (exact beacon coordinates, production code, deployment playbooks, and internal calibration data) are withheld for security. The manuscript focuses on methodology and architecture.

---

## Future Work

- WebRTC-based timing for NATed/proxied clients
- NTP stratum correlation for satellite link detection
- Machine learning on per-user RTT patterns (with caution: ML can be gamed)
- Beacon-signed responses with timestamp/nonce validation

---

## License

MIT License — See [LICENSE](LICENSE)

---

## Citation

If you use this work in research, please cite:

```bibtex
@misc{sentinel2026,
  title={Sentinel: Probabilistic Physical Context Signal via Network Multilateration},
  author={Elizabeth Efeelobari Letam},
  year={2026},
  howpublished={\\url{https://github.com/leemeeooo/sentinel}},
  note={Accessed: 2026-05-20}
}
```

---

## Contact

For questions, collaborations, or security disclosures: efeelobarielizabeth@gmail.com

---

*"The question was never 'What is the best platform for this?' The question was 'Given that the assets live in Google Workspace, what is the best protection layer we can build without leaving that boundary?'"*
