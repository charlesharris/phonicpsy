# Walking Skeleton — phonicpsy

**Phase:** 1
**Generated:** 2026-09-15

## Capability Proven End-to-End

> One sentence: the smallest user-visible capability that exercises the full stack.

`bin/phonicpsy export --seed 42` writes `out/arpeggiator-42.rb`, a self-contained Sonic Pi
program that opens in the Sonic Pi editor and plays a scale-locked, grid-aligned arpeggio —
produced with no Sonic Pi process running, by a path that touches every layer the project will
ever have (CLI → Session → Theory → Generator → IR → Emitter → Validator → Transport → disk).

The live-OSC half of the stack (`play` / `stop` through the Sonic Pi cue server) is the second
slice, layered onto this skeleton in plan `01-03` without changing any decision below.

## Architectural Decisions

| Decision | Choice | Rationale |
|---|---|---|
| Language / runtime | Ruby 3.4.4 (floor 3.2) | PROJECT.md constraint — generators read like the Sonic Pi DSL they emit. Verified installed (`ruby -v`, 01-RESEARCH.md Environment Availability). |
| Musical representation | Structured IR (`IR::Pattern` of `IR::Event`, with `IR::Expr` nodes for symbolic randomness) — generators NEVER build Sonic Pi strings | ARCHITECTURE.md "The Central Decision"; IR-01/IR-02/IR-03. String building is the Emitter's exclusive job. |
| Randomness model | Two independent streams, never correlated: a Ruby `Random.new(seed)` resolves *structure* at emission time; Sonic Pi's own RNG (reset by `use_random_seed <seed>` as the first statement inside the `live_loop`) resolves *symbolic* `Expr` nodes forever at playback | ARCHITECTURE.md Determinism Architecture; GEN-02/GEN-03; PITFALLS.md Pitfall 7. |
| Pitch source | Sonic Pi's own `chord_degree(degree, root, scale_name, 3)` — scale-locked by construction | GEN-04/GEN-06. Verified at `western_theory.rb:936`; `Chord.resolve_degree` stacks every-other-note *within the named scale*, so an out-of-scale pitch is structurally impossible. No interval arithmetic in this project's Ruby. |
| Rhythm source | Sonic Pi's own `spread(onsets, steps)` (Toussaint Euclidean), emitted — empty slots ARE the rests | D-13; GEN-05/GEN-07/GEN-08 satisfied by one mechanism. Verified at `core.rb:1758`. |
| Transport seam | Unauthenticated OSC cue server, UDP `localhost:4560`, plus a repo-root `runner.rb` the user runs by hand in a Sonic Pi buffer | 01-RESEARCH.md, verified against the installed Sonic Pi v5.0.0. The token-gated `/run-code` channel is out of scope (REQUIREMENTS.md Out of Scope). |
| Loop identity | `live_loop :phonicpsy_<generator_name>` — derived from the generator name alone, stable across every re-push | D-21; EMIT-04. No seed, no instance id, no `--as` override in Phase 1. |
| Panic mechanism | A Time State flag (`:phonicpsy_stop`) checked by the emitted loop on every grid step; `phonicpsy stop` sets it via its own OSC cue | LOOP-01; D-08. Verified `stop` semantics (`core.rb:1565`) only halt the calling thread, so an outside-in kill message does not exist — the check must be *inside* emitted code. |
| Parameter model | Plain Ruby keyword arguments with defaults (`scale:`, `root:`, `progression:`, `bpm:`, `grid:`, `onsets:`, `synth:`, `release:`) — no declaration DSL, no runtime introspection | D-03. The Phase 2 param DSL (LOOP-02/03/04) must remain real work; a keyword arg becomes a declared param later with no rewrite. |
| CLI | `bin/phonicpsy` with `export` / `play` / `stop` subcommands, stdlib `optparse`, thin wrapper over `Phonicpsy::Session` | D-05/D-06. Keeps the core dependency list at `osc-ruby` alone and startup fast (PROJECT.md inner-loop constraint). |
| Directory layout | `lib/phonicpsy/{theory,ir,generators,emit,transport}/` + `session.rb`; `bin/phonicpsy`; `runner.rb` at repo root; `out/` gitignored | ARCHITECTURE.md Recommended Project Structure, Phase-1 subset. `theory/` knows nothing of Sonic Pi or the IR; `transport/` knows nothing of `ir/` or `generators/`. |
| Test runner | Minitest (stdlib), pinned `~> 5.25` in `test/test_helper.rb`, driven by `rake test` | STACK.md Testing Generated Code — golden-file snapshot tests and seeded-determinism tests are string comparisons; no RSpec, no snapshot gem. |
| Syntax gate | stdlib `Ripper.sexp` (returns `nil` on invalid source) before any write or push | EMIT-03. In-process, sub-millisecond; verified this research session. |
| "Deployment" | None — single-user, local-only (PROJECT.md Scope). The equivalent full-stack run command is `bin/phonicpsy export` (no Sonic Pi needed) and `bin/phonicpsy play` (Sonic Pi + `runner.rb` running). | |

## Stack Touched in Phase 1

- [ ] Project scaffold — `Gemfile`, `Rakefile`, `.gitignore`, `lib/phonicpsy.rb`, `test/test_helper.rb`
- [ ] Entry point — `bin/phonicpsy` with a real `export` subcommand parsed by `optparse`
- [ ] Domain core — `theory/` + `ir/` + `generators/arpeggiator.rb` producing a real `IR::Pattern`
- [ ] Rendering — `emit/sonic_pi_emitter.rb` turning that `IR::Pattern` into valid Sonic Pi source
- [ ] Persistence (one real write) — `transport/file_exporter.rb` atomically writing `out/<generator>-<seed>.rb`
- [ ] External process (one real round trip) — `transport/osc_client.rb` ↔ `runner.rb` over UDP 4560, with ack
- [ ] Full-stack run command — `bin/phonicpsy export --seed 42` (offline) and `bin/phonicpsy play --seed 42` (live)

## Out of Scope (Deferred to Later Slices)

> Anything that is *not* in the skeleton. Be explicit — this list prevents future phases from
> re-litigating Phase 1's minimalism.

- Parameter declaration DSL, runtime parameter inspection, setting values without editing source (Phase 2 — LOOP-02/03/04)
- Hot reload, `listen`/`pry` session, favorites (Phase 2 — LOOP-06/07, LIB-07)
- A second or third generator; finite-loop vs endless-piece forms (Phase 3 — LIB-01/02/03)
- Layered parts, shared-tempo sync, register separation, an `--as` loop-name override (Phase 4 — LIB-04/05/06)
- `resolve/resolver.rb` and streamed emission — IR-03 is satisfied by the IR's *shape* in this phase (symbolic `Expr` nodes survive to the Emitter untouched), verified by review, not by a running resolver (v2 — STRM-01..04)
- Plugin registry / `inherited` self-registration — Phase 1 has exactly one generator, wired by a direct `require` (v2 — PLUG-01..03)
- The prose/LLM front-end (v2 — PROSE-01..03)
- `IR::Expr` node types beyond `Choice` and `Weighted` (no `RandRange`, no `TickRing`) — D-04 scopes Phase 1's symbolic surface to exactly what the arpeggiator uses
- Measuring the true OSC truncation ceiling — a conservative fixed 8192-byte cap stands in (D-20)

## Subsequent Slice Plan

Each later phase adds one vertical slice on top of this skeleton without altering the
architectural decisions above:

- **Phase 2 — Knobs and the Fast Loop:** promote the keyword args to declared `IR::Param`
  descriptors; add runtime listing/setting and a seed re-roll; keep one long-lived process alive
  so edit-to-sound is seconds. Reuses `Session` and the IR unchanged.
- **Phase 3 — A Library Worth Listening To:** generators two and three by divergence from
  `arpeggiator.rb`; finite-loop and endless-piece forms as `IR::Pattern` loop metadata the Emitter
  already reads.
- **Phase 4 — Layered Parts:** multiple concurrent `IR::Pattern`s, each with its own stable loop
  name; the loop-identity noun is promoted from *generator* to *part* at that point (see the
  assumption-delta decision recorded in `01-03-PLAN.md`).
