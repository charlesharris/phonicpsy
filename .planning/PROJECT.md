# phonicpsy

## What This Is

A Ruby layer that composes Sonic Pi programs. Pluggable generators built from music
theory — scales, rhythmic grammars, weighted choice — emit valid Sonic Pi code that
either runs as a self-contained `live_loop` or gets streamed into a live Sonic Pi
session over OSC. It's a personal playground for the author, not a product: the
audience is one person who wants to tinker with generative music and hear the result
immediately.

## Core Value

Twist a knob or edit a generator and hear the result in seconds — the tool disappears
and you're just playing.

## Requirements

### Validated

(None yet — ship to validate)

### Active

- [ ] Generators emit syntactically valid, runnable Sonic Pi code
- [ ] Generation is deterministic and seedable — same seed, same output
- [ ] Generated code can be pushed into a running Sonic Pi instance and heard immediately
- [ ] Generated code can be exported as `.rb` files to keep and open in the Sonic Pi editor
- [ ] A generator can emit self-contained generative code (`live_loop` with its own randomness) that Sonic Pi runs forever unattended
- [ ] A generator can stream fresh code into Sonic Pi over OSC while playback continues
- [ ] Generators are pluggable — adding a new one does not require changing the core
- [ ] Parameters are exposed as knobs that can be twisted and re-rolled without editing code
- [ ] Editing a generator's code and hearing the result is a fast loop (hot-reload, minimal ceremony)
- [ ] Generators can produce loops, layerable single parts (bass / drums / lead), and endless generative pieces
- [ ] Optional LLM front-end translates prose descriptions into a parameter set

### Out of Scope

- **LLM writing Sonic Pi Ruby directly** — musical intelligence lives in deterministic code; an LLM only ever produces a parameter set, never emitted code. Keeps output valid and reproducible.
- **Structured song forms (intro/verse/chorus/bridge)** — the unit of generation is loops, parts, and endless pieces. Arrangement over minutes is a different problem.
- **Multi-user anything** — auth, accounts, sharing, deployment. This is a single-person playground running locally.
- **Audio rendering / DSP** — Sonic Pi (and SuperCollider beneath it) makes the sound. This layer only ever produces code.
- **Replacing or forking Sonic Pi** — it is the runtime, used as-is through its public interfaces.
- **A polished GUI** — nothing that slows the iteration loop down. Interface stays as thin as it can be.

## Context

- **Sonic Pi** (https://sonic-pi.net/) is a live-coding environment with a Ruby-flavored DSL. Code lives in buffers; running a buffer plays it; `live_loop` provides forever-running, hot-swappable loops. It exposes an OSC interface, and `sonic-pi-tool` is the established CLI for driving a running instance — both are the integration seams this project builds on.
- The generator layer is Ruby to keep generated code and generating code in the same language, minimizing the mental context-switch while iterating. The tradeoff — Ruby's algorithmic-music library ecosystem is thinner than Python's (music21, mingus, mido) — was accepted knowingly.
- Generator ideas floated during questioning, as flavor rather than commitment: 16-bit Final Fantasy–inspired loops, noise walls, "weirdest version of a described vibe." The module name being the vibe is a legitimate design.
- Novelty and surprise are welcome outcomes but are not the success bar. Iteration speed is.
- Existing repo is empty apart from a README — greenfield.

## Constraints

- **Tech stack**: Generators written in Ruby — matches Sonic Pi's DSL, so emitted and emitting code read alike.
- **Dependencies**: Requires a locally installed, running Sonic Pi for the live path. File export must work without it.
- **Compatibility**: Emitted code must be valid Sonic Pi, driven only through public interfaces (OSC, `sonic-pi-tool`) — no forking or patching Sonic Pi itself.
- **Performance**: Knob-twist to sound is the inner loop and must feel instant; generator-edit to sound is the outer loop and must stay fast.
- **Scope**: Single-user, local-only. No deployment target.

## Key Decisions

| Decision | Rationale | Outcome |
|----------|-----------|---------|
| Algorithmic/music-theory core rather than LLM-generated code | Deterministic, seedable, reproducible; the musicality stays under the author's control | — Pending |
| LLM confined to an optional prose → parameters front-end | Keeps the weird-input path available without putting a nondeterministic component in the emission path | — Pending |
| Ruby for the generator layer | Same language as the Sonic Pi DSL; low context-switch cost while iterating, accepting a thinner music-library ecosystem | — Pending |
| Support both self-contained generative code and OSC streaming | Self-contained code is the default and needs no supervisor; streaming covers pieces that must evolve beyond one buffer | — Pending |
| Both live-push and `.rb` file export | Live push is for iterating, file export is for keeping what works | — Pending |
| Pluggable generator modules over one monolithic generator | The playground framing means the value is in trying many generator ideas cheaply | — Pending |

## Evolution

This document evolves at phase transitions and milestone boundaries.

**After each phase transition** (via `/gsd-transition`):
1. Requirements invalidated? → Move to Out of Scope with reason
2. Requirements validated? → Move to Validated with phase reference
3. New requirements emerged? → Add to Active
4. Decisions to log? → Add to Key Decisions
5. "What This Is" still accurate? → Update if drifted

**After each milestone** (via `/gsd-complete-milestone`):
1. Full review of all sections
2. Core Value check — still the right priority?
3. Audit Out of Scope — reasons still valid?
4. Update Context with current state

---
*Last updated: 2026-09-15 after initialization*
