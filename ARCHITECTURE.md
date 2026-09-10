# Xhctx — Design Rationale

> **Note.** This document is the original motivation and design
> principles behind the `Xhctx` extension. It intentionally does **not**
> duplicate technical content — every concrete architectural decision
> (CSRs, event fabric, context lifecycle, exception delivery, ISA
> compatibility, reference microarchitecture, and rationale/prior-art
> comparison) lives in the formal specification, one chapter per page
> under [`modules/ROOT/pages/`](modules/ROOT/pages/), assembled into a
> single document at [`src/Xhctx.adoc`](src/Xhctx.adoc). This file used
> to carry both; keeping the same material in two places, in two
> languages, drifting independently, was a maintenance hazard rather
> than a feature — see [README.md](README.md) for how to build the
> formal spec into a PDF.

## 1. Motivation

`Xhctx` is an **experimental RISC-V extension** that replaces the
classical interrupt model (`trap → ISR → mret`) with a model of
**persistent execution contexts activated by events**.

The classical model is:

```
Event
  ↓
Trap
  ↓
ISR
  ↓
mret
```

The proposed model is:

```
Event
  ↓
Event Fabric
  ↓
Persistent context
  ↓
Abandonment
  ↓
Next context
```

The unit of scheduling ceases to be the interrupt and becomes the
**Execution Context** — architecturally, a RISC-V hart (see the formal
spec's Hart Model chapter for why, and where that framing deliberately
diverges from a standard hart).

The primary goal is a microcontroller architecture with:

- Deterministic, minimal activation latency.
- Zero software-managed register save/restore.
- No ISR prologue/epilogue.
- No `mret` as a control-transfer mechanism.
- Contexts resident entirely in hardware.
- An Event Fabric with priority-based arbitration.
- A viable implementation on a small RV32E core.

## 2. Design Principles

1. Contexts are persistent: they are never created or destroyed during
   execution.
2. Events activate contexts; no ephemeral ISRs exist.
3. Software never writes the currently active context directly.
4. All execution state is fully banked.
5. A context switch consists solely of selecting a different bank.
6. Hardware arbitrates events; it does not implement an RTOS scheduler.
7. Behavior must be deterministic and easily verifiable.
8. The ISA must remain compatible with RV32E wherever possible.

Everything that follows from these eight principles — how they cash
out into actual CSRs, wire formats, and hardware behavior, including
the trade-offs and open issues that showed up along the way — is
worked out in the formal specification, not repeated here.
