# 01 — Linux CPU & Memory Metric Collection (Fundamentals)

Theory first. The goal is to *reason* about any collector (Datadog's, Syspeek's,
or one you write), not to memorize an API.

---

## Part A — CPU

### A1. The kernel never reports "CPU %"

Linux exposes **cumulative CPU time**, i.e. monotonically increasing counters.
Percentage is *derived* by sampling twice and dividing deltas:

```
usage_fraction = (counter_now - counter_prev) / (wall_time_now - wall_time_prev)
```

Consequences that drive every design decision:

- **Every first sample is useless.** There is no delta yet; a collector must
  discard the first reading and re-baseline whenever the counter is reset.
- **The interval defines the meaning.** 1 s and 15 s give different answers for
  bursty work; a reading is only valid over the window it was measured.
- **Counters can go backwards** (process exit, PID reuse, 32-bit wrap on some
  kernels). A correct collector clamps or resets; it never emits a negative %.

### A2. Two counter scopes

- **System**: `/proc/stat` first line `cpu` = aggregate of all cores;
  `cpu0`, `cpu1`, … = per core. Fields:
  `user, nice, system, idle, iowait, irq, softirq, steal, guest, guest_nice`.
- **Per process**: `/proc/<pid>/stat` `utime` + `stime` (user + system time).

`guest` is *already included* inside `user`; adding it again double-counts. The
standard "total" is the first 8 fields only.

### A3. Units: ticks vs seconds

Counters are in **clock ticks** (`USER_HZ`). Convert with `getconf CLK_TCK`
(commonly 100 → 10 ms, but *not guaranteed*). Rules:

- If both numerator and denominator are ticks, the unit **cancels** — you may
  skip conversion safely (this is what Syspeek does).
- If you mix ticks with wall-clock seconds (Datadog's process path does), you
  must divide by `CLK_TCK`.

### A4. Normalization convention (the important choice)

Given `(process_ticks_now - prev) / (all_core_ticks_now - prev)`:

- **×100 only** → percent of *whole-machine* capacity. One busy thread on a
  4-core box ≈ 25%. (Datadog's *system* check.)
- **×100 × ncores** → `top`-style, per-core-normalized. The same thread ≈ 100%.
  (Datadog's *process* check, and Syspeek.)

Both are correct; they answer different questions. Pick one and label it.
Cap the top-style value at `ncores × 100` so runaway math can't lie.

### A5. "How many cores?" is not obvious

`hardware_concurrency()` / `nproc` / `/proc/stat`'s `cpuN` count can disagree
inside **cgroups / CPU affinity / containers**. If you scale by the wrong core
count, every per-process percentage is wrong. A robust collector derives it from
the same source it measures and caches it (re-probing is expensive).

---

## Part B — Memory

### B1. Memory is a snapshot, not a counter

There is no delta math. You read the current state and report a gauge.

- **System**: `/proc/meminfo` — `MemTotal`, `MemFree`, **`MemAvailable`**,
  `Buffers`, `Cached`, `Slab`, `SReclaimable`, `PageTables`, `Committed_AS`, …
- **Process**: `/proc/<pid>/status` (`VmRSS`, `VmSize`, `VmSwap`) and
  `/proc/<pid>/statm` (page counts: `size resident shared text lib data dt`).

### B2. `MemFree` ≠ `MemAvailable` (the classic trap)

Linux uses idle RAM for page cache, which is reclaimable.

- `MemFree` = truly untouched RAM. Looks alarmingly small on healthy systems.
- `MemAvailable` = free + reclaimable cache, i.e. what a new workload can get
  without swapping. This is the honest "free memory" for capacity questions.

A host that reports `used = Total − Free` is *including cache*; a host that
reports `used = Total − Available` is closer to "pressure". Name it explicitly —
both are useful, silently swapping them is a bug.

### B3. RSS vs VMS vs Swap (per process)

- **RSS** (resident): pages actually in RAM now. The number users care about.
- **VMS** (virtual): address space reserved — includes untouched mappings,
  often huge and misleading.
- **Swap**: pages pushed to swap. On modern cgroup systems it can be pressure
  signal; often zero.

`statm` is in **pages**, so multiply by `PAGE_SIZE` (`getconf PAGE_SIZE`).

### B4. Shared memory is genuinely hard

A process's RSS double-counts shared pages (libraries, COW, tmpfs). Two
processes can each "show" the same physical page. Hence: summing RSS across a
process group over-counts real memory. Datadog exposes this via `statm`'s
`shared` field; Syspeek currently sums RSS and should at least acknowledge it.

---

## Part C — Failure modes every collector must decide on

| Situation | What it looks like | Sensible behavior |
|---|---|---|
| First sample / new entity | no previous counter | skip emit, mark "warming" |
| Counter goes backwards | negative delta | clamp to 0 **or** re-baseline |
| PID reused (old id, new proc) | identity changed | key by `(pid, starttime)` |
| Process exited mid-window | missing file | drop entity, don't zero-fill |
| Unreadable file | `EACCES`/`ENOENT` | treat as absent; don't crash |
| Zombie process | `State == Z` | skip or report as zero |
| Kernel thread | no cmdline | usually skip |
| Partial success (half the reads fail) | mixed | emit what succeeded, log the rest |

The through-line: **never turn missing data into a zero**. Zero means "measured,
and it was zero". Absence is a different fact.

---

## Part D — Sampling interval trade-offs

- Longer interval → smoother, cheaper, but hides bursts and delays alerts.
- Shorter interval → responsive, but jitter dominates the delta and I/O cost
  rises linearly with process count.
- Production collectors **tier** intervals: coarse for host/infra, fine for
  realtime/process views. Datadog uses 15 s (infra), 10 s (process), 2 s
  (realtime). Syspeek uses 1 s for a single focused group — fine because N is
  tiny, but it must scale its work to the group size, not to all processes.

---

## Anchor (optional further reading)

- System CPU: `pkg/collector/corechecks/system/cpu/cpu/`
- System memory: `pkg/collector/corechecks/system/memory/`
- Per-process `/proc` parsing: `pkg/process/procutil/process_linux.go`
- Syspeek CPU/mem: `cpp/src/CpuSampler.cpp`, `cpp/src/LinuxProcess.cpp`
