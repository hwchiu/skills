# Investigation Workflow — Event-Driven Root-Cause Analysis

> This document describes the **step-by-step process** the agent follows when a Kubernetes incident is reported. Each phase builds on the previous one.

---

## Pre-Flight: Gather Incident Context

Before any Prometheus query, collect:

| Item | How to Get | Variable |
|------|-----------|---------|
| Event time | Alert annotation, user report, `kubectl get events` | `$EVENT_TIME` |
| Affected node | `kubectl describe pod $POD \| grep Node:` | `$NODE` |
| Affected namespace/pod | Alert labels or user input | `$NAMESPACE`, `$POD` |
| Prometheus URL | Cluster config / environment | `$PROMETHEUS_URL` |
| node-exporter instance label format | `count by(instance)(node_uname_info)` | Confirm `$NODE` format |

```bash
# Convert event time to unix timestamps
T=$(date -d "$EVENT_TIME" +%s)
PRE_START=$((T - 1800))   # 30 min before
PRE_END=$((T - 300))      # 5 min before
NEAR_START=$((T - 300))   # 5 min before
NEAR_END=$((T + 120))     # 2 min after
POST_START=$((T + 120))
POST_END=$((T + 1800))    # 30 min after
```

---

## Phase 1: Node Health Triage (2 minutes)

Run all 6 queries simultaneously as instant queries at `$T`.

```promql
# Q1: CPU available?
1 - avg by (instance) (
  rate(node_cpu_seconds_total{mode="idle", instance="$NODE"}[5m])
)

# Q2: Memory available?
node_memory_MemAvailable_bytes{instance="$NODE"} /
node_memory_MemTotal_bytes{instance="$NODE"}

# Q3: Disk saturated?
max by (instance, device) (
  rate(node_disk_io_time_seconds_total{instance="$NODE"}[5m])
)

# Q4: OOM kills?
increase(node_vmstat_oom_kill{instance="$NODE"}[10m])

# Q5: Pod restarts on node?
topk(5, increase(kube_pod_container_status_restarts_total{node="$NODE"}[10m]))

# Q6: Node conditions?
kube_node_status_condition{node="$NODE", status="true"}
```

### Triage Decision Tree

```
Q1 CPU > 90%? ──YES──→ [Path A: CPU Pressure]
     │NO
     ▼
Q2 Memory < 10%? ──YES──→ [Path B: Memory Pressure]
     │NO
     ▼
Q3 Disk IO > 90%? ──YES──→ [Path C: I/O Saturation]
     │NO
     ▼
Q4 OOM kills > 0? ──YES──→ [Path B: Memory + OOM]
     │NO
     ▼
Q5 Restarts > 2? ──YES──→ [Path D: Pod Instability]
     │NO
     ▼
Q6 Condition != Ready? ──YES──→ [Path E: Node Condition]
     │NO
     ▼
[Path F: Subtle / Network / Kernel]
```

---

## Phase 2: Path-Specific Deep Dive

### Path A: CPU Pressure

**Goal:** Determine whether it is CPU limit, iowait, steal, or noisy-neighbour.

```promql
# A1: CPU mode breakdown
rate(node_cpu_seconds_total{instance="$NODE"}[5m])

# A2: iowait specifically?
avg by (instance) (rate(node_cpu_seconds_total{mode="iowait", instance="$NODE"}[5m]))
# iowait > 0.2 → jump to Path C (I/O Saturation)

# A3: steal? (cloud VM)
avg by (instance) (rate(node_cpu_seconds_total{mode="steal", instance="$NODE"}[5m]))
# steal > 0.05 → noisy neighbour on hypervisor

# A4: softirq? (network storm)
avg by (instance) (rate(node_cpu_seconds_total{mode="softirq", instance="$NODE"}[5m]))
# softirq > 0.05 → check network metrics (Path F)

# A5: Which containers are consuming most CPU?
topk(5,
  rate(container_cpu_usage_seconds_total{node="$NODE", container!=""}[5m])
)

# A6: Are containers throttled?
topk(5,
  rate(container_cpu_cfs_throttled_periods_total{node="$NODE", container!=""}[5m]) /
  rate(container_cpu_cfs_periods_total{node="$NODE", container!=""}[5m])
)

# A7: Processes blocked (D-state)?
node_procs_blocked{instance="$NODE"}
# blocked > 5 → probably I/O causing CPU iowait → jump to Path C
```

**Findings matrix:**

| Signal | Meaning | Next Action |
|--------|---------|------------|
| High user CPU + throttled containers | CPU limit too low | Recommend raising limits |
| High system CPU + high context switches | Lock contention / too many threads | Check container thread count |
| High iowait | Storage bottleneck masking as CPU | Path C |
| High steal | Hypervisor contention | Alert cloud provider |
| High softirq (NET_RX/NET_TX) | NIC interrupt affinity issue | Path F network |

---

### Path B: Memory Pressure

```promql
# B1: Working set of top consumers
topk(5,
  container_memory_working_set_bytes{node="$NODE", container!=""}
)

# B2: Usage vs limit ratio
topk(5,
  container_memory_working_set_bytes{node="$NODE", container!=""} /
  on (namespace, pod, container)
  kube_pod_container_resource_limits{resource="memory", node="$NODE"}
)

# B3: Any container near/over 95% of limit?
container_memory_working_set_bytes{node="$NODE", container!=""} /
on (namespace, pod, container)
kube_pod_container_resource_limits{resource="memory", node="$NODE"} > 0.95

# B4: Memory growth trajectory (leak detection)
deriv(container_memory_working_set_bytes{
  node="$NODE", container!=""
}[30m]) > 0

# B5: Node-level paging activity
rate(node_vmstat_pgmajfault{instance="$NODE"}[5m])
rate(node_vmstat_pgsteal_direct{instance="$NODE"}[5m])

# B6: Slab cache leak?
node_memory_SUnreclaim_bytes{instance="$NODE"}
# Compare current value vs 1-day average

# B7: OOM termination confirmation
kube_pod_container_status_last_terminated_reason{reason="OOMKilled", node="$NODE"}
```

**Findings matrix:**

| Signal | Meaning | Next Action |
|--------|---------|------------|
| Single container at 100% limit | OOM from limit too low | Raise memory limit |
| Multiple containers near limit simultaneously | Node over-committed | Adjust node memory requests or add nodes |
| `deriv` positive and accelerating | Memory leak in application | Capture heap dump, escalate to dev |
| `SUnreclaim` growing | Kernel slab leak (dentries/inodes) | Check for directory-heavy workloads; may need node restart |
| `pgsteal_direct > 0` | Severe pressure; latency spikes expected | Urgent: cordon node, drain pods |

---

### Path C: I/O Saturation

```promql
# C1: Which devices are saturated?
rate(node_disk_io_time_seconds_total{instance="$NODE"}[5m])

# C2: Latency per device
rate(node_disk_read_time_seconds_total{instance="$NODE"}[5m]) /
  (rate(node_disk_reads_completed_total{instance="$NODE"}[5m]) + 0.001) * 1000

# C3: Which containers are the I/O writers?
topk(5,
  rate(container_fs_writes_bytes_total{node="$NODE", container!=""}[5m])
)

# C4: Filesystem free space?
node_filesystem_avail_bytes{instance="$NODE", fstype!~"tmpfs|overlay"} /
node_filesystem_size_bytes{instance="$NODE", fstype!~"tmpfs|overlay"}

# C5: Inode exhaustion?
node_filesystem_files_free{instance="$NODE"} /
node_filesystem_files{instance="$NODE"}

# C6: Filesystem readonly (error recovery)?
node_filesystem_readonly{instance="$NODE"}

# C7: Blocked processes (waiting for I/O)
node_procs_blocked{instance="$NODE"}
```

**Findings matrix:**

| Signal | Meaning | Next Action |
|--------|---------|------------|
| Single device at 100% util, high latency | Storage device saturated | Identify top writer, rate-limit or migrate storage |
| Multiple devices at high util | Workload spread incorrectly | Review StorageClass / PV affinity |
| Filesystem near full | Space exhaustion | Identify large files, expand PV |
| Inode near full | Small-file storm (logs, temp files) | Identify pod writing many small files |
| `readonly == 1` | Filesystem error → remounted RO | Immediate node cordon; fsck required |

---

### Path D: Pod Instability

```promql
# D1: Which pods are restarting?
topk(10,
  increase(kube_pod_container_status_restarts_total{node="$NODE"}[30m])
)

# D2: Exit reason
kube_pod_container_status_last_terminated_reason{node="$NODE"}

# D3: Is it OOM? (see Path B)
kube_pod_container_status_last_terminated_reason{reason="OOMKilled", node="$NODE"}

# D4: Readiness probe failures?
kube_pod_container_status_ready{node="$NODE"} == 0

# D5: Is kubelet PLEG healthy?
histogram_quantile(0.99,
  rate(kubelet_pleg_relist_duration_seconds_bucket{instance="$NODE"}[5m])
)

# D6: Container runtime errors?
rate(kubelet_runtime_operations_errors_total{instance="$NODE"}[5m])

# D7: Image pull failures?
kube_pod_container_status_waiting_reason{reason="ImagePullBackOff", node="$NODE"}
kube_pod_container_status_waiting_reason{reason="ErrImagePull", node="$NODE"}
```

---

### Path E: Node Condition

```promql
# E1: Which conditions are true?
kube_node_status_condition{node="$NODE", status="true"}

# E2: MemoryPressure → go to Path B
kube_node_status_condition{node="$NODE", condition="MemoryPressure", status="true"}

# E3: DiskPressure → go to Path C
kube_node_status_condition{node="$NODE", condition="DiskPressure", status="true"}

# E4: PIDPressure → check fork rate
kube_node_status_condition{node="$NODE", condition="PIDPressure", status="true"}
rate(node_forks_total{instance="$NODE"}[5m])

# E5: NetworkUnavailable → go to Path F
kube_node_status_condition{node="$NODE", condition="NetworkUnavailable", status="true"}

# E6: kubelet heartbeat gap (node unreachable)
kube_node_status_condition{node="$NODE", condition="Ready", status="false"}
```

---

### Path F: Subtle / Network / Kernel

Run when no obvious signal from Paths A–E.

```promql
# F1: Conntrack exhaustion
node_nf_conntrack_entries{instance="$NODE"} /
node_nf_conntrack_entries_limit{instance="$NODE"}

# F2: TCP retransmits
rate(node_netstat_Tcp_RetransSegs{instance="$NODE"}[5m])

# F3: NIC packet drops (ring buffer overflow)
rate(node_network_receive_drop_total{instance="$NODE", device!~"lo|veth.*"}[5m])

# F4: NIC errors
rate(node_network_receive_errs_total{instance="$NODE"}[5m])

# F5: Kernel NMI / hardware errors
increase(node_edac_uncorrectable_errors_total{instance="$NODE"}[30m])

# F6: NTP sync lost
node_timex_sync_status{instance="$NODE"}
abs(node_timex_offset_seconds{instance="$NODE"}) > 0.1

# F7: File descriptor exhaustion
node_filefd_allocated{instance="$NODE"} /
node_filefd_maximum{instance="$NODE"}

# F8: Entropy starvation (apps blocked on /dev/random)
node_entropy_available_bits{instance="$NODE"} < 128

# F9: CPU thermal throttling
increase(node_cpu_package_throttles_total{instance="$NODE"}[10m])

# F10: Scheduler wait time (cgroup starvation)
rate(node_schedstat_waiting_seconds_total{instance="$NODE"}[5m])
```

---

## Phase 3: Cross-Signal Timeline Construction

After gathering signals, build a timeline table:

```markdown
| Time offset | Source | Metric | Value | Delta from baseline |
|-------------|--------|--------|-------|-------------------|
| -25m | node    | MemAvailable ratio           | 45%      | normal (baseline 48%) |
| -12m | node    | MemAvailable ratio           | 22%      | ↓ declining        |
| -8m  | node    | vmstat_oom_kill              | +1       | ↑ first OOM event  |
| -5m  | pod     | container_memory_failcnt     | rate +3  | ↑ cgroup hitting limit |
| -3m  | pod     | working_set/limit            | 0.98     | ↑ near-limit       |
| 0    | k8s     | pod restart count            | +1       | EVENT              |
| +2m  | node    | MemAvailable ratio           | 38%      | ↑ recovering       |
| +5m  | pod     | working_set bytes            | -200MB   | ↑ pod restarted fresh |
```

**Causation test:**
1. Does the anomaly appear in the `NEAR` window (`-5m to +2m`)?
2. Was it absent in the `PRE` window (`-30m to -5m`)?
3. Does it occur on `$NODE` but not on other nodes?

If all 3 are YES → strong causal candidate.

---

## Phase 4: Hypothesis Ranking

List all candidate hypotheses and score each:

| Hypothesis | Pre signal? | Near signal? | Node-specific? | 2+ metrics? | Matches known pattern? | Score |
|-----------|------------|-------------|---------------|------------|----------------------|-------|
| Container OOM | Yes (-12m) | Yes | Yes | Yes | Yes | 10 |
| Node disk full | No | No | — | No | No | 0 |
| CPU throttle | Yes | Yes | Yes | No | Yes | 7 |

**Threshold:** Score ≥ 6 → root cause. Score 3–5 → contributing factor.

---

## Phase 5: Remediation Recommendations

Based on confirmed root cause:

| Root Cause | Short-term | Long-term |
|-----------|-----------|----------|
| Container OOM (limit too low) | `kubectl set resources deployment/$D --limits=memory=2Gi` | VPA, load test to determine right-size |
| CPU throttle | Raise CPU limit or remove limit | HPA on custom metric, benchmark CFS |
| Disk saturation | Identify top writer, rate-limit logging | Use separate log volume, log shipping agent |
| Conntrack exhaustion | `sysctl -w net.netfilter.nf_conntrack_max=524288` | Use `ipvs` kube-proxy mode, disable conntrack for certain services |
| Kernel slab leak | Cordon + drain + restart node | Upgrade kernel, check for known dentry/inode leak bugs |
| NTP drift | `chronyc makestep` | Add NTP monitoring alert, ensure reliable NTP sources |
| ECC uncorrectable errors | Cordon node immediately | Replace memory DIMMs, report to hardware team |
| PLEG overloaded | Reduce pod density on node | Tune kubelet `--max-pods`, upgrade container runtime |

---

## Quick Reference: Key Thresholds

| Metric | Warning | Critical |
|--------|---------|---------|
| CPU utilisation | > 80% | > 95% |
| CPU iowait | > 10% | > 30% |
| CPU steal | > 3% | > 10% |
| Memory available ratio | < 20% | < 10% |
| Memory working_set/limit | > 80% | > 95% |
| Disk IO utilisation | > 70% | > 90% |
| Disk read latency (SSD) | > 5ms | > 20ms |
| Disk read latency (HDD) | > 30ms | > 100ms |
| Filesystem free | < 20% | < 10% |
| Inode free | < 20% | < 5% |
| Conntrack utilisation | > 70% | > 90% |
| TCP retransmit rate | > 1% | > 5% |
| NTP offset | > 50ms | > 1s |
| Node load / CPU count | > 1.0 | > 2.0 |
| procs_blocked | > 2 | > 10 |
| OOM kills | > 0 | > 3 in 10m |
| PLEG relist p99 | > 500ms | > 1s |
| CPU throttle ratio | > 25% | > 50% |
