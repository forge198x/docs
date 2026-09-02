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

## The assembler is the language service

*Captured 2026-09-02, after the Asm198x browser playground shipped.*

The playground (asm198x.github.io/playground/) is `crates/asm198x-web`, a
`wasm-bindgen` shell over the assembler library with three calls — `assemble`,
`listing`, `dialects` — each returning the CLI's own contract-shaped JSON. It
came together in a day because nothing in it was new: the engine has no
filesystem dependency, the site already checked the assembler out at its
release tag, and Play198x had already worked out the excluded-crate pattern.
Two things it showed are worth holding onto for Forge198x.

**Highlighting is where a separate editor layer starts to go wrong.** Asm198x
covers twenty-four dialects. Highlighting them with a grammar per dialect — a
TextMate file, a Tree-sitter grammar, a regex table — means twenty-four
grammars that drift from the assembler the moment a dialect learns a new word,
and the assembler learns new words in most releases. The alternative is that
the assembler answers the question: a `tokens(dialect, source)` call returning
spans classified by the parser that decides what they mean — label, mnemonic,
directive, macro name, register, comment — with errors underlined by the same
diagnostics the CLI prints. Highlighting, completion, go-to-definition and
diagnostics are then one thing, the assembler answering questions about a
buffer, and an LSP server is one transport for it with wasm another. This is
the "Forge owns only the editing UX; the language semantics come from Asm198x"
line in §"The loop", taken to its end: the editor should not know what a Z80
label looks like.

**What that asks of Asm198x.** The semantic AST (`crates/asm198x/src/ast.rs`)
is source-preserving — it is what the formatter round-trips through — so it
already carries the spans a token service would classify. Coverage is the gap,
not design: as of v0.0.57 the Z80 dialects and a migrating subset lower
through the AST, and the rest still lower directly behind the dialect
boundary. Asm198x's own roadmap (`decisions/roadmap-sequencing.md`) already
lists this consumer under the name *language surface*, gated on the AST and
staged CPU tier by CPU tier — "finish the AST for a CPU tier before the
AST-consumers scale across it". So a `tokens` call is an Asm198x change that
arrives per tier, not a Forge198x one, and nothing on the Forge side should
assume it exists yet.

**Three fronts on one back.** Once the join exists, the same shell serves:

- the Asm198x playground, as a demonstration and as the first place a break in
  the join shows up (the site builds the module from the release tag, so it is
  the integration test);
- Code198x in-lesson editing, the unit's sample loaded, edited, assembled and
  run beside the text — the front with an audience;
- the Forge198x workbench itself, whatever editor it extends.

The edit → assemble → run half of that loop in a browser is one piece short:
an Emu198x wasm build. The join is `assemble()`'s `bytes` and `origin` going
into whichever core loads, which is why the shell returns the CLI's contract
rather than anything page-specific. Delivery today is each site building the
module from the tag; an npm package becomes the answer when a consumer is on a
different build pipeline, and that is a hosting decision to take deliberately.

**Consequence for the front-end stance above.** "Prefer extending an existing
editor" still holds, and this sharpens why: an editor extension that is a thin
LSP client over an assembler-provided language service is the boring shape.
The thing to avoid is the extension growing its own understanding of any
dialect. Drift trigger: *"add a syntax file for dialect X"* — stop, and ask
the assembler for tokens instead.

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

- **Asm198x matured past the early-subset slice** — *met as of v0.0.57
  (2026-09-02)*: twenty-four dialects across the 6502, Z80, 68xx, 68000 and
  16-bit families, differential against the reference assemblers, and a
  browser build. Still open on the Asm198x side for Forge specifically: the
  token service in §"The assembler is the language service".
- **Emu198x's debug surface settled** — the query/step/memory/disasm/watch tools
  above stable enough to build against without churn. Not yet assessed here.
- **A stable symbol/listing format from Asm198x** for the bridge in §"The loop"
  — *met*: Debug198x, the NDJSON sidecar Asm198x writes and Emu198x reads,
  frozen at v1 on 2026-08-18 and graduated to its own project.

When those hold, revisit `forge198x-ide.md`, lift the deferral there, and start
the flagship workspace.
