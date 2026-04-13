# Pod & Container Metrics — Deep Reference

> Covers `kube-state-metrics`, `cadvisor` (kubelet `/metrics/cadvisor`), and kubelet metrics.
> Replace `$NODE`, `$NAMESPACE`, `$POD`, `$CONTAINER` with actual label values.

---

## 1. Finding Relevant Pods

```promql
# All pods on a specific node
kube_pod_info{node="$NODE"}

# Pods that are not Running
kube_pod_status_phase{node="$NODE", phase!="Running"}

# Pods with non-zero restarts on a node
kube_pod_container_status_restarts_total{node="$NODE"} > 0

# Recent pod restarts (last 30 minutes)
increase(kube_pod_container_status_restarts_total{node="$NODE"}[30m]) > 0
```

---

## 2. Pod Resource Usage

### CPU

```promql
# CPU usage rate per container (cores)
rate(container_cpu_usage_seconds_total{node="$NODE", container!=""}[5m])

# CPU throttling ratio per container — % of time the container was throttled
rate(container_cpu_cfs_throttled_seconds_total{node="$NODE", container!=""}[5m]) /
rate(container_cpu_cfs_periods_total{node="$NODE", container!=""}[5m])

# Number of throttled periods (absolute count)
rate(container_cpu_cfs_throttled_periods_total{node="$NODE", container!=""}[5m])

# CPU usage vs CPU limit
rate(container_cpu_usage_seconds_total{node="$NODE", container!=""}[5m]) /
on (namespace, pod, container)
kube_pod_container_resource_limits{resource="cpu", node="$NODE"}
```

**Interpretation:**
- Throttle ratio > 25% → container CPU limit is too tight; application will experience periodic latency spikes even if average CPU looks normal
- CPU limit not set (`kube_pod_container_resource_limits` returns no result) → container can starve neighbours

### Memory

```promql
# Working set memory (what matters for OOM — excludes reclaimable cache)
container_memory_working_set_bytes{node="$NODE", container!=""}

# RSS memory
container_memory_rss{node="$NODE", container!=""}

# Cache memory (page cache — can be reclaimed)
container_memory_cache{node="$NODE", container!=""}

# Memory usage vs memory limit ratio
container_memory_working_set_bytes{node="$NODE", container!=""} /
on (namespace, pod, container)
kube_pod_container_resource_limits{resource="memory", node="$NODE"}

# Memory mapped files (mmap-heavy apps: databases, ML models)
container_memory_mapped_file{node="$NODE", container!=""}

# Container memory swap usage (if swap is enabled on the node)
container_memory_swap{node="$NODE", container!=""}

# OOM kill events at container level
kube_pod_container_status_last_terminated_reason{reason="OOMKilled", node="$NODE"}
```

**Interpretation:**
- `working_set / limit > 0.9` → container is near OOM kill threshold
- `OOMKilled` in terminated reason + matching spike in `node_vmstat_oom_kill` → confirms OOM as root cause

### Ephemeral Storage

```promql
# Container rootfs usage
container_fs_usage_bytes{node="$NODE", container!=""}

# Container rootfs limit
container_fs_limit_bytes{node="$NODE", container!=""}

# Logs size on disk (written via stdout/stderr to node journal)
container_fs_writes_bytes_total{node="$NODE", container!=""}

# Ephemeral volume usage (emptyDir, projected, etc.)
kube_pod_ephemeral_storage_request_bytes{node="$NODE"}
```

---

## 3. Pod Lifecycle & Scheduler

### Pod Status

```promql
# Pod phase distribution on node
count by (phase) (kube_pod_status_phase{node="$NODE"})

# Pods stuck in Pending (could be resource pressure or unschedulable)
kube_pod_status_phase{node="$NODE", phase="Pending"}

# Container readiness (0 = not ready)
kube_pod_container_status_ready{node="$NODE"}

# Container waiting reason
kube_pod_container_status_waiting_reason{node="$NODE"}

# Container terminated reason (OOMKilled, Error, Completed, etc.)
kube_pod_container_status_last_terminated_reason{node="$NODE"}

# Restart count with time filter
increase(kube_pod_container_status_restarts_total{node="$NODE"}[10m])
```

### Pod Scheduling Latency

```promql
# Time from pod creation to scheduled (scheduling queue latency)
# kube-scheduler metrics — not per-node but useful context
histogram_quantile(0.99,
  rate(scheduler_pod_scheduling_duration_seconds_bucket[5m])
)

# Unschedulable pods (no node fits)
kube_pod_status_unschedulable

# Scheduler queue depth
scheduler_pending_pods

# Scheduler attempts and errors
rate(scheduler_schedule_attempts_total[5m])
rate(scheduler_schedule_attempts_total{result="error"}[5m])
```

### Pod Disruption & Eviction

```promql
# Evicted pods (node pressure eviction)
kube_pod_status_reason{reason="Evicted", node="$NODE"}

# Preempted pods (higher priority pod displaced this one)
kube_pod_status_reason{reason="Preempting", node="$NODE"}

# Node-pressure eviction events (from kube-state-metrics events)
kube_events_total{reason="Evicting", involved_object_kind="Pod"}

# PodDisruptionBudget violations
kube_poddisruptionbudget_status_disruptions_allowed == 0
```

---

## 4. Container Runtime & cgroups

### cgroup CPU Accounting

```promql
# Total CPU periods observed by the cgroup
rate(container_cpu_cfs_periods_total{node="$NODE", container!=""}[5m])

# Periods throttled (container exceeded its CPU quota)
rate(container_cpu_cfs_throttled_periods_total{node="$NODE", container!=""}[5m])

# Throttle ratio per container
rate(container_cpu_cfs_throttled_periods_total{node="$NODE", container!=""}[5m]) /
rate(container_cpu_cfs_periods_total{node="$NODE", container!=""}[5m])
```

### cgroup Memory Accounting

```promql
# cgroup memory failures (allocation rejected by cgroup limit)
container_memory_failcnt{node="$NODE", container!=""}

# Rate of memory limit hits
rate(container_memory_failcnt{node="$NODE", container!=""}[5m])
```

**Interpretation:**
- `memory_failcnt` rate > 0 → container is hitting its memory limit frequently; even if not OOMKilled yet, latency is degraded as the kernel tries to reclaim memory within the cgroup

### Process & Thread Counts

```promql
# Process count inside containers (runaway fork or thread leak)
container_processes{node="$NODE", container!=""}

# Thread count
container_threads{node="$NODE", container!=""}

# Threads limit per container (from cgroup pids.max)
container_threads_max{node="$NODE", container!=""}
```

### Container Network (Per-Pod via Pause Container)

```promql
# Pod-level network receive/transmit (via pause/infra container)
rate(container_network_receive_bytes_total{node="$NODE", container=""}[5m])
rate(container_network_transmit_bytes_total{node="$NODE", container=""}[5m])

# Network errors at pod level
rate(container_network_receive_errors_total{node="$NODE", container=""}[5m])
rate(container_network_transmit_errors_total{node="$NODE", container=""}[5m])

# Packet drops at pod level
rate(container_network_receive_packets_dropped_total{node="$NODE", container=""}[5m])
rate(container_network_transmit_packets_dropped_total{node="$NODE", container=""}[5m])
```

### Container File System I/O

```promql
# Container I/O read/write bytes (overlayfs layer)
rate(container_fs_reads_bytes_total{node="$NODE", container!=""}[5m])
rate(container_fs_writes_bytes_total{node="$NODE", container!=""}[5m])

# Container I/O operations count
rate(container_fs_reads_total{node="$NODE", container!=""}[5m])
rate(container_fs_writes_total{node="$NODE", container!=""}[5m])
```

---

## 5. Kubelet Health

```promql
# Kubelet operation errors (pod sync, volume mount, etc.)
rate(kubelet_runtime_operations_errors_total{instance="$NODE"}[5m])

# Volume mount errors
rate(kubelet_volume_stats_available_bytes{instance="$NODE"}[5m])

# Kubelet pod lifecycle event generator latency
histogram_quantile(0.99,
  rate(kubelet_pod_worker_duration_seconds_bucket{instance="$NODE"}[5m])
)

# Kubelet image pull latency
histogram_quantile(0.99,
  rate(kubelet_image_pull_duration_seconds_bucket{instance="$NODE"}[5m])
)

# PLEG (Pod Lifecycle Event Generator) relist latency — high value → kubelet struggling
histogram_quantile(0.99,
  rate(kubelet_pleg_relist_duration_seconds_bucket{instance="$NODE"}[5m])
)

# PLEG relist interval — if > 3s, kubelet may be overwhelmed
histogram_quantile(0.99,
  rate(kubelet_pleg_relist_interval_seconds_bucket{instance="$NODE"}[5m])
)

# Number of running pods the kubelet is tracking
kubelet_running_pods{instance="$NODE"}

# Number of running containers
kubelet_running_containers{instance="$NODE"}
```

**Interpretation:**
- PLEG latency > 1s or PLEG relist interval > 3s → kubelet is overloaded; this can cause pods to appear unhealthy even when processes are fine
- `runtime_operations_errors` spike → container runtime (containerd/CRI-O) issues

---

## 6. Storage / PVC / Volume

```promql
# PVC bound/unbound status
kube_persistentvolumeclaim_status_phase{phase!="Bound"}

# PV available/total capacity
kube_persistentvolume_capacity_bytes
kube_persistentvolume_status_phase

# Volume attach errors (CSI driver issues)
storage_operation_duration_seconds_count{status="fail-unknown"}

# Volume plugin latency (attach/detach/mount)
histogram_quantile(0.99,
  rate(storage_operation_duration_seconds_bucket[5m])
)
```

---

## 7. Pod Quality of Service (QoS) & Priority

```promql
# QoS class (Guaranteed/Burstable/BestEffort) — determines eviction order
kube_pod_status_qos_class{node="$NODE"}

# Priority class values
kube_pod_spec_priorityClassName{node="$NODE"}

# Pods without resource requests (BestEffort — evicted first)
kube_pod_container_resource_requests{resource="memory", node="$NODE"} == 0
```

---

## 8. DaemonSet & System Pod Health

```promql
# DaemonSet pods not scheduled or not available
kube_daemonset_status_number_unavailable
kube_daemonset_status_number_misscheduled

# Critical system pods (kube-proxy, CNI, CSI) restart count
increase(
  kube_pod_container_status_restarts_total{
    node="$NODE",
    namespace=~"kube-system|monitoring"
  }[10m]
) > 0
```
