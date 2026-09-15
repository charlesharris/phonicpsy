# Project Research Summary

**Project:** phonicpsy  
**Domain:** Algorithmic-composition Ruby generator layer over Sonic Pi  
**Researched:** 2026-09-15  
**Confidence:** MEDIUM-HIGH overall

## Executive Summary

Phonicpsy is a Ruby layer for algorithmic music generation that emits and drives Sonic Pi code. The recommended approach is a **structured intermediate representation (IR)** that separates musical decisions (generators) from code emission (Emitter), enabling both self-contained generative loops (Sonic Pi's own RNG) and streamed code push (Ruby's RNG) from a single generator implementation.

**Critical integration finding (resolves PROJECT.md's assumption):** `sonic-pi-tool` and all its rewrites are **broken against current Sonic Pi v5.0.0**. The assumed integration path (token-gated `/run-code` channel) requires a dynamically-allocated port and 32-bit auth token, with no stable discovery mechanism for external processes. The verified, working path is the unauthenticated **cue-server runner** (port 4560, documented, stable): paste a small `live_loop` once that listens for OSC cues and evals the payload.

**Two major risk areas:** (1) Dual-mode RNG management — Sonic Pi's randomness and Ruby's generator randomness must be independently seeded via the IR so "same seed, same output" holds per-mode; (2) Generated code quality — syntactically valid code that produces undifferentiated noise requires deliberate musicality constraints (scale quantization, rhythmic grid, register separation, density limits, weighted choice) baked into shared helpers.

## Key Findings

### Recommended Stack

**Core technologies:**
- **Ruby 4.0.x** — matches Sonic Pi's DSL syntax
- **`osc-ruby` 1.1.5** — actively maintained, simple synchronous API; reject `ruby-osc` (requires EventMachine)
- **Hand-rolled music-theory core** (~100-150 lines) — scales, chords, rhythms as pure data; Sonic Pi already provides `scale`, `chord`, `spread` for execution
- **`pry` + `listen`** — fast-iteration loop (Pry session holds OSC client; optional file watcher auto-reloads)

**Critical finding:** Do NOT use `sonic-pi-tool`, `sonic-pi-cli` gem, or existing Sonic Pi CLIs. They hardcode pre-v4 protocol (port 4557, no token) and are broken. Use `osc-ruby` + cue-server runner pattern (port 4560, untokenized, documented, stable).

**What Sonic Pi provides for free (do NOT reimplement):**
- `scale`, `chord`, `ring` — scale/chord lookup
- `spread(k, n)` — Euclidean rhythm distribution
- `live_loop`, `sync`, `cue` — loop and sync primitives
- `use_random_seed` — seed Sonic Pi's RNG

### Expected Features

**Must have (table stakes):**
- Seeded, deterministic generation (Ruby-side `Random.new(seed)`, Sonic Pi-side `use_random_seed` in code)
- Scale/chord vocabulary (thin wrapper over Sonic Pi)
- Rhythm generation including Euclidean distributions
- Syntactically valid Sonic Pi Ruby code emission
- File export + OSC live push
- Self-contained `live_loop` wrapping
- Parameters as knobs, re-rollable without code editing

**Should have (competitive/high-payoff):**
- Plugin registry (add generator N without touching core)
- A/B re-rolling (keep snapshots, iterate)
- Weighted choice / Markov sequencing (prefer over uniform)
- Layerable parts (bass/drums/lead with shared context)
- Musicality helpers: scale quantization, grid alignment, register separation, density/rest limits

**Defer to v2+ (or driven by use):**
- Streamed mode (architecture supports, v1 can be self-contained-only)
- Generative grammars (L-systems)
- Optional LLM prose→parameters front-end
- Numeric/sequence generators

**Do NOT build (anti-features):**
- Visual/grid UI (contradicts "no polished GUI")
- Audio DSP / real-time feedback
- Multi-user sharing / plugin marketplace
- Full song-structure engine

### Architecture Approach

Use a **structured Intermediate Representation (IR)** separating musical decisions from code emission. Core components:

1. **Music Theory Core** — pure data + functions (scales, chords, rhythms). Zero knowledge of Sonic Pi.
2. **Generators (plugins)** — build `IR::Pattern` using params + seeded RNG. Emit `Expr` nodes (`Choice`, `RandRange`, `Weighted`) for decisions, not `rand` calls.
3. **IR** (Pattern/Event/Param/Expr) — lightweight value objects. `Expr` knows how to resolve (RNG) and render symbolically (Sonic Pi `choose`/`rrand`).
4. **Resolver** (streamed only) — walks IR with seeded `Random`, replaces `Expr` nodes with concrete values.
5. **Sonic Pi Emitter** — renders Pattern to Ruby string. Two modes: `:symbolic` (unresolved `Expr` become `choose`/`rrand`) for self-contained; `:literal` (concrete values) for streamed.
6. **Transport** — File exporter and OSC client (cue-server runner pattern, port 4560).
7. **Session/Runner** — orchestrates generator + seed + params + mode + transport.

**Dual-mode design:** Both modes share one generator, one IR. Only difference: Resolver runs before Emitter in streamed mode. Determinism is per-mode guarantee (replay self-contained, replay streamed), not cross-mode.

**Determinism:** One master seed scoped strictly per engine. Ruby's `Random.new(master_seed)` for generator decisions. Sonic Pi's `use_random_seed master_seed` (first line inside `live_loop`) for Sonic Pi randomness.

### Critical Pitfalls

1. **OSC handshake fragility** — `/run-code` requires token discovery with no stable API. **Prevention:** Use cue-server runner pattern (port 4560). One-time buffer paste: `live_loop :phonicpsy_runner { code = sync "/osc*/phonicpsy-run"; eval(code) }`. Spike this early.

2. **Errors in pushed code vanish silently** — Sonic Pi has no OSC error-reply protocol. **Prevention:** Pre-flight validation using `Ripper.sexp` and character-count guard (~8KB UDP ceiling).

3. **Generated code sounds like noise** — Missing constraints: scale quantization, grid alignment, register separation, density/rest, weighted choice. **Prevention:** Bake scale/grid/weighted-choice into shared toolkit. Make "sounds good" explicit UAT gate.

4. **"Same seed, same output" breaks** — Two RNG streams, `Kernel.rand` global state, hash iteration order, float accumulation. **Prevention:** Thread explicit `Random.new(seed)` through generators. Always emit `use_random_seed`. Never build structures from unordered sources. Automate seed-diff checks.

5. **Runaway loops you can't stop** — Computed `sleep` values can go to zero/negative. **Prevention:** Centralize sleep emission through helper that clamps and forces float division. Static-check every `live_loop` body has syntactic `sleep`. Build panic command early.

6. **Building framework before a generator sounds good** — Investing in plugin architecture before one generator is listenable. **Prevention:** Ship one concrete, hand-built, good-sounding generator first. Only then extract plugin contract. Let generator #2 be a copy-paste divergence.

## Implications for Roadmap

Suggested phases reflect one principle: **ship audible sound first, structure it second, extend it third, polish iteration fourth.**

### Phase 1: Sound, Now
**Rationale:** Validate core loop (edit→hear) before architecture. Ship one hand-crafted generator producing listenable output via file export and OSC live push, self-contained mode only.

**Delivers:**
- Music Theory Core (scales, chords, rhythm helpers)
- One hand-built generator with musicality-first design (apply Pitfall 6 constraint list)
- File export + OSC live push via cue-server runner (port 4560, untokenized)
- Pre-flight syntax validation + size guard
- Explicit musicality test (generator is good, not just "runs")

**Addresses:** Valid code emission, deterministic generation, dual transports, `live_loop` wrapping, edit→hear loop.

**Avoids:** Pitfalls 1 (uses verified cue-server), 2/3 (pre-flight validation), 6 (musicality constraints), 9 (no framework yet).

**Research spike needed:** Verify cue-server runner integration against installed Sonic Pi version (log format, headless behavior).

### Phase 2: Structure & Plugins
**Rationale:** Generalize Phase 1's proof so adding generators #2-N is cheap. Extract plugin contract from real generators.

**Delivers:**
- Full `IR::Expr` hierarchy with resolve/render methods
- `Generator` base class with `inherited` registry + `param` DSL
- Refactor Phase 1 generator to emit `Expr` nodes instead of hardcoded strings
- 2-3 new generators proving the plugin contract
- Parameter randomize/re-roll operations
- Minimal REPL for generator control
- A/B re-rolling UI (keep N snapshots)
- Stable loop-name contract (re-pushes replace, not stack)

**Addresses:** Pluggability, named parameters/knobs, A/B re-rolling.

**Avoids:** Pitfall 4 (stable loop names), Pitfall 9 (contract from working generators, not up-front), Pitfall 10 (no depth mandate, only Pitfall 6 constraint list).

### Phase 3: Streamed Mode
**Rationale:** Prove dual-mode architecture by running Phase 2 generator unmodified in streamed mode.

**Delivers:**
- Resolver (IR walk with seeded RNG, replaces `Expr` with concrete)
- Emitter literal-mode path (concrete values)
- OSC push scheduler (bar/phrase chunks, timing to avoid gaps)
- Session mode switch (self-contained vs. streamed)
- Optional LLM prose→parameters front-end (parameter output only, never code)

**Addresses:** Continuous code push while playback runs, optional LLM front-end.

**Research spike needed:** Chunk-size and push-timing heuristics (empirical validation that playback doesn't gap). LLM schema-constrained output validation.

### Phase 4: Ergonomics & Extension
**Rationale:** Now architecture is proven, invest in fast feedback loop.

**Delivers:**
- Hot-reload (watch generator source, re-push on save)
- Interactive knob-twisting REPL
- Expanded generator library (flavor generators)
- Layerable part composition (shared master-seed context)
- Endless generative patterns (incommensurate loops, drift)
- Numeric/sequence generators
- Stop-all/panic control
- Optional favorites capture

**Addresses:** Fast iteration, layerable parts, endless pieces, parameter hot-reload.

**Research spike needed:** File-watcher latency validation (`listen` + `rb-fsevent` sub-second?). Panic command mechanics (re-push `stop` in each loop).

### Phase Ordering Rationale

1. **Phase 1 before Phase 2:** Sound first. Phase 1 proves core loop works for real generator. Phase 2 safely extracts structure.
2. **Phase 2 before Phase 3:** Plugin interface proven on 2-3 generators before committing dual-mode Resolver/Emitter separation.
3. **Phase 3 before Phase 4:** Streamed mode optional depth built after self-contained solid. Ergonomics depend on stable foundation.
4. **Musicality gate at Phase 1 end, enforced in Phase 2:** Prevents framework-for-building-bad-generators trap.

### Research Flags

**Phases needing research-phase during planning:**
- **Phase 1:** Sonic Pi cue-server integration (log format, token/port discovery, headless vs. GUI behavior). Spike against installed version early.
- **Phase 3:** OSC scheduler and chunk timing (empirical validation of gap-free streaming). LLM schema-constrained output validation.
- **Phase 4:** File-watcher latency (`listen` performance). Panic command semantics.

**Phases with standard patterns (skip research-phase):**
- **Phase 1:** File export, Ripper validation (stdlib Ruby)
- **Phase 2:** Plugin registry via `inherited` hook (idiomatic Ruby)
- **Phase 3:** LLM via official `anthropic` gem (vendor-maintained)
- **Phase 4:** Hot-reload, file watching (`listen` gem, mature)

## Confidence Assessment

| Area | Confidence | Notes |
|------|------------|-------|
| **Stack** | HIGH | Verified against sonic-pi-net/sonic-pi source. Rubygems.org API queried live. `osc-ruby` actively maintained (2025-11-16). `sonic-pi-tool` broken status corroborated across repos/issues. |
| **Features** | MEDIUM-HIGH | Landscape surveyed (TidalCycles, Strudel, FoxDot, Overtone, etc.). Dependencies inferred. "Sounds like noise" constraints cross-checked but not single source. |
| **Architecture** | MEDIUM-HIGH | Structured IR + dual-mode is synthesis of standard patterns, novel integration. Sonic Pi specifics HIGH (verified directly). Dual-RNG coordination MEDIUM (requires validation). |
| **Pitfalls** | MEDIUM-HIGH | Sonic Pi integration gotchas HIGH (verified). Musicality/determinism MEDIUM (practitioner consensus, not single source). Framework/theory pitfalls MEDIUM (pattern-based, not domain-specific). |

**Overall confidence:** MEDIUM-HIGH

### Gaps to Address

1. **Verified cue-server spike needed:** Recommended by Sonic Pi author but PROJECT.md assumed `sonic-pi-tool`. Early Phase 1 work should spike actual integration against installed version.

2. **LLM schema validation:** Official `anthropic` gem supports schema-constrained output, but project hasn't used it. Phase 3 should spike prose→parameter test before full integration.

3. **Musicality metrics:** Pitfall 6 constraint list are domain heuristics, not specs. Phase 2 should document checklist; "sounds good" remains subjective UAT.

4. **Determinism diff-check:** Pitfall 7 recommends automating seed-diff checks. Phase 1 should include simple version (binary-identical); Phase 2+ refines if precision issues emerge.

5. **Chunk-size heuristics:** No specific guidance for Phase 3's scheduler. Needs empirical validation with real generators.

## Sources

### Primary (HIGH confidence)

- **sonic-pi-net/sonic-pi** GitHub repo — direct clone and source read:
  - Tag `v5.0.0` (released 2026-08-07, current stable); `dev` HEAD (commit `cba6a37`, 2026-09-12)
  - Files: `app/api/src/sonicpi_api.cpp`, `app/server/ruby/bin/daemon.rb`, `app/server/ruby/lib/sonicpi/osc_api.rb`, tutorial files on OSC and randomization
  - **Topics:** OSC protocol (both surfaces), token auth requirement, cue-server documentation, determinism/randomness patterns
- **rubygems.org API** — live queries for `osc-ruby`, `ruby-osc`, `music_theory`, `listen`, `pry`, `anthropic`
  - **Topics:** gem maintenance status, version history, download counts
- **GitHub API** — repo activity for `aberant/osc-ruby`, `lpil/sonic-pi-tool`, `emlyn/sonic-pi-tool`, `Widdershin/sonic-pi-cli`
  - **Topics:** last-push dates, maintenance status
- **abhishekjairath/sonic-pi-mcp** — working reference implementation of cue-server runner pattern (pushed 2026-03-22)

### Secondary (MEDIUM confidence)

- GitHub issue `sonic-pi-net/sonic-pi#3330` — community discussion and maintainer commentary on token requirement
- Kevin Murphy — "Revisiting Calling Sonic Pi From Ruby" blog post — practitioner account of protocol breakage
- TidalCycles, Strudel, FoxDot, Overtone, Orca, Gibber ecosystem documentation — feature and architecture patterns
- Sonic Pi GitHub issues #767, #794, #146 — community-reported integration gotchas
- Ruby `Random` class documentation — RNG independence semantics

### Tertiary (LOWER confidence, needs validation)

- `spx` gem (released Nov 2025) — newer Ruby CLI for Sonic Pi; worth studying for token handling but not directly examined
- Generative-music craft patterns — synthesized from cross-checked practitioner consensus, not single source

---

*Research completed: 2026-09-15*  
*Synthesized by: gsd-research-synthesizer*  
*Ready for roadmap phase planning: yes*
