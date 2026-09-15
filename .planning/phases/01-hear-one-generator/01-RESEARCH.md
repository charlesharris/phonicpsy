# Phase 1: Hear One Generator - Research

**Researched:** 2026-09-15
**Domain:** Ruby-side music generation (theory/IR/emitter) driving Sonic Pi v5.x over its unauthenticated OSC cue server
**Confidence:** HIGH — the phase's single riskiest unknown (the OSC integration seam) was verified twice: once by prior research reading the `sonic-pi-net/sonic-pi` source tree, and again this session by reading the **actually-installed** Sonic Pi v5.0.0 app bundle's source directly, plus reading the `osc-ruby` 1.1.5 gem's own source after downloading it fresh from rubygems.org.

<user_constraints>
## User Constraints (from CONTEXT.md)

### Locked Decisions

**The First Generator**
- **D-01:** Generator #1 is an **arpeggiator over a chord** — walks a chord's notes in a seeded order across a rhythmic grid. Chosen because it is the fastest path to listenable sound while exercising scale + chord + grid + weighted choice together, and because Phase 3 builds generators 2 and 3 by divergence from it. `ARCHITECTURE.md` already uses `arpeggiator.rb` as its worked example.
- **D-02:** The arpeggiator moves through a **short chord progression**, not a single held chord, because a static single chord fights Success Criterion 4's listenable bar and leaves Phase 3's endless-piece form nothing to evolve.
- **D-03:** The progression is a **plain Ruby keyword argument with a default** — not a param-declaration DSL and not runtime-introspectable. This is a deliberate boundary: Phase 2's LOOP-02/03/04 (declared params with names and ranges, runtime inspection, setting values without editing source) must remain real work. Nothing built here is thrown away when the param DSL lands — a keyword arg becomes a declared param later. — **Reversibility:** reversible — the keyword arg is the eventual param's call signature.
- **D-04:** Seeded randomness lives in **note order and rests only**. Pitch per step is a weighted `Choice` over the current chord's notes; a `Weighted` node produces the rests GEN-08 requires. **Rhythm does not vary per seed** — onsets stay on a fixed Euclidean grid so the loop grooves consistently and GEN-07's grid alignment stays demonstrable. This exercises `Choice`, `Weighted` and `spread`, which is the IR-02 surface Phase 1 needs.

**Command Surface**
- **D-05:** Primary entry point is a **CLI binary, `bin/phonicpsy`, with subcommands** (`play`, `export`, `stop`). It is a thin wrapper over a `Session` object so Phase 2's pry-based hot-reload loop reuses the same objects rather than reimplementing them. Satisfies Success Criterion 1's "run one command" literally.
- **D-06:** Argument parsing uses **`optparse` from stdlib** — no CLI gem. Keeps the core path's dependency list at `osc-ruby` alone and keeps startup fast, which matters because PROJECT.md makes the inner loop a performance constraint.
- **D-07:** With no `--seed`, the CLI **rolls a random seed and always prints it**, so re-running explores and any run you liked can be recovered by passing its seed back. This makes GEN-01's determinism useful day-to-day rather than only test-visible, and it is what Phase 2's LOOP-05 seed re-roll builds on.
- **D-08:** `phonicpsy stop` (LOOP-01) sends a **distinct OSC cue that the shipped runner buffer handles** by stopping the running loops. Same transport as push, nothing extra to install, reachable from any shell the moment things get loud (Pitfall 5).

**Sound Defaults**
- **D-09:** Scale and root are **`--scale` / `--root` flags** (plain keyword args per D-03, not a param DSL). *User explicitly asked for these to be changeable rather than fixed.* Nearly free to implement: per GEN-04 the emitted code calls Sonic Pi's own `scale`, so the theory core passes the name through and only needs the degree formula for *choosing* pitches during generation.
- **D-10:** Default when neither flag is passed: **A minor pentatonic**. Chosen as the default specifically because five notes with no semitone clashes make it very hard for the first sound the tool ever produces to be the unlistenable one (Pitfall 6).
- **D-11:** `--scale` accepts **any scale name Sonic Pi itself knows** (~40 per the user's original framing — **corrected by this research to 157, see Common Pitfalls / Assumptions Log**), validated against a list in the theory core. An unknown name **fails loudly at the CLI**, not at Sonic Pi runtime. Costs no more than hardcoding three, since the formula table is data.
- **D-12:** Default tempo and grid: **110 BPM on a 16th-note grid**. Mid-tempo and unhurried; a 16th grid gives the Euclidean `spread` enough slots to make an interesting pattern without becoming a blur, and leaves room for rests.
- **D-13:** **The Euclidean `spread` onsets ARE the density cap** (GEN-08). Empty slots in the pattern are the rests, so they land musically rather than randomly. One mechanism satisfies GEN-05, GEN-07 and GEN-08 together. No separate rest probability, no post-generation notes-per-bar filter.
- **D-14:** Default synth is **`:prophet`** — warm and forgiving of short releases, very hard to make harsh, so the untuned default run clears Criterion 4.

**Transports and Failure Feedback**
- **D-15:** The runner buffer **wraps its `eval` in `begin`/`rescue` and acks back over OSC** — success or the error message — to a UDP port the CLI listens on briefly. This is the direct mitigation for Pitfall 2 ("errors in pushed code vanish into the void"), which is the most likely day-one failure. Requires a receive socket in the CLI. — **Reversibility:** costly — the ack protocol is a contract between `runner.rb` and the OSC client; changing its shape later means changing both sides plus any Phase 2 code that waits on it.
- **D-16:** Push **pings first and fails with instructions**: before sending code, send a cheap ping cue and wait a short timeout. No ack means print `Sonic Pi runner not responding — open runner.rb in Sonic Pi and hit Run` **including the file's absolute path**. This is what makes Criterion 1's "no manual step beyond having the runner buffer running" honest — the one manual step names itself the moment it's missing.
- **D-17:** The runner buffer ships as **`runner.rb` in the repo root, run manually** in Sonic Pi (EMIT-06). Versioned in git, depends on nothing inside Sonic Pi's install layout. Explicitly **not** installed into Sonic Pi's `init.rb` — that would write into Sonic Pi's own config, against the project's "public interfaces only" constraint.
- **D-18:** Exported `.rb` files carry a header comment with **generator, seed, and parameter values only — no timestamp, no absolute path**. Anything varying between runs would break the byte-identical diff that Success Criterion 3 is defined by. The header must contain everything needed to regenerate the file and nothing else.
- **D-19:** Exports default to **`out/<generator>-<seed>.rb`**, overridable with `--out`. The seed in the filename makes a directory of exports self-describing, different seeds never collide, and re-exporting the same seed overwrites with identical bytes — itself a spot-check of Criterion 3. `out/` is gitignored.
- **D-20:** EMIT-07's payload ceiling is a **conservative fixed cap around 8KB**, below the ~8–9KB silent-truncation point PITFALLS.md identifies, failing loudly with the actual byte count. A Phase 1 arpeggiator is far under it; this is a guardrail that costs nothing now and catches the failure the day a generator grows.
- **D-21:** `live_loop` names derive from the **generator name only** — `live_loop :phonicpsy_arpeggiator` — stable across every re-push regardless of seed or params (EMIT-04). No seed in the name, no `--as` override, no layering affordance in Phase 1. Deliberate layering is Phase 4's scope. — **Reversibility:** reversible — adding an override flag later is additive.

  **Note for planner/verifier:** the user initially chose generator-name-plus-seed here. That was surfaced as a direct conflict with EMIT-04 and Criterion 5 (every re-roll would leave the previous loop playing, making `stop` the only way back to silence), and the user resolved it to the strict stable-name form above. **No requirement changes were made.** EMIT-04 and Criterion 5 stand as written.

### Claude's Discretion

The user made an explicit choice on every question asked — no "you decide" answers. The following remain unspecified and are the planner's/researcher's call:

- Ack timeout duration and the exact OSC paths for push / ping / stop / ack
- Note release and sustain values, octave range, and loop length in bars
- Whether any FX (e.g. reverb) appear in the emitted code
- The specific default chord progression and its length
- Euclidean onset count (e.g. 7-in-16) for the default pattern
- How the walk order over chord notes is structured
- Exactly how the seed-diff determinism check is run in plan 01-03, and what the listening pass consists of

### Deferred Ideas (OUT OF SCOPE)

- **Full parameter DSL with declared names and ranges** — considered as a pull-forward for the progression argument (D-03); belongs to Phase 2 (LOOP-02/03/04).
- **`--as` flag for deliberately naming a loop so two can play at once** — considered during the EMIT-04 discussion (D-21); layering is Phase 4's scope (LIB-04/05/06).
- **Saving the last seed to a file for a `--last` replay flag** — considered during the seed discussion (D-07); this is Phase 2's favorites feature (LIB-07).
- **`phonicpsy runner --print` / clipboard copy of `runner.rb`** — considered for runner ergonomics (D-17); a nicety once the manual paste gets old, not needed for Phase 1.
- **Spiking the true OSC truncation ceiling** — considered for D-20; the conservative cap is sufficient until a generator approaches it.
- **Per-seed rhythmic variation and weighted octave jumps** — considered for D-04; rhythm stays fixed in Phase 1 to keep GEN-07 demonstrable. Octave movement is a natural fit for Phase 4's register separation (LIB-06).
</user_constraints>

<phase_requirements>
## Phase Requirements

| ID | Description | Research Support |
|----|-------------|------------------|
| GEN-01 | Same seed and same parameters produce byte-identical emitted code across runs | Determinism Architecture below; `use_random_seed` idiom verified against official tutorial; seed-diff check pattern in Validation section |
| GEN-02 | Generator-side randomness uses a seeded `Random` instance derived from the master seed, never global `Kernel.rand` | Code Examples — "Threading an explicit `Random.new(seed)`"; Pitfall — Determinism traps |
| GEN-03 | Self-contained emitted code carries a `use_random_seed` derived from the master seed as the first statement inside the `live_loop` body | Code Examples — verified `use_random_seed` placement, quoted from Sonic Pi's own A.09 tutorial |
| GEN-04 | Generators compose from scale and chord primitives that emit Sonic Pi's own `scale`/`chord` rather than reimplementing them in Ruby | `chord()` and `scale` signatures verified directly from installed Sonic Pi v5.0.0 source below |
| GEN-05 | Generators compose from rhythm primitives, including Euclidean patterns emitted as Sonic Pi's `spread` | `spread()` signature and verified worked example below |
| GEN-06 | Generated pitches are quantized to the generator's declared scale | Don't Hand-Roll — scale degree lookup vs. raw MIDI |
| GEN-07 | Generated note onsets align to a declared rhythmic grid | `spread()` ring output is inherently grid-aligned — see Code Examples |
| GEN-08 | Random selection is weighted rather than uniform, and a density limit caps notes per bar so output contains rests | D-13's `spread` onsets = density cap; `Weighted` IR node pattern below |
| IR-01 | Generators build a structured representation of musical events; they never emit Sonic Pi source strings directly | Architecture Patterns — minimal IR shape for Phase 1 |
| IR-02 | The representation holds randomness symbolically (choice, range, weighted) as data rather than as already-resolved values | Architecture Patterns — `Choice`/`Weighted` nodes, scoped to Phase 1's needs only |
| IR-03 | One generator implementation can drive both self-contained emission and future streamed emission without being rewritten | Architecture Patterns — IR shape rationale (design property, verified by review) |
| EMIT-01 | An emitter renders the intermediate representation into Sonic Pi source | Architecture Patterns — Emitter responsibility and anti-pattern boundary |
| EMIT-02 | Self-contained mode renders symbolic randomness as Sonic Pi randomness calls inside the emitted `live_loop` | Code Examples — symbolic rendering of `Choice`/`Weighted` to `choose`/weighted array |
| EMIT-03 | Emitted source is syntax-validated before it is pushed or written, and invalid source fails loudly rather than silently | Code Examples — `Ripper.sexp` verified behavior this session |
| EMIT-04 | `live_loop` names are stable across re-pushes, so re-pushing replaces the running loop instead of stacking a second one | D-21; Common Pitfalls — Pitfall 4 |
| EMIT-05 | Generated code can be pushed into a running Sonic Pi over the OSC cue server and is audible without any copy-paste | Sonic Pi Integration Seam section — verified against installed v5.0.0 |
| EMIT-06 | The repo ships the Sonic Pi runner buffer that receives cued code, as a versioned artifact rather than pasted-in setup lore | Code Examples — runner.rb pattern, extended with D-15's ack + D-08's stop cue |
| EMIT-07 | Payload size is checked against the OSC datagram ceiling before sending; an oversized push reports an error instead of failing silently | Common Pitfalls — Pitfall 3, D-20's 8KB cap |
| EMIT-08 | Generated code can be written to a `.rb` file that opens and runs in the Sonic Pi editor, with no Sonic Pi process required | Architecture Patterns — File Exporter is a pure `String` -> disk write, no Sonic Pi dependency |
| LOOP-01 | A single command stops every running loop | D-08; Code Examples — verified `stop` semantics inside `live_loop` |
</phase_requirements>

## Summary

Phase 1's architecture and stack were already researched in depth (`ARCHITECTURE.md`, `STACK.md`, `PITFALLS.md`, `SUMMARY.md`) — that work remains current and is the primary reference for the planner. This document adds three things: (1) it resolves that research against CONTEXT.md's 21 locked decisions so the planner has one coherent picture instead of two documents to reconcile; (2) it **re-verifies the phase's one required research spike** — the OSC cue-server seam — against the Sonic Pi binary **actually installed on this machine** (v5.0.0, matching the version the prior research targeted) rather than only against a cloned source tree; and (3) it pulls exact, quoted signatures for `spread`, `chord`, `use_random_seed`, `Ripper.sexp`, and the `osc-ruby` gem's non-EventMachine server class directly from primary sources read this session, because Plan 01-01/01-02 need these exact shapes, not summaries of them.

**Primary recommendation:** Build the runner buffer and OSC transport first (Plan 01-02 can be sequenced before or alongside 01-01 without risk — the integration seam is now double-verified), because everything else in the phase is ordinary Ruby that has no external dependency risk. Use the plain (non-EventMachine) `OSC::Server` class from `osc-ruby` for the ack/ping listener in D-15/D-16 — confirmed this session to need only Ruby's `socket` and `thread` stdlib, not EventMachine. Treat D-11's "~40 scale names" as **superseded**: the installed Sonic Pi actually defines 157, and the theory core's validation list should be the full set (given verbatim below), not a guessed subset.

**One correction to a locked decision's stated premise (not a decision reversal):** D-11 assumed Sonic Pi has "~40" scale names. Reading `scale.rb` directly shows 157 (including many Turkish makam scales layered on the same 33-name Western/pentatonic/exotic core the user likely had in mind). This does not change D-11's decision (validate against Sonic Pi's real list, fail loudly on an unknown name) — it only changes what that list contains. Flagged here rather than silently "fixed" because the planner should decide whether Phase 1 validates against the full 157 or a curated subset, and CONTEXT.md's "Claude's Discretion" section does not cover this.

## Architectural Responsibility Map

This project has no web/browser tiers — it's a local CLI process driving an external, unmodified GUI application over UDP. The standard tier vocabulary is adapted accordingly: "Client/CLI" replaces Browser, "Generation Core" and "Transport" replace API/Backend, "External Process" (Sonic Pi, out of this project's control) replaces CDN, and "Filesystem" replaces Database.

| Capability | Primary Tier | Secondary Tier | Rationale |
|------------|-------------|----------------|-----------|
| Argument parsing, seed rolling, subcommand dispatch | Client/CLI (`bin/phonicpsy`) | — | D-05/D-06; thin wrapper, no logic lives here per ARCHITECTURE.md's Session pattern |
| Scale/chord/rhythm formulas, degree lookups | Generation Core (`theory/`) | — | Must have zero knowledge of Sonic Pi or OSC (ARCHITECTURE.md); testable in plain `irb` |
| Pattern building (arpeggiator logic, `Choice`/`Weighted` node placement) | Generation Core (`generators/`, `ir/`) | — | IR-01: generators build structured data, never strings |
| IR → Sonic Pi source string rendering | Generation Core (`emit/`) | — | EMIT-01/02; only place string-building/heredocs are allowed |
| Pre-flight syntax + size validation | Transport (`transport/` or a shared validator) | Generation Core | EMIT-03/07; must run before either transport, so it is shared, not duplicated per-transport |
| `.rb` file export | Transport (`transport/file_exporter.rb`) | Filesystem | EMIT-08; pure `String` -> disk write, testable without Sonic Pi |
| OSC push, ping, ack-listen, stop-cue | Transport (`transport/osc_client.rb`) | External Process (Sonic Pi) | EMIT-05, LOOP-01, D-15/D-16; never imports `ir/` or `generators/` per ARCHITECTURE.md |
| `eval`-and-ack loop, stop-flag handling | External Process (Sonic Pi's `runner.rb`) | — | EMIT-06; the *only* code this project places inside Sonic Pi, versioned in the repo but executed by Sonic Pi's own Ruby VM, not this project's |
| Exported `.rb` files, `runner.rb` | Filesystem | — | `out/` gitignored (D-19); `runner.rb` versioned in repo root (D-17) |

## Standard Stack

### Core

| Library | Version | Purpose | Why Standard |
|---------|---------|---------|--------------|
| Ruby | 3.4.4 (installed) | Generator language, CLI runtime | `[VERIFIED: ruby -v, this session]` — output: `ruby 3.4.4 (2025-05-14 revision a38531fd3f) +PRISM [arm64-darwin24]`. Satisfies STACK.md's stated floor of 3.2+; the 4.0.x figure in STACK.md is a "current stable" aspiration, not a requirement — nothing in this phase needs a Ruby feature newer than 3.2. |
| `osc-ruby` gem | 1.1.5 | Send/receive OSC over UDP to Sonic Pi's cue server | `[VERIFIED: gem source, this session]` — fetched fresh from rubygems.org (`gem fetch osc-ruby -v 1.1.5`) and read `lib/osc-ruby.rb`: the default require list is `socket`, `thread`, plus the gem's own files — **`em_server.rb` (the EventMachine-dependent server) is not required by default**. `OSC::Server` (plain `server.rb`) uses only stdlib `UDPSocket` + `Thread.fork`, confirming STACK.md's claim that EventMachine is optional, not load-bearing for this phase's send-and-listen-for-ack use case. |
| Hand-rolled music-theory core | n/a | Scale/chord degree formulas, Euclidean grid helper | `[VERIFIED]` — no maintained Ruby equivalent exists (STACK.md's live rubygems.org query); confirmed again this session by reading Sonic Pi's own `scale.rb` and `western_theory.rb`, which is the reference implementation this project's formulas should match in *shape* (interval arrays) without depending on Sonic Pi's Ruby process. |

### Supporting

| Library | Version | Purpose | When to Use |
|---------|---------|---------|-------------|
| `pry` | 0.16.0 | Interactive session for later manual verification / debugging during Phase 1 build-out | `[VERIFIED: gem list --local, this session]` — already installed locally. Not required by Phase 1's CLI path (that's `optparse` per D-06) but useful for the researcher/planner's own spike work against the runner buffer. |
| Minitest | stdlib (6.0.6 also present) | Golden-file snapshot tests, seeded-determinism tests | `[VERIFIED: gem list --local, this session]` — ships with Ruby regardless; a newer standalone version is also installed. No RSpec needed. |
| Ripper | stdlib | Syntax-validate emitted source (EMIT-03) | `[VERIFIED: ruby -e, this session]` — see Code Examples below for exact confirmed behavior. |

### Alternatives Considered

| Instead of | Could Use | Tradeoff |
|------------|-----------|----------|
| `osc-ruby`'s plain `OSC::Server` for the ack/ping listener | Raw `UDPSocket` + `IO.select` + `OSC::OSCPacket.messages_from_network` (also in the gem, used internally) | The plain `OSC::Server#run` blocks the calling thread in a receive loop with **no built-in timeout** — see Common Pitfalls below. For D-16's "wait a short timeout" ping, bypassing `OSC::Server` and driving the raw socket + `IO.select(..., timeout)` directly is very likely the better fit; `OSC::Server` is a better fit if the CLI is willing to run the listener in a background `Thread` for the whole `push` command's lifetime instead. |
| Hand-rolled OSC packet encode/decode | `osc-ruby`'s `Message`/`OSCPacket` classes | Already the STACK.md recommendation; reconfirmed — the gem's `OSC::Message#encode` and `OSC::OSCPacket.messages_from_network` are small, readable, and remove an entire category of binary-format bugs. |

**Installation:**
```bash
gem install osc-ruby   # not currently installed locally — see Environment Availability
# pry and a Minitest are already present; no action needed there
```

**Version verification:** `osc-ruby` 1.1.5 confirmed present on rubygems.org and successfully fetched this session (`gem fetch osc-ruby -v 1.1.5` → `Downloaded osc-ruby-1.1.5`). Ruby 3.4.4 confirmed installed and meets the project's stated floor.

## Package Legitimacy Audit

> The GSD `package-legitimacy check` seam supports only `npm`/`pypi`/`crates` ecosystems — it has no `rubygems` mode (confirmed by invoking it this session; it rejected `--ecosystem rubygems` with a usage error). Verification below is manual, via `gem fetch`/`gem list` against the live rubygems.org registry plus direct source inspection, which is the ecosystem-appropriate substitute per this protocol's Step 2.

| Package | Registry | Age | Downloads | Source Repo | Verdict | Disposition |
|---------|----------|-----|-----------|-------------|---------|-------------|
| `osc-ruby` | rubygems.org | Gem released 2024-07-21; repo pushed 2025-11-16 (per STACK.md's same-day `gh api` check) | 62,552 (per STACK.md's same-day rubygems.org API check) | `github.com/aberant/osc-ruby` | OK | Approved — re-verified this session by fetching and reading the actual gem source; no native extensions, no postinstall equivalent (Ruby gems don't have npm-style postinstall scripts as a general mechanism), pure-Ruby stdlib-only requires |
| `pry` | rubygems.org | Long-established, actively maintained (STACK.md, same-day check) | High (STACK.md) | `github.com/pry/pry` | OK | Approved — already installed locally, not a new dependency risk |
| Minitest | Ruby stdlib | Ships with Ruby | n/a | ruby-lang.org | OK | Approved — stdlib, no registry risk |
| Ripper | Ruby stdlib | Ships with Ruby | n/a | ruby-lang.org | OK | Approved — stdlib, no registry risk |
| `optparse` | Ruby stdlib | Ships with Ruby | n/a | ruby-lang.org | OK | Approved — `[VERIFIED: gem list --local, this session]` shows `optparse (default: 0.6.0)` |

**Packages removed due to [SLOP] verdict:** none.
**Packages flagged as suspicious [SUS]:** none. `osc-ruby` is the only new runtime dependency Phase 1 adds, and it has now been verified two independent ways (rubygems.org live metadata in prior research, direct gem-source read this session).

## Architecture Patterns

Full architecture guidance is in `.planning/research/ARCHITECTURE.md` and remains authoritative for Phase 1 **except its OSC transport description**, which is stale (see the staleness warning already carried into this phase's CONTEXT.md). The material below is Phase-1-scoped: what to actually build, not the full 4-phase design.

### System Architecture Diagram

```
   bin/phonicpsy (CLI: play | export | stop)
          │
          │ builds
          ▼
   Session (seed, params, mode=self-contained-only for Phase 1)
          │
          │ generator.pattern(seed:, progression:, scale:, root:, ...)
          ▼
   Arpeggiator (generators/arpeggiator.rb)
          │  uses Theory::Scale, Theory::Chord, Theory::Rhythm.spread_pattern
          │  builds IR::Pattern of IR::Event{ pitch: Choice, gate: Weighted }
          ▼
   IR::Pattern  ────────────────────────────────────────────────────┐
          │                                                          │
          │ Emitter.render(pattern, seed:) -> String                │ (same Pattern,
          ▼                                                          │  same Emitter —
   Sonic Pi source string (a live_loop body)                         │  IR-03's dual-mode
          │                                                          │  property, unexercised
          ├─ pre-flight: Ripper.sexp(code) — fail loud if nil        │  until Phase 3 but
          ├─ pre-flight: code.bytesize > ~8KB — fail loud             │  must not be broken)
          │                                                          │
          ▼                                                          ▼
   ┌─────────────────────┐                          ┌──────────────────────────┐
   │  File Exporter       │                          │  OSC Client                │
   │  write out/<gen>-    │                          │  1. ping /phonicpsy-ping  │
   │  <seed>.rb            │                          │  2. wait ack (timeout)    │
   │  (no Sonic Pi needed) │                          │  3. push /phonicpsy-run   │
   └─────────────────────┘                          │  4. wait run-ack/error     │
                                                       └──────────────┬─────────────┘
                                                                      │ UDP :4560
                                                                      ▼
                                            ┌───────────────────────────────────────┐
                                            │  Sonic Pi (external, unmodified, v5.x) │
                                            │  runner.rb — resident live_loop:       │
                                            │    sync "/osc*/phonicpsy-run" -> eval  │
                                            │    begin/rescue -> ack back over OSC   │
                                            │    sync "/osc*/phonicpsy-stop" -> stop │
                                            └───────────────────────────────────────┘
```

Verified against the **installed** app: Sonic Pi.app version `5.0.0` (`[VERIFIED: /Applications/Sonic Pi.app/Contents/Info.plist, CFBundleShortVersionString, this session]`), matching the version the prior `STACK.md`/`ARCHITECTURE.md` research targeted (also v5.0.0). The default cue-in port is 4560, confirmed three independent ways this session: `daemon.rb` line 1145 (`"osc-cues" => 4560`), `spider-server.rb` line 133 (`osc_cues_port = ARGV[5] ? ARGV[5].to_i : 4560`), and the shipped tutorial doc `etc/doc/tutorial/12.1-Receiving-OSC.md` ("By default when Sonic Pi is launched it listens to port 4560").

### Recommended Project Structure

Phase-1-scoped subset of ARCHITECTURE.md's full structure (no `resolve/`, no `Expr` hierarchy beyond what D-04 needs — see Suggested Build Order note in ARCHITECTURE.md, "not every generator needs `Expr` nodes on day one"):

```
lib/phonicpsy/
├── theory/
│   ├── scale.rb          # degree formulas + the verified 157-name validation list
│   ├── chord.rb           # progression as an array of [root, quality] pairs
│   └── rhythm.rb           # Euclidean helper wrapping the same math as Sonic Pi's spread
├── ir/
│   ├── event.rb            # pitch (Choice), gate (Weighted), duration (literal)
│   ├── pattern.rb           # ordered Events + loop metadata
│   └── expr.rb              # ONLY Choice and Weighted for Phase 1 (D-04) — no RandRange, no TickRing yet
├── generators/
│   └── arpeggiator.rb        # the one Phase 1 generator (D-01/D-02)
├── emit/
│   └── sonic_pi_emitter.rb    # Pattern -> live_loop source string; owns use_random_seed placement (GEN-03)
├── transport/
│   ├── validator.rb            # Ripper.sexp + bytesize check (EMIT-03/07), shared by both transports
│   ├── file_exporter.rb        # EMIT-08
│   └── osc_client.rb            # EMIT-05, LOOP-01, D-15/D-16 ping+ack+stop
└── session.rb                   # D-05's thin-wrapper target; seed rolling (D-07)

bin/phonicpsy       # optparse subcommands: play, export, stop (D-05/D-06)
runner.rb            # repo root, versioned, pasted into Sonic Pi manually (D-17/EMIT-06)
out/                  # gitignored export destination (D-19)
```

### Pattern 1: The verified runner buffer, extended for D-15/D-16/D-08

**What:** A resident `live_loop` inside Sonic Pi that receives cued code and evals it, acking success/failure back, and separately handling a stop cue.
**When to use:** This is `runner.rb` (EMIT-06), the one artifact this project places inside Sonic Pi.
**Example** (base pattern `[VERIFIED: abhishekjairath/sonic-pi-mcp, sonic-pi-queue.rb, per STACK.md]`; port and `sync` path syntax `[VERIFIED: /Applications/Sonic Pi.app/Contents/Resources/etc/doc/tutorial/12.1-Receiving-OSC.md, this session]`; `use_osc`/`osc` send-back syntax `[VERIFIED: same tutorial file, 12.2-Sending-OSC.md]`; ack-timeout and exact cue-path naming for ping/stop are `[ASSUMED]` — Claude's Discretion per CONTEXT.md):

```ruby
# runner.rb — paste into a Sonic Pi buffer once, hit Run, leave it running.
# Sonic Pi automatically prefixes incoming OSC with /osc, so a message this
# project sends to path "/phonicpsy-run" is synced here as "/osc*/phonicpsy-run".

use_osc "localhost", 4559          # ack destination — the CLI's listen port (D-15); exact port TBD by planner

live_loop :phonicpsy_runner do
  use_real_time
  code = sync "/osc*/phonicpsy-run"
  begin
    eval(code[0])
    osc "/phonicpsy-ack", "ok"
  rescue Exception => e
    osc "/phonicpsy-ack", "error: #{e.message}"
  end
end

live_loop :phonicpsy_stopper do
  use_real_time
  sync "/osc*/phonicpsy-stop"
  # Per Sonic Pi's own `stop` semantics (verified below): `stop` only halts the
  # *current* live_loop's thread. Stopping "every running loop" (LOOP-01) means
  # this buffer must know every loop name this project has ever pushed and
  # re-run a `stop`-carrying redefinition of each — the exact stop-all mechanic
  # is Claude's Discretion; STACK.md's recommended shape is a shared set/get flag
  # each generator's live_loop checks every iteration.
end
```

**Verified `stop` semantics** `[VERIFIED: /Applications/Sonic Pi.app/Contents/Resources/app/server/ruby/lib/sonicpi/lang/core.rb:1565-1600, this session]` — the doc string quoted verbatim: *"Stops the current thread or if not in a thread, stops the current run. Does not stop any running synths triggered previously in the run/thread or kill any existing sub-threads."* and the shipped example: *"Stopping live loops... live_loop :foo ... stop # live loop :foo will now stop and no longer loop"*. This confirms `stop` called from **inside** a specific `live_loop`'s body stops **that** loop — a true "stop everything" panic command must therefore either be baked into every pushed generator's loop body (checking a shared flag) or the runner buffer must track and individually signal every loop name it has ever started, since there is no single OSC message that halts all jobs from outside (the token-gated `/stop-all-jobs` channel is out of scope per the phase's research spike boundary).

### Pattern 2: Non-blocking-with-timeout ack listener

**What:** The CLI process needs to wait "a short timeout" (D-16) for a ping-ack before deciding Sonic Pi's runner isn't listening, and again after a push for the run-ack (D-15).
**When to use:** `transport/osc_client.rb`.
**Example** (all API calls `[VERIFIED: osc-ruby 1.1.5 gem source, lib/osc-ruby/server.rb and osc_packet.rb, read this session]`):

```ruby
require "socket"
require "osc-ruby"

def wait_for_osc_ack(port, timeout_seconds)
  socket = UDPSocket.new
  socket.bind("", port)
  ready = IO.select([socket], nil, nil, timeout_seconds)
  return nil unless ready # timed out — no response

  data, = socket.recvfrom(16_384)
  messages = OSC::OSCPacket.messages_from_network(data)
  messages.first
ensure
  socket.close
end
```

**Why not `OSC::Server#run` directly:** `[VERIFIED: server.rb, read this session]` — `OSC::Server#run` calls `start_dispatcher` (forks a background `Thread` that pops from an internal `Queue` and dispatches to registered matchers) and then calls `start_detector` **synchronously in the calling thread**, which loops forever on `@socket.recvfrom(16384)` with no timeout parameter anywhere in the class. Using it as-is for a "wait N seconds, then fail with instructions" ping would require running it in a background `Thread` and separately implementing the timeout/join logic — strictly more code than driving the raw `UDPSocket` + `IO.select` directly, for a one-shot ping/ack that doesn't need the class's message-address-pattern-matching machinery.

### Anti-Patterns to Avoid

- **Trying to make `OSC::Server#run` do timeout-bounded single-message waits.** It's built for a long-lived listener with registered handlers, not a request/response ping. See Pattern 2 above.
- **Validating `--scale` against a hand-typed list of "the common ones."** Sonic Pi's real list is 157 names (verified below); a curated subset silently rejects a valid Sonic Pi scale name, which is exactly the "fails loudly at the wrong layer" problem D-11 exists to prevent. Either embed the full verified list or explicitly scope down with a documented reason — don't guess a subset.
- **Building the stop-all mechanism as a single OSC message the runner "just handles" without a per-loop flag or registry.** Verified `stop` semantics show it only affects the loop it's called from — see Pattern 1.
- Everything already listed in `ARCHITECTURE.md`'s five anti-patterns remains current (generators building Sonic Pi strings directly, resolving randomness inside the generator, framework-weight autoloading, trying to correlate the two RNG streams, one monolithic `live_loop`).

## Don't Hand-Roll

| Problem | Don't Build | Use Instead | Why |
|---------|-------------|-------------|-----|
| Euclidean rhythm distribution | A Bjorklund/Euclidean algorithm implementation | Sonic Pi's own `spread(num_accents, size, rotate:)`, emitted directly into the generated code | `[VERIFIED: /Applications/Sonic Pi.app/Contents/Resources/app/server/ruby/lib/sonicpi/lang/core.rb:1758-1796, this session]` — `spread` is already the Toussaint 2005 algorithm, returns a `ring` of booleans, and the doc's own worked example (`(spread 3, 8) #=> (ring true, false, false, true, false, false, true, false)`) is quoted verbatim from the source's `doc:` block. GEN-05/GEN-07/GEN-08 are satisfied by emitting a call to this function, not reimplementing it in Ruby. |
| Chord voicing / MIDI note lookup | A chord-interval table plus MIDI arithmetic | Sonic Pi's own `chord(tonic, name)`, emitted directly | `[VERIFIED: /Applications/Sonic Pi.app/Contents/Resources/app/server/ruby/lib/sonicpi/lang/western_theory.rb:965-1010, this session]` — returns `.ring` of MIDI note numbers; the theory core only needs the tonic + quality *name* to pass through (GEN-04), not the interval math. |
| OSC packet encoding/decoding | Manual byte-packing of OSC's type-tag-string format | `osc-ruby`'s `OSC::Message#encode` / `OSC::OSCPacket.messages_from_network` | `[VERIFIED: message.rb, osc_packet.rb, read this session]` — small, correct, already handles the address+type-tag+args framing and 4-byte padding. |
| Ruby-side syntax validation of generated code | A hand-rolled Sonic Pi DSL parser or shelling to `ruby -c` | stdlib `Ripper.sexp(code)` | `[VERIFIED: ruby -e 'require "ripper"; ...', this session]` — confirmed it returns a parse-tree Array for valid code and `nil` for invalid code (exact outputs quoted in Code Examples below). In-process, sub-millisecond, no subprocess. |
| Sonic Pi's own random stream reset | Trying to synchronize Ruby's RNG with Sonic Pi's | `use_random_seed <n>` as the first line inside the emitted `live_loop`, per GEN-03 | `[VERIFIED: /Applications/Sonic Pi.app/Contents/Resources/etc/doc/tutorial/A.09-randomisation.md, this session]` — official tutorial's own worked `live_loop :random_riff` example places `use_random_seed` as the first statement inside the loop body, matching ARCHITECTURE.md's recommendation exactly. |

**Key insight:** every "don't hand-roll" item in this phase already exists as a verified, documented Sonic Pi DSL function or a small, source-read gem method. There is no case in Phase 1 where custom implementation is genuinely necessary for anything except the theory-core *formulas* (which pick pitches during generation) and the IR/Emitter glue (which is inherent to the architecture, not a reimplementation of something Sonic Pi already provides).

## Common Pitfalls

The seven pitfalls PITFALLS.md maps to this phase (1, 2, 3, 4, 5, 6, 7, 9) remain fully applicable and are not repeated in full here — see `.planning/research/PITFALLS.md`. Two additions from this session's direct verification:

### Pitfall A: D-11's "~40 scale names" undercounts by roughly 4x

**What goes wrong:** If the planner or a generator author hand-types a "common scales" list based on the ~40 figure in CONTEXT.md, `--scale` will reject valid Sonic Pi scale names the moment a user tries anything outside the Western/common set (e.g., any of the ~120 Turkish makam scale names Sonic Pi also defines).
**Why it happens:** The ~40 figure was a reasonable guess at "scale, planning time" but Sonic Pi's `scale.rb` layers a large Turkish makam scale library (`hicaz`, `segah`, `kurdili_hicazkar`, etc., many with numbered variants) on top of the ~33 Western/pentatonic/exotic names most people mean by "Sonic Pi's scales."
**How to avoid:** `[VERIFIED: /Applications/Sonic Pi.app/Contents/Resources/app/server/ruby/lib/sonicpi/scale.rb, `SCALE = lambda{...}` hash keys, read directly this session]` — the full verified list of 157 names is: `diatonic ionian major dorian phrygian lydian mixolydian aeolian minor locrian hex_major6 hex_dorian hex_phrygian hex_major7 hex_sus hex_aeolian minor_pentatonic yu major_pentatonic gong egyptian shang jiao zhi ritusen whole_tone whole chromatic harmonic_minor melodic_minor_asc hungarian_minor octatonic messiaen1 messiaen2 messiaen3 messiaen4 messiaen5 messiaen6 messiaen7 super_locrian hirajoshi kumoi neapolitan_major bartok bhairav locrian_major ahirbhairav enigmatic neapolitan_minor pelog augmented2 scriabin harmonic_major melodic_minor_desc romanian_minor hindu iwato melodic_minor diminished2 marva melodic_major indian spanish prometheus diminished todi leading_whole augmented purvi chinese acem_asiran acem_kurdi acemli_rast acoustic altered bayati bayati_2 bayati_araban bestenigar blues_major blues_minor buselik buselik_2 byzantine cargah double_harmonic dugah dugah_2 evcara evcara_2 evcara_3 evcara_4 evic evic_2 ferahfeza ferahfeza_2 ferahnak gulizar hicaz hicaz_2 hicaz_humayun hicaz_humayun_2 hicazkar hicazkar_2 huseyni huseyni_2 huzzam huzzam_2 isfahan isfahan_2 karcigar kurdi kurdili_hicazkar kurdili_hicazkar_2 kurdili_hicazkar_3 kurdili_hicazkar_4 kurdili_hicazkar_5 lydian_dominant lydian_minor mahur muhayyer neva neva_2 nihavend nihavend_2 phrygian_dominant rast saba sedaraban sedaraban_2 segah segah_2 sehnaz sehnaz_2 sehnaz_3 sehnaz_4 sevkefza sevkefza_2 sevkefza_3 sultani_yegah sultani_yegah_2 suzidil suzidil_2 suznak suznak_2 tahir tahir_2 ussak uzzal uzzal_2 yegah yegah_2 zirguleli_hicaz zirguleli_hicaz_2 zirguleli_suznak zirguleli_suznak_2 zirguleli_suznak_3`. The planner should decide (this is not covered by CONTEXT.md's Claude's Discretion list) whether Phase 1 embeds this full list or a documented, deliberately curated subset — either is defensible, but it must be a decision, not an accident of an undercounted estimate.
**Phase to address:** Plan 01-01 (theory primitives) — this is exactly where the scale-name validation list per D-11 gets written.

### Pitfall B: `osc-ruby`'s plain `OSC::Server#run` has no timeout — don't reach for it for D-16's ping

**What goes wrong:** A natural first read of `osc-ruby`'s docs suggests `OSC::Server.new(port).run` as "the" way to listen for OSC in Ruby. Used naively for D-16's "ping, wait a short timeout, fail with instructions" flow, it blocks forever with no way to bound the wait.
**Why it happens:** `[VERIFIED: server.rb, read this session]` — `start_detector` runs the receive loop synchronously in the calling thread; there is no timeout parameter on `#run`, `#add_method`, or anywhere else in the class.
**How to avoid:** Use the raw `UDPSocket` + `IO.select(..., timeout)` + `OSC::OSCPacket.messages_from_network` pattern shown in Code Examples / Pattern 2 above for the ping/ack round-trips specifically. `OSC::Server` remains fine for anything that's genuinely meant to run forever (nothing in Phase 1 needs that on the CLI side — the CLI only ever waits briefly for an ack, it doesn't run a persistent listener).
**Phase to address:** Plan 01-02 (transports) — this is exactly D-15/D-16's implementation.

## Code Examples

All verified against the actually-installed Sonic Pi v5.0.0 and the freshly-downloaded `osc-ruby` 1.1.5 gem this session.

### `spread` — verified signature and output

```ruby
# Source: /Applications/Sonic Pi.app/Contents/Resources/app/server/ruby/lib/sonicpi/lang/core.rb:1758
# spread(num_accents, size, *args) — args_h supports rotate:
(spread 3, 8)                #=> (ring true, false, false, true, false, false, true, false)
(spread 3, 8, rotate: 1)     #=> (ring true, false, false, true, false, true, false, false)
```

### `chord` — verified signature

```ruby
# Source: /Applications/Sonic Pi.app/Contents/Resources/app/server/ruby/lib/sonicpi/lang/western_theory.rb:965
# chord(tonic_or_name, *opts) -> ring of MIDI note numbers
puts (chord :e, :minor)   #=> (ring 64, 67, 71)
```

### `use_random_seed` — verified placement idiom

```ruby
# Source: /Applications/Sonic Pi.app/Contents/Resources/etc/doc/tutorial/A.09-randomisation.md (official tutorial)
live_loop :random_riff do
  use_random_seed 10300
  use_synth :prophet
  s = [0.125, 0.25, 0.5].choose
  8.times do
    r = [0.125, 0.25, 1, 2].choose
    n = (scale :e3, :minor).choose
    co = rrand(30, 100)
    play n, release: r, cutoff: co
    sleep s
  end
end
```
This is the exact idiom GEN-03 requires the Emitter to reproduce: `use_random_seed` as the **first statement inside** the `live_loop` body.

### `Ripper.sexp` — verified pre-flight validation behavior

```ruby
# Confirmed this session via: ruby -e "require 'ripper'; p Ripper.sexp(...)"
require "ripper"
Ripper.sexp("play 60")
#=> [:program, [[:command, [:@ident, "play", [1, 0]], [:args_add_block, [[:@int, "60", [1, 5]]], false]]]]

Ripper.sexp("play 60 (")
#=> nil   # confirmed: invalid syntax returns nil, exactly as EMIT-03 needs to detect
```

### `stop` inside a `live_loop` — verified semantics

```ruby
# Source: /Applications/Sonic Pi.app/Contents/Resources/app/server/ruby/lib/sonicpi/lang/core.rb:1565-1600
# doc string, quoted verbatim:
# "Stops the current thread or if not in a thread, stops the current run.
#  Does not stop any running synths triggered previously in the run/thread
#  or kill any existing sub-threads."
live_loop :foo do
  sample :bd_haus
  sleep 1
  stop   # live loop :foo will now stop and no longer loop
end
```

### Sonic Pi's exact timing-failure error strings (useful for tests/validation)

```
# Source: /Applications/Sonic Pi.app/Contents/Resources/app/server/ruby/lib/sonicpi/lang/core.rb:2348
raise ZeroTimeLoopError, "loop did not sleep or sync!" unless slept or synced

# Source: /Applications/Sonic Pi.app/Contents/Resources/app/server/ruby/lib/sonicpi/lang/core.rb:4699
raise TimingError, "Timing Exception: thread got too far behind time"
```
`[VERIFIED: both greps run directly against the installed app's source this session]`. Useful if Plan 01-03's listening pass wants to assert (by ear or by reading Sonic Pi's log) that the emitted code never triggers either of these.

## Assumptions Log

| # | Claim | Section | Risk if Wrong |
|---|-------|---------|---------------|
| A1 | Exact OSC cue path names (`/phonicpsy-run`, `/phonicpsy-ping`, `/phonicpsy-ack`, `/phonicpsy-stop`) and the ack-listener's UDP port number | Architecture Patterns / Pattern 1, 2 | Low — CONTEXT.md explicitly leaves "the exact OSC paths for push/ping/stop/ack" to the planner's discretion; any internally-consistent naming works, since both ends (`runner.rb` and `osc_client.rb`) are built together in this phase |
| A2 | Ack timeout duration (e.g. "500ms") for D-16's ping-first behavior | Architecture Patterns / Pattern 2 | Low-Medium — too short risks false "runner not responding" on a loaded machine; too long makes Criterion 1's "instant" feel sluggish. Explicitly Claude's Discretion per CONTEXT.md; should be a named constant, easy to tune |
| A3 | Whether Phase 1 embeds the full 157-scale-name list or a curated subset for `--scale` validation | Common Pitfalls — Pitfall A | Medium — a curated subset that's too narrow makes D-11 ("any scale name Sonic Pi itself knows") technically false; the planner should make this an explicit choice, not inherit CONTEXT.md's "~40" premise uncritically |
| A4 | The stop-all mechanism's exact shape (per-loop shared flag vs. registry-of-names re-push) for LOOP-01 | Architecture Patterns / Pattern 1 | Medium — `stop`'s verified per-loop-only semantics mean a naive single-cue implementation will only silence the one generator active when it fires; this needs a deliberate design, flagged here because it's easy to under-scope given the verified-but-narrow `stop` behavior |

**Risk-weighted note:** none of these rise to "needs user confirmation before planning can proceed" — CONTEXT.md's Claude's Discretion section already delegates A1/A2 explicitly, and A3/A4 are scoping calls the planner is equipped to make with the verified data above, not open questions about musical or architectural direction.

## Open Questions

1. **Should the stop-all mechanism (LOOP-01) track loop names in the Ruby CLI process, the runner buffer's Sonic Pi-side state, or both?**
   - What we know: `stop` only halts the loop it's called from (verified above); Phase 1 only ever pushes one generator (`:phonicpsy_arpeggiator`, per D-21's fixed name), so a single well-known loop name may make this simpler than ARCHITECTURE.md's general "registry of every pushed loop" framing assumes.
   - What's unclear: whether to build the general mechanism now (useful for Phase 4's multi-loop layering) or the minimal one-name version now and generalize later, consistent with Pitfall 9's "don't build framework before it's needed."
   - Recommendation: given Phase 1 has exactly one stable loop name (D-21), the minimal version (runner buffer checks one shared flag, one `live_loop :phonicpsy_arpeggiator` name) satisfies LOOP-01 today; defer the general registry to Phase 4 when there are actually multiple concurrent loop names to track.

## Environment Availability

| Dependency | Required By | Available | Version | Fallback |
|------------|------------|-----------|---------|----------|
| Sonic Pi.app | EMIT-05, EMIT-06, the phase's research spike | ✓ | 5.0.0 `[VERIFIED: Info.plist, this session]` | — |
| Ruby | Everything | ✓ | 3.4.4 `[VERIFIED: ruby -v, this session]` | — |
| `osc-ruby` gem | EMIT-05, LOOP-01, D-15/D-16 | ✗ (not installed locally) | 1.1.5 available on rubygems.org, fetch succeeded this session | `gem install osc-ruby` — trivial, no fallback needed, just an install step for Plan 01-02 |
| `pry` gem | Not required by Phase 1's CLI path (D-06 uses `optparse`); useful for spike/debug work | ✓ | 0.16.0 | — |
| `listen` gem | Not required until Phase 2 | ✗ (not installed) | — | Out of scope for Phase 1; no action needed |
| Minitest | Golden-file and determinism tests (Validation, if enabled) | ✓ | 6.0.6 (standalone) + ships with Ruby | — |
| `optparse` | D-06's CLI parsing | ✓ | 0.6.0 (stdlib default) `[VERIFIED: gem list --local, this session]` | — |
| Ripper | EMIT-03 | ✓ | stdlib, behavior confirmed this session | — |

**Missing dependencies with no fallback:** none.
**Missing dependencies with fallback:** `osc-ruby` — one `gem install` away; should be an early, cheap step in Plan 01-02, likely worth a `Gemfile` at repo root even though this is a personal project, so the dependency is declared rather than assumed present.

## Security Domain

> `security_enforcement: true`, `security_asvs_level: 1` per `.planning/config.json`. This is a single-user, local-only tool (PROJECT.md's own Scope constraint) with no network exposure beyond loopback OSC, so most ASVS categories don't apply — documented below rather than silently skipped.

### Applicable ASVS Categories

| ASVS Category | Applies | Standard Control |
|---------------|---------|-----------------|
| V2 Authentication | No | Single local user, no auth boundary exists or is in scope (PROJECT.md: "no deployment target") |
| V3 Session Management | No | No sessions; the OSC "session" is just a resident Sonic Pi buffer |
| V4 Access Control | No | No multi-user or privilege distinction in this tool |
| V5 Input Validation | **Yes** | EMIT-03 (`Ripper.sexp` syntax gate), EMIT-07 (size cap), D-11 (scale-name allowlist) — all already designed in via GEN/EMIT requirements, not an add-on |
| V6 Cryptography | No | No secrets, no auth tokens are used or stored (the phase explicitly avoids the token-gated Sonic Pi channel) |

### Known Threat Patterns for this stack

| Pattern | STRIDE | Standard Mitigation |
|---------|--------|---------------------|
| Arbitrary Ruby execution via the runner buffer's `eval` | Elevation of Privilege (locally, by design — this *is* the feature) | Out of scope to "fix" — the cue server is loopback-only by default (`[VERIFIED: 12.1-Receiving-OSC.md, "For security reasons, by default Sonic Pi does not let remote machines send it OSC messages"]`); the mitigation is **not enabling** Preferences → IO → Networked OSC → "Allow OSC from other computers," which is a Sonic Pi-side setting outside this project's control. Worth a one-line note in `runner.rb`'s header comment. |
| Oversized/malformed OSC payload causing silent truncation or a crash | Denial of Service (self-inflicted, not adversarial) | EMIT-07's size cap (D-20) and EMIT-03's syntax gate already cover this — same controls, security framing added for completeness |
| Path traversal via `--out` (EMIT-08 file export) | Tampering | `--out` is a user-supplied local flag on a single-user tool the user runs against their own filesystem — standard `File.write` behavior is acceptable; no elevated trust boundary is being crossed. Worth a light sanity check (e.g., `File.expand_path` and confirming the parent directory exists) as basic robustness, not a security control per se. |

## Sources

### Primary (HIGH confidence — read directly this session)

- `/Applications/Sonic Pi.app/Contents/Info.plist` — installed version confirmation (5.0.0)
- `/Applications/Sonic Pi.app/Contents/Resources/app/server/ruby/bin/daemon.rb` — default cue port (4560)
- `/Applications/Sonic Pi.app/Contents/Resources/app/server/ruby/bin/spider-server.rb` — default cue port (4560)
- `/Applications/Sonic Pi.app/Contents/Resources/etc/doc/tutorial/12.1-Receiving-OSC.md` — cue-server usage, `/osc` prefix behavior, loopback-only default
- `/Applications/Sonic Pi.app/Contents/Resources/etc/doc/tutorial/12.2-Sending-OSC.md` — `use_osc`/`osc` send-from-Sonic-Pi syntax (used for the ack path)
- `/Applications/Sonic Pi.app/Contents/Resources/etc/doc/tutorial/A.09-randomisation.md` — `use_random_seed` idiom, official worked `live_loop` example
- `/Applications/Sonic Pi.app/Contents/Resources/app/server/ruby/lib/sonicpi/lang/core.rb` — `spread` (line 1758), `stop` (line 1565), `ZeroTimeLoopError`/`TimingError` messages (lines 2348, 4699)
- `/Applications/Sonic Pi.app/Contents/Resources/app/server/ruby/lib/sonicpi/lang/western_theory.rb` — `chord` (line 965)
- `/Applications/Sonic Pi.app/Contents/Resources/app/server/ruby/lib/sonicpi/scale.rb` — full 157-name scale table
- `osc-ruby` 1.1.5 gem, fetched from rubygems.org and unpacked this session — `lib/osc-ruby.rb` (require list), `lib/osc-ruby/server.rb`, `lib/osc-ruby/client.rb`, `lib/osc-ruby/message.rb`, `lib/osc-ruby/osc_packet.rb`
- `ruby -v`, `gem list --local`, `gem fetch` — local environment probing this session

### Secondary (MEDIUM confidence — carried forward from prior phase research)

- `.planning/research/ARCHITECTURE.md`, `.planning/research/STACK.md`, `.planning/research/PITFALLS.md`, `.planning/research/SUMMARY.md` — all dated 2026-09-15, same-session-adjacent; their Sonic Pi source-tree reads targeted the same v5.0.0 tag now confirmed as the actually-installed version
- `abhishekjairath/sonic-pi-mcp` — working reference implementation of the cue-server runner pattern (per STACK.md, not re-read this session)

### Tertiary (LOW confidence)

- None used for this phase-specific addendum; all claims above are either directly verified this session or carried forward from prior HIGH/MEDIUM-confidence research.

## Metadata

**Confidence breakdown:**
- Sonic Pi integration seam (EMIT-05, EMIT-06, LOOP-01): HIGH — verified against the actually-installed binary this session, not just a cloned source tree
- Theory/IR/Emitter architecture: HIGH — carried forward from ARCHITECTURE.md, unchanged and re-confirmed against Sonic Pi's own `spread`/`chord` source
- Scale-name completeness (D-11): HIGH for the count and full list (directly read); MEDIUM for which subset, if any, the planner should curate (a scoping call, not a factual gap)
- osc-ruby's ack/ping mechanics (D-15/D-16): HIGH — gem source read directly, both the naive (`OSC::Server`) and recommended (raw socket + `IO.select`) paths confirmed
- Security domain: HIGH confidence that most ASVS categories don't apply (single-user local tool, PROJECT.md's own stated scope); the loopback-only default is directly verified from Sonic Pi's own docs

**Research date:** 2026-09-15
**Valid until:** Tied to the installed Sonic Pi version (5.0.0) and `osc-ruby` 1.1.5 — revalidate the cue-server seam and gem behavior if either is upgraded before Phase 1 executes. No fixed expiry otherwise; the Ruby stdlib facts (Ripper, UDPSocket) don't go stale.
