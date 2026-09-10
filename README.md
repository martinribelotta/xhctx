# Xhctx — RISC-V Hart-Context Extension

[![Build specification](https://github.com/martinribelotta/xhctx/actions/workflows/build-pdf.yml/badge.svg)](https://github.com/martinribelotta/xhctx/actions/workflows/build-pdf.yml)
[![Latest release](https://img.shields.io/github/v/release/martinribelotta/xhctx)](https://github.com/martinribelotta/xhctx/releases/latest)
[![License: CC BY 4.0](https://img.shields.io/badge/License-CC%20BY%204.0-lightgrey.svg)](https://creativecommons.org/licenses/by/4.0/)

`Xhctx` ("Hart Context") is an experimental, non-ratified RISC-V
extension that replaces the classical interrupt model
(`trap → ISR → mret`) with **persistent execution contexts, activated
by events, that behave architecturally as RISC-V harts**. Hart
selection is arbitrated entirely by hardware — there is no software
scheduler, no ISR prologue/epilogue, and no register save/restore: all
architectural state is fully banked per hart, so a "context switch" is
just selecting a different bank.

> **Status: draft and development.** This is an individual, early-stage
> proposal, not endorsed by or affiliated with RISC-V International.
> Everything is subject to change. Feedback is welcome — see
> [Feedback and contributing](#feedback-and-contributing) below.

## Key ideas

- **Contexts are harts.** Each of the (reference: 16) contexts has its
  own PC, register file, `mhartid`, `mepc`/`mcause` — but `mret` does
  not exist in this model, because no operation in `Xhctx` restores a
  precisely-suspended computation and continues it. Every reactivation
  is a fresh dispatch driven by an event.
- **Fully preemptive, fixed-priority arbitration**, implemented in
  hardware (an Event Fabric + per-hart FIFOs with mandatory bypass for
  minimum latency), not a programmable RTOS scheduler.
- **No new instruction encoding.** Everything is accessed through
  existing `CSRRW`/`CSRRS`/`CSRRC` on new CSR addresses, plus a
  dependency on the standard `Smcsrind` extension for cross-hart
  introspection (`mepc[ctx_id]`, `mcause[ctx_id]`, and friends) instead
  of burning one raw CSR address per hart.
- **U-mode can still request service from an M-mode hart** — `ecall`
  from U-mode is a defined, resumable request channel to a
  firmware-configured destination hart, distinct from the terminal
  fault path used for genuine exceptions.

The reasoning behind each of these is in the spec itself, not repeated
here — see [Reading the spec](#reading-the-spec).

## Reading the spec

The formal specification is written chapter-by-chapter as AsciiDoc
pages under [`modules/ROOT/pages/`](modules/ROOT/pages/), assembled
into one document by [`src/Xhctx.adoc`](src/Xhctx.adoc):

1. [Introduction and Motivation](modules/ROOT/pages/intro.adoc)
2. [The Xhctx Hart Model](modules/ROOT/pages/hart-model.adoc)
3. [Control and Status Registers](modules/ROOT/pages/csrs.adoc)
4. [Event Fabric and Routing](modules/ROOT/pages/event-fabric.adoc)
5. [Software-Sourced Events](modules/ROOT/pages/software-events.adoc)
6. [Priority and Preemption](modules/ROOT/pages/priority-preemption.adoc)
7. [Context (Hart) Lifecycle](modules/ROOT/pages/context-lifecycle.adoc)
8. [Exception Delivery](modules/ROOT/pages/exception-delivery.adoc)
9. [Programmer's Model](modules/ROOT/pages/programmers-model.adoc)
10. [RV32E ISA Compatibility](modules/ROOT/pages/compatibility.adoc)
11. [Reference Microarchitecture](modules/ROOT/pages/microarchitecture.adoc)
12. [Open Issues](modules/ROOT/pages/open-issues.adoc)
13. [Rationale and Comparison with Prior Art](modules/ROOT/pages/rationale.adoc)

For the *why* behind the project rather than the normative detail, see
[ARCHITECTURE.md](ARCHITECTURE.md).

The easiest way to read it, though, is the built PDF — see the
[latest release](https://github.com/martinribelotta/xhctx/releases/latest),
or build it yourself below.

## Building the spec

This repo uses RISC-V International's own specification tooling
([`riscv/docs-spec-template`](https://github.com/riscv/docs-spec-template):
Antora + AsciiDoc, built with `asciidoctor-pdf` inside a Docker
container). Requires Docker and `make`.

```bash
git clone --recurse-submodules https://github.com/martinribelotta/xhctx.git
cd xhctx
make build
```

> **Windows/Git Bash note:** MSYS rewrites the `-w /build` Docker
> argument into an invalid Windows path. Prefix the command instead:
> `MSYS_NO_PATHCONV=1 MSYS2_ARG_CONV_EXCL="*" make build`.

Output lands in `build/`: an ARC-style-named PDF and an HTML rendering.
CI ([`.github/workflows/build-pdf.yml`](.github/workflows/build-pdf.yml))
builds both on every push/PR and attaches the PDF to a
[GitHub Release](https://github.com/martinribelotta/xhctx/releases)
whenever a `v*` tag is pushed.

## Repository layout

| Path | Purpose |
| --- | --- |
| `ARCHITECTURE.md` | Motivation and design principles (non-technical) |
| `modules/ROOT/pages/` | Spec chapters, one file per chapter |
| `src/Xhctx.adoc` | PDF/HTML assembler entry point |
| `modules/ROOT/nav.adoc` | Chapter navigation/ordering |
| `docs-resources/` | Shared RISC-V doc tooling (git submodule) |
| `Makefile`, `scripts/` | Build tooling, adapted from `docs-spec-template` |

## Versioning

Releases follow RISC-V's ARC lifecycle scheme, `vMAJOR.FRAC` (e.g.
`v0.0`, `v0.01`, ... `v0.6` = development-complete, `v0.8` = stabilized,
`v0.9` = frozen, `v0.99` = ratification-ready, `v1.0` = ratified) — not
semantic versioning. See
[`scripts/release-info.sh`](scripts/release-info.sh) for the exact
rules.

## Feedback and contributing

This is an early-stage, individual proposal. The intended next step is
requesting feedback on the public
[`isa-dev`](https://groups.google.com/a/groups.riscv.org/g/isa-dev)
mailing list. Until then, issues and discussion on this repository are
welcome.

## License

Licensed under [CC BY 4.0](https://creativecommons.org/licenses/by/4.0/)
— see [LICENSE](LICENSE).
