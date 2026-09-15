# Architecture Research

**Domain:** Ruby layer that algorithmically generates and drives Sonic Pi (generative/live-coding music tooling)
**Researched:** 2026-09-15
**Confidence:** MEDIUM-HIGH (component decomposition, IR design, and plugin pattern are HIGH confidence — standard patterns applied to a well-understood problem; Sonic Pi OSC/token specifics are MEDIUM — verify against the installed version's `spider.log` early in Phase 1)

## Standard Architecture

### System Overview

```
┌──────────────────────────────────────────────────────────────────────────┐
│                              SESSION / RUNNER                            │
│   owns: master seed · selected generator · current params · mode         │
│         (self-contained | streamed) · transport (file | OSC)             │
└───────────────┬───────────────────────────────────────────┬──────────────┘
                │ instantiate(seed, params)                  │ REPL/CLI knob edits
                ▼                                             │
┌───────────────────────────────┐                             │
│   GENERATOR REGISTRY (plugin) │◀────────────────────────────┘
│   subclasses self-register    │
│   via `inherited`             │
└───────────────┬───────────────┘
                │ .pattern(params) -> IR::Pattern
                ▼
┌───────────────────────────────┐        ┌───────────────────────────────┐
│        MUSIC THEORY CORE      │───────▶│              IR                │
│  Scale, Chord, Interval,      │  used  │  Pattern / Event / Param       │
│  Rhythm grid, voicings        │  by    │  + symbolic Expr nodes:        │
│  (pure data + pure functions) │  gens  │  Choice, RandRange, Weighted,  │
└───────────────────────────────┘        │  TickRing                     │
                                          └───────┬───────────────┬────────┘
                                                  │               │
                              self-contained path │               │ streamed path
                                                  │               ▼
                                                  │      ┌─────────────────┐
                                                  │      │    RESOLVER     │
                                                  │      │ walks IR, calls │
                                                  │      │ Expr#resolve(rng)│
                                                  │      │ seeded Ruby RNG │
                                                  │      └────────┬────────┘
                                                  │               │ concrete IR chunk
                                                  ▼               ▼
                                          ┌───────────────────────────────┐
                                          │       SONIC PI EMITTER        │
                                          │  IR -> Sonic Pi source string │
                                          │  mode: :symbolic | :literal   │
                                          └───────┬───────────────┬────────┘
                                                  │               │
                                    live_loop text│               │small play/pattern chunk
                                                  ▼               ▼
                                          ┌────────────┐   ┌────────────────┐
                                          │ FILE EXPORT│   │  OSC CLIENT     │
                                          │  write .rb │   │ /run-code push, │
                                          │            │   │ token+port      │
                                          │            │   │ discovery,      │
                                          │            │   │ push scheduler  │
                                          └────────────┘   └────────────────┘
                                                  │               │
                                                  ▼               ▼
                                          ┌───────────────────────────────┐
                                          │           SONIC PI            │
                                          │ (external process, unmodified)│
                                          └───────────────────────────────┘
```

The originally proposed decomposition (theory → generators → IR → emitter → transport) is **directionally correct** but incomplete in two ways worth calling out explicitly, because both are load-bearing for the dual-mode requirement:

1. It's missing a **Resolver** as a distinct component. Without it, "streamed mode" gets bolted onto the Emitter as a special case, which is exactly the trap that forces generators to be written twice (see Q3 below).
2. It's missing an explicit **Session/Runner** orchestration layer that owns the seed, the current param set, and the mode. Without this, "twist a knob and hear it" has no natural home — it either leaks into the Emitter (wrong layer) or into each generator (duplicated).

### Component Responsibilities

| Component | Responsibility | Depends on | Knows nothing about |
|-----------|-----------------|------------|----------------------|
| **Music Theory Core** | Scales, chords, intervals, note-name/MIDI conversion, rhythmic grids (e.g. Euclidean patterns), voicing helpers. Pure data + pure functions. | Nothing | Sonic Pi, OSC, IR, randomness |
| **IR** (`Pattern`, `Event`, `Param`, `Expr`) | The structured musical representation. Plain Ruby value objects. `Expr` subclasses (`Choice`, `RandRange`, `Weighted`, `TickRing`) represent *not-yet-resolved* randomness as data. | Nothing (theory only for type shape, not behavior) | Sonic Pi syntax, OSC, generators |
| **Generator (plugin)** | One musical idea. Declares `Param`s, takes a seed, builds an `IR::Pattern` using theory helpers and `Expr` nodes wherever the idea is "pick one of..." rather than "always this." | Theory Core, IR | Sonic Pi syntax, OSC, transport, whether it's running self-contained or streamed |
| **Generator Registry** | Tracks every loaded generator subclass; supports listing/lookup by name. | Generator base class | Everything downstream |
| **Resolver** | Walks an `IR::Pattern`, evaluates every `Expr` node against a seeded `Random` instance, returns a fully concrete `IR::Pattern` (no `Expr` nodes left). Used only by streamed mode. | IR | Sonic Pi syntax, generators, transport |
| **Sonic Pi Emitter** | Renders an `IR::Pattern` to a Sonic Pi source string. Two render modes: `:symbolic` (unresolved `Expr` nodes become literal Sonic Pi randomness calls — `choose`, `rrand`, `ring...tick`) and `:literal` (every field is already a concrete value, so it just prints notes/durations). | IR | Music theory, generators, OSC, files |
| **Transport — File Exporter** | Writes an emitted string to a `.rb` file on disk. Works with no running Sonic Pi. | Emitter output (a string) | IR, generators |
| **Transport — OSC Client** | Discovers Sonic Pi's live port + auth token, sends `/run-code`, and (streamed mode only) runs the push scheduler that decides *when* the next chunk is due. | Emitter output (a string) | IR, generators |
| **Session/Runner** | Orchestrates the above: holds the master seed, the active generator + its current params, the mode (self-contained/streamed), and the transport (file/OSC). This is where "twist a knob" and "re-roll" live. | Everything above | — (this is the top) |

## Recommended Project Structure

```
lib/phonicpsy/
├── theory/
│   ├── scale.rb          # Scale.new(:minor_pentatonic, root: :c4) -> degree lookups
│   ├── chord.rb           # Chord voicings, extensions
│   ├── interval.rb        # note-name <-> MIDI, transposition
│   └── rhythm.rb           # Euclidean/grid pattern generators (returns boolean/weight arrays)
├── ir/
│   ├── event.rb            # note/duration/velocity/synth — fields may hold literals OR Expr
│   ├── pattern.rb           # ordered/nested collection of Events, carries loop metadata
│   ├── param.rb             # Param descriptor: name, type, range/options, default, description
│   └── expr.rb              # Choice, RandRange, Weighted, TickRing — each has #resolve(rng) and
│                             # #to_sonic_pi (symbolic rendering)
├── generators/
│   ├── generator.rb          # base class: `inherited` registry hook, `param` DSL, #pattern contract
│   ├── arpeggiator.rb         # one file per generator, one class per file
│   ├── drum_grid.rb
│   └── ...
├── resolve/
│   └── resolver.rb            # Resolver.resolve(pattern, rng) -> concrete Pattern
├── emit/
│   └── sonic_pi_emitter.rb    # Emitter.render(pattern, mode:, seed:) -> String
├── transport/
│   ├── file_exporter.rb
│   └── osc_client.rb           # token/port discovery, /run-code push, streamed scheduler
└── session.rb                   # Session — the Runner described above

bin/phonicpsy                      # CLI/REPL entry point (knob-twisting UI lives here, thin)
```

### Structure Rationale

- **`theory/` has zero knowledge of Sonic Pi or IR.** It should be usable and testable in a plain `irb` session with nothing else loaded. This is what keeps generator logic honest — musical decisions are made here or in generators, never smuggled into string templates in the emitter.
- **`ir/` is the seam the whole dual-mode design pivots on** — see Q2/Q3 below. It stays deliberately tiny: value objects plus four `Expr` subclasses. Resist the urge to grow it into a general "composition engine"; it only needs to represent what generators actually produce.
- **`generators/` is one-file-per-class by convention**, not enforced by code — this is what makes "adding a generator doesn't touch the core" true in practice, not just in principle.
- **`resolve/` and `emit/` are separate directories even though `resolve/` is small**, because they answer different questions ("what value?" vs "what text?") and keeping them apart is exactly what lets one generator serve both modes.
- **`transport/` never imports `ir/` or `generators/`** — it only ever receives a finished string (plus, for OSC, timing metadata). This means transport can be tested by pointing it at literal strings without any music-generation machinery running.

## The Central Decision: Structured IR vs. Direct Sonic Pi String Emission

**Recommendation: structured IR, not direct string emission. This is not close.**

### The case for direct emission (steelmanned, then rejected)

A generator could just build a Sonic Pi source string directly — heredocs with interpolated values, maybe some string-building helpers. This is the fastest possible path to "I typed something and heard sound," which matters given the project's Core Value is iteration speed. It also sidesteps designing an IR at all. For a single self-contained generator this genuinely is less code.

### Why it loses anyway

- **It cannot serve both emission modes without duplication.** Direct string emission bakes the choice of *where randomness resolves* into the string-building code itself. A generator that writes `"choose([:e4,:g4,:c5])"` into its output has committed to symbolic (self-contained) rendering; making that same idea stream would mean writing a second version of the generator that calls Ruby's `rand` and interpolates a concrete note instead. That's precisely the duplication the project needs to avoid, and it scales linearly with every generator you add.
- **It's untestable as music.** Testing a string-emitting generator means either regex-matching Sonic Pi source (brittle, tests the wrong thing) or launching Sonic Pi and listening (not a unit test). Testing an IR-emitting generator means asserting on Ruby objects: "this pattern has 8 events, the third is a rest, the pitch field is a `Choice` over these five degrees." That's a real, fast, CI-friendly test.
- **It makes "twist a knob" expensive.** With an IR, changing a parameter means rebuilding the `Pattern` (cheap, pure Ruby) and re-rendering (a string join). With direct string emission, "parameters" are just interpolation points in a string template, which works right up until a knob needs to change the *structure* (e.g., "generate 12 events instead of 8," "add a rest 30% of the time") rather than a value — at which point the string-building code degenerates into ad hoc conditionals scattered through a heredoc.
- **The IR cost is small and paid once.** The IR here is not "build a general music representation library" — it's four or five value-object classes. Given four requirements (self-contained, streamed, testable, pluggable) all point the same direction, this is a rare case where the "more architected" answer is also the pragmatic one for a fast-iteration personal tool, not a premature-abstraction trap.

**Where direct string building still belongs:** the Sonic Pi Emitter itself. Once you're turning an `IR::Event` into `play :e4, release: 0.3`, that's string formatting, not music logic, and heredocs/string interpolation are exactly the right tool there. The rule is: **generators never touch a Sonic Pi keyword; the Emitter never makes a musical decision.**

## The Dual-Mode Emission Problem (the hard part)

**Recommendation: one generator, one IR, one Emitter — the only thing that differs between self-contained and streamed mode is whether a Resolver runs before the Emitter.**

The insight that makes this work: self-contained and streamed mode disagree about *when* randomness gets resolved, not about *what the music is*. So push that single disagreement into exactly one seam, and let everything else be shared.

Concretely:

1. A generator's `#pattern(params)` method never calls `rand` directly and never resolves a choice. It builds an `IR::Pattern` where any field that represents "pick one of these" is an `Expr` node (`Choice.new([:e4, :g4, :c5])`, `RandRange.new(0.1, 0.4)`, `Weighted.new(kick: 0.7, rest: 0.3)`), not a resolved value. This is the entire discipline the generator author has to follow — build data, don't roll dice.
2. **Self-contained path:** the `Pattern` (still full of `Expr` nodes) goes straight to the Emitter in `:symbolic` mode. Every `Expr` node knows how to render itself as literal Sonic Pi source (`Choice#to_sonic_pi` → `"choose([:e4, :g4, :c5])"`, `RandRange#to_sonic_pi` → `"rrand(0.1, 0.4)"`). The result is a `live_loop` whose body contains real Sonic Pi randomness calls — Sonic Pi's own RNG drives it forever, no Ruby process required after the single push.
3. **Streamed path:** the same `Pattern` goes to the Resolver first. The Resolver walks it with a seeded `Random` instance and replaces every `Expr` node with a concrete value (`Choice#resolve(rng)` picks one, `RandRange#resolve(rng)` samples). The now-fully-concrete `Pattern` goes to the same Emitter, this time in `:literal` mode (which is really just the default — with no `Expr` nodes left, the Emitter's dispatch on each field falls through to "print the literal value" automatically). This produces a small, non-randomized chunk of Sonic Pi code — a bar or phrase of already-decided notes — which gets pushed over OSC. The Runner calls `generator.pattern(params) → Resolver.resolve → Emitter.render → OscClient.push` again before the previous chunk finishes playing, advancing the same `Random` instance each time so the stream continues rather than restarting.

This means a generator author writes the musical idea exactly once. Whether it ends up as a self-contained `live_loop` or a stream of pushed chunks is a decision made by the *Session*, at the moment of running it, not by the generator. A generator that only ever produces literal values (no `Expr` nodes at all) is legal too — the Emitter just renders it the same way in both modes — but a generator that wants Sonic Pi's own randomness to carry the self-contained version forever *and* wants Ruby to drive the streamed version needs to express that randomness as `Expr` nodes, not as calls.

**One deliberate asymmetry to keep:** self-contained rendering emits a *loop* (Sonic Pi owns the repetition, via `live_loop`), while streamed rendering emits a *chunk* (Ruby owns the repetition, by calling the pipeline again). The IR's `Pattern` should therefore carry a small amount of loop metadata (e.g., "this pattern repeats indefinitely" vs "this pattern is one bar") that the Emitter uses to decide whether to wrap output in `live_loop do ... end` or leave it as a bare block — this is a rendering-mode detail, not something that forks the generator or the IR shape.

**Scope discipline for Phase 1:** not every generator needs `Expr` nodes on day one. It is completely fine, and recommended, for the first generator(s) to be pattern-of-literals-only, self-contained-only, hand-tuned to sound good, while the `Expr`/Resolver machinery is built in a later phase. The IR shape supports this from the start (an `Expr` field is just an optional case of "field value"), so nothing gets thrown away when streamed mode is added — see Suggested Build Order.

## Plugin/Module Registration

**Recommendation: `inherited`-hook self-registration, plus a directory convention loaded by a plain `Dir.glob` + `require` at boot. No Zeitwerk, no explicit registration file, no gem-based plugin framework.**

This is the idiomatic low-ceremony Ruby pattern for exactly this shape of problem (a handful to a few dozen leaf subclasses, single process, no need for lazy loading or namespacing across gems). Rails-style autoloading (Zeitwerk) solves problems — reloading, huge codebases, naming convention enforcement across a framework — that don't exist here and would add a dependency and a constant-resolution mental model for no benefit.

Shape:

```ruby
# lib/phonicpsy/generators/generator.rb
module Phonicpsy
  class Generator
    class << self
      def registry
        @registry ||= []
      end

      def inherited(subclass)
        super
        Generator.registry << subclass
      end

      def param(name, type:, default:, range: nil, options: nil, desc: nil)
        params << IR::Param.new(name:, type:, default:, range:, options:, desc:)
      end

      def params
        @params ||= []
      end

      def generator_name
        name.split('::').last
             .gsub(/([a-z\d])([A-Z])/, '\1_\2').downcase
      end
    end

    attr_reader :seed, :param_values

    def initialize(seed:, **overrides)
      @seed = seed
      @param_values = self.class.params.to_h { |p| [p.name, overrides.fetch(p.name, p.default)] }
    end

    # Every subclass implements this. Returns an IR::Pattern.
    def pattern
      raise NotImplementedError
    end
  end
end
```

```ruby
# lib/phonicpsy/generators/arpeggiator.rb
module Phonicpsy
  class Arpeggiator < Generator
    param :root,     type: :note,   default: :e3
    param :scale,    type: :symbol, default: :minor_pentatonic
    param :density,  type: :float,  default: 0.6, range: 0.0..1.0
    param :note_pool_size, type: :int, default: 5, range: 3..8

    def pattern
      degrees = Theory::Scale.new(param_values[:scale], root: param_values[:root])
                              .degrees(param_values[:note_pool_size])
      IR::Pattern.loop(steps: 8) do |i|
        IR::Event.new(
          pitch: IR::Expr::Choice.new(degrees),
          gate:  IR::Expr::Weighted.new(true => param_values[:density], false => 1 - param_values[:density])
        )
      end
    end
  end
end
```

```ruby
# lib/phonicpsy/generators.rb — loaded once at boot
Dir.glob(File.join(__dir__, 'generators', '*.rb')).sort.each { |f| require f }
```

`Generator.registry` now holds every loaded subclass; a CLI/REPL lists them by `generator_name`. Adding a new generator is: drop a file in `generators/`, subclass `Generator`, declare params, implement `#pattern`. Nothing else changes — the registry, the CLI listing, and the param introspection all update automatically because they read off the class, not off a hand-maintained list.

## Parameter/Knob Model

**Recommendation: a class-level `param` DSL (shown above) producing `IR::Param` descriptors, with values resolved at instantiation time and randomization implemented as "sample from the descriptor's declared domain using the seeded RNG," never as ad hoc `rand` calls scattered in generator code.**

`IR::Param` is a plain descriptor:

```ruby
Param = Struct.new(:name, :type, :default, :range, :options, :desc, keyword_init: true) do
  def sample(rng)
    case type
    when :float, :int then rng.rand(range)
    when :symbol       then options.sample(random: rng)
    when :note          then options ? options.sample(random: rng) : rng.rand(range)
    end
  end
end
```

This gives three operations for free, all driven off the same descriptor:

- **Introspect:** `SomeGenerator.params` → list for a CLI/REPL to render as sliders/choices, with types and legal domains, no code editing required.
- **Randomize:** `SomeGenerator.params.to_h { |p| [p.name, p.sample(rng)] }` — produces a full random param set from one seeded RNG call per param, in declaration order, so it's reproducible.
- **Re-roll one knob:** re-sample a single param's `sample(rng)` while leaving the rest of `param_values` untouched — this is what "twist a knob" is, mechanically.

**Seed composition with params:** `Generator.new(seed:, **param_overrides)` takes the *base* seed and the *resolved* param values as independent inputs — params are not derived from the seed, they're given (defaults, user overrides, or a randomize step that itself consumed the seed). This means "same seed + same params = same output" holds structurally: the seed only ever drives (a) the one-time randomize step, if used, and (b) whatever `Expr` resolution happens downstream (self-contained: baked into `use_random_seed`; streamed: the Resolver's `Random.new(seed)`). Nothing about the pipeline consumes entropy outside of those two places, so there is no hidden nondeterminism to reconcile.

## Determinism Architecture

**The two RNGs (Ruby's and Sonic Pi's) are never made to agree with each other, and should not be.** They're different algorithms consuming different numbers of draws in different orders — attempting to make "seed 42 in Ruby" and "seed 42 in Sonic Pi" produce correlated output is both practically unachievable and pointless, since the two streams are never compared to each other, only replayed against themselves.

**Recommendation: one master seed per session, scoped strictly per engine, with an explicit emission convention for where `use_random_seed` goes.**

- The Session holds a single integer **master seed**. It feeds two independent consumers:
  1. A `Random.new(master_seed)` instance in Ruby, used for anything the Ruby layer resolves: param randomization and, in streamed mode, the Resolver's `Expr#resolve` calls.
  2. The literal integer passed to Sonic Pi's `use_random_seed master_seed` in emitted self-contained code, governing Sonic Pi's own RNG for every symbolic `choose`/`rrand`/`ring` call left in the emitted text.
- **"Same seed, same output" is a per-mode guarantee, not a cross-mode one.** Replaying self-contained mode with the same seed + params + generator code always emits byte-identical Sonic Pi source (the Emitter is a pure function of the IR, and the IR-building generator is a pure function of seed+params). Replaying streamed mode with the same seed + params + generator code always resolves the same sequence of concrete chunks, because `Random.new(seed)` is itself deterministic and the Resolver walks the IR in a fixed order. Nothing requires (or should attempt to make) the self-contained run and the streamed run of the *same* generator sound identical to each other — they're two different consumers of the same musical idea, correctly free to diverge.
- **Emission convention for `use_random_seed` placement:** put it as the **first line inside the `live_loop do...end` body**, not once outside the loop. Sonic Pi's own documentation pattern for live-tweakable, reproducible loops resets the seed every iteration this way — each pass through the loop restarts from the same point in the stream. This matters specifically for this project because it means editing a generator and re-pushing its output mid-performance doesn't shift *future* iterations unpredictably relative to past ones; the seed's meaning stays stable under live editing, which is the same property the project wants from the Ruby side.
- **What a "reproducibility unit" actually is:** (master seed, resolved param values, generator class + its source). The first two are handled by the architecture above; the third is a lighter-weight concern — if bit-for-bit reproducibility across time (not just within a session) matters later, tag exported files with a content hash of the generator source, but this is a minor, deferrable concern for a single-user playground, not something to build machinery for up front.

## Suggested Build Order (coarse phases)

Framed as 4 substantial, dependency-ordered chunks. **Phase 1 must produce audible sound** — it deliberately defers the hardest architecture (structured `Expr`/Resolver dual-mode machinery) so that the first working slice is "hear a generative loop," not "IR is done."

### Phase 1 — Sound, now
**Goal: a single hand-built generator, self-contained mode only, both transports.**
- Music Theory Core: just enough to be useful (scale + note-name lookups, one or two rhythm helpers).
- A minimal `IR::Event`/`IR::Pattern` (literal fields only — no `Expr` nodes yet; that abstraction isn't needed until Phase 3).
- One Sonic Pi Emitter that renders a `Pattern` into a `live_loop` block, with Ruby-computed literal values (scale notes, etc.) interpolated as arrays and Sonic Pi's own `choose`/`rrand` written directly into the loop body template for the "feel" — no need to route this through the IR as `Expr` nodes yet; the Emitter can hardcode the randomness syntax for this first generator.
- File export (write `.rb`, open manually in Sonic Pi — this is the fastest possible path to sound and has zero dependency on OSC working).
- A minimal OSC client: discover Sonic Pi's port/token (parse from its log — verify the exact mechanism against the installed Sonic Pi version, since token auth changed around v4), single `/run-code` push of the whole `live_loop` text.
- **Exit criterion: run one command, hear a generative loop, both via exported file opened in Sonic Pi and via a direct OSC push.**

### Phase 2 — Structure & plugins
**Goal: formalize what Phase 1 proved out, so adding generator #2 through #N is cheap.**
- Full `IR::Expr` hierarchy (`Choice`, `RandRange`, `Weighted`, `TickRing`) with `#to_sonic_pi` (symbolic rendering).
- Refactor the Phase 1 generator to build its randomness as `Expr` nodes instead of hardcoded template text; Emitter's symbolic-mode rendering replaces the hand-written randomness syntax.
- `Generator` base class with `inherited` registry + `param` DSL; port Phase 1's generator onto it; add 2-3 more generators to prove the plugin story.
- Parameter randomize/re-roll operations, wired into a minimal CLI (even just an `irb`-driven session is fine here — the REPL polish is Phase 4).
- **Exit criterion: adding a new generator is "create a file, subclass, declare params, implement `#pattern`" — nothing in `session.rb`, the CLI, or the Emitter needs to change.**

### Phase 3 — Streamed mode
**Goal: prove the dual-mode design by running an existing (Phase 2) generator, unmodified, in streamed mode.**
- Resolver (`Expr#resolve(rng)` walk).
- Emitter's literal-mode path (likely already falls out of Phase 2's dispatch logic — confirm/harden it).
- OSC push scheduler: decide chunk size (a bar/phrase) and push timing so playback never gaps waiting on Ruby.
- Session-level mode switch (self-contained vs streamed) for a given generator + seed + params.
- **Exit criterion: pick any Phase 2 generator, run it streamed instead of self-contained, hear continuous playback, with zero changes to that generator's code.**

### Phase 4 — Ergonomics & extension
**Goal: the fast feedback loop the project's Core Value is about, plus breadth.**
- Hot-reload: file-watch generator source, re-instantiate, re-push on save (self-contained: re-push whole loop; streamed: next chunk picks up the change).
- REPL/CLI knob-twisting UI (list params, mutate live, re-roll, switch seed).
- Expand the generator library (this is where "16-bit JRPG loop," "noise wall," etc. get built — cheap now because the plugin system is proven).
- Optional LLM prose → params front-end (strictly parameter-set output, never code — per the project's explicit out-of-scope boundary on LLM-written Sonic Pi).
- **Exit criterion: editing a generator's `#pattern` and hearing the change is sub-few-seconds, with no manual restart of the Sonic Pi connection.**

**Dependency logic for this ordering:** IR must exist before generators can be plugin-ized against it (Phase 2 before real plugin architecture); the plugin/param model must exist before the Resolver has multiple real generators worth proving dual-mode against (Phase 2 before Phase 3); streamed mode's OSC scheduler is a strict superset of Phase 1's single-push OSC client, so it reuses rather than replaces that code. Ergonomics (Phase 4) depends on everything else existing because hot-reload and knob-twisting are UI over the pipeline, not new pipeline stages.

## Anti-Patterns

### Anti-Pattern 1: Generators that build Sonic Pi strings directly

**What people do:** Write the musical idea as a Ruby method that heredocs/interpolates Sonic Pi source directly, skipping the IR "because it's simpler for now."
**Why it's wrong:** Forces every generator to be rewritten (not refactored — rewritten) the day streamed mode needs it, defeats testability, and makes "twist a knob" require touching string-template code instead of data.
**Do this instead:** Generators return `IR::Pattern`. String-building lives only in the Emitter.

### Anti-Pattern 2: Resolving randomness inside the generator

**What people do:** Generator calls `rand` or `Array#sample` directly while building its pattern, "just to make streamed mode easy."
**Why it's wrong:** This silently forecloses self-contained mode for that generator — there's no way to un-resolve a value back into "Sonic Pi should pick this live, forever." It also breaks reproducibility if the generator is constructed more than once per session (e.g., re-rolled) without care.
**Do this instead:** Generator emits `Expr` nodes for anything that should vary; resolution is the Resolver's job, invoked (or not) by the Session depending on mode.

### Anti-Pattern 3: Framework-weight autoloading for a dozen files

**What people do:** Reach for Zeitwerk, Rails' `eager_load`, or a gem-based plugin framework for generator discovery.
**Why it's wrong:** Solves problems (hot constant reloading across a large multi-gem codebase, enforced naming conventions) this project doesn't have, and adds a dependency plus a new mental model to a project whose stated value is a short iteration loop.
**Do this instead:** `Dir.glob(...).sort.each { |f| require f }` at boot, `inherited` hook for registration.

### Anti-Pattern 4: Trying to make Ruby's RNG and Sonic Pi's RNG produce the "same" randomness

**What people do:** Attempt to seed both engines such that the same seed yields correlated or identical musical output across self-contained and streamed runs.
**Why it's wrong:** The two engines never need to be compared to each other — only each to its own past runs — so this is wasted design effort chasing a guarantee nobody needs.
**Do this instead:** Scope determinism per-engine (see Determinism Architecture above); document plainly that "same seed" means "reproducible within a mode," not "identical across modes."

### Anti-Pattern 5: One global `live_loop` per session instead of composable parts

**What people do:** Build the whole piece as a single monolithic self-contained loop, because it's easier than managing several.
**Why it's wrong:** Directly conflicts with the stated requirement that generators produce "loops, layerable single parts (bass/drums/lead), and endless generative pieces" — a monolith can't be layered or independently re-rolled.
**Do this instead:** Each `Generator` instance produces one `Pattern`/one `live_loop` (or one streamed voice); layering is the Session running multiple generator instances concurrently, each with its own OSC push (Sonic Pi's `live_loop :name` namespacing already supports this — give each generator instance a unique loop name derived from its class + an instance id).

## Integration Points

### External Services / Processes

| Integration | Pattern | Notes |
|---|---|---|
| Sonic Pi (running instance) | OSC `/run-code` to the port Sonic Pi listens on, with a security token | Sonic Pi v4+ requires a token in addition to the port, both discoverable by parsing Sonic Pi's own log (`spider.log`, path OS-dependent) on startup — the widely-referenced `sonic-pi-cli`/`sonic-pi-tool` Ruby gem predates this token requirement (last compatible with 2.7–3.2) and should not be relied on as-is; verify the discovery mechanism against the installed Sonic Pi version early in Phase 1 and implement a thin OSC client directly (an `osc-ruby`/`ruby-osc` gem, or raw UDP + minimal OSC packet encoding, is enough — no need for a full framework) |
| Sonic Pi (file loading) | Write a `.rb` file to disk; user opens/runs it in the Sonic Pi editor | No API surface needed — this path has no version-compatibility risk and should be the first transport built (Phase 1) |
| Sonic Pi receiving OSC | Sonic Pi can also *receive* arbitrary OSC as cues inside a `live_loop` (`sync "/osc*/trigger"` style) | Not needed for the core push model above, but worth knowing as an escape hatch if a future generator wants Sonic Pi-side reactive behavior instead of Ruby pushing fresh code each time |

### Internal Boundaries

| Boundary | Communication | Notes |
|---|---|---|
| Theory ↔ Generator | Direct method calls, plain Ruby values | Theory is a library, not a service; no boundary ceremony needed |
| Generator ↔ IR | Generator constructs IR value objects | IR classes should have no behavior beyond `#resolve`/`#to_sonic_pi` on `Expr` — keep them dumb |
| Generator ↔ Emitter | **Never direct** — always via IR | This is the boundary the whole dual-mode design protects; a generator must never import or reference the Emitter |
| Resolver ↔ IR | Resolver reads `Expr` nodes, returns a new `Pattern` with concrete values | Should not mutate the input `Pattern` in place — return a new one, so the original (still-symbolic) `Pattern` stays available for a self-contained render if the Session ever switches modes mid-session |
| Emitter ↔ Transport | Emitter returns a `String`; transport never inspects it beyond passing it along | Keeps transport testable without any music-generation code loaded |
| Session ↔ everything | Session is the only component allowed to know about *all* the others | Everything else should be importable and testable in isolation |

## Sources

- [Sonic Pi tutorial — Receiving OSC](https://github.com/sonic-pi-net/sonic-pi/blob/dev/etc/doc/tutorial/12.1-Receiving-OSC.md) — MEDIUM confidence (official docs, dev branch)
- [Sonic Pi tutorial — Sending OSC](https://github.com/sonic-pi-net/sonic-pi/blob/dev/etc/doc/tutorial/12.2-Sending-OSC.md) — MEDIUM confidence (official docs, dev branch)
- [Sonic Pi tutorial — Randomisation (A.09)](https://github.com/sonic-pi-net/sonic-pi/blob/stable/etc/doc/tutorial/A.09-randomisation.md) — HIGH confidence (official docs, stable branch); source for the `use_random_seed`-inside-`live_loop` reproducibility idiom
- [GitHub issue — Clarifying usage of OSC /run-code in latest versions (4+)](https://github.com/sonic-pi-net/sonic-pi/issues/3330) — MEDIUM confidence (community/maintainer discussion, confirms token requirement changed in v4); verify directly against installed version before building the OSC client
- [Revisiting Calling Sonic Pi From Ruby — Kevin Murphy](https://kevinjmurphy.com/posts/revisiting-calling-sonic-pi-from-ruby/) — MEDIUM confidence (practitioner blog, corroborates `sonic-pi-cli` gem history and OSC approach)
- [sonic-pi-cli gem (Nick Johnstone), RubyGems](https://bundler.rubygems.org/gems/sonic-pi-cli/versions/0.0.4) — MEDIUM confidence; version history confirms it targets Sonic Pi 2.7–3.2, predating v4 token auth
- [The Inherited Hook Method in Ruby — AppSignal Blog](https://blog.appsignal.com/2019/09/03/inherited-hook-method-and-parenting.html) — HIGH confidence (well-established idiomatic Ruby pattern, corroborated across multiple independent sources found)
- [Refactoring Ruby: From Subclass to Registry — The Miners/Medium](https://medium.com/codeminer-42/refactoring-ruby-from-subclass-to-registry-b6fe4db58b18) — HIGH confidence, corroborating source for the registry pattern

---
*Architecture research for: Ruby-based Sonic Pi generative music driver*
*Researched: 2026-09-15*
