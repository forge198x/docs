# Forge198x — design notes (deferred capture)

Forge198x is **deferred** — gated on Asm198x maturity and a stable Emu198x debug
surface. These are notes captured now per the decision's instruction to "capture
the idea and its constraints" without building; they are a sketch for whoever
starts the build, not a commitment to specifics. The binding *why* and the
constraints live in [`../../decisions/forge198x-ide.md`](https://github.com/198x)
(umbrella `decisions/forge198x-ide.md`) — read it first; this only adds the
concrete *how* shape.

## What Forge198x is

The integrator: one **edit → assemble → run → debug** loop over tools the family
already has. A thin front-end over Asm198x (assemble/disassemble) and Emu198x
(run/debug) — it never reimplements either. The human face of capabilities the
tool surface already exposes to agents.

## The loop, mapped to real surfaces

Each stage is a call the front-end makes to an existing surface — and the same
call an agent can make directly (that symmetry is the contract, not a bonus).

1. **Edit** — vintage source (per-CPU dialect) in the editor. Forge owns only
   the editing UX; the language semantics come from Asm198x.
2. **Assemble** — invoke the `asm198x` CLI on the source. Out: the binary
   (ROM/disk image), a listing, and a symbol table. Errors map back to source
   spans. Disassembly (`asm198x` once it disassembles) drives the reverse view.
3. **Run** — hand the assembled image to Emu198x for the target machine and run
   it. Against today's MCP surface that is roughly: `insert_media` / `load_media`
   → `run_frames` / `run_until_pc`, then `save_screenshot` to show output.
4. **Debug** — drive Emu198x's inspection surface and join it to Asm198x's
   symbols/listing so the human (and agent) see *source*, not raw addresses:
   - `query_cpu` / `query` — registers and machine state.
   - `step` / `run_until_pc` / `run_frames` — execution control.
   - `memory_read` / `disasm` — memory and code views, addresses resolved to
     symbols via the assembler's symbol table.
   - `watch_memory` / breakpoints — change and stop conditions.

The load-bearing seam is the **symbol/listing bridge**: Asm198x emits where each
label and source line landed; Emu198x reports what the machine is doing at an
address; Forge maps between them. Both halves must be stable before this is real.

## Front-end stance

Prefer **extending an existing editor** (a VS Code extension, or a TUI over the
scriptable/MCP interfaces) over a bespoke cross-platform app — boring technology,
rescue-beats-replace at the tooling layer. A from-scratch IDE needs justification
recorded in the decision record, not assumed here. No stack is chosen yet.

## Agent-native parity (hard rule)

Every workbench capability must be reachable through Emu198x's MCP/scriptable
surface and Asm198x's CLI — Forge adds no human-only path that bypasses the tool
surface. If a feature would need one, the fix is to expose it on the surface, so
agents keep parity with the IDE. This is why the loop above is written as tool
calls: the IDE is a presentation of them.

## What it is NOT

Not an emulator or assembler (it consumes them); not a backend; not bespoke by
default; not driven by Code198x's curriculum (one input, not master); not a
source of human-only capability. See the decision record for the full list.

## What would un-defer it

The build waits on its dependencies being stable enough to integrate. Concrete
signals to watch:

- **Asm198x matured past the early-subset slice** — disassembly exists, the 6502
  dialect reaches real source-compatibility (expressions, directives, segments,
  macros), and a second CPU has proven the engine/dialect seam. (As of v0.0.2 it
  is an early-subset 6502 assembler only.)
- **Emu198x's debug surface settled** — the query/step/memory/disasm/watch tools
  above stable enough to build against without churn.
- **A stable symbol/listing format from Asm198x** for the bridge in §"The loop".

When those hold, revisit `forge198x-ide.md`, lift the deferral there, and start
the flagship workspace.
