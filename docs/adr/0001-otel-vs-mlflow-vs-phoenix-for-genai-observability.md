# 1. Bare OTel vs MLflow vs Phoenix as an observability backend for agentic AI

## Status

Proposed

## Context

abox (`feat/otel-demo`, release 0.11.36) ships kagent, MLflow and Arize Phoenix. This ADR compares
them as observability backends for an agent that calls models and tools. The agent is only the
test input; the subject is the tools.

One scenario is followed through all three: a kagent `k8s-agent` session (kagent 0.10.1, model
`claude-sonnet-5-5`) of 17 spans, with 3 model calls and 9 tool calls, 21.7 s. kagent exports
`gen_ai.*` spans over OTLP to one collector, which sent the same traces to MLflow 3.14 and Phoenix
12.0.10.

"Bare OTel" means the collector plus a generic trace backend and the `gen_ai.*` conventions, with
no GenAI-aware UI. abox has no such backend, so this column uses the production kagent spans in
Victoria Traces as a sample. Their structure is the same as the lab session's (same instrumentation,
same span names). Production data was used for structure only; no content is reproduced.

Marks: **O** observed in this lab, **S** observed in the production sample, **B** background
knowledge, not checked.

## Decision

Phoenix is the observability backend for agentic AI. Agents keep emitting `gen_ai.*` OTLP through
the OTel collector, so they stay independent of the backend.

- **Phoenix wins.** It is the lightest of the two GenAI tools, takes OTLP directly, and has what
  matters most here: tokens and cost per call, in a tree with typed AGENT, LLM and TOOL spans.
- **Bare OTel is not enough.** It needs a separate trace backend to be deployed and run (abox has
  none). The data is the same, but there are no span types, no token or cost view and no agent
  tree, so it is noisy to read. It stays as the transport, not the backend.
- **MLflow is not needed.** Its trace view is comparable to Phoenix, but it is a large platform that
  needs a bridge and clearly more memory. Without the ML lifecycle features (model registry,
  model packaging and serving) it adds weight and nothing else. Choose it only if those features
  are wanted.

## Comparison

### 1. Getting the traces in (deployment)

| | Bare OTel | MLflow | Phoenix |
|---|-----------|--------|---------|
| Ingest | OTLP native | OTLP/HTTP only, at `/v1/traces` with an `x-mlflow-experiment-id` header (O) | OTLP gRPC 4317 and HTTP 6006 (O) |
| Backend to deploy | A separate trace backend must be deployed and run; none in abox | Included | Included |
| Collector change needed | None | A gRPC-to-HTTP bridge and a header per route (O) | None (O) |
| Storage | Backend specific | SQL store and artifacts, 10Gi PVC (O) | Postgres, 20Gi PVC (O) |
| Auth | Backend specific | None configured | On by default; the collector was rejected until it was turned off (O) |
| Startup problems | - | Probe timeouts and OOM kills needed chart patches (O) | None |

### 2. What the trace looks like

Same session, opened in each tool.

| | Bare OTel | MLflow | Phoenix |
|---|-----------|--------|---------|
| Tree | Parent/child spans as emitted; no GenAI view (S) | Same 17-span tree (O) | Same 17-span tree (O) |
| Span types | None in the data (S) | AGENT, LLM, TOOL from `gen_ai.*`; HTTP wrapper spans untyped (O) | AGENT, LLM, TOOL (`span_kind`); wrapper spans UNKNOWN (O) |
| Tool calls | Arguments inside the model completion; tool time only as a gap between spans (S) | Own TOOL spans with duration (O) | Own TOOL spans with duration (O) |
| Prompt and completion | Raw attributes, read by script (S) | Input and output on LLM spans (O) | Input and output on LLM spans (O) |
| Trace-level input/output | None (S) | Empty, root is an HTTP span (O) | Empty, same reason (O) |
| Session grouping | `gen_ai.conversation.id`, no UI (S) | `mlflow.trace.session` set from it automatically (O) | Session set from the same conversation ID (O) |
| Noise | Three wrapper spans per model call (S) | HTTP wrapper spans (O) | HTTP wrapper spans (O) |

The tree and the span types come from kagent's spans; both tools render what is there. The empty
trace-level input and output is a source limit, not a tool difference.

### 3. What it tells you (tokens and cost)

| | Bare OTel | MLflow | Phoenix |
|---|-----------|--------|---------|
| Tokens per call | On spans, including cache (S) | On spans (O) | On spans, including cache-read (O) |
| Tokens per trace | Not provided | Total in trace metadata: 54.7k in, 2.8k out (O) | Tokens and cost per trace |
| Cost | No field; needs a price table (S) | Computed at ingest from a built-in price list; nothing stored, as the list lacks `claude-sonnet-5-5` (O) | Per LLM span in a `span_costs` table; about $0.137 for the session (O) |

Cost in both tools depends on the model being in the tool's price table.

### 4. What it costs to run

One reading per tool after the same few sessions, from the container cgroup (`anon` is memory
excluding page cache). No metrics server was available and nothing was measured under load.

| | Bare OTel | MLflow | Phoenix |
|---|-----------|--------|---------|
| Memory, anonymous | Not measured | 1.9 GiB (limit 2 GiB) | 0.47 GiB, plus 87 MiB for Postgres (total memory) |
| Requests / limits | - | 768Mi, 250m / 2Gi, 1 CPU | 1Gi, 500m / 2Gi, 1 CPU; Postgres 256Mi / 512Mi |
| Restarts | - | 2 on OOM/probe during setup | 0 |

MLflow ran close to its limit and was OOM-killed at 1Gi (O); part of its restarts came from large
queries run inside the pod.

## Extras (outside observability)

| | Bare OTel | MLflow | Phoenix |
|---|-----------|--------|---------|
| Evals, datasets and experiments | None | Yes (B) | Yes (B) |
| Prompt management and playground | None | Prompt registry (B) | Versioned prompts and playground (B) |
| Model registry and model packaging/serving | None | Core of the product (B) | No (B) |

## Consequences

- One collector feeding one GenAI backend keeps the agents independent of the tool, since they
  only emit `gen_ai.*` OTLP.
- Phoenix needs less to run and takes OTLP without a bridge; MLflow needs the bridge and used
  several times the memory for the same input.
- Keep the price table current in whichever tool is used, or cost will be missing for new models.
- Trace-level previews stay empty until kagent puts input and output on the root span.

## Limits

- Bare OTel is a production sample, not the same traces, and its resource use was not measured.
- One agent, a few sessions, one memory reading per tool; not a load test. MLflow's figure includes
  a few restarts and queries run inside the pod.
- Multi-agent delegation and MLflow custom pricing were not checked.
- The catch-all route also sent Astronomy Shop spans into the same MLflow experiment and Phoenix
  project.
