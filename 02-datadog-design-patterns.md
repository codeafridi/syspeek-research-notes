# 02 — Datadog Design Patterns Worth Adopting

Patterns observed in the Datadog Agent that make a collector trustworthy at
scale. These are the *reasons* behind the code, not the code.

---

## P1. Sample twice, emit a delta, never emit the raw counter as a rate

CPU is a counter; the useful metric is a **rate/percentage over a window**.
Store the previous sample per entity and emit only when a valid delta exists.

- Implication: the first observation of any entity is *silent*, and that is
  correct. Marking it "warming up" is honest; emitting 0% is a lie.

## P2. Key every baseline by entity identity, not by position

A baseline is only valid while the thing it measures is the **same thing**.
Identity must include something that survives PID reuse:

- process → `(pid, start_time)` (+ cmdline for exec-in-place),
- system → nothing (it never restarts; but handle counter wrap).

This is the single most important pattern for correctness under churn. It also
lets the collector treat "membership changed" and "baseline invalid" as
**separate** events instead of conflating them.

## P3. Never fabricate data for missing reads

Missing file, permission denied, process gone, zombie — all are *absence*, not
zero. Use explicit sentinels (e.g. `-1` = "no permission") and skip rather than
fill. A dashboard full of confident zeros is worse than visible gaps.

## P4. Degrade gracefully and keep collecting

A collector runs unattended forever. If one sub-reading fails, log it and emit
the rest. Only fail the whole cycle when *nothing* usable came back. The agent
should never take down the host it is measuring.

## P5. Tier the sampling rate by value

Cheap, coarse metrics (host CPU/mem) can run slowly (15 s). Rich, high-churn
metrics (per process, realtime) need faster sampling (2–10 s) and more
optimization. Different intervals for different cost/benefit is expected, not a
bug.

## P6. Optimize the hot path deliberately

The per-process path runs over hundreds/thousands of files every few seconds.
Wins that generalize:

- reuse a cached handle/directory instead of re-`open`ing `/proc`,
- pre-size and reuse buffers; avoid per-line string allocation,
- bound work for pathological inputs (e.g. hash only the first N args),
- prefer kernel fast paths when available, with a safe fallback.

## P7. Snapshot semantics for gauges, rate semantics for counters

Memory/limits are gauges (last value wins). CPU ticks, IO bytes, context
switches are counters (must be differentiated). Emitting a counter as a gauge
produces an ever-rising line; emitting a gauge as a rate produces nonsense. The
metric type is part of its meaning.

## P8. Concurrency is folded in, not double-counted

CPU accounting includes guest time inside user time; irq/softirq overlap; iowait
sits on the idle side. The collector's "total" must be an intentional set of
fields, documented, not "sum everything you saw". (Datadog excludes guest from
its total for exactly this reason.)

## P9. Platform differences are explicit

Linux, Windows, macOS, cgroup v1/v2, container runtimes all differ. The clean
shape is: one portable interface, platform-specific implementations, and
unit-tests per platform. Avoid `#ifdef` spaghetti in the calculation layer.

## P10. Tests anchor the arithmetic

Fake the raw inputs (fixed counter samples) and assert the emitted values. This
locks down unit math, failure paths, and platform branches without needing a
live host. Datadog does this for both system checks and the `/proc` parser.

---

## The one-sentence summary

A durable collector is: **delta-based, identity-keyed, absence-honest,
rate-vs-gauge-aware, tiered, and tested against fake inputs.**
