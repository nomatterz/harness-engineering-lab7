# Lab 7 — Observability for agentic AI: bare OTel vs MLflow vs Phoenix

A learning lab from the fwdays harness-engineering course, run in the abox repo (branch
`feat/otel-demo`, release 0.11.36). It compares bare OpenTelemetry, MLflow and Arize Phoenix as
observability backends for an agent, using traces from a kagent `k8s-agent` session sent through
one OTel collector to MLflow and Phoenix. Bare OTel is represented by a production kagent trace
sample, since abox has no OTel trace backend.

**Result:** Phoenix. It is lighter than MLflow, takes OTLP directly and shows tokens, cost and a
typed agent tree. Bare OTel needs a separate backend and shows none of that; MLflow is comparable
in the trace view but heavier, and is only worth it with its ML lifecycle features. See the ADR.

## Where to look

- **`docs/adr/0001-otel-vs-mlflow-vs-phoenix-for-genai-observability.md`** — the deliverable:
  decision and comparison.
- **`CHANGELOG.md`** — what was done, step by step.

## Reproduce

Prerequisites: abox deployed from the `feat/otel-demo` release on a local KinD cluster (8 CPU,
24 GiB), and a LiteLLM endpoint.

1. Enable kagent tracing (`otel.tracing` on the kagent HelmRelease) towards the
   `mlflow/otel-collector` bridge, and pause Flux on it, or the release reverts the change.
2. Add a catch-all route to the bridge collector that exports to MLflow (experiment 4, header
   `x-mlflow-experiment-id`) and to Phoenix (`phoenix-svc.phoenix.svc.cluster.local:4317`).
3. Turn Phoenix auth off (`enableAuth: false`, `disableBasicAuth: true`) so the collector can write.
4. Create Secret `litellm-personal` and a ModelConfig in `kagent` (as in lab4) and set `k8s-agent`
   to it.
5. Send prompts to `k8s-agent` from the kagent UI, then open the trace in MLflow and Phoenix.

## Notes

- The catch-all route also sends Astronomy Shop spans to the same MLflow experiment and Phoenix
  project.
- Production kagent spans were used for span structure only; no content is reproduced.
