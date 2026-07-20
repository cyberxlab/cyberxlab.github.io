---
title: "Evaluating Linux Scheduler Latency: Practical Metrics and Measurement Techniques"
date: 2026-07-20T00:15:00Z
draft: false
tags: ["Linux", "Kernel", "Performance", "eBPF"]
categories: ["Technical Practices"]
author: "Cyber·X·Lab"
description: "A comprehensive guide to measuring and evaluating CPU scheduler runqueue latency on Linux using modern eBPF tools and low-overhead proc filesystem metrics."
summary: "Learn how to diagnose CPU starvation and evaluate scheduler latency on Linux. This guide covers eBPF-based tracing with runqlat and low-overhead stats from the proc filesystem."
toc: true
---

## Background and Motivation

In high-performance computing and enterprise microservices, application latency is often the defining metric for user experience and system reliability. When systems experience performance degradation, engineers typically investigate database queries, network round-trips, or disk I/O. However, a significant yet often overlooked source of tail latency is **CPU scheduler runqueue latency** (frequently referred to as scheduler latency).

Scheduler latency is the elapsed time between when a thread becomes ready to run (runnable) and when the Linux kernel scheduler actually assigns it to a CPU core to execute. Under heavy CPU load or misconfigured thread priorities, threads can spend an excessive amount of time waiting in the runqueue. This leads to subtle latency spikes that are notoriously difficult to capture using standard system monitors like `top` or `htop`. 

For instance, a transaction log writer or a critical network event-loop thread waiting for 50 milliseconds in a CPU runqueue will directly translate into a 50-millisecond latency spike for downstream clients, regardless of how fast the underlying storage or network is. To build highly responsive systems, platform and SRE engineers must know how to measure, evaluate, and tune scheduler latency under production constraints.

## Prerequisites

Before diving into the implementation steps, ensure your environment meets the following requirements:

1. **Operating System & Kernel**: Linux kernel 4.9+ (kernel 5.4+ is highly recommended to leverage Compile-Once Run-Everywhere BPF features).
2. **Privileges**: Root access (`sudo`) is required for running eBPF-based tools and tracing tools.
3. **Toolchains Installed**:
   - `bcc-tools` or `bpftrace` for dynamic tracing.
   - Python 3.x for custom scripting and low-overhead parsing.
4. **Development Utilities**: `clang`, `llvm`, and `libbpf` headers if you plan to compile custom eBPF programs from source.

## Step 1: Preparation

To accurately evaluate scheduler metrics, we first need to identify the built-in hooks and statistics provided by the Linux kernel.

### 1. The Low-Overhead Kernel Source: `/proc/[pid]/schedstat`

Every process in Linux exposes scheduler statistics via `/proc/[pid]/schedstat`. Reading this file does not require root privileges (though you must have permission to read the process's `/proc` directory) and incurs virtually zero system overhead, making it ideal for continuous production monitoring.

The format of `/proc/[pid]/schedstat` consists of three space-separated integers:
```bash
$ cat /proc/12345/schedstat
1000523951 1204002 542
```

Where:
1. **`se.sum_exec_runtime`**: Time spent executing on the CPU (in nanoseconds).
2. **`se.statistics.run_delay`**: Time spent waiting in the runqueue to be scheduled (in nanoseconds). This is our primary target metric.
3. **`se.statistics.nr_switches`**: Number of times this task was scheduled (both voluntary and involuntary context switches).

### 2. Global System-Wide Stats: `/proc/schedstat`

For a system-wide view, `/proc/schedstat` provides raw counters for each CPU core. This includes the total runqueue wait time and tasks run.

## Step 2: Core Implementation

To measure these metrics effectively in production, we implement two approaches:
1. A **low-overhead Python script** that continuously samples `/proc/[pid]/schedstat` to calculate running latency percentages.
2. An **eBPF-based command** leveraging `bpftrace` or `runqlat` to capture nanosecond-precise scheduling histograms.

### Method A: Low-Overhead SchedStat Parser (Python)

Save the following script as `sched_latency_monitor.py`. This script monitors a specific target PID, calculating the CPU percentage, latency percentage, and sleep percentage over a configurable interval.

```python
#!/usr/bin/env python3
import sys
import time
import os

if len(sys.argv) < 2:
    print("Usage: ./sched_latency_monitor.py <PID> [interval_seconds]")
    sys.exit(1)

pid = int(sys.argv[1])
interval = float(sys.argv[2]) if len(sys.argv) > 2 else 1.0
schedstat_path = f"/proc/{pid}/schedstat"

if not os.path.exists(schedstat_path):
    print(f"Error: PID {pid} or {schedstat_path} does not exist.")
    sys.exit(1)

def read_schedstat():
    try:
        with open(schedstat_path, 'r') as f:
            line = f.readline().strip()
            if not line:
                return None
            parts = list(map(int, line.split()))
            # Return (exec_time_ns, wait_time_ns, switches)
            return parts[0], parts[1], parts[2]
    except IOError:
        return None

print(f"Monitoring Scheduler Latency for PID {pid} at {interval}s intervals...")
print(f"{'TIMESTAMP':<20} | {'%CPU':<6} | {'%LAT (Wait)':<11} | {'%SLP':<6} | {'SWITCHES/s':<10}")
print("-" * 65)

prev_stats = read_schedstat()
if not prev_stats:
    print("Failed to read initial statistics.")
    sys.exit(1)

prev_time = time.time()

try:
    while True:
        time.sleep(interval)
        curr_stats = read_schedstat()
        curr_time = time.time()
        
        if not curr_stats:
            print("Process terminated.")
            break
            
        dt = curr_time - prev_time
        dt_ns = dt * 1e9
        
        d_exec = curr_stats[0] - prev_stats[0]
        d_wait = curr_stats[1] - prev_stats[1]
        d_switches = curr_stats[2] - prev_stats[2]
        
        # Calculate percentages relative to elapsed wall time
        cpu_pct = (d_exec / dt_ns) * 100.0
        lat_pct = (d_wait / dt_ns) * 100.0
        
        # SLP% represents the remaining time where the thread was voluntarily or involuntarily off the runqueue
        slp_pct = 100.0 - (cpu_pct + lat_pct)
        if slp_pct < 0:
            slp_pct = 0.0
            
        switches_sec = d_switches / dt
        
        timestamp = time.strftime('%Y-%m-%d %H:%M:%S')
        print(f"{timestamp:<20} | {cpu_pct:>5.1f}% | {lat_pct:>10.1f}% | {slp_pct:>5.1f}% | {switches_sec:>10.1f}")
        
        prev_stats = curr_stats
        prev_time = curr_time
except KeyboardInterrupt:
    print("\nMonitoring stopped.")
```

### Method B: High-Precision eBPF Histograms (`bpftrace`)

For fine-grained latency distribution details (such as identifying tail latency at the 99th percentile), use `bpftrace`. This one-liner instruments the scheduler tracepoints `sched_wakeup`, `sched_wakeup_new`, and `sched_switch` to calculate the exact wait-time distribution.

```bash
sudo bpftrace -e '
tracepoint:sched:sched_wakeup,
tracepoint:sched:sched_wakeup_new {
    @start[args.pid] = nsecs;
}
tracepoint:sched:sched_switch {
    $prev_state = args.prev_state;
    // If the previous process is still runnable but context switched, track its start
    if ($prev_state == 0) {
        @start[args.prev_pid] = nsecs;
    }
    
    // Check if the next process has an active start timestamp
    $s = @start[args.next_pid];
    if ($s > 0) {
        $delta = nsecs - $s;
        @latency_us = lhist($delta / 1000, 0, 10000, 500); // 0 to 10ms histogram, step 500us
        delete(@start[args.next_pid]);
    }
}
END {
    clear(@start);
}
'
```

## Step 3: Verification and Tuning

To verify your metrics pipeline, you can generate synthetic CPU contention using the standard `stress-ng` utility:

```bash
# Spawn 4 workers spinning on CPU to saturate the system
stress-ng --cpu 4 --timeout 30s
```

Run your Python monitor or the `bpftrace` script simultaneously in another terminal. You will observe the `%LAT (Wait)` percentage spike from nearly 0% up to 40%–80%, depending on core count, as processes compete for CPU cycles.

### Tuning Strategies

If your evaluation reveals unacceptable scheduler latency (e.g., latency exceeding 5ms on interactive/critical microservices), apply the following tuning steps:

1. **Adjust Niceness/Priority**:
   Use `nice` or `renice` to decrease a process's nice value (increasing CPU priority).
   ```bash
   sudo renice -n -10 -p <PID>
   ```

2. **Configure Real-Time Scheduling (SCHED_FIFO / SCHED_RR)**:
   For critical real-time threads (e.g., database log writers), assign a real-time scheduling policy.
   ```bash
   sudo chrt -f -p 99 <PID>
   ```
   *Warning: A thread scheduled under SCHED_FIFO with priority 99 can easily starve other processes if it goes into an infinite loop.*

3. **CPU Pinning (Affinity)**:
   Pin critical application processes to dedicated CPU cores, isolating them from systemic background noise.
   ```bash
   taskset -pc 2,3 <PID>
   ```

4. **Sysctl Kernel Parameter Tuning**:
   Tuning CFS (Completely Fair Scheduler) parameters can drastically affect responsiveness:
   - `/proc/sys/kernel/sched_latency_ns`: Target scheduler period.
   - `/proc/sys/kernel/sched_min_granularity_ns`: Minimum execution time a task gets before preemption.
   - `/proc/sys/kernel/sched_wakeup_granularity_ns`: Ability of woken tasks to preempt running ones.

## Best Practices Summary

- **Establish a Baseline**: Monitor scheduler latency under normal workloads. A healthy production system should keep scheduler latency below 1–2 milliseconds for critical threads.
- **Isolate Critical Threads**: Apply CPU pinning (`taskset`) and adjust priority (`renice` or `chrt`) to keep low-latency application workloads separated from bulk background processes.
- **Minimize Tracking Overhead**: Use Python or bash to read `/proc/[pid]/schedstat` for continuous profiling. Save heavyweight eBPF tracing tools for deep diagnostic drills.
- **Alert on Latency Ratios**: Set SRE alerts when the ratio of `%LAT / (%CPU + %LAT)` exceeds 20% on primary execution threads, as this signals systemic CPU starvation.

## References

- [Linux bcc/BPF Run Queue (Scheduler) Latency - Brendan Gregg](https://www.brendangregg.com/blog/2016-10-08/linux-bcc-runqlat.html)
- [eBPF Tutorial 9: Capturing Scheduling Latency and Recording as Histogram](https://eunomia.dev/tutorials/9-runqlat/)
- [SchedLat: Low-Overhead Script for Measuring CPU Scheduling Latency - Tanel Poder](https://tanelpoder.com/posts/schedlat-low-tech-script-for-measuring-cpu-scheduling-latency-on-linux/)
