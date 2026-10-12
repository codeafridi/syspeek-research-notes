# Observability & SRE Research Notes

Working notes toward building **Syspeek** into a serious competitor to the
Datadog Agent, using the Datadog Agent source as the reference implementation.

Principle: these notes hold **concepts, trade-offs, and things worth porting** —
not code walkthroughs, not function dumps. Cite the source tree only as an
anchor for further reading.

Source trees (reference only, never modified):
- `~/Research/datadog-agent/` — reference implementation
- `~/Research/Syspeek/` — the project being improved

---

## Topics

| # | Topic | Status | Notes |
|---|-------|--------|-------|
| 1 | Linux CPU & memory metric collection | in progress | `01-cpu-memory-fundamentals.md` |

## Cross-cutting

| File | Purpose |
|------|---------|
| `02-datadog-design-patterns.md` | Observable/collection patterns worth adopting |
| `03-syspeek-improvements.md` | Prioritized backlog of changes for Syspeek |

---

## How to use these notes

- `01` is theory: read it to understand *why* the numbers look the way they do.
- `02` is engineering taste: patterns that make a collector trustworthy at scale.
- `03` is action: the concrete, prioritized improvements to make in Syspeek.

_Last updated: topic 1 first pass._
