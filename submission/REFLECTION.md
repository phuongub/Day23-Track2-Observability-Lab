# Day 23 Lab Reflection

**Student:** Trần Việt Phương
**Submission date:** 2026-05-11
**Lab repo URL:** https://github.com/phuongub/Day23-Track2-Observability-Lab.git
---

## 1. Hardware + setup output

Paste output of `python3 00-setup/verify-docker.py`:

```json
{
  "docker": {
    "ok": true,
    "version": "29.4.0"
  },
  "compose_v2": {
    "ok": true,
    "version": "5.1.1"
  },
  "ram_gb_available": 7.35,
  "ram_ok": true,
  "required_ports": [
    8000, 9090, 9093, 3000, 3100, 16686, 4317, 4318, 8888
  ],
  "bound_ports": [],
  "all_ports_free": true
}
```

---

## 2. Track 02 — Dashboards & Alerts

### 6 essential panels (screenshot)

Drop `submission/screenshots/dashboard-overview.png`.

### Burn-rate panel

Drop `submission/screenshots/slo-burn-rate.png`.

### Alert fire + resolve

| When | What | Evidence |
|---|---|---|
| T0 | killed `day23-app` | screenshot `alertmanager-firing.png` |
| T0+90s | `ServiceDown` fired | Alertmanager UI shows firing |
| T1 | restored app | `make alert` script auto-restores |
| T1+60s | alert resolved | Alertmanager UI shows resolved |

> Note: Slack webhook uses placeholder URL (`REPLACE/ME/HERE`) because no free Slack workspace was configured. Alertmanager UI screenshots serve as fire/resolve evidence. In production this would route to `#observability` via `slack_api_url`.

### One thing surprised me about Prometheus / Grafana

Dashboards loaded but all panels showed "No data" even though Prometheus was scraping successfully. Root cause: the datasource provisioning YAML omitted `uid: prometheus`, so Grafana auto-generated a UID that didn't match the hardcoded `uid: "prometheus"` in dashboard JSON. Dashboards-as-code is powerful, but UID coupling between datasource and panel target is a silent failure mode.

---

## 3. Track 03 — Tracing & Logs

### One trace screenshot from Jaeger

Drop `submission/screenshots/jaeger-trace.png` showing `embed-text → vector-search → generate-tokens` spans under `POST /predict`.

### Log line correlated to trace

Paste the log line and the trace_id it links to:

```json
{"model": "llama3-mock", "input_tokens": 4, "output_tokens": 54, "quality": 0.82, "duration_seconds": 0.1761, "trace_id": "22d41b49c94cba2e5f2442802814d366", "event": "prediction served", "level": "info", "timestamp": "2026-05-11T14:28:14.545933Z"}
```

Trace ID: `22d41b49c94cba2e5f2442802814d366`

### Tail-sampling math

Policy configured in `otel-config.yaml`:
- **keep-errors**: status_code = ERROR → 100% kept
- **keep-slow**: latency > 2000 ms → 100% kept
- **probabilistic-1pct**: 1% of healthy traces kept

If the service produces N traces/sec with a 1% error rate:
- Error traces kept: `0.01N × 1.0 = 0.01N`
- Slow traces kept (assume <1%): `≈ 0`
- Healthy traces kept: `0.99N × 0.01 = 0.0099N`
- **Total kept: ~0.02N** (≈ 2% of all traces)
- **Dropped: ~98%** cost reduction while retaining all errors.

For the lab demo the probabilistic rate was temporarily raised to 100% so the healthy trace would appear in Jaeger for the screenshot; in production it stays at 1%.

---

## 4. Track 04 — Drift Detection

### PSI scores

Paste `04-drift-detection/reports/drift-summary.json`:

```json
{
  "prompt_length": {
    "psi": 3.461,
    "kl": 1.7982,
    "ks_stat": 0.702,
    "ks_pvalue": 0.0,
    "drift": "yes"
  },
  "embedding_norm": {
    "psi": 0.0187,
    "kl": 0.0324,
    "ks_stat": 0.052,
    "ks_pvalue": 0.133853,
    "drift": "no"
  },
  "response_length": {
    "psi": 0.0162,
    "kl": 0.0178,
    "ks_stat": 0.056,
    "ks_pvalue": 0.086899,
    "drift": "no"
  },
  "response_quality": {
    "psi": 8.8486,
    "kl": 13.5011,
    "ks_stat": 0.941,
    "ks_pvalue": 0.0,
    "drift": "yes"
  }
}
```

### Which test fits which feature?

| Feature | Best test | Why |
|---|---|---|
| `prompt_length` | **KS** | Continuous count data; KS is non-parametric and detects any distributional shift without binning assumptions. |
| `embedding_norm` | **KS** | 1D continuous scalar derived from high-D embeddings; KS handles the tail behavior well. MMD would be better if we compared raw embedding vectors. |
| `response_length` | **PSI** | Discrete/count-like; PSI works well when values can be binned into quantiles. |
| `response_quality` | **PSI** | Score in [0,1], naturally binned; PSI is the industry standard for scorecard drift. |

> Note: Evidently HTML report generation failed due to Python 3.14 incompatibility (`np.float_` removed in NumPy 2.0). The JSON summary above satisfies checkpoint 16; checkpoint 17 is documented but unscreenshotted due to dependency issue.

---

## 5. Track 05 — Cross-Day Integration

### Which prior-day metric was hardest to expose? Why?

Day 20's llama.cpp serving metrics would be hardest. llama.cpp's built-in HTTP server doesn't emit Prometheus-format metrics natively — you need to patch in a sidecar or scrape its JSON stats endpoint and translate. The stub script (`monitor-day20-llama-cpp.py`) emits `day20_llamacpp_tokens_per_second` and `day20_llamacpp_queue_depth`, but wiring real llama.cpp would require either modifying the C++ server or running a separate metrics translator process.

Day 19 Qdrant is easier — it already exposes `/metrics` in Prometheus format if you enable the telemetry API.

---

## 6. The single change that mattered most

The most impactful single change was **fixing the Grafana datasource UID mismatch**.

`datasources.yml` provisioned Prometheus without an explicit `uid`, so Grafana auto-generated one. The three Day-23 dashboard JSON files hardcoded `"uid": "prometheus"`. Result: every panel queried a non-existent datasource and rendered "No data" even though Prometheus was scraping correctly and the queries themselves were valid.

This cost more debugging time than any other issue because the failure was silent — no error banners, just empty panels. The fix was a one-line addition (`uid: prometheus` in `datasources.yml`) plus recreating the Grafana container. It taught me that "dashboards-as-code" is only as robust as the cross-file contract between provisioning config and dashboard JSON. If I were designing this from scratch, I'd generate both files from a single source of truth (a small Python/Terraform script) so UIDs can never drift.

This connects directly to the deck's §4 "Grafana + Dashboards-as-Code" principle: provisioning removes click-ops, but you still need to validate the wiring. A future improvement would be a CI check (`make lint-dashboards`) that validates every `datasource.uid` in dashboard JSON against the actual provisioned datasources.
