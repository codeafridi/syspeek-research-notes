# Notes

## Topic 1: Linux CPU & Memory Metrics (Datadog vs Syspeek)

What we covered:
- CPU is cumulative tick counters, not a percent. Percent = delta ticks / delta time; first sample is useless.
- System CPU: /proc/stat (first `cpu` line = all cores). Per process: /proc/<pid>/stat utime+stime.
- Ticks convert via `getconf CLK_TCK` (usually 100). If both sides are ticks, units cancel.
- `top`-style percent multiplies by core count; Datadog system check does not. Pick one and label it.
- Memory is a snapshot. /proc/meminfo MemFree != MemAvailable (Available includes reclaimable cache).
- Per process: VmRSS (real) vs VmSize (virtual), /proc/<pid>/statm is in pages (x PAGE_SIZE).
- Summing RSS across a group double-counts shared pages.
- Datadog: gopsutil for host metrics, own parser (procutil) for per-process, tiered intervals (15s/10s/2s).
- Syspeek: 1s sampling of one focused process group; CPU baseline resets whenever group PIDs change.

Implementation ideas for Syspeek (by priority):
1. Per-PID CPU baselines keyed by (pid, starttime). Seed new pids, drop gone, no full reset on membership change.
2. Derive core count from /proc/stat cpuN lines (or affinity/cgroup), not hardware_concurrency().
3. Group discovery should cost O(group), not O(all processes).
4. UI: keep "no data" vs "warming" vs "value" separate; never render missing as 0.
5. Be honest that summed RSS includes shared pages (or use statm shared field).
6. Optional host context line: overall CPU% + MemAvailable, slower interval.
7. Tier sampling: cheap counters every tick, metadata (exe/cmdline/group/docker/k8s) on a slower TTL.
8. Tests with fixed fake counter samples: first sample, membership change, counter reset, PID reuse, unreadable file.
