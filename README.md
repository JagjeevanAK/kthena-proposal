# Proposal: Kthena Router Benchmark Framework

## Summary

Kthena Router is the data-plane component that receives OpenAI-compatible inference
requests, resolves the target model and route, applies traffic policy, chooses an
in-cluster backend, and forwards requests to model-serving pods. As the router grows
to support more routing strategies, prefill-decode disaggregation, rate limiting,
fairness scheduling, KV-cache-aware placement, and third-party backends, the project
needs a reproducible benchmark framework that can answer a simple question for each
release: did router performance improve, stay stable, or regress?

This proposal defines a benchmark framework for Kthena Router that is runnable on a
developer machine, in a Kubernetes cluster, and in CI. The framework will include a
load generator, scenario definitions, mock and real backend support, Prometheus-based
metrics collection, profile capture, result aggregation, and a report template. It
will measure throughput, latency, tail latency, time to first token (TTFT), time per
output token (TPOT), request errors, scheduler/plugin cost, CPU, memory, and, when
available, GPU utilization. Benchmark results will be stored as structured artifacts
so maintainers can compare releases and validate upstream optimizations with
before/after numbers.

This proposal tracks the feature request in issue #942.

## Motivation

Router performance has a direct effect on user-visible inference latency and on the
amount of traffic a Kthena installation can serve with fixed cluster resources.
Without a shared benchmark, performance claims are hard to reproduce because each
developer may use different prompt sizes, output sizes, backend behavior, routing
rules, concurrency, hardware, and metric collection windows. This makes it difficult
to identify bottlenecks, review optimization PRs, or detect regressions before a
release.

LLM routing also differs from generic HTTP proxy benchmarking. Realistic scenarios
must model streaming responses, long-running requests, different prompt and response
token distributions, varying concurrency, model-specific routing, rate limits,
prefill-decode flows, KV-cache locality, backend saturation, and failures. The
framework should therefore be specific to Kthena Router while reusing standard tools
and Kubernetes-native observability where possible.

### Goals

- Provide a reusable Kthena Router benchmark framework under `benchmark/kthena-router`.
- Support local, Kubernetes Job, and CI execution modes.
- Define versioned scenario configs for common LLM routing workloads.
- Generate OpenAI-compatible completion and chat-completion traffic.
- Support both mock backends and real inference backends such as vLLM and SGLang.
- Measure throughput, latency, p50/p90/p95/p99, TTFT, TPOT, request success rate,
  scheduler/plugin latency, router CPU, router memory, and backend GPU metrics when
  available.
- Collect router Prometheus metrics, backend metrics, Kubernetes pod resource usage,
  and optional `pprof` profiles in a single run artifact.
- Produce machine-readable results for CI gates and human-readable reports for
  release notes and optimization PRs.
- Define an end-to-end runbook covering cluster setup, benchmark execution, result
  interpretation, and cleanup.
- Identify clear bottlenecks and guide upstream optimization PRs with before/after
  evidence.

### Non-Goals

- This proposal does not define an official Kthena Router service-level objective.
- This proposal does not require GPU hardware for every benchmark scenario.
- This proposal does not replace existing unit, integration, or e2e tests.
- This proposal does not implement a general-purpose benchmark platform for all
  Kthena components.
- This proposal does not require benchmarking every possible model, inference engine,
  tokenizer, or third-party provider.
- This proposal does not guarantee identical absolute results across heterogeneous
  hardware; it focuses on reproducibility within a documented environment and on
  comparable trends across commits and releases.

## Proposal

Introduce a benchmark framework with four layers:

1. Workload generation: sends realistic OpenAI-compatible requests to Kthena Router.
2. Scenario orchestration: installs or points to backends, applies Kthena resources,
   runs a configured workload, and controls warmup/measurement/cooldown phases.
3. Metrics and profile collection: gathers client-side results, router metrics,
   backend metrics, Kubernetes resource usage, and optional profiles.
4. Result aggregation and reporting: writes normalized JSON/CSV/Markdown artifacts
   and compares current results with a baseline.

The initial implementation may reuse the existing `benchmark/kthena-router`
scaffold, which currently packages an SGLang `bench_serving.py`-based load generator
as a container and Kubernetes Job. The framework should evolve that scaffold into a
Kthena-specific benchmark suite with stable scenarios, reproducible outputs, and CI
integration.

### User Stories

#### Story 1: Maintainer Checks a Router Optimization PR

A maintainer wants to review a router scheduling optimization. They run a baseline
benchmark against `main`, apply the PR, run the same scenarios, and receive a
comparison report showing throughput, p99 latency, TTFT, TPOT, CPU, memory, and
scheduler plugin cost. The report shows whether the change improves the target
scenario without regressing other scenarios.

#### Story 2: Contributor Runs a Local Smoke Benchmark

A contributor without GPU access wants to check whether a router change introduces a
major performance regression. They run a local or Kind-based mock-backend smoke
scenario that completes in minutes and produces a concise result file.

#### Story 3: Release Manager Produces Release Benchmark Results

A release manager runs the full benchmark suite on a documented cluster profile
before release. The generated report is attached to release notes and includes the
exact Kthena commit, image tags, scenario configs, cluster size, backend type, and
result tables.

#### Story 4: Maintainer Investigates a Bottleneck

A maintainer sees high tail latency in the benchmark report. They rerun the scenario
with `--profile=true`, capture router CPU and heap profiles, and use the flamegraph
to identify hot code paths in request parsing, scheduling, datastore access, or
response streaming.

## Design Details

### Repository Structure

The framework will live under `benchmark/kthena-router`:

```text
benchmark/kthena-router/
  README.md
  Makefile
  Dockerfile
  go.mod or pyproject.toml
  cmd/
    kthena-router-bench/
      main.go
  pkg/
    loadgen/
      client.go
      openai.go
      scheduler.go
      stream.go
      latency.go
    scenario/
      config.go
      loader.go
      validate.go
    metrics/
      prometheus.go
      kube.go
      nvidia.go
      pprof.go
      aggregate.go
    report/
      compare.go
      markdown.go
      json.go
    backend/
      mock/
        server.go
        deployment.yaml
      vllm/
        values.yaml
      sglang/
        values.yaml
  scenarios/
    smoke.yaml
    throughput-sweep.yaml
    latency-sweep.yaml
    streaming-chat.yaml
    multi-backend-random.yaml
    multi-backend-least-request.yaml
    kvcache-aware-shared-prefix.yaml
    prefill-decode-disaggregation.yaml
    rate-limit.yaml
    backend-failure.yaml
  manifests/
    job.yaml
    rbac.yaml
    configmap.yaml
    servicemonitor.yaml
    mock-backend.yaml
  scripts/
    run-local.sh
    run-kind.sh
    run-cluster.sh
    collect-artifacts.sh
    compare.sh
  reports/
    template.md
  results/
    .gitignore
```

The exact implementation language can be Go or Python. Go is preferred for tight
integration with Kubernetes APIs, Kthena types, Prometheus parsing, and CI packaging.
Python is acceptable for a load-generator-only first milestone if the interface and
artifacts remain stable.

### Benchmark CLI

The benchmark binary will provide a stable command interface:

```bash
kthena-router-bench run \
  --scenario benchmark/kthena-router/scenarios/smoke.yaml \
  --target http://kthena-router.kthena-system.svc.cluster.local \
  --namespace kthena-benchmark \
  --output results/smoke-$(date +%Y%m%d-%H%M%S) \
  --prometheus-url http://prometheus.monitoring.svc:9090 \
  --profile=false
```

Additional commands:

```bash
kthena-router-bench validate --scenario scenarios/smoke.yaml
kthena-router-bench report --input results/run.json --output results/report.md
kthena-router-bench compare --baseline results/main.json --candidate results/pr.json
kthena-router-bench mock-backend --listen :8000 --profile mock-backend.yaml
```

### Scenario Configuration

Scenarios will be declarative YAML files. Each file describes the request shape,
arrival pattern, routing setup, backend shape, metrics collection, and pass/fail
thresholds.

```yaml
apiVersion: benchmark.kthena.volcano.sh/v1alpha1
kind: RouterBenchmarkScenario
metadata:
  name: streaming-chat-shared-prefix
description: Streaming chat workload with repeated system prompts and variable outputs.
target:
  scheme: http
  host: kthena-router.kthena-system.svc.cluster.local
  port: 80
  path: /v1/chat/completions
traffic:
  mode: poisson
  requestRate: 100
  maxConcurrency: 256
  warmupSeconds: 60
  durationSeconds: 300
  cooldownSeconds: 30
requests:
  api: chat_completions
  model: deepseek-ai/DeepSeek-R1-Distill-Qwen-7B
  stream: true
  prompt:
    distribution: generated_shared_prefix
    groups: 64
    promptsPerGroup: 16
    systemPromptTokens: 512
    userPromptTokens:
      distribution: lognormal
      mean: 1024
      p95: 4096
  outputTokens:
    distribution: fixed
    value: 256
routing:
  modelRoute: ds7b-route
  strategy: kvcache-aware
  backends: 8
metrics:
  scrapeIntervalSeconds: 5
  collectRouterMetrics: true
  collectBackendMetrics: true
  collectKubernetesMetrics: true
  collectProfiles: false
thresholds:
  errorRateMax: 0.01
  p99LatencySecondsMax: 20
  p99TTFTSecondsMax: 3
  routerCPUCoreMax: 2
```

Scenario validation will reject missing target fields, unsupported distributions,
invalid concurrency values, negative durations, and threshold definitions that cannot
be evaluated.

### Workload Model

The load generator will support:

- Fixed QPS.
- Poisson arrival process for production-like request spacing.
- Closed-loop concurrency mode for saturation testing.
- Warmup, measurement, and cooldown phases.
- Non-streaming `/v1/completions` requests.
- Non-streaming `/v1/chat/completions` requests.
- Streaming chat-completion requests with TTFT and token timing extraction.
- Prompt distributions:
  - fixed token length,
  - uniform token length,
  - lognormal token length,
  - dataset-backed prompts,
  - generated shared-prefix prompts for KV-cache-aware routing.
- Output distributions:
  - fixed,
  - uniform,
  - lognormal,
  - scenario-defined sequence.

Token lengths should be generated through a tokenizer when available. For mock-only
CI scenarios, deterministic synthetic text may be used with an approximate token
ratio, but reports must mark those results as approximate.

### Backend Modes

The benchmark framework will support three backend modes.

#### Mock Backend

The mock backend provides deterministic, GPU-free test behavior for local runs and
CI. It will expose OpenAI-compatible endpoints and support:

- configurable TTFT delay,
- configurable per-token delay,
- configurable output token count,
- streaming and non-streaming responses,
- configurable error rate,
- configurable status-code failure injection,
- backend identity headers for routing validation,
- Prometheus metrics for request count, active requests, TTFT, TPOT, and errors.

The mock backend allows router overhead to be isolated from model inference cost.

#### Real Backend

Real-backend scenarios will use existing Kthena examples and model-serving manifests
for vLLM and SGLang where possible. These scenarios measure router behavior in a
more realistic environment but are expected to be hardware-dependent.

#### Third-Party Backend Stub

As Kthena adds third-party model backend support, the benchmark should include a
stub mode that emulates an external backend over HTTP with configurable network
latency and rate limits. This keeps the benchmark ready for future routing paths
without depending on paid external providers.

### Metrics

The framework will collect client-side, router-side, backend-side, and cluster-side
signals.

Client-side metrics:

- offered QPS,
- completed QPS,
- success count,
- failure count,
- timeout count,
- HTTP status distribution,
- throughput in requests/second,
- input tokens/second,
- output tokens/second,
- latency p50/p90/p95/p99/max,
- TTFT p50/p90/p95/p99/max,
- TPOT p50/p90/p95/p99/max,
- inter-token latency for streaming responses,
- request body size and response body size summaries.

Router Prometheus metrics:

- `kthena_router_requests_total`,
- `kthena_router_request_duration_seconds`,
- `kthena_router_request_prefill_duration_seconds`,
- `kthena_router_request_decode_duration_seconds`,
- `kthena_router_tokens_total`,
- `kthena_router_scheduler_plugin_duration_seconds`,
- `kthena_router_rate_limit_exceeded_total`,
- `kthena_router_active_downstream_requests`,
- `kthena_router_active_upstream_requests`,
- `kthena_router_fairness_queue_size`,
- `kthena_router_fairness_queue_duration_seconds`.

Backend and runtime metrics:

- backend active requests,
- backend queue length,
- backend TTFT and TPOT when exposed,
- KV cache utilization when exposed,
- GPU utilization and GPU memory usage when available,
- CPU and memory for backend pods.

Cluster metrics:

- router pod CPU usage,
- router pod memory working set,
- router restarts,
- backend pod CPU and memory,
- node CPU and memory saturation,
- network transmit and receive bytes if available.

### Result Artifacts

Each run will write a self-contained output directory:

```text
results/2026-05-12T10-30-00Z-smoke/
  run.json
  summary.csv
  report.md
  scenario.yaml
  environment.json
  prometheus/
    router.json
    backend.json
    kube.json
  profiles/
    router-cpu.pb.gz
    router-heap.pb.gz
  logs/
    benchmark.log
    router.log
    mock-backend.log
```

`run.json` will be the source of truth for automated comparison:

```json
{
  "scenario": "smoke",
  "kthenaCommit": "abc123",
  "routerImage": "ghcr.io/volcano-sh/kthena-router:abc123",
  "startedAt": "2026-05-12T10:30:00Z",
  "durationSeconds": 300,
  "client": {
    "offeredQPS": 50,
    "completedQPS": 49.7,
    "errorRate": 0.0,
    "latencySeconds": {
      "p50": 0.42,
      "p90": 0.71,
      "p95": 0.9,
      "p99": 1.4,
      "max": 2.1
    },
    "ttftSeconds": {
      "p50": 0.12,
      "p90": 0.2,
      "p95": 0.24,
      "p99": 0.37
    },
    "tpotSeconds": {
      "p50": 0.018,
      "p90": 0.025,
      "p95": 0.03,
      "p99": 0.048
    }
  },
  "router": {
    "cpuCoresAvg": 0.8,
    "cpuCoresMax": 1.6,
    "memoryMiBAvg": 220,
    "memoryMiBMax": 310,
    "schedulerPluginP99Seconds": {
      "kvcache-aware": 0.008
    }
  },
  "thresholds": {
    "passed": true,
    "failures": []
  }
}
```

### Report Format

The generated Markdown report will include:

- benchmark purpose and scenario name,
- Kthena commit, image tags, benchmark tool version, and cluster information,
- workload configuration,
- backend configuration,
- throughput and latency tables,
- TTFT and TPOT tables,
- CPU, memory, and GPU tables,
- router metric summaries,
- pass/fail threshold results,
- before/after comparison when a baseline is supplied,
- flamegraph/profile links when profiling is enabled,
- analysis notes and identified bottlenecks.

The report should be suitable for attaching directly to a PR, issue, or release
artifact.

### CI Integration

CI should use a two-tier benchmark strategy.

#### Tier 1: Smoke Benchmark

Runs on PRs or on a scheduled workflow with mock backends:

- Kind cluster or local process mode.
- Short duration, for example 60 seconds.
- Small concurrency, for example 16 to 64.
- No GPU required.
- Validates that the benchmark framework, manifests, mock backend, result writer,
  and basic router path are working.
- Fails on high error rate, severe latency regression, or missing artifacts.

#### Tier 2: Nightly or Release Benchmark

Runs on dedicated infrastructure:

- Longer duration, for example 5 to 30 minutes per scenario.
- Multiple routing strategies.
- Real backends when hardware is available.
- Captures Prometheus metrics and profiles.
- Uploads artifacts for maintainers.
- Compares results with the latest accepted baseline.

Recommended GitHub Actions workflow names:

- `router-benchmark-smoke.yaml`
- `router-benchmark-nightly.yaml`

The benchmark should not make normal PR CI depend on GPU availability.

### Benchmark Scenarios

The initial scenario set should include:

| Scenario | Purpose | Backend | CI Tier |
| --- | --- | --- | --- |
| `smoke` | Validate benchmark plumbing and basic router forwarding | mock | Tier 1 |
| `throughput-sweep` | Find max sustainable QPS under fixed latency thresholds | mock or real | Tier 2 |
| `latency-sweep` | Measure p50-p99 latency across request rates | mock or real | Tier 2 |
| `streaming-chat` | Measure TTFT, TPOT, and stream stability | mock or real | Tier 2 |
| `multi-backend-random` | Measure routing overhead with multiple backends | mock | Tier 1/Tier 2 |
| `multi-backend-least-request` | Measure scheduling overhead and balance quality | mock | Tier 2 |
| `kvcache-aware-shared-prefix` | Measure KV-cache-aware plugin overhead and benefit | mock or real | Tier 2 |
| `prefill-decode-disaggregation` | Measure PD connector and phase-level latency | real or specialized mock | Tier 2 |
| `rate-limit` | Measure request rejection behavior and overhead | mock | Tier 2 |
| `backend-failure` | Measure tail latency and error behavior during backend failure | mock | Tier 2 |

### Profiling

The framework will optionally capture profiles during the measurement window:

- CPU profile from the router debug endpoint,
- heap profile,
- goroutine profile,
- mutex/block profiles if enabled,
- benchmark process CPU and heap profile if useful.

If pprof endpoints are not currently exposed or are not enabled by default, the
implementation should add a gated configuration option for benchmark and debug
environments. Profiles must not expose sensitive request payloads.

### Runbook

The benchmark documentation must include an end-to-end runbook:

1. Install prerequisites: Go, Docker or compatible container tool, `kubectl`, Kind
   for local clusters, Helm if needed, and Prometheus for metrics collection.
2. Build Kthena Router image:
   ```bash
   make docker-build-router IMG_ROUTER=ghcr.io/volcano-sh/kthena-router:bench
   ```
3. Start local cluster:
   ```bash
   kind create cluster --name kthena-bench
   ```
4. Install Kthena CRDs and router manifests.
5. Deploy mock backends or real model-serving backends.
6. Apply the benchmark scenario resources.
7. Run a smoke scenario:
   ```bash
   kthena-router-bench run \
     --scenario benchmark/kthena-router/scenarios/smoke.yaml \
     --output benchmark/kthena-router/results/smoke
   ```
8. Run a comparison:
   ```bash
   kthena-router-bench compare \
     --baseline benchmark/kthena-router/results/main/run.json \
     --candidate benchmark/kthena-router/results/pr/run.json
   ```
9. Read the report:
   - Check error rate first.
   - Check completed QPS against offered QPS.
   - Check p95/p99 latency and TTFT.
   - Check TPOT for streaming workloads.
   - Check router CPU and memory.
   - Check scheduler plugin p99 when routing strategies are enabled.
   - Check backend saturation before attributing latency to the router.
10. Collect artifacts and clean up the namespace or cluster.

### Optimization Workflow

When a benchmark surfaces a hotspot, the mentee and maintainers should use the
following workflow:

1. Confirm the issue is reproducible with the same scenario and environment.
2. Identify whether the bottleneck is in the client, router, backend, Kubernetes, or
   cluster infrastructure.
3. Capture profiles and relevant Prometheus windows.
4. Open or update a tracking issue with:
   - scenario name,
   - baseline run artifact,
   - observed regression or bottleneck,
   - profile summary,
   - suspected code path.
5. Implement a narrowly scoped optimization PR.
6. Attach before/after benchmark results to the PR.
7. Add a unit, integration, or benchmark test when the optimization fixes a specific
   code path that can regress independently.

Likely router hotspots to investigate include:

- request body parsing and reserialization,
- streaming response proxying,
- scheduler plugin execution,
- datastore locking and pod lookup,
- tokenizer overhead in routing plugins,
- Redis calls for KV-cache-aware routing,
- Prometheus label cardinality or hot-path metric updates,
- active request accounting,
- rate-limit checks,
- logging overhead under high QPS.

## Test Plan

The benchmark framework itself requires tests because inaccurate benchmark tools are
worse than no benchmark.

Unit tests:

- scenario YAML parsing and validation,
- request payload generation,
- token distribution generation,
- latency histogram aggregation,
- TTFT and TPOT calculation,
- threshold evaluation,
- baseline comparison logic,
- Markdown and JSON report generation.

Integration tests:

- load generator against mock backend,
- streaming response parser against mock backend,
- Prometheus scraper against mock metric endpoints,
- Kubernetes metrics collector with fake or envtest clients,
- artifact writer and report generation from a complete mock run.

E2E tests:

- Kind-based smoke benchmark with Kthena Router and mock backends,
- CI smoke workflow that uploads `run.json` and `report.md`,
- optional nightly real-backend benchmark on dedicated infrastructure.

Validation criteria:

- benchmark run exits non-zero when thresholds fail,
- result artifacts are complete and schema-valid,
- repeated mock-backend runs stay within an agreed variance band,
- compare command correctly flags regressions and improvements,
- smoke scenario completes within normal PR CI time limits.

## Milestones

### Milestone 1: Design and Baseline Skeleton

- Finalize this proposal with maintainers.
- Audit existing `benchmark/kthena-router` scaffold.
- Define result schema and scenario schema.
- Add smoke scenario and report template.
- Document local run instructions.

### Milestone 2: Load Generator and Mock Backend

- Implement OpenAI-compatible non-streaming and streaming load generation.
- Implement deterministic mock backend.
- Measure latency, TTFT, TPOT, throughput, and errors.
- Add unit and integration tests.
- Produce first smoke report.

### Milestone 3: Kubernetes Orchestration and Metrics

- Add Kubernetes Job manifests and RBAC.
- Add Prometheus and Kubernetes resource metric collection.
- Add artifact collection and result aggregation.
- Add Kind-based smoke CI.

### Milestone 4: Scenario Suite

- Add throughput, latency, multi-backend, routing strategy, rate-limit, shared-prefix,
  and backend-failure scenarios.
- Add real backend runbook for vLLM and SGLang.
- Add baseline comparison command.

### Milestone 5: Profiling and Optimization

- Add optional router profile capture.
- Run full benchmark suite.
- Identify clear hotspots.
- Work with maintainers to land optimization PRs with before/after numbers.
- Publish final benchmark report.

## Risks and Mitigations

| Risk | Impact | Mitigation |
| --- | --- | --- |
| Results vary across hardware | Hard to compare absolute numbers | Record environment metadata and compare only within documented profiles |
| GPU availability blocks CI | PR checks become unreliable | Keep GPU-free smoke tests separate from nightly/release benchmarks |
| Mock backend hides real inference behavior | Optimizations may not transfer to production | Include real-backend scenarios for release benchmarks |
| Benchmark client becomes bottleneck | Results understate router capacity | Report client CPU/memory and support distributed load generators later |
| Prompt generation is not representative | Results miss real-world traffic patterns | Include multiple prompt distributions and dataset-backed scenarios |
| High-cardinality metrics overload Prometheus | Metrics collection skews benchmark | Keep labels bounded and scrape only required windows |
| Profiles expose sensitive data | Security concern | Use synthetic prompts by default and avoid profile modes in shared clusters |
| CI flakiness causes noisy regressions | Maintainers lose trust in benchmarks | Use tolerant thresholds for smoke CI and trend-based nightly comparisons |

## Open Questions

- Should the first implementation be Go-only, or should it wrap the existing
  SGLang `bench_serving.py` path while adding Kthena-specific orchestration?
- Which cluster profile should be the official release benchmark environment?
- Which scenarios should be required for release qualification versus optional
  analysis?
- Should benchmark baselines be committed to the repository, stored as GitHub
  Actions artifacts, or published externally?
- Should pprof endpoints be enabled through the existing debug server or a separate
  benchmark-only flag?
- What threshold should CI use for severe regression detection in mock-backend runs?

## Alternatives

### Use Only SGLang `bench_serving.py`

The current benchmark scaffold already packages an SGLang benchmark script. This is
useful for initial load generation, but it does not provide a complete Kthena Router
benchmark framework. It lacks Kthena scenario schemas, router-specific metrics,
Kubernetes resource aggregation, profile capture, standard reports, CI thresholds,
and baseline comparison. The proposed framework can still reuse it during early
implementation if doing so accelerates delivery.

### Use Generic HTTP Tools

Tools such as `wrk`, `hey`, or `vegeta` are excellent for generic HTTP services but
do not natively model LLM streaming, TTFT, TPOT, output-token timing, shared-prefix
traffic, or OpenAI-compatible request and response semantics. They can be useful for
microbenchmarks but are insufficient as the primary benchmark framework.

### Use Only Unit Benchmarks

Go benchmarks for scheduler plugins, datastore lookups, or tokenization paths are
valuable and should be added for isolated hotspots. However, they do not measure
end-to-end router behavior under realistic concurrency, streaming, backend
saturation, or Kubernetes resource constraints. They should complement, not replace,
the proposed framework.

### Require Real Models for All Runs

Real models provide the most realistic data, but requiring them for every run would
make the benchmark expensive, slow, and difficult to run in CI. Mock backends provide
fast and reproducible router-focused coverage, while real-backend scenarios remain
available for release and optimization validation.
