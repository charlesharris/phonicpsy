# Phase 1: Hear One Generator - Context

**Gathered:** 2026-09-15
**Status:** Ready for planning

<domain>
## Phase Boundary

One hand-built generator taken all the way to audible sound: theory primitives, a minimal
IR, the emitter, the shipped Sonic Pi runner buffer, live OSC push, `.rb` file export, and
the panic command. 20 requirements (GEN-01..08, IR-01..03, EMIT-01..08, LOOP-01).

The phase delivers **one** generator that sounds good, plus the whole path from Ruby to
speakers. It does not deliver a parameter system (Phase 2), more generators (Phase 3), or
layered parts (Phase 4).

</domain>

<decisions>
## Implementation Decisions

### The First Generator

- **D-01:** Generator #1 is an **arpeggiator over a chord** — walks a chord's notes in a
  seeded order across a rhythmic grid. Chosen because it is the fastest path to listenable
  sound while exercising scale + chord + grid + weighted choice together, and because
  Phase 3 builds generators 2 and 3 by divergence from it. `ARCHITECTURE.md` already uses
  `arpeggiator.rb` as its worked example.
- **D-02:** The arpeggiator moves through a **short chord progression**, not a single held
  chord, because a static single chord fights Success Criterion 4's listenable bar and
  leaves Phase 3's endless-piece form nothing to evolve.
- **D-03:** The progression is a **plain Ruby keyword argument with a default** — not a
  param-declaration DSL and not runtime-introspectable. This is a deliberate boundary:
  Phase 2's LOOP-02/03/04 (declared params with names and ranges, runtime inspection,
  setting values without editing source) must remain real work. Nothing built here is
  thrown away when the param DSL lands — a keyword arg becomes a declared param later.
  — **Reversibility:** reversible — the keyword arg is the eventual param's call signature.
- **D-04:** Seeded randomness lives in **note order and rests only**. Pitch per step is a
  weighted `Choice` over the current chord's notes; a `Weighted` node produces the rests
  GEN-08 requires. **Rhythm does not vary per seed** — onsets stay on a fixed Euclidean
  grid so the loop grooves consistently and GEN-07's grid alignment stays demonstrable.
  This exercises `Choice`, `Weighted` and `spread`, which is the IR-02 surface Phase 1 needs.

### Command Surface

- **D-05:** Primary entry point is a **CLI binary, `bin/phonicpsy`, with subcommands**
  (`play`, `export`, `stop`). It is a thin wrapper over a `Session` object so Phase 2's
  pry-based hot-reload loop reuses the same objects rather than reimplementing them.
  Satisfies Success Criterion 1's "run one command" literally.
- **D-06:** Argument parsing uses **`optparse` from stdlib** — no CLI gem. Keeps the core
  path's dependency list at `osc-ruby` alone and keeps startup fast, which matters because
  PROJECT.md makes the inner loop a performance constraint.
- **D-07:** With no `--seed`, the CLI **rolls a random seed and always prints it**, so
  re-running explores and any run you liked can be recovered by passing its seed back.
  This makes GEN-01's determinism useful day-to-day rather than only test-visible, and it
  is what Phase 2's LOOP-05 seed re-roll builds on.
- **D-08:** `phonicpsy stop` (LOOP-01) sends a **distinct OSC cue that the shipped runner
  buffer handles** by stopping the running loops. Same transport as push, nothing extra to
  install, reachable from any shell the moment things get loud (Pitfall 5).

### Sound Defaults

- **D-09:** Scale and root are **`--scale` / `--root` flags** (plain keyword args per D-03,
  not a param DSL). *User explicitly asked for these to be changeable rather than fixed.*
  Nearly free to implement: per GEN-04 the emitted code calls Sonic Pi's own `scale`, so
  the theory core passes the name through and only needs the degree formula for *choosing*
  pitches during generation.
- **D-10:** Default when neither flag is passed: **A minor pentatonic**. Chosen as the
  default specifically because five notes with no semitone clashes make it very hard for
  the first sound the tool ever produces to be the unlistenable one (Pitfall 6).
- **D-11:** `--scale` accepts **any scale name Sonic Pi itself knows** (~40), validated
  against a list in the theory core. An unknown name **fails loudly at the CLI**, not at
  Sonic Pi runtime. Costs no more than hardcoding three, since the formula table is data.
- **D-12:** Default tempo and grid: **110 BPM on a 16th-note grid**. Mid-tempo and
  unhurried; a 16th grid gives the Euclidean `spread` enough slots to make an interesting
  pattern without becoming a blur, and leaves room for rests.
- **D-13:** **The Euclidean `spread` onsets ARE the density cap** (GEN-08). Empty slots in
  the pattern are the rests, so they land musically rather than randomly. One mechanism
  satisfies GEN-05, GEN-07 and GEN-08 together. No separate rest probability, no
  post-generation notes-per-bar filter.
- **D-14:** Default synth is **`:prophet`** — warm and forgiving of short releases, very
  hard to make harsh, so the untuned default run clears Criterion 4.

### Transports and Failure Feedback

- **D-15:** The runner buffer **wraps its `eval` in `begin`/`rescue` and acks back over
  OSC** — success or the error message — to a UDP port the CLI listens on briefly. This is
  the direct mitigation for Pitfall 2 ("errors in pushed code vanish into the void"), which
  is the most likely day-one failure. Requires a receive socket in the CLI.
  — **Reversibility:** costly — the ack protocol is a contract between `runner.rb` and the
  OSC client; changing its shape later means changing both sides plus any Phase 2 code
  that waits on it.
- **D-16:** Push **pings first and fails with instructions**: before sending code, send a
  cheap ping cue and wait a short timeout. No ack means print `Sonic Pi runner not
  responding — open runner.rb in Sonic Pi and hit Run` **including the file's absolute
  path**. This is what makes Criterion 1's "no manual step beyond having the runner buffer
  running" honest — the one manual step names itself the moment it's missing.
- **D-17:** The runner buffer ships as **`runner.rb` in the repo root, run manually** in
  Sonic Pi (EMIT-06). Versioned in git, depends on nothing inside Sonic Pi's install
  layout. Explicitly **not** installed into Sonic Pi's `init.rb` — that would write into
  Sonic Pi's own config, against the project's "public interfaces only" constraint.
- **D-18:** Exported `.rb` files carry a header comment with **generator, seed, and
  parameter values only — no timestamp, no absolute path**. Anything varying between runs
  would break the byte-identical diff that Success Criterion 3 is defined by. The header
  must contain everything needed to regenerate the file and nothing else.
- **D-19:** Exports default to **`out/<generator>-<seed>.rb`**, overridable with `--out`.
  The seed in the filename makes a directory of exports self-describing, different seeds
  never collide, and re-exporting the same seed overwrites with identical bytes — itself a
  spot-check of Criterion 3. `out/` is gitignored.
- **D-20:** EMIT-07's payload ceiling is a **conservative fixed cap around 8KB**, below the
  ~8–9KB silent-truncation point PITFALLS.md identifies, failing loudly with the actual
  byte count. A Phase 1 arpeggiator is far under it; this is a guardrail that costs nothing
  now and catches the failure the day a generator grows.
- **D-21:** `live_loop` names derive from the **generator name only** —
  `live_loop :phonicpsy_arpeggiator` — stable across every re-push regardless of seed or
  params (EMIT-04). No seed in the name, no `--as` override, no layering affordance in
  Phase 1. Deliberate layering is Phase 4's scope.
  — **Reversibility:** reversible — adding an override flag later is additive.

  **Note for planner/verifier:** the user initially chose generator-name-plus-seed here.
  That was surfaced as a direct conflict with EMIT-04 and Criterion 5 (every re-roll would
  leave the previous loop playing, making `stop` the only way back to silence), and the
  user resolved it to the strict stable-name form above. **No requirement changes were
  made.** EMIT-04 and Criterion 5 stand as written.

### Claude's Discretion

The user made an explicit choice on every question asked — no "you decide" answers. The
following remain unspecified and are the planner's/researcher's call:

- Ack timeout duration and the exact OSC paths for push / ping / stop / ack
- Note release and sustain values, octave range, and loop length in bars
- Whether any FX (e.g. reverb) appear in the emitted code
- The specific default chord progression and its length
- Euclidean onset count (e.g. 7-in-16) for the default pattern
- How the walk order over chord notes is structured
- Exactly how the seed-diff determinism check is run in plan 01-03, and what the
  listening pass consists of

</decisions>

<canonical_refs>
## Canonical References

**Downstream agents MUST read these before planning or implementing.**

### Project scope and requirements
- `.planning/PROJECT.md` — Core value, constraints, out-of-scope list, and the **corrected**
  Sonic Pi integration seam (see staleness warning below)
- `.planning/REQUIREMENTS.md` — The 20 v1 requirements this phase must satisfy
  (GEN-01..08, IR-01..03, EMIT-01..08, LOOP-01), plus the Out of Scope table
- `.planning/ROADMAP.md` §"Phase 1: Hear One Generator" — Goal, 5 success criteria, the
  3-plan breakdown, and the research spike definition

### Architecture
- `.planning/research/ARCHITECTURE.md` — **Read with the warning below.** Authoritative on:
  the structured-IR-over-direct-string-emission decision (§"The Central Decision"), the
  component responsibility table, the `lib/phonicpsy/` project structure, the
  generator-never-touches-a-Sonic-Pi-keyword rule, and the five anti-patterns
- `.planning/research/ARCHITECTURE.md` §"Anti-Patterns" — Especially #1 (generators building
  Sonic Pi strings) and #2 (resolving randomness inside the generator)

  > ⚠️ **STALENESS WARNING — do not follow the OSC transport parts of this file.**
  > `ARCHITECTURE.md` predates the integration-seam correction (commit `d0e0890`). Its
  > system diagram (lines 54–57) and its Transport/OSC Client row (line 83) still describe
  > `/run-code` with token and port discovery. **That path is verified broken against Sonic
  > Pi v5.x and is explicitly out of scope in REQUIREMENTS.md.** Everything else in the file
  > — IR, Resolver, Emitter, structure, anti-patterns — remains current and authoritative.

### Sonic Pi integration seam (the corrected, verified path)
- `.planning/research/STACK.md` §"Sonic Pi Integration Seams" and §"Version Compatibility" —
  The unauthenticated cue server on port 4560, the resident runner-buffer pattern, and
  `osc-ruby` 1.1.5 as the client
- `.planning/research/PITFALLS.md` §"Pitfall 1" — Why `/run-code` loses and why the cue
  listener wins; includes Sam Aaron's own recommended runner-buffer shape
- `.planning/research/SUMMARY.md` — The correction stated compactly, with the runner
  buffer's `live_loop` sketch

### Pitfalls this phase must actively mitigate
- `.planning/research/PITFALLS.md` §"Pitfall 2" — Errors in pushed code vanish (→ D-15)
- `.planning/research/PITFALLS.md` §"Pitfall 3" — Silent truncation past ~8–9KB (→ D-20)
- `.planning/research/PITFALLS.md` §"Pitfall 4" — Pushing doesn't cleanly replace (→ D-21)
- `.planning/research/PITFALLS.md` §"Pitfall 5" — Unstoppable runaway `live_loop` (→ D-08)
- `.planning/research/PITFALLS.md` §"Pitfall 6" — Correct but unlistenable (→ D-10, D-13, D-14)
- `.planning/research/PITFALLS.md` §"Pitfall 7" — Where determinism quietly breaks (→ D-18)
- `.planning/research/PITFALLS.md` §"Pitfall 9" — Framework before sound (→ D-03)

### Tooling
- `.planning/research/STACK.md` §"Testing Generated Code" — `Ripper.sexp` for in-process
  syntax validation (EMIT-03), Minitest golden-file and seeded-determinism test patterns
- `.planning/research/STACK.md` §"What NOT to Use" — `sonic-pi-tool`, `sonic-pi-cli`,
  `music_theory`, `ruby-osc`, and Zeitwerk are all ruled out with reasons

</canonical_refs>

<code_context>
## Existing Code Insights

**The repository is greenfield** — `README.md` is the only tracked file outside
`.planning/` and `.claude/`. There are no codebase maps in `.planning/codebase/`.

### Reusable Assets
None — every file in this phase is new.

### Established Patterns
No in-repo patterns exist yet. The patterns this phase must follow come from
`ARCHITECTURE.md` rather than from code:
- **Generators never touch a Sonic Pi keyword; the Emitter never makes a musical decision.**
  This is the load-bearing rule for IR-01.
- **`theory/` has zero knowledge of Sonic Pi or the IR** — it must be usable in a plain
  `irb` with nothing else loaded.
- **`transport/` never imports `ir/` or `generators/`** — it only receives a finished
  string, so it is testable against literal strings with no music machinery running.
- **One file per generator class**, by convention rather than enforcement.

### Integration Points
- **Sonic Pi (external process, unmodified)** — reached only over UDP/OSC to the cue server
  on port 4560. The two processes never share a Ruby VM, so this project's Ruby version is
  independent of Sonic Pi's vendored one.
- **`runner.rb`** — the repo-root artifact pasted into a Sonic Pi buffer once per Sonic Pi
  session. It is the *only* code this project places inside Sonic Pi, and it is a
  first-class versioned deliverable (EMIT-06), not setup lore.

**Because this phase establishes every pattern the rest of the project inherits, the
structure choices here are worth getting right the first time — Phases 2–4 all build on
the `Session` object and the IR shape introduced here.**

</code_context>

<specifics>
## Specific Ideas

- **"Can we make this something we can change?"** — the user's response when asked to pick
  a single fixed scale and root. The instinct generalizes: where a musical default is
  arbitrary, prefer a flag with a good default over a hardcoded constant — while keeping
  those flags as plain keyword args, not the Phase 2 param system (D-03).
- The user consistently chose the **safe, requirement-honoring option** when a trade-off
  was surfaced: strict stable loop names over layering, conservative size cap over
  measuring the real ceiling, manual `runner.rb` over writing into Sonic Pi's `init.rb`.
  Downstream agents should read ambiguity in that direction.
- PROJECT.md's floated flavors — 16-bit Final Fantasy loops, noise walls, "weirdest version
  of a described vibe" — were considered and **not** chosen for generator #1. They remain
  live candidates for Phase 3's generators 2 and 3.

</specifics>

<deferred>
## Deferred Ideas

- **Full parameter DSL with declared names and ranges** — considered as a pull-forward for
  the progression argument (D-03); belongs to Phase 2 (LOOP-02/03/04).
- **`--as` flag for deliberately naming a loop so two can play at once** — considered
  during the EMIT-04 discussion (D-21); layering is Phase 4's scope (LIB-04/05/06).
- **Saving the last seed to a file for a `--last` replay flag** — considered during the
  seed discussion (D-07); this is Phase 2's favorites feature (LIB-07).
- **`phonicpsy runner --print` / clipboard copy of `runner.rb`** — considered for runner
  ergonomics (D-17); a nicety once the manual paste gets old, not needed for Phase 1.
- **Spiking the true OSC truncation ceiling** — considered for D-20; the conservative cap
  is sufficient until a generator approaches it.
- **Per-seed rhythmic variation and weighted octave jumps** — considered for D-04; rhythm
  stays fixed in Phase 1 to keep GEN-07 demonstrable. Octave movement is a natural fit for
  Phase 4's register separation (LIB-06).

</deferred>

---

*Phase: 1-Hear One Generator*
*Context gathered: 2026-09-15*
