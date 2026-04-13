---
name: prometheus-k8s-debugging
description: Use when debugging Kubernetes node or pod issues using Prometheus metrics. Covers node-level (kernel/OS) metrics from node_exporter, pod/container metrics, event-time correlation, and systematic root-cause analysis workflows.
---

# Prometheus Kubernetes Debugging — Node & Pod Root-Cause Analysis

## Overview

A systematic, event-driven workflow for diagnosing Kubernetes node and pod failures using Prometheus API queries. The agent correlates **event timestamps** with metric anomalies across multiple dimensions — from high-level pod health down to kernel-level subsystems — to pinpoint root causes.

**Core principle:** Every hypothesis must be validated by at least two independent metric signals before being declared the root cause. Time correlation is the primary evidence-linking mechanism.

**Target environment:**
- Kubernetes cluster with `kube-state-metrics` and `node-exporter` deployed
- Prometheus API accessible at `$PROMETHEUS_URL`
- Incident has a known approximate timestamp (`$EVENT_TIME`, RFC3339 or Unix)

---

## When to Use

- A node becomes `NotReady` or is evicted unexpectedly
- Pods are OOMKilled, CrashLooping, or stuck in Pending
- Workload latency spikes or throughput drops appear on a specific node
- CI/CD alerts fire and you need to identify the faulty node/pod
- You need to correlate a Kubernetes event with system-level behaviour
- **「幫我分析這個節點在 XX 時間點前後發生了什麼」** — triggers full node deep-dive

**When NOT to use:**
- Cluster-wide capacity planning (use separate capacity tooling)
- Application-level tracing (use Jaeger/Tempo)
- Log-only incidents with no metric signals

---

## Prometheus API Conventions

All queries use the HTTP API. The agent MUST use these patterns:

```bash
# Instant query at a specific timestamp
curl -sG "$PROMETHEUS_URL/api/v1/query" \
  --data-urlencode "query=<PROMQL>" \
  --data-urlencode "time=<UNIX_TIMESTAMP>"

# Range query around an event window
curl -sG "$PROMETHEUS_URL/api/v1/query_range" \
  --data-urlencode "query=<PROMQL>" \
  --data-urlencode "start=<START_UNIX>" \
  --data-urlencode "end=<END_UNIX>" \
  --data-urlencode "step=15s"

# Helper: convert ISO8601 to unix
date -d "2024-01-15T10:30:00Z" +%s
```

### Event Window Strategy

Define three time windows relative to `$EVENT_TIME`:

| Window | Purpose | Offset |
|--------|---------|--------|
| `PRE` | Baseline / normal state | `-30m` to `-5m` |
| `NEAR` | Leading indicators & causation signals | `-5m` to `+2m` |
| `POST` | Impact / consequence confirmation | `+2m` to `+30m` |

**Rule:** A metric that spikes in `NEAR` but was normal in `PRE` is a **candidate cause**. A metric that only changes in `POST` is an **effect**.

---

## Core Workflow

```
Phase 1: Event Anchoring      → establish $EVENT_TIME, $NODE, $NAMESPACE/$POD
Phase 2: Node Health Triage   → quick 6-metric snapshot of the affected node
Phase 3: Deep Node Analysis   → kernel/OS layer investigation
Phase 4: Pod/Workload Analysis → container and scheduler investigation
Phase 5: Cross-Signal Correlation → build cause-effect timeline
Phase 6: Hypothesis Ranking   → score candidates, declare root cause
```

See [investigation-workflow.md](./investigation-workflow.md) for the complete decision tree and phase-by-phase prompts.

---

## Metric Categories

| Category | Reference | Scope |
|----------|-----------|-------|
| Node compute & saturation | [node-metrics.md §2](./node-metrics.md#2-compute--cpu-saturation) | Node |
| Node memory subsystem | [node-metrics.md §3](./node-metrics.md#3-memory-subsystem) | Node |
| Node disk & I/O | [node-metrics.md §4](./node-metrics.md#4-disk--io-subsystem) | Node |
| Node network | [node-metrics.md §5](./node-metrics.md#5-network-subsystem) | Node |
| Kernel internals | [node-metrics.md §6](./node-metrics.md#6-kernel-internals--os-health) | Node |
| Pod resource usage | [pod-metrics.md §2](./pod-metrics.md#2-pod-resource-usage) | Pod |
| Pod lifecycle events | [pod-metrics.md §3](./pod-metrics.md#3-pod-lifecycle--scheduler) | Pod |
| Container runtime | [pod-metrics.md §4](./pod-metrics.md#4-container-runtime--cgroups) | Pod |
| Ready-made query recipes | [query-cookbook.md](./query-cookbook.md) | Both |

---

## Quick-Start: Minimum Viable Investigation

When time is limited, run these 6 queries first (replace `$NODE` and `$T` with actuals):

```promql
# 1. Was the node under CPU pressure?
rate(node_cpu_seconds_total{mode="idle", instance="$NODE"}[5m]) < 0.1

# 2. Was memory critically low (including page cache pressure)?
(node_memory_MemAvailable_bytes{instance="$NODE"} /
 node_memory_MemTotal_bytes{instance="$NODE"}) < 0.1

# 3. Was disk I/O saturated?
rate(node_disk_io_time_seconds_total{instance="$NODE"}[5m]) > 0.9

# 4. Were there OOM kills?
increase(node_vmstat_oom_kill{instance="$NODE"}[10m]) > 0

# 5. Were pods restarting?
increase(kube_pod_container_status_restarts_total{node="$NODE"}[10m]) > 2

# 6. Was the kubelet reporting node conditions?
kube_node_status_condition{node="$NODE", status="true", condition!="Ready"}
```

If any of these trigger, proceed to the relevant deep-dive section in [node-metrics.md](./node-metrics.md) or [pod-metrics.md](./pod-metrics.md).

---

## Root-Cause Scoring Rubric

After collecting signals, score each hypothesis:

| Criterion | Score |
|-----------|-------|
| Metric anomaly precedes event by 1–10 min | +3 |
| Anomaly occurs on the exact affected node only | +2 |
| Two or more independent metrics confirm | +2 |
| Known failure mode for this metric combination | +1 |
| Anomaly disappears after remediation | +2 |
| Metric anomaly is on a different node or time | -2 |

**Threshold:** Score ≥ 6 → declare root cause. Score 3–5 → contributing factor. Score < 3 → coincidence.

---

## Output Format

The agent MUST produce a structured report with these sections:

```markdown
## Incident Summary
- Event time: <ISO8601>
- Affected node: <name>
- Affected pods: <list>

## Evidence Timeline
| Time offset | Metric | Value | Significance |
|-------------|--------|-------|--------------|
| -8m  | node_vmstat_oom_kill | +3  | OOM pressure building |
| -3m  | container_memory_working_set_bytes | 98% limit | Near-limit |
| 0    | kube_pod_container_status_restarts_total | +1 | Pod restarted |

## Root Cause
**Primary:** <hypothesis> (score: N)
**Contributing:** <hypothesis> (score: N)

## Recommended Actions
1. <action>
2. <action>
```
