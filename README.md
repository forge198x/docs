# Forge198x documentation

Documentation for [Forge198x](https://github.com/forge198x), the 198x family's
development workbench.

Forge198x is **deferred** — gated on Asm198x, Build198x, and Emu198x surfaces maturing — so for now this repo holds design notes rather than user docs.

## Scope (eventually)

- How the edit → assemble/build → run → debug loop works.
- How Forge198x drives Asm198x (assemble), Build198x (convert/package), and Emu198x (run / debug) over their existing interfaces, without reimplementing them.
- The agent-native contract: every workbench capability reachable through the
  tool surface, not human-only.

## Not here

- The decision to pursue Forge198x and its constraints — the umbrella decision record `198x/decisions/forge198x-ide.md`.
- Hardware facts — the umbrella `reference/` library.

## Design notes

[`design-notes.md`](design-notes.md) — the integrator loop (edit → assemble/build → run → debug) sketched against the intended Asm198x, Build198x, and Emu198x tool surfaces, the front-end stance, the agent-native contract, and the concrete signals that would un-defer the build. Captured now; the build still waits.
