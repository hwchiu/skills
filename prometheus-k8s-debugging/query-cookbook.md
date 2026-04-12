# PromQL Query Cookbook — Scenario-Based Recipes

> Ready-to-run queries for common Kubernetes node/pod failure scenarios.
> All queries are parameterised. Set variables before running:
> ```bash
> NODE="your-node-name-or-ip:9100"
> NAMESPACE="default"
> POD="your-pod-name"
> T=$(date -d "2024-01-15T10:30:00Z" +%s)   # event time as unix timestamp
> START=$((T - 1800))   # 30 min before
> END=$((T + 1800))     # 30 min after
> ```

---

## Scenario 1: Node Went NotReady

**Hypothesis checklist:** memory pressure → OOM → kernel panic | disk full → kubelet crash | network partition | NTP drift | hardware error

```promql
# 1a. Memory pressure? (should trigger MemoryPressure condition first)
node_memory_MemAvailable_bytes{instance="$NODE"} /
node_memory_MemTotal_bytes{instance="$NODE"}

# 1b. OOM events near event time?
increase(node_vmstat_oom_kill{instance="$NODE"}[5m])

# 1c. Disk pressure?
node_filesystem_avail_bytes{instance="$NODE", mountpoint="/"} /
node_filesystem_size_bytes{instance="$NODE", mountpoint="/"}

# 1d. Node conditions
kube_node_status_condition{node="$NODE", status="true"}

# 1e. Kubelet PLEG stalled?
histogram_quantile(0.99,
  rate(kubelet_pleg_relist_duration_seconds_bucket{instance="$NODE"}[5m])
)

# 1f. Processes in D-state (stuck in I/O)?
node_procs_blocked{instance="$NODE"}

# 1g. NTP sync lost?
node_timex_sync_status{instance="$NODE"}

# 1h. Hardware errors (ECC)?
increase(node_edac_uncorrectable_errors_total{instance="$NODE"}[10m])

# 1i. CPU throttling (entire node thermal)?
increase(node_cpu_package_throttles_total{instance="$NODE"}[10m])
```

---

## Scenario 2: Pod OOMKilled

**Hypothesis checklist:** container working set exceeds limit | node-level OOM (limit set too high) | memory leak | sudden spike in data volume

```promql
# 2a. Which containers were OOMKilled?
kube_pod_container_status_last_terminated_reason{reason="OOMKilled", node="$NODE"}

# 2b. Container working set vs limit ratio (should be near 1.0 before kill)
container_memory_working_set_bytes{namespace="$NAMESPACE", pod="$POD"} /
on (namespace, pod, container)
kube_pod_container_resource_limits{resource="memory", namespace="$NAMESPACE", pod="$POD"}

# 2c. Was the node itself memory-pressured?
increase(node_vmstat_oom_kill{instance="$NODE"}[10m])
node_memory_MemAvailable_bytes{instance="$NODE"} /
node_memory_MemTotal_bytes{instance="$NODE"}

# 2d. Was there direct memory reclaim? (node-level pressure sign)
rate(node_vmstat_pgsteal_direct{instance="$NODE"}[5m])

# 2e. Memory growth trend (is it a leak?)
deriv(container_memory_working_set_bytes{
  namespace="$NAMESPACE", pod="$POD"
}[30m])

# 2f. Memory failcnt (cgroup hit limit before kill)
rate(container_memory_failcnt{namespace="$NAMESPACE", pod="$POD"}[5m])

# 2g. RSS vs working set gap (large gap = excessive page cache inside container)
container_memory_working_set_bytes{namespace="$NAMESPACE", pod="$POD"} -
container_memory_rss{namespace="$NAMESPACE", pod="$POD"}
```

---

## Scenario 3: CPU Throttling / Latency Spikes

**Hypothesis checklist:** CPU limit too tight | CPU noisy neighbour | I/O wait inflating latency | GC pause | lock contention

```promql
# 3a. Throttle ratio per container
rate(container_cpu_cfs_throttled_periods_total{
  namespace="$NAMESPACE", pod="$POD", container!=""
}[5m]) /
rate(container_cpu_cfs_periods_total{
  namespace="$NAMESPACE", pod="$POD", container!=""
}[5m])

# 3b. Node CPU iowait (I/O rather than CPU pressure)
avg by (instance) (rate(node_cpu_seconds_total{mode="iowait", instance="$NODE"}[5m]))

# 3c. Node CPU steal (cloud host is stealing cycles)
avg by (instance) (rate(node_cpu_seconds_total{mode="steal", instance="$NODE"}[5m]))

# 3d. Load saturation
node_load1{instance="$NODE"} /
count by (instance) (node_cpu_seconds_total{mode="idle", instance="$NODE"})

# 3e. Context switches rate (high = lock contention / too many threads)
rate(node_context_switches_total{instance="$NODE"}[5m])

# 3f. Processes in D-state
node_procs_blocked{instance="$NODE"}

# 3g. Scheduler throttle (cgroup)
rate(container_cpu_cfs_throttled_seconds_total{
  namespace="$NAMESPACE", pod="$POD"
}[5m])
```

---

## Scenario 4: Disk I/O Saturation

**Hypothesis checklist:** disk throughput limit hit | too many small random I/Os | log storm | overlayfs metadata storms | disk hardware failure

```promql
# 4a. Disk utilisation per device
rate(node_disk_io_time_seconds_total{instance="$NODE"}[5m])

# 4b. Average read/write latency
rate(node_disk_read_time_seconds_total{instance="$NODE"}[5m]) /
  (rate(node_disk_reads_completed_total{instance="$NODE"}[5m]) + 0.001) * 1000

rate(node_disk_write_time_seconds_total{instance="$NODE"}[5m]) /
  (rate(node_disk_writes_completed_total{instance="$NODE"}[5m]) + 0.001) * 1000

# 4c. Queue depth
rate(node_disk_io_time_weighted_seconds_total{instance="$NODE"}[5m])

# 4d. Which container is writing most?
topk(5,
  rate(container_fs_writes_bytes_total{node="$NODE", container!=""}[5m])
)

# 4e. Which container is reading most?
topk(5,
  rate(container_fs_reads_bytes_total{node="$NODE", container!=""}[5m])
)

# 4f. CPU iowait (process-level impact of I/O saturation)
avg by (instance) (rate(node_cpu_seconds_total{mode="iowait", instance="$NODE"}[5m]))

# 4g. Filesystem nearly full?
node_filesystem_avail_bytes{instance="$NODE", fstype!~"tmpfs|overlay"} /
node_filesystem_size_bytes{instance="$NODE", fstype!~"tmpfs|overlay"}

# 4h. Inode exhaustion?
node_filesystem_files_free{instance="$NODE", fstype!~"tmpfs|overlay"} /
node_filesystem_files{instance="$NODE", fstype!~"tmpfs|overlay"}

# 4i. Filesystem went readonly?
node_filesystem_readonly{instance="$NODE"} == 1
```

---

## Scenario 5: Network Issues / Pod Connectivity

**Hypothesis checklist:** conntrack exhaustion | NIC ring buffer overflow | MTU mismatch | CNI pod | kube-proxy iptables issues | DNS failure

```promql
# 5a. Conntrack table saturation
node_nf_conntrack_entries{instance="$NODE"} /
node_nf_conntrack_entries_limit{instance="$NODE"}

# 5b. Packet drops on host NIC
rate(node_network_receive_drop_total{instance="$NODE", device!~"lo|veth.*"}[5m])
rate(node_network_transmit_drop_total{instance="$NODE", device!~"lo|veth.*"}[5m])

# 5c. NIC errors
rate(node_network_receive_errs_total{instance="$NODE"}[5m])
rate(node_network_transmit_errs_total{instance="$NODE"}[5m])

# 5d. TCP retransmits (congestion or loss)
rate(node_netstat_Tcp_RetransSegs{instance="$NODE"}[5m])

# 5e. SYN backlog overflow (service not accepting fast enough)
rate(node_netstat_TcpExt_TCPReqQFullDrop{instance="$NODE"}[5m])

# 5f. TIME_WAIT exhaustion (port reuse issues)
node_sockstat_TCP_tw{instance="$NODE"}

# 5g. Pod-level network drops
rate(container_network_receive_packets_dropped_total{node="$NODE", container=""}[5m])
rate(container_network_transmit_packets_dropped_total{node="$NODE", container=""}[5m])

# 5h. DNS resolution failures (CoreDNS)
rate(coredns_dns_responses_total{rcode="SERVFAIL"}[5m])
rate(coredns_dns_responses_total{rcode="NXDOMAIN"}[5m])

# 5i. kube-proxy sync latency
histogram_quantile(0.99,
  rate(kubeproxy_sync_proxy_rules_duration_seconds_bucket[5m])
)
```

---

## Scenario 6: Pod Stuck in Pending / Scheduling Failure

```promql
# 6a. Pods in Pending state
kube_pod_status_phase{phase="Pending"}

# 6b. Unschedulable pods
kube_pod_status_unschedulable > 0

# 6c. Scheduler errors
rate(scheduler_schedule_attempts_total{result="error"}[5m])

# 6d. Node allocatable resources remaining
kube_node_status_allocatable{node="$NODE"}

# 6e. Total requests on node vs allocatable
sum by (node) (kube_pod_container_resource_requests{resource="cpu", node="$NODE"}) /
on (node) kube_node_status_allocatable{resource="cpu", node="$NODE"}

sum by (node) (kube_pod_container_resource_requests{resource="memory", node="$NODE"}) /
on (node) kube_node_status_allocatable{resource="memory", node="$NODE"}

# 6f. Node taints blocking scheduling
kube_node_spec_taint{node="$NODE"}

# 6g. PVC not bound (pod waiting for storage)
kube_persistentvolumeclaim_status_phase{phase!="Bound"}
```

---

## Scenario 7: Workload Throttled by Resource Quotas

```promql
# 7a. Namespace CPU quota utilisation
sum by (namespace) (
  rate(container_cpu_usage_seconds_total{container!=""}[5m])
) /
on (namespace)
kube_resourcequota{type="hard", resource="limits.cpu"}

# 7b. Namespace memory quota utilisation
sum by (namespace) (
  container_memory_working_set_bytes{container!=""}
) /
on (namespace)
kube_resourcequota{type="hard", resource="limits.memory"}

# 7c. Namespace pod count quota
count by (namespace) (kube_pod_info) /
on (namespace)
kube_resourcequota{type="hard", resource="pods"}
```

---

## Scenario 8: Node Exporter / Monitoring Stack Health

```promql
# 8a. Node exporter up?
up{job="node-exporter", instance="$NODE"}

# 8b. Scrape duration (slow scrape = node load issue)
scrape_duration_seconds{job="node-exporter", instance="$NODE"}

# 8c. Prometheus storage pressure
prometheus_tsdb_storage_blocks_bytes
prometheus_tsdb_head_series

# 8d. Target scrape failures
rate(scrape_samples_scraped[5m]) == 0
```

---

## Utility: Cross-Node Comparison

When a single node misbehaves, compare it against the cluster to confirm anomaly:

```promql
# CPU usage all nodes — spot the outlier
1 - avg by (instance) (rate(node_cpu_seconds_total{mode="idle"}[5m]))

# Memory pressure all nodes
node_memory_MemAvailable_bytes /
node_memory_MemTotal_bytes

# Disk I/O saturation all nodes
max by (instance) (rate(node_disk_io_time_seconds_total[5m]))

# Network drops all nodes
sum by (instance) (
  rate(node_network_receive_drop_total{device!~"lo|veth.*"}[5m]) +
  rate(node_network_transmit_drop_total{device!~"lo|veth.*"}[5m])
)

# OOM kills all nodes
sum by (instance) (increase(node_vmstat_oom_kill[10m]))
```

---

## Utility: Time-Range Execution Template

```bash
#!/bin/bash
# Run a PromQL range query and output CSV for the event window
PROMETHEUS_URL="http://prometheus:9090"
NODE="node1:9100"
EVENT_TIME="2024-01-15T10:30:00Z"
T=$(date -d "$EVENT_TIME" +%s)
START=$((T - 1800))
END=$((T + 1800))
STEP="15s"

QUERY='rate(node_cpu_seconds_total{mode="iowait",instance="'"$NODE"'"}[5m])'

curl -sG "$PROMETHEUS_URL/api/v1/query_range" \
  --data-urlencode "query=$QUERY" \
  --data-urlencode "start=$START" \
  --data-urlencode "end=$END" \
  --data-urlencode "step=$STEP" \
  | jq -r '
    .data.result[] |
    .metric as $m |
    .values[] |
    [(.[0] | todate), .[1]] |
    @csv
  '
```
