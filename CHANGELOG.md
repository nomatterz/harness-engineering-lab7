# Changelog

1. Deployed abox from the `feat/otel-demo` release (0.11.36) on local KinD (Colima, 8 CPU, 24 GiB), with a placeholder credentials Secret for ngrok-operator (not used by the lab).
2. Read production kagent spans in Victoria Traces (`kagent-trace-inspector`) as the bare-OTel evidence: span hierarchy, attributes, token fields, session grouping. Structure only; abox has no OTel trace backend.
3. Enabled kagent tracing (`otel.tracing` on the kagent HelmRelease) towards the `mlflow/otel-collector` bridge.
4. Added a catch-all route to the bridge collector that sends traces to MLflow experiment 4 (`kagent`) and to Phoenix (gRPC 4317). Paused Flux on `kagent`, `mlflow/otel-collector` and `phoenix` so these edits stay.
5. Disabled Phoenix auth (`enableAuth: false`, `disableBasicAuth: true`) so the collector can write without additional setup.
6. Created Secret `litellm-personal` and ModelConfig `litellm-claude-sonnet-5` in `kagent`, and set `k8s-agent` to it.
7. Ran k8s-agent sessions from the kagent UI.
8. Compared the same session in both tools: 17 spans (3 LLM calls, 9 tool calls, 21.7 s). Same tree, span types, session ID (from the conversation ID) and per-trace token total in both.
9. Compared cost: Phoenix prices each LLM span into a `span_costs` table (about $0.137 for the session). MLflow computes `mlflow.llm.cost` at ingest from a built-in price list that lacks `claude-sonnet-5-5`, so no cost was stored.
10. Wrote ADR 0001.
