# 03 — Syspeek Improvement Backlog

What to port from the Datadog Agent's approach to make Syspeek a serious
competitor. Ordered by impact. Theory and rationale in `01`/`02`.

Context: Syspeek tracks **one focused application's process group** (CPU + RSS,
plus Docker/k8s badges). That narrow scope is its edge — it can afford 1 s
sampling and rich context, but it must be exactly right and cheap per sample.

---

## S1 — Per-PID CPU baselines instead of one group sum  ★ highest value

**Problem.** The CPU baseline is a single summed tick count for the whole group,
and it is *invalidated whenever the group's PID list changes*. Any focused app
that spawns/reaps a child (browsers, Electron, IDEs, shells running commands)
therefore discards its baseline and shows "Warming up" — repeatedly, forever,
under churn. The displayed CPU is then either missing or one interval stale.

**Change.** Keep a per-entity baseline map keyed by process identity
`(pid, start_time)` plus the previous total-CPU counter:
- entity present in both samples → add its delta,
- entity newly seen → seed it, exclude it from this sample (no full reset),
- entity gone → drop it.

**Payoff.** Continuous CPU across churn; PID reuse handled for free; group
membership and baseline validity become independent concerns.

**Anchor:** Datadog's per-process `lastProcs[pid]` map and its "skip first
sighting" rule; Syspeek's current single-sum reset.

---

## S2 — Derive core count from the measured source, not `hardware_concurrency()`

**Problem.** The `top`-style scaling multiplies by `std::thread::hardware_concurrency()`,
which can disagree with the core count actually represented in `/proc/stat`
(cgroups, CPU affinity, containers). Wrong count → wrong percentage everywhere.

**Change.** Derive the scaling factor from the same view you measure (e.g. count
`cpuN` lines in `/proc/stat`, or affinity/cgroup-aware query). Cache it and
refresh rarely.

**Payoff.** Correct percentages on laptops, CI containers, and CPU-limited
sandboxes — the environments where a HUD is most useful.

---

## S3 — Make group discovery O(group), not O(all processes)

**Problem.** Discovering the focused app's process group currently enumerates
*every* PID and reads parent info per PID on each refresh/cache-miss. On a busy
desktop this is a full process-table scan just to find a handful of relatives.

**Change.** Walk up from the focus PID to the group root using only parent reads
(cheap), then collect descendants by a single `/proc` pass shared across the
sample — or cache parentage with a short TTL. Avoid re-reading the world per
focus change.

**Payoff.** Predictable cost independent of system process count; no periodic
CPU spikes from Syspeek itself (ironic for a CPU monitor).

---

## S4 — Be explicit about absence vs zero in the UI

**Problem.** "Warming up", `--`, and a real `0%` currently can blur together for
a user.

**Change.** Keep the three states distinct and visible (no data / warming /
value), and never render a missing read as `0`. Optionally show *why* (no focus,
permission, process gone).

**Payoff.** Trust. Users forgive gaps; they do not forgive silently wrong
numbers.

---

## S5 — Surface shared-memory reality (or stop implying RSS is total)

**Problem.** Summing per-process RSS double-counts shared pages. For a
multi-process app (Chromium family) the group RSS can exceed actual physical
usage.

**Change.** Either label it clearly as "sum of RSS (includes shared)", or use the
`shared` field from `/proc/<pid>/statm` to present a more honest estimate.

**Payoff.** Avoids the most common complaint about process-group memory meters.

---

## S6 — Add a compact host context strip (optional)

**Problem.** Per-app CPU% is meaningless without a baseline of overall system
load. A 40% app on an idle 12-core box is very different from 40% on a loaded one.

**Change.** Add an optional tiny host line: overall CPU% and `MemAvailable`
(not `MemFree`), sampled on a slower tier (e.g. 5–15 s).

**Payoff.** Turns a single number into context; teaches users the Free vs
Available distinction by showing the right one.

---

## S7 — Tier and cache: don't sample what didn't change

**Change.** Sample the cheap per-process counters every tick, but refresh
process metadata (exe, cmdline, group membership, Docker/k8s state) on a slower
TTL. Reuse buffers/handles across ticks.

**Payoff.** Lower steady-state overhead; smoother UI; room to lower the interval
for fast-changing CPU without paying metadata cost each tick.

---

## S8 — Lock the arithmetic with tests

**Change.** Add tests that feed **fixed fake counter samples** and assert the
emitted percentage/memory — including: first sample, membership change, counter
reset, PID reuse, unreadable file, zero cores. Syspeek already has unit tests;
extend them to cover the new per-PID baseline (S1) explicitly.

**Payoff.** The correctness properties in `01`/`02` become enforced, not hoped.

---

## Sequencing

1. **S1** (correctness under churn) — biggest user-visible win.
2. **S8** (tests around S1) — lock it in.
3. **S2** (correct scaling) and **S3** (bounded cost).
4. **S4**/**S5** (honesty), then **S6**/**S7** (polish and positioning).

---

## Deliberately out of scope

- Host-wide multi-metric export / time-series backend (Syspeek's value is
  local, single-app, always-on-top).
- cgroup/container per-container metrics beyond what the Docker/k8s badges need.
- eBPF or kernel-side collection — unnecessary at N = one group.
