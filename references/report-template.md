# Diagnostic Report Template

Use this template when producing a final performance report. Adapt sections as needed based on the available artifacts and the scope of the investigation. Delete placeholder text before delivering.

---

## 1. Executive Summary

<!--
State the core finding in 2–3 sentences. Include: the primary symptom, the root cause, the confidence level, and the single most impactful recommendation.
-->

**Root cause:** [one sentence]
**Confidence:** High | Medium | Low
**Top recommendation:** [one sentence]

---

## 2. Scope and Limitations

| Item | Value |
|:---|:---|
| JDK version | |
| GC algorithm | |
| `-Xmx` / `-Xms` | |
| Container memory limit | |
| Container CPU quota | |
| Concurrency configuration | |
| Recording type | Full / Partial (ring buffer) |
| Analysis date | |

**Known limitations:**

- [ ] One or more recordings are partial (ring buffer truncation)
- [ ] No JMC HTML available for corroboration
- [ ] No access to source code for hotspot correlation
- [ ] Container diagnostics unavailable (non-containerized deployment)

---

## 3. Artifact Inventory

| Artifact | File | Duration/Coverage | Notes |
|:---|:---|:---|:---|
| JFR recording | | | |
| Application log | | | |
| JMC HTML report | | | |
| Source code | | | |

---

## 4. Comparative Evidence by Scenario

<!--
Use this section when comparing multiple runs (e.g., sequential vs concurrent, before vs after).
Add or remove columns as needed.
-->

| Metric | Scenario A | Scenario B | Delta | Notes |
|:---|:---|:---|:---|:---|
| Wall-clock duration | | | | |
| Peak RSS | | | | |
| Peak heap used | | | | |
| Native memory committed | | | | |
| GC pause count | | | | |
| GC total pause time | | | | |
| GC max pause | | | | |
| GC P99 pause | | | | |
| Top allocation class | | | | |
| Top allocation site | | | | |
| Avg JVM CPU load | | | | |
| Top hot method (CPU) | | | | |
| VT pinning events | | | | |
| Exception rate | | | | |

---

## 5. Technical Diagnosis

### 5.1 Memory Analysis

<!--
Describe heap vs native memory breakdown. Include peak values, growth patterns, and dominant allocators.
-->

### 5.2 GC Analysis

<!--
Describe GC behavior: cause distribution, pause characteristics, configuration adequacy.
-->

### 5.3 CPU and Method Analysis

<!--
Describe CPU saturation, hot methods, and whether CPU cost correlates with allocation or I/O.
-->

### 5.4 Concurrency and Threading

<!--
Describe thread behavior: lock contention, virtual thread pinning, thread pool utilization.
-->

### 5.5 I/O and Network

<!--
If applicable, describe file/socket I/O patterns and their contribution to latency.
-->

---

## 6. Causal Model

| Role | Finding | Evidence Source | Confidence |
|:---|:---|:---|:---|
| **Symptom** | | JFR / Logs | |
| **Mechanism** | | JFR | |
| **Code path** | | JFR + Code | |
| **Root cause** | | Code + Tests | |
| **Amplifier** | | JFR + Code | |
| **Ruled out** | | JFR / Logs | |

---

## 7. Recommended Next Steps

| Priority | Recommendation | Expected Impact | Risk | Preserves Guarantee |
|:---|:---|:---|:---|:---|
| P0 | | | | |
| P1 | | | | |
| P2 | | | | |

**Open questions:**

- [ ] [List anything that requires further investigation or a targeted experiment]
