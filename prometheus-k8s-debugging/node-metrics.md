# Node Exporter Metrics — Deep Reference

> All queries assume `instance` label matches the Kubernetes node name or its IP:port.
> Replace `$NODE` with the actual `instance` label value found in your Prometheus.

---

## 1. Finding the Right Instance Label

```promql
# List all node_exporter instances
count by (instance) (node_uname_info)

# Map hostname to instance label
node_uname_info{nodename="$HOSTNAME"}
```

---

## 2. Compute / CPU Saturation

### CPU Utilisation by Mode

```promql
# Per-mode CPU breakdown (idle, user, system, iowait, softirq, steal)
rate(node_cpu_seconds_total{instance="$NODE"}[5m])

# System-wide CPU utilisation (non-idle)
1 - avg by (instance) (rate(node_cpu_seconds_total{mode="idle", instance="$NODE"}[5m]))

# CPU iowait — indicates processes blocked waiting for disk/network I/O
avg by (instance) (rate(node_cpu_seconds_total{mode="iowait", instance="$NODE"}[5m]))

# CPU steal — hypervisor stealing time from this VM (cloud env signal)
avg by (instance) (rate(node_cpu_seconds_total{mode="steal", instance="$NODE"}[5m]))

# CPU softirq — high value → network/storage interrupt storm
avg by (instance) (rate(node_cpu_seconds_total{mode="softirq", instance="$NODE"}[5m]))
```

**Interpretation:**
- `iowait > 20%` combined with disk saturation → I/O bottleneck
- `steal > 5%` → noisy-neighbour or under-provisioned cloud instance
- `softirq > 5%` → NIC packet storm, check [§5 Network](#5-network-subsystem)

### CPU Run Queue & Load

```promql
# Load average (1/5/15 min) vs CPU count
node_load1{instance="$NODE"}
node_load5{instance="$NODE"}
node_load15{instance="$NODE"}

# Number of logical CPUs on the node
count by (instance) (node_cpu_seconds_total{mode="idle", instance="$NODE"})

# Run queue saturation: load1 / cpu_count > 1.5 → saturated
node_load1{instance="$NODE"} /
  count by (instance) (node_cpu_seconds_total{mode="idle", instance="$NODE"})

# Processes in uninterruptible sleep (D-state) — indicator of I/O or lock stall
node_procs_blocked{instance="$NODE"}

# Total runnable processes in queue
node_procs_running{instance="$NODE"}
```

**Interpretation:**
- `procs_blocked > 5` sustained → serious I/O or kernel lock contention
- `load1 / cpu_count > 2` → CPU saturated, scheduling latency increases

### Context Switches & Interrupts

```promql
# Context switch rate — high value with low CPU util → lock contention / spinlock
rate(node_context_switches_total{instance="$NODE"}[5m])

# Hardware interrupt rate — NIC/disk IRQ count
rate(node_intr_total{instance="$NODE"}[5m])
```

---

## 3. Memory Subsystem

### Basic Availability

```promql
# Available memory ratio (includes reclaimable cache)
node_memory_MemAvailable_bytes{instance="$NODE"} /
node_memory_MemTotal_bytes{instance="$NODE"}

# Committed memory ratio (how much the kernel has promised to apps)
node_memory_Committed_AS_bytes{instance="$NODE"} /
node_memory_MemTotal_bytes{instance="$NODE"}

# Slab cache (kernel data structures) — leak indicator
node_memory_Slab_bytes{instance="$NODE"}
node_memory_SReclaimable_bytes{instance="$NODE"}
node_memory_SUnreclaim_bytes{instance="$NODE"}

# Kernel stack memory consumption
node_memory_KernelStack_bytes{instance="$NODE"}
```

### Virtual Memory & Paging

```promql
# Page fault rate — major faults indicate swapping-in from disk
rate(node_vmstat_pgmajfault{instance="$NODE"}[5m])

# Minor page faults (normal, but watch for sudden spike)
rate(node_vmstat_pgfault{instance="$NODE"}[5m])

# Pages paged-in/out from swap
rate(node_vmstat_pgpgin{instance="$NODE"}[5m])
rate(node_vmstat_pgpgout{instance="$NODE"}[5m])

# Swap usage
node_memory_SwapFree_bytes{instance="$NODE"} /
node_memory_SwapTotal_bytes{instance="$NODE"}

# Page scan rate — kswapd actively reclaiming → memory pressure
rate(node_vmstat_pgscan_kswapd{instance="$NODE"}[5m])
rate(node_vmstat_pgsteal_kswapd{instance="$NODE"}[5m])

# Direct reclaim — application blocked waiting for memory reclaim (very bad)
rate(node_vmstat_pgsteal_direct{instance="$NODE"}[5m])
```

**Interpretation:**
- `pgmajfault` spike → working set > physical RAM, node is swapping
- `pgsteal_direct > 0` → process threads are being forced to reclaim memory themselves; expect severe latency spikes
- `SUnreclaim` growth over time → kernel slab leak (e.g. dentry/inode cache never freed)

### OOM & Memory Pressure

```promql
# OOM kill events (requires kernel ≥ 4.9 + node_exporter vmstat collector)
increase(node_vmstat_oom_kill{instance="$NODE"}[5m])

# Transparent Huge Pages allocation failures
rate(node_vmstat_thp_fault_fallback{instance="$NODE"}[5m])
rate(node_vmstat_thp_collapse_alloc_failed{instance="$NODE"}[5m])

# NUMA allocation failures (multi-socket nodes)
node_memory_numa_allocation_fail_total{instance="$NODE"}
```

### Huge Pages

```promql
# HugePages usage (databases/JVMs often require these)
node_memory_HugePages_Free{instance="$NODE"}
node_memory_HugePages_Total{instance="$NODE"}
node_memory_HugePages_Rsvd{instance="$NODE"}
```

---

## 4. Disk / I/O Subsystem

### Throughput & IOPS

```promql
# Read/write throughput (bytes/s) per device
rate(node_disk_read_bytes_total{instance="$NODE"}[5m])
rate(node_disk_written_bytes_total{instance="$NODE"}[5m])

# IOPS
rate(node_disk_reads_completed_total{instance="$NODE"}[5m])
rate(node_disk_writes_completed_total{instance="$NODE"}[5m])

# Merged I/O operations (OS-level I/O scheduler coalescing)
rate(node_disk_reads_merged_total{instance="$NODE"}[5m])
rate(node_disk_writes_merged_total{instance="$NODE"}[5m])
```

### Latency & Saturation

```promql
# Average I/O latency per operation (ms) — critical indicator
rate(node_disk_read_time_seconds_total{instance="$NODE"}[5m]) /
  rate(node_disk_reads_completed_total{instance="$NODE"}[5m]) * 1000

rate(node_disk_write_time_seconds_total{instance="$NODE"}[5m]) /
  rate(node_disk_writes_completed_total{instance="$NODE"}[5m]) * 1000

# Disk utilisation — percentage of time device was busy (saturation)
rate(node_disk_io_time_seconds_total{instance="$NODE"}[5m])

# I/O queue depth — sustained > 1 → device saturated
rate(node_disk_io_time_weighted_seconds_total{instance="$NODE"}[5m])

# Discards (SSDs/thin-provisioned volumes)
rate(node_disk_discards_completed_total{instance="$NODE"}[5m])
```

**Interpretation:**
- Read latency > 20ms on SSD or > 100ms on HDD → I/O bottleneck
- `io_time` approaching 1.0 → disk 100% busy, writes/reads queuing
- High queue depth with high latency → RAID controller or storage backend issue

### Filesystem & Inodes

```promql
# Filesystem free space ratio
node_filesystem_avail_bytes{instance="$NODE", fstype!~"tmpfs|overlay"} /
node_filesystem_size_bytes{instance="$NODE", fstype!~"tmpfs|overlay"}

# Inode free ratio — can run out before disk space
node_filesystem_files_free{instance="$NODE", fstype!~"tmpfs|overlay"} /
node_filesystem_files{instance="$NODE", fstype!~"tmpfs|overlay"}

# Filesystem readonly (remounted due to error)
node_filesystem_readonly{instance="$NODE"}
```

**Interpretation:**
- `filesystem_readonly = 1` → filesystem encountered errors and remounted RO; pods will fail writes
- Inode exhaustion (`files_free / files < 0.05`) causes `ENOSPC` even with space available

---

## 5. Network Subsystem

### Bandwidth & Packets

```promql
# Interface throughput (bytes/s)
rate(node_network_receive_bytes_total{instance="$NODE", device!~"lo|veth.*|docker.*|cilium.*"}[5m])
rate(node_network_transmit_bytes_total{instance="$NODE", device!~"lo|veth.*|docker.*|cilium.*"}[5m])

# Packet rate
rate(node_network_receive_packets_total{instance="$NODE", device!~"lo"}[5m])
rate(node_network_transmit_packets_total{instance="$NODE", device!~"lo"}[5m])
```

### Errors & Drops

```promql
# RX/TX errors — hardware or driver issues
rate(node_network_receive_errs_total{instance="$NODE"}[5m])
rate(node_network_transmit_errs_total{instance="$NODE"}[5m])

# RX/TX drops — ring buffer exhaustion or CPU too slow to drain
rate(node_network_receive_drop_total{instance="$NODE"}[5m])
rate(node_network_transmit_drop_total{instance="$NODE"}[5m])

# FIFO overruns — kernel ring buffer overflow
rate(node_network_receive_fifo_total{instance="$NODE"}[5m])
rate(node_network_transmit_fifo_total{instance="$NODE"}[5m])

# Carrier errors — physical link issues
rate(node_network_carrier_changes_total{instance="$NODE"}[5m])
```

### TCP / Socket State

```promql
# TCP connection states
node_sockstat_TCP_tw{instance="$NODE"}       # TIME_WAIT connections
node_sockstat_TCP_alloc{instance="$NODE"}    # allocated sockets
node_sockstat_TCP_inuse{instance="$NODE"}    # in-use TCP sockets
node_sockstat_UDPLITE_inuse{instance="$NODE"}

# Total socket memory usage (bytes)
node_sockstat_sockets_used{instance="$NODE"}

# TCP retransmit rate — congestion / packet loss indicator
rate(node_netstat_Tcp_RetransSegs{instance="$NODE"}[5m])

# TCP SYN backlog overflow — syn flood or slow accept()
rate(node_netstat_TcpExt_TCPReqQFullDrop{instance="$NODE"}[5m])

# TCP out-of-order packets
rate(node_netstat_TcpExt_TCPOFOQueue{instance="$NODE"}[5m])

# Conntrack table usage (critical for service mesh / iptables-based services)
node_nf_conntrack_entries{instance="$NODE"}
node_nf_conntrack_entries_limit{instance="$NODE"}

# Conntrack table utilisation ratio
node_nf_conntrack_entries{instance="$NODE"} /
node_nf_conntrack_entries_limit{instance="$NODE"}
```

**Interpretation:**
- `TCP_tw > 10000` → high connection churn; TIME_WAIT accumulation
- `TCPReqQFullDrop > 0` → accept backlog overflow; service is too slow to accept connections
- `conntrack_entries / limit > 0.8` → conntrack table near full; new connections will be silently dropped (common in high-traffic Kubernetes nodes with iptables kube-proxy)

---

## 6. Kernel Internals / OS Health

### File Descriptors

```promql
# System-wide file descriptor usage
node_filefd_allocated{instance="$NODE"}
node_filefd_maximum{instance="$NODE"}

# Utilisation ratio
node_filefd_allocated{instance="$NODE"} /
node_filefd_maximum{instance="$NODE"}
```

**Interpretation:** FD exhaustion causes `EMFILE`/`ENFILE` errors; pods may fail to open sockets or files.

### Entropy & Random

```promql
# Entropy pool — low value causes applications blocked on /dev/random
node_entropy_available_bits{instance="$NODE"}
```

### System Calls & Kernel Errors

```promql
# Fork rate — rapid process creation (fork bombs, runaway init containers)
rate(node_forks_total{instance="$NODE"}[5m])

# Interrupts per second (total)
rate(node_intr_total{instance="$NODE"}[5m])

# Softirq breakdown by type (NET_RX, NET_TX, BLOCK, TIMER, etc.)
rate(node_softirqs_total{instance="$NODE"}[5m])

# NMI (non-maskable interrupts) — hardware errors (ECC memory, CPU machine checks)
rate(node_softirqs_total{softirq="NMI", instance="$NODE"}[5m])
```

### Hardware Errors (EDAC / MCE)

```promql
# Memory correctable errors (ECC) — precursor to uncorrectable failure
node_edac_correctable_errors_total{instance="$NODE"}
node_edac_uncorrectable_errors_total{instance="$NODE"}

# CPU machine check errors (requires mcelog or rasdaemon)
node_cpu_core_throttles_total{instance="$NODE"}
node_cpu_package_throttles_total{instance="$NODE"}
```

**Interpretation:**
- `uncorrectable_errors > 0` → hardware memory failure; node should be cordoned immediately
- `cpu_package_throttles > 0` → thermal throttling; investigate cooling

### Time Synchronization

```promql
# NTP offset (seconds) — large offset causes certificate errors, log correlation issues
node_timex_offset_seconds{instance="$NODE"}

# NTP sync status (0 = unsynchronised)
node_timex_sync_status{instance="$NODE"}

# Estimated error bound
node_timex_estimated_error_seconds{instance="$NODE"}
```

### Kernel Scheduler & Cgroups

```promql
# Scheduler latency (requires schedstats; not always enabled)
# High value → CPU contention between cgroups
rate(node_schedstat_waiting_seconds_total{instance="$NODE"}[5m])
rate(node_schedstat_timeslices_total{instance="$NODE"}[5m])

# Cgroup CPU throttling — pods being throttled by their CPU limits
# (Query in pod-metrics.md §4)
```

---

## 7. Node-Level Kubernetes Integration

```promql
# Node resource capacity and allocatable
kube_node_status_capacity{node="$NODE"}
kube_node_status_allocatable{node="$NODE"}

# Node conditions (Ready, MemoryPressure, DiskPressure, PIDPressure, NetworkUnavailable)
kube_node_status_condition{node="$NODE", status="true"}

# Node taint presence (scheduled for eviction/maintenance?)
kube_node_spec_taint{node="$NODE"}

# Kubelet certificate expiry
kubelet_certificate_manager_client_expiration_renew_errors{instance="$NODE"}

# Kubelet API server communication errors
rest_client_requests_total{instance="$NODE", code=~"5.."}
```

---

## 8. Anomaly Baseline Queries

Use these to detect recent changes compared to historical averages:

```promql
# CPU usage now vs 1-day average (z-score style)
(
  1 - avg by (instance) (rate(node_cpu_seconds_total{mode="idle", instance="$NODE"}[5m]))
) /
(
  avg_over_time(
    (1 - avg by (instance) (rate(node_cpu_seconds_total{mode="idle", instance="$NODE"}[5m])))[1d:5m]
  )
)

# Memory available now vs 1-day average
node_memory_MemAvailable_bytes{instance="$NODE"} /
avg_over_time(node_memory_MemAvailable_bytes{instance="$NODE"}[1d])
```
