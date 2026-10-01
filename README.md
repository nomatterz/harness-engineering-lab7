# Lab 7 — Observability for agentic AI: bare OTel vs MLflow vs Phoenix

A learning lab from the fwdays harness-engineering course, run in the abox repo (branch
`feat/otel-demo`, release 0.11.36). It compares bare OpenTelemetry, MLflow and Arize Phoenix as
observability backends, using traces from a kagent `k8s-agent` session sent through one OTel
collector to MLflow and Phoenix. Bare OTel is represented by a production kagent trace sample,
since abox has no OTel trace backend.

**Result:** Phoenix. It is lighter than MLflow, takes OTLP directly and shows tokens, cost and a
typed agent tree. Bare OTel needs a separate backend and shows none of that; MLflow is comparable
in the trace view but heavier, and is only worth it with its ML lifecycle features.

## Where to look

- **`docs/adr/0001-otel-vs-mlflow-vs-phoenix-for-genai-observability.md`** — the deliverable:
  decision and comparison.
- **`CHANGELOG.md`** — what was done, step by step, including the setup.
