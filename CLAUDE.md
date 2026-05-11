# CLAUDE.md

This file provides guidance to Claude Code (claude.ai/code) when working with code in this repository.

## Project Overview

This is a **7-service Docker Compose lab** that builds the open-source observability stack for a mock AI inference service. It is the integrative day of AICB Phase 2, Track 2, wiring telemetry across all prior days (16-22).

The stack comprises: Prometheus, Grafana, Loki, Jaeger, Alertmanager, OpenTelemetry Collector, and an instrumented FastAPI app.

## Common Commands

All orchestration is via the **Makefile** (`C:\Users\LEGION\Day23-Track2-Observability-Lab\Makefile`). There is no `npm`, `pytest`, or traditional build system.

| Command | Description |
|---------|-------------|
| `make setup` | One-time: copy `.env`, pull 6 Docker images, run `verify-docker.py` |
| `make up` | Start the 7-service stack via `docker compose up -d` |
| `make down` | Stop the stack (preserves volumes) |
| `make restart` | `make down` + `make up` |
| `make logs` | Tail logs from all services |
| `make smoke` | Health-check all 7 services via `curl` |
| `make load` | Run locust load test (10 users, 60s) against `http://localhost:8000` |
| `make alert` | Kill app, wait 90s for alert, restore, wait for resolve |
| `make trace` | Generate one traced `POST /predict` request and print its `trace_id` |
| `make drift` | Run drift detection script (PSI / KL / KS) |
| `make demo` | End-to-end: load, alert, trace, drift |
| `make verify` | **Rubric gate** — exits 0 only if all 10 checkpoints pass |
| `make lint-dashboards` | Validate Grafana dashboard JSON files |
| `make clean` | `docker compose down -v` (DESTRUCTIVE — removes volumes) |

### Running a Single Test / Verification

There is no traditional test suite. To run a specific verification step, invoke the relevant `make` target or the underlying script directly:

- `python3 scripts/verify.py` — full rubric gate
- `python3 scripts/lint-dashboards.py 02-prometheus-grafana/grafana/dashboards/*.json`
- `curl -fsS http://localhost:8000/healthz` — app health
- `curl -sS http://localhost:8000/metrics | grep inference_requests_total` — metric presence

## High-Level Architecture

The system is a **mock LLM inference service** instrumented with the Three Pillars of Observability (metrics, traces, logs), plus SLO-based alerting and statistical drift detection.

### Data Flows

```
┌─────────────┐     OTLP/gRPC     ┌──────────────────┐     OTLP/gRPC    ┌──────────┐
│  FastAPI App │ ────────────────→ │  OTel Collector  │ ───────────────→ │  Jaeger  │
│  (day23-app) │                   │  (tail-sampling)  │                 │ (traces) │
│  :8000       │ ←─────────────── │  :4317/:4318     │                 │ :16686   │
└──────┬───────┘     /metrics      └──────────────────┘                 └──────────┘
       │                                     │
       │                                     │ (filelog receiver, not wired by default)
       │                                     ↓
       │                              ┌──────────┐
       │                              │   Loki   │
       │                              │  (logs)  │
       │                              │  :3100   │
       │                              └──────────┘
       │
       │  /metrics scrape (15s)
       ↓
┌──────────────┐     alert rules    ┌───────────────┐     webhook     ┌─────────┐
│  Prometheus  │ ────────────────→ │  Alertmanager  │ ─────────────→ │  Slack  │
│  :9090       │                   │  :9093         │                └─────────┘
└──────┬───────┘                   └────────────────┘
       │
       │  datasource
       ↓
┌──────────────────────────────────────────────────────────────┐
│                        Grafana :3000                         │
│  Dashboards: ai-service-overview, slo-burn-rate, cost-tokens │
│  Datasources: Prometheus, Loki, Jaeger                       │
└──────────────────────────────────────────────────────────────┘
```

### Key Design Decisions

- **Metrics-as-code and dashboards-as-code**: Prometheus rules and Grafana dashboard JSON are committed to the repo and mounted read-only into containers.
- **Mock inference with seeded randomness**: The FastAPI app (`01-instrument-fastapi/app/`) simulates LLM latency, token counts, and quality scores deterministically so results are reproducible for grading.
- **Tail-sampling in OTel Collector**: Configured to keep all errors + slow traces + 1% of healthy traces, achieving ~97% cost reduction while retaining signal.
- **Loki is not receiving logs by default**: The stack intentionally omits Promtail / filelog receiver to stay at 7 services and avoid cross-platform bind-mount fragility. Adding it is a common homework extension.
- **Stub scrapers for prior days**: `05-integration/` contains scripts that emit fake metrics for Days 16-22 if those systems are not running, ensuring the cross-day dashboard always renders.

### Metric Families (exposed at `/metrics`)

- `inference_requests_total{model,status}` — Counter (RED: rate + errors)
- `inference_latency_seconds_bucket{model}` — Histogram (RED: duration)
- `inference_active_gauge` — Gauge (USE: in-flight requests)
- `gpu_utilization_percent` — Gauge (USE: GPU util, simulated)
- `inference_tokens_total{model,direction}` — Counter (AI-specific 4th pillar)
- `inference_quality_score{model}` — Gauge (eval-as-metric stub)

### Grading Verification

The rubric gate (`make verify` / `scripts/verify.py`) checks 10 automated checkpoints:
1. `setup-report.json` exists
2. App `/healthz` reachable
3. `/metrics` exposes `inference_requests_total`
4. Prometheus, Grafana, Alertmanager reachable
5. 3 Day-23 dashboards loaded in Grafana
6. Jaeger, Loki, OTel Collector reachable
7. `drift-summary.json` exists with at least one drifted feature
8. `submission/REFLECTION.md` exists and is non-trivial (>500 chars)

Manual checkpoints (screenshots + reflection text) are described in `rubric.md`.
