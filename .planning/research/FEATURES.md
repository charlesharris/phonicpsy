# Feature Research

**Domain:** Algorithmic/generative music tooling — a Ruby generator layer driving Sonic Pi
**Researched:** 2026-09-15
**Confidence:** MEDIUM-HIGH (broad ecosystem coverage via web search; no library-version-pinned docs lookups needed since this is a survey of concepts, not API integration)

## Landscape Surveyed

| Tool | Language/Model | What it's for | Relevance here |
|------|-----------------|----------------|-----------------|
| **Sonic Pi** (built-ins) | Ruby DSL | The runtime this project targets | Source of "already free" features |
| **TidalCycles** | Haskell, pattern-algebra DSL | Pattern manipulation for live coding | Pattern transformation vocabulary (`euclid`, randomness operators, mini-notation) |
| **Strudel** | JavaScript port of Tidal | Browser live coding | Confirms Tidal's pattern model is portable/valuable outside Haskell; same feature set, different host language |
| **Orca / ORCΛ** | Esoteric 2D grid language | Visual procedural sequencing, sends MIDI/OSC | Different UI paradigm (grid, not code) — mostly informs anti-features (don't build a grid IDE) |
| **FoxDot** | Python over SuperCollider | Player-object pattern live coding | Pattern algebra (`PRange`, `PDur`, `.every`, `.shuffle`) — a Python analog to what a Ruby layer could offer |
| **Overtone** | Clojure over SuperCollider | Programmable/collaborative music, has its own scale/chord/rhythm library + metronome | Closest architectural cousin: a host-language layer with music-theory helpers, feeding a synthesis engine — validates the "generator layer with a vocabulary of scales/chords/rhythms" shape |
| **Gibber** | JavaScript, browser | Audiovisual live coding | Confirms sample-accurate scheduling + minimal-ceremony editing is the standard UX bar |
| **Euterpea** | Haskell DSL | Music research/composition library, note-level + signal-level | Confirms Euclidean rhythms and most "generative" primitives are typically bolted on, not core — nobody ships this as a monolith |
| **Sonic Pi generator projects (GitHub)** | Ruby, various | E.g. "Figurate Numbers" (infinite integer-sequence generators via Enumerator), song-collection repos | Confirms the ecosystem convention: small composable Ruby modules that emit values/patterns consumed by Sonic Pi's own DSL, not full alternate runtimes |

**Recurring pattern across every tool surveyed:** none of them try to replace the host synthesis/sequencing engine. Each adds a *generation and transformation vocabulary* (scales, patterns, randomness, Euclidean rhythms, weighted choice) on top of an engine that already handles timing, playback, and sound. This validates the PROJECT.md architecture directly: phonicpsy should be a thin generation layer, not a competing runtime.

## Feature Landscape

### Table Stakes (Users Expect These)

Without these, the tool is not "generative music tooling," it's just scripts.

| Feature | Why Expected | Complexity | Sonic Pi already does this? |
|---------|--------------|------------|------------------------------|
| Seeded, deterministic generation (same seed → same output) | Every tool surveyed treats this as foundational — Sonic Pi's own `use_random_seed` makes randomness deterministic by default; TidalCycles/Strudel randomness is seedable too. Without this, "twist a knob and hear it again" is impossible and A/B comparison is meaningless. | LOW | **Partially** — Sonic Pi's `use_random_seed` + its RNG stream gives deterministic *Sonic Pi-side* randomness for free. But the Ruby generator layer needs its *own* seeded RNG (e.g. Ruby's `Random.new(seed)`) for decisions made before code is ever emitted (which scale, which rhythm grammar, which motif). Two independent seed spaces to manage — this is real generator-layer work, not free. |
| Scale/chord/interval vocabulary | Every algorithmic tool (Overtone, Euterpea, Sonic Pi itself) ships one; it's the minimum unit of "not just noise." | LOW | **Yes, fully free** — `scale`, `chord`, `ring`, note-name parsing are core Sonic Pi built-ins with dozens of scale/chord types already implemented and tested. The Ruby layer should *choose* scale/chord degrees and *emit* `scale(...)`/`chord(...)` calls — never reimplement music theory tables. |
| Rhythm/timing generation (some way to produce a sequence of durations/gaps) | Table stakes across Tidal, FoxDot, Orca, Overtone — a generator with no rhythmic variation is a chord player, not a sequencer. | LOW-MEDIUM | **Partially** — Sonic Pi has `sleep`, `spread` (Euclidean bool ring), and pattern-through-`ring`/`Enumerator` idioms. The *choice* of rhythm (which Euclidean params, which grammar) is generator-layer logic; the *execution* (`sleep`, `live_loop` timing) is free. |
| Code emission that is syntactically valid, runnable Sonic Pi Ruby | Stated directly in PROJECT.md as a requirement; if emitted code doesn't run, nothing else matters. | MEDIUM | No — this is inherently the generator layer's job. Complexity comes from string-templating Ruby safely (quoting, indentation, nested blocks) rather than from anything conceptually hard. Template-based emission (ERB or plain string builders) keeps this tractable. |
| A way to actually hear the result (file export and/or live push) | Already a stated requirement (`.rb` export + OSC push via `sonic-pi-tool`-style client). Every surveyed tool's core value prop is "instant audible feedback" — Gibber and Strudel both lead with zero-latency audition. | LOW-MEDIUM | **Yes for playback** — Sonic Pi does all sound-making. The layer only needs to get text into Sonic Pi: either write a `.rb` file the user opens, or send it via OSC (`run-file`/`eval-stdin`-style, per `sonic-pi-tool`/`sonic-pi-cli` prior art). Both are well-trodden integration paths — not novel engineering. |
| Live loop wrapping (self-contained generative code that runs forever unattended) | Explicit requirement. Sonic Pi's `live_loop` is the standard idiom for this across the whole ecosystem's Sonic Pi-specific projects. | LOW | **Yes, fully free** — `live_loop` with internal randomization is a native Sonic Pi construct; the generator layer's job is only to emit a well-formed `live_loop do ... end` block with its own seeded random calls inside. |
| Parameter/knob exposure (values that can be re-rolled or tweaked without hand-editing code) | Explicit requirement, and the entire practical difference between "a script" and "a generator with a UI surface," per FoxDot's Player-object kwargs and Overtone's function-argument model. | LOW-MEDIUM | No — this is purely a Ruby-layer concern (a generator's public interface: named params with ranges/choices). Nothing in Sonic Pi provides this; Sonic Pi doesn't know about "generators" as a concept at all. |
| Pluggability (adding a generator doesn't touch the core) | Explicit requirement; also how every multi-generator ecosystem project on GitHub is structured (small composable modules/mixins, not one giant file). | LOW-MEDIUM | No — architectural concern for the Ruby layer (a `Generator` base class/interface, a registry). Standard Ruby patterns apply (duck typing, module registration); no novel research needed here. |

**Dependency note:** code emission depends on having *something* to emit (scale/rhythm decisions), which depends on the seeded RNG existing first. Practical build order: seeded RNG → music-theory vocabulary (thin wrapper/pass-through to Sonic Pi's own names) → rhythm generation → code emission → file export → OSC push → `live_loop` wrapping → parameter/knob interface → plugin registry. Knob exposure and plugin registry are somewhat independent of each other and could be built in either order.

### Differentiators (What Makes It Fun, Not Just Functional)

These aren't required for the tool to work, but they're what separates "a code generator" from "a plaything you keep coming back to." Prioritize ones that shorten the iteration loop (Core Value) over ones that just add musical sophistication.

| Feature | Value Proposition | Complexity | Sonic Pi already does this? |
|---------|--------------------|------------|------------------------------|
| Euclidean rhythm generation (Bjorklund's algorithm: distribute *k* hits over *n* steps as evenly as possible) | The single most bang-for-buck rhythm technique in the entire surveyed landscape — TidalCycles' `euclid`/`euclidFull` and Sonic Pi's own `spread` are both direct implementations of the same 300 BC algorithm; instantly produces musically-plausible, non-random-sounding rhythms from two integers. | LOW | **Yes — `spread` in Sonic Pi already implements Bjorklund's algorithm** (Euclidean-distributed ring of bools). The generator layer's job is just to *choose* the (k, n) pairs (possibly seeded/weighted) and emit `spread(k, n)`, not to reimplement the algorithm. This is the clearest "emit, don't reimplement" case in the whole survey. |
| Weighted random choice / Markov chains for melodic and rhythmic sequencing | Research literature (Markov chain composition, n-th order chains) treats this as the standard technique for "surprising but coherent" sequences — a step up from uniform `choose`/`rrand`. Directly serves the "generative surprise" goal called out in PROJECT.md context. | MEDIUM | No — Sonic Pi gives `choose`/`dice`/`one_in`/`shuffle` (uniform or simple-probability choice only). Weighted choice and Markov transition tables are generator-layer logic; a small weighted-sampler and an optional transition-matrix structure are the concrete deliverables. Order-1 Markov chains (state = last note/duration) are a reasonable ceiling for "fun" without becoming a research project — order-2+ has diminishing returns for a personal tool. |
| Generative grammars (L-systems / production-rule expansion for motifs and phrases) | Produces structured-but-surprising output beyond what pure randomness gives; a natural fit for "endless generative pieces" since grammars can self-similarly expand indefinitely. | MEDIUM-HIGH | No — not present in Sonic Pi or in any surveyed tool as a first-class primitive (closest analog: FoxDot's pattern *functions*, which compose but don't rewrite). This is the most novel/riskiest differentiator — good candidate for "v2, after the simpler generators prove the harness works." |
| Parameter interpolation/morphing between two seeds or two parameter sets | None of the surveyed *live-coding* tools do this natively (they're built for discrete re-evaluation, not continuous morphing) — this would be a genuine differentiator specific to a personal tool that isn't optimized for performance-under-pressure. Enables "slowly drift from vibe A to vibe B" which pure re-seeding can't. | MEDIUM | No — entirely a Ruby-layer concern: lerp/crossfade across generator parameter values (tempo, density, scale-degree weights) across N regenerations, or crossfading between two `live_loop`s. Worth sequencing after knob-twisting (table stakes) is solid, since morphing is just "knob-twisting between two named points" implemented on top of the same parameter interface. |
| A/B re-rolling (regenerate just one part/seed, hear it against the previous take, revert instantly) | Directly serves the Core Value ("twist a knob... hear the result in seconds"). This is a UX/workflow feature more than a musical-algorithm feature — the "differentiator" isn't the music theory, it's the interaction loop. | LOW-MEDIUM | No — needs history of the last N seeds/parameter-sets per generator and a quick "regenerate" vs. "revert" action. Cheap to build (an array of past parameter snapshots) and disproportionately high value for the stated Core Value — should be prioritized early despite being a "differentiator" not "table stakes." |
| "Favorites" capture (bookmark a seed + parameter set that sounded good, name it, recall it later) | Solves the well-known generative-music problem: you stumble into something great, then lose it three re-rolls later. Every serious generative-music workflow (per Eno-style systems research and practitioner writeups) eventually needs a way to freeze a good result. | LOW | No — simple persistence: serialize `{generator_name, seed, params}` to a file (YAML/JSON), reload by name. Trivially cheap relative to its value; strongly recommend for v1 once re-rolling exists, since favorites without re-rolling has nothing to bookmark. |
| Layerable part composition (bass/drums/lead generated independently, combined with shared tempo/key context) | Explicit requirement. Overtone's shared-metronome model and Sonic Pi's own multiple-`live_loop`-with-shared-`use_bpm`/cue pattern are the established mechanism — this is well-trodden, not novel. | MEDIUM | **Partially** — Sonic Pi gives synchronization primitives for free (`live_loop`, `sync`, `cue`, shared `use_bpm`). The generator layer's job is to make each part-generator aware of a shared *context* object (key, scale, tempo, seed-per-part-but-derived-from-a-master-seed) so parts sound like they belong together rather than coincidentally overlapping. Master-seed-derives-per-part-seeds is the concrete technique (e.g. `Random.new(seed).hash_with(:bass)` style derivation) — cheap once the seeding table-stakes exists. |
| Live parameter nudging while playing (change a knob value and have the *currently running* `live_loop` pick it up next cycle, without a full restart) | This is exactly what `live_loop`'s hot-swap semantics already give for free at the Sonic Pi level — Sonic Pi's whole design premise is that redefining a `live_loop` mid-play doesn't stop it. | LOW (given OSC push exists) | **Yes, almost entirely free** — as long as the generator emits a `live_loop` and the harness re-pushes edited code over OSC, Sonic Pi's own hot-reload does the "nudging" work. The only generator-layer piece is making sure a parameter change regenerates *just that loop's code*, not a full-program re-emit that would glitch other loops. |
| Endless generative piece patterns (long-duration, slowly-evolving unattended pieces — Eno-style incommensurable loop lengths, slowly-drifting probabilities) | Explicit requirement category. Eno's own technique — loops of deliberately non-coincident lengths so the combined pattern rarely exactly repeats — is directly implementable as "pick loop lengths that are coprime/incommensurate" and is cheap relative to its payoff. | MEDIUM | No — Sonic Pi provides the infinite-loop *mechanism* (`live_loop` running forever) for free; the "doesn't obviously repeat" *quality* is a generator-layer sequencing decision (incommensurate lengths, slow parameter drift via `tick`/counters, occasional low-probability mutation). This is a technique to document/encode as a generator pattern, not a subsystem to build. |
| Numeric/mathematical sequence generators (figurate numbers, integer sequences, cellular automata → pitch/rhythm mapping) | Precedent exists directly in the Sonic Pi Ruby ecosystem (the "Figurate Numbers" GitHub project maps infinite integer sequences to Sonic Pi via `Enumerator`), and fits the "weirdest version of a vibe" flavor floated in PROJECT.md. | LOW-MEDIUM | No — pure Ruby `Enumerator`/lazy-sequence logic, mapped to scale degrees or durations before emission. Fun, cheap, and idiomatically very "Ruby" — a good example generator to build early to prove the plugin architecture, not because it's foundational. |

### Anti-Features (Commonly Attractive, Actually Harmful Here)

Beyond PROJECT.md's existing exclusions (song forms, LLM-written code, multi-user, DSP, GUI), these are domain-specific traps that look appealing in generative-music tooling but reliably kill iteration speed or scope-creep a personal project into an unfinished framework.

| Feature | Why It's Tempting | Why It's Actually Harmful Here | Alternative |
|---------|--------------------|-----------------------------------|-------------|
| A visual/grid sequencer UI (Orca-style) | Orca and similar tools look immediately appealing — "just draw the pattern." | Building and maintaining a custom UI directly contradicts PROJECT.md's "no polished GUI" constraint and is a multi-week subsystem (grid rendering, editing, input handling) for a single-person tool where the code *is* the interface. Also duplicates what the Sonic Pi editor buffer already provides as a text surface. | Keep the interface as Ruby method calls / a minimal REPL or CLI; if a visual element is ever wanted, a read-only visualization of what's about to play (not an editable grid) is far cheaper. |
| Training a Markov model or ML model from a corpus of existing songs | "Feed it real music and it'll sound more human" is the most common ask in every generative-music adjacent survey and forum thread. | Corpus collection, training pipeline, and model management are a different (much larger) project than "seeded, hand-authored generators." It also reintroduces non-determinism/opacity that conflicts with the "deterministic, seedable" table stakes — a trained model's output isn't auditable/tunable the way a hand-built weighted-choice table is. | Hand-author weighted-choice tables and small order-1 Markov transition matrices directly from music theory (this is literally what Overtone and most hobbyist tools do) — tunable, inspectable, and reproducible without a training step. |
| A general-purpose pattern-algebra DSL (a la TidalCycles' mini-notation: writing a whole new parser/interpreter for pattern strings) | TidalCycles/Strudel's expressive power is genuinely appealing, and "just build a mini pattern language in Ruby" seems like a natural next step once you have generators. | This is a multi-year research project (Tidal itself represents years of Haskell type-theory work on `Pattern a = Arc -> [Event a]`); a hand-rolled clone will be a worse, buggier version. It also duplicates Sonic Pi's own `ring`/`Enumerator`-based pattern idioms, which already cover most practical needs for a personal tool. | Use plain Ruby arrays/Enumerators/rings as the pattern representation (what the Sonic Pi ecosystem already does) and add small composable transformation methods (`rotate`, `euclid`, `every_nth`) as plain methods, not a parsed language. |
| Real-time audio analysis / feedback (listening to the audio output to adjust generation) | "Make it responsive to what's actually playing" sounds like it'd close the loop nicely. | Explicitly out of scope (DSP exclusion in PROJECT.md) and adds an entire audio-analysis subsystem plus a latency-sensitive feedback loop — directly threatens "iteration must feel instant." | If adaptive behavior is wanted, drive it from the generator's own internal state (counters, tick, elapsed cycles) rather than listening to output. |
| A generic plugin marketplace / discovery system for generators | Once pluggability exists, "let people share generators" is a natural-feeling next step for an ecosystem-shaped tool. | Multi-user and sharing are explicitly out of scope; building discovery/installation/versioning infrastructure for a single-user local tool is pure overhead with zero users to serve it. | A generator is just a Ruby file in a known directory, `require`d and registered by convention — copy-paste sharing (a gist, a folder) is sufficient for "personal playground." |
| Full song-structure/arrangement engine bolted onto "endless pieces" (sections, transitions, builds/drops) | It's tempting to let "endless generative piece" creep into "actually just write a structured song," since the line feels blurry once you have layered parts. | Explicitly out of scope (song forms) — and the failure mode is specific: once you add "section B starts at bar 32," you've silently built a sequencer/DAW timeline, which is a different, much bigger tool with its own set of table-stakes (arrangement view, automation lanes) that don't serve a live-coding personal tool. | Keep "endless" pieces as parameter-drift-over-time within one continuously-running structure (Eno's incommensurate-loops technique), not scripted sections. If song-like structure is ever wanted, that's a deliberate v2+ scope decision, not something to let in sideways. |
| Precise inter-generator quantization/sync guarantees (sample-accurate multi-part alignment, custom scheduler) | Gibber's "sample-accurate sequencing" and Overtone's metronome look like a bar worth matching. | Sonic Pi already provides `live_loop`/`sync`/`cue`/shared-BPM synchronization at the runtime level — building a *second*, competing scheduling/timing system in the Ruby layer (to control exactly when OSC messages land) is redundant engineering against a problem Sonic Pi has already solved, and risks fighting Sonic Pi's own timing model. | Emit code that leans on Sonic Pi's native `sync`/`cue`/shared-BPM constructs for part alignment; the generator layer only decides *what* to emit, never *when* it plays at the sample level. |
| Undo/redo history graph or full DAW-style project/session management | Feels like a natural companion to "favorites" and A/B re-rolling. | Session/project management (multiple named projects, branching history, etc.) is disproportionate infrastructure for a single-person tool where "favorites" (name + recall a good seed) already solves 90% of the real need. | Flat favorites file per generator is enough; skip versioned history/branching until a concrete pain point demands it. |

## Feature Dependencies

```
Seeded RNG (Ruby-side)
    └──requires──> nothing (foundational)

Scale/chord/interval vocabulary (thin wrapper over Sonic Pi's scale/chord)
    └──requires──> nothing (foundational, mostly pass-through)

Rhythm generation (incl. Euclidean via `spread`)
    └──requires──> Seeded RNG

Code emission (valid Sonic Pi Ruby)
    └──requires──> Seeded RNG, Scale/chord vocabulary, Rhythm generation
                       └──requires──> (all upstream, since emission needs something to emit)

File export (.rb)
    └──requires──> Code emission

OSC live push
    └──requires──> Code emission

live_loop wrapping (self-contained generative code)
    └──requires──> Code emission

Parameter/knob interface
    └──requires──> Code emission (knobs must map to emission inputs)

Plugin registry (pluggable generators)
    └──requires──> Code emission (defines the contract every generator implements)
    └──enhances──> everything downstream (differentiators become "just another generator")

A/B re-rolling
    └──requires──> Parameter/knob interface, Seeded RNG

Favorites capture
    └──requires──> A/B re-rolling (nothing to bookmark without a "current result" concept)

Parameter interpolation/morphing
    └──requires──> Parameter/knob interface
    └──enhances──> A/B re-rolling (morph instead of hard-switch)

Live parameter nudging while playing
    └──requires──> OSC live push, live_loop wrapping, Parameter/knob interface

Layerable part composition
    └──requires──> Code emission, Plugin registry
    └──enhances──> Endless generative piece patterns (parts need to coexist for long-running pieces to feel full)

Endless generative piece patterns (Eno-style incommensurate loops)
    └──requires──> live_loop wrapping, Rhythm generation
    └──enhances──> Layerable part composition

Weighted choice / Markov sequencing
    └──requires──> Seeded RNG
    └──enhances──> Rhythm generation, melodic generators generally

Generative grammars (L-systems)
    └──requires──> Seeded RNG, Weighted choice (grammars commonly need probabilistic rule selection)

Numeric/sequence generators (figurate numbers, CA)
    └──requires──> Scale/chord vocabulary (needs a mapping target), Code emission

CONFLICTS:
Visual grid UI ──conflicts──> "no polished GUI" constraint (PROJECT.md)
Real-time audio analysis ──conflicts──> "no DSP" constraint (PROJECT.md)
Generic plugin marketplace ──conflicts──> "no multi-user" constraint (PROJECT.md)
Full song-structure engine ──conflicts──> "no song forms" constraint (PROJECT.md)
Custom scheduler/quantization ──conflicts──> Sonic Pi's own live_loop/sync/cue (redundant, fights the runtime)
```

### Dependency Notes

- **Code emission requires the full upstream vocabulary set:** there's no meaningful way to build "emit valid Sonic Pi code" before you have *something musical* to decide (scale, rhythm, seed). Sequence these together as the true v1 core, not as separable milestones.
- **Plugin registry enhances everything downstream:** it doesn't gate the differentiators technically, but building it early means every differentiator (Euclidean generator, Markov generator, numeric-sequence generator) is developed as "just another plugin" rather than needing retrofitting later. Strongly recommend building the registry interface right after the core emission pipeline, before writing the second generator.
- **A/B re-rolling requires the knob interface, not the other way around:** knobs (named, re-rollable parameters) are the substrate; re-rolling is a workflow built on top. Building re-rolling without a real knob interface first just produces ad hoc one-off re-run scripts.
- **Favorites requires re-rolling to exist first:** a favorites feature only has value once there's a "current take" concept to compare against and capture — building it before re-rolling exists means guessing at the data model.
- **Layerable parts and endless pieces reinforce each other but aren't strictly sequential:** either can be built first, but an endless piece with only one part is thin, and layered parts without any long-running/drifting behavior are just a static combo. Consider building minimal versions of both roughly together once single-generator loops work solidly.
- **Custom scheduler conflicts, doesn't just duplicate:** this is flagged as a conflict rather than a mere redundancy because building a second timing authority risks actively fighting Sonic Pi's own `live_loop` cycle timing (races, drift) — worth stating explicitly so it's rejected outright rather than half-built and debugged.

## MVP Definition

### Launch With (v1)

Minimum to validate "twist a knob, hear it in seconds" — matches PROJECT.md's Active requirements almost 1:1.

- [ ] Seeded, deterministic Ruby-side RNG per generator — without this, nothing is reproducible or comparable
- [ ] Thin scale/chord/interval vocabulary that emits Sonic Pi's own `scale`/`chord`/`ring` calls — reuse, don't reimplement
- [ ] Basic rhythm generation, including Euclidean rhythms via emitted `spread(k, n)` — cheapest high-payoff musical technique available, and Sonic Pi already implements the algorithm
- [ ] Code emission producing syntactically valid Sonic Pi Ruby (templated string generation)
- [ ] File export to `.rb`
- [ ] OSC live push to a running Sonic Pi instance (`sonic-pi-tool`-style eval/run-file)
- [ ] `live_loop` wrapping for self-contained generative code
- [ ] Named parameter/knob interface per generator, re-rollable without editing code
- [ ] Plugin registry — a minimal `Generator` interface/base so new generators register without touching the core
- [ ] A/B re-rolling (keep last N parameter snapshots, re-roll vs. revert)

### Add After Validation (v1.x)

Add once the core loop is proven fun to use, triggered by actually wanting them mid-tinkering rather than speculatively.

- [ ] Favorites capture (name + persist a good seed/parameter set) — add once re-rolling makes "I lost that one" a real annoyance
- [ ] Weighted choice / order-1 Markov sequencing — add once uniform `choose`/`rrand`-driven generators start feeling samey
- [ ] Layerable part composition with shared master-seed-derived context — add once single-loop generators feel solid and "I want bass+drums together" becomes the itch
- [ ] Live parameter nudging on already-running loops (edit-in-place over OSC) — add once OSC push is reliable and re-editing feels slower than it should
- [ ] Endless generative piece patterns (incommensurate loop lengths, slow parameter drift) — add once layered parts exist, since a satisfying "endless piece" needs more than one part
- [ ] Numeric/mathematical sequence generators (figurate numbers, cellular automata) — cheap, fun proof of the plugin architecture; add opportunistically, not because anything depends on it

### Future Consideration (v2+)

Genuinely interesting but higher-risk/higher-effort; defer until the core harness has proven itself worth extending.

- [ ] Parameter interpolation/morphing between two seeds or parameter sets — defer because it's only valuable once there are enough favorites/parameter-sets worth morphing between
- [ ] Generative grammars (L-systems) for motif/phrase generation — defer as the highest-complexity, highest-risk differentiator; worth a spike once simpler generators have validated the harness
- [ ] Optional LLM prose→parameters front-end — explicitly listed as an Active requirement in PROJECT.md but architecturally independent of everything else here (it only needs the parameter/knob interface to target); safe to build any time after knobs exist, but not needed to validate the core value

## Feature Prioritization Matrix

| Feature | User Value | Implementation Cost | Priority |
|---------|------------|----------------------|----------|
| Seeded RNG | HIGH | LOW | P1 |
| Scale/chord vocabulary (thin wrapper) | HIGH | LOW | P1 |
| Rhythm generation incl. Euclidean (`spread`) | HIGH | LOW | P1 |
| Code emission (valid Ruby) | HIGH | MEDIUM | P1 |
| File export | HIGH | LOW | P1 |
| OSC live push | HIGH | LOW-MEDIUM | P1 |
| `live_loop` wrapping | HIGH | LOW | P1 |
| Parameter/knob interface | HIGH | LOW-MEDIUM | P1 |
| Plugin registry | HIGH | LOW-MEDIUM | P1 |
| A/B re-rolling | HIGH | LOW-MEDIUM | P1 |
| Favorites capture | MEDIUM-HIGH | LOW | P2 |
| Weighted choice / Markov | MEDIUM-HIGH | MEDIUM | P2 |
| Layerable part composition | HIGH | MEDIUM | P2 |
| Live parameter nudging | MEDIUM | LOW (given OSC push) | P2 |
| Endless generative piece patterns | HIGH | MEDIUM | P2 |
| Numeric/sequence generators | MEDIUM | LOW-MEDIUM | P2 |
| Parameter interpolation/morphing | MEDIUM | MEDIUM | P3 |
| Generative grammars (L-systems) | MEDIUM | MEDIUM-HIGH | P3 |
| LLM prose→parameters front-end | MEDIUM | MEDIUM | P3 |

**Priority key:**
- P1: Core loop — without these, there's nothing to iterate on
- P2: Makes it genuinely fun and covers the three stated generation units (loops, parts, endless pieces)
- P3: Real but deferrable — validate the core first

## Competitor Feature Analysis

| Feature | TidalCycles/Strudel | Overtone | FoxDot | phonicpsy approach |
|---------|----------------------|----------|--------|----------------------|
| Pattern representation | Custom type-theoretic pattern algebra (`Pattern a = Arc -> [Event a]`) | Plain Clojure data + scale/chord/rhythm functions | Python `Pattern` class with pattern methods/functions | Plain Ruby arrays/`Enumerator`/Sonic Pi `ring` — no custom pattern algebra (explicit anti-feature above) |
| Euclidean rhythms | First-class `euclid`/`euclidFull` operators in the pattern language | Not a first-class primitive; built from rhythm functions | Available via pattern functions | Emit Sonic Pi's own `spread` — zero reimplementation |
| Randomness/seeding | `rand` oscillator + probability operators; seeding not prominently documented | Standard Clojure `rand`/seeded PRNGs | Python `random` module, seedable | Ruby `Random.new(seed)` per generator, explicit and central (table stakes) |
| Synchronization across parts | Cycle-based global clock shared by all patterns | Shared metronome object | Player-object `.every`/shared clock | Sonic Pi's native `live_loop`/`sync`/`cue`/shared `use_bpm` — emitted, not reimplemented |
| Code emission vs. direct execution | Executes patterns directly (no separate "emission" step — Tidal/Strudel *are* the runtime) | Executes directly against SuperCollider (no emission step) | Executes directly against SuperCollider (no emission step) | **Structurally different**: phonicpsy *emits* Ruby text that Sonic Pi then runs — this is the load-bearing architectural choice (per PROJECT.md), not shared by any single surveyed tool. Closest precedent is the "generator writes Enumerator-driven values, Sonic Pi's own DSL plays them" convention seen in Sonic Pi-specific GitHub projects (e.g. Figurate Numbers), not the live-coding language tools generally. |
| UI paradigm | Text editor (Tidal via any editor+Haskell; Strudel via browser REPL) | Text editor (any Clojure REPL) | Text editor (custom Python IDE) | Text editor / Ruby scripts + thin CLI — matches the field's convention; explicitly rejects Orca's grid-UI paradigm |

## Sources

- [TidalCycles Randomness docs](https://tidalcycles.org/docs/reference/randomness/)
- [TidalCycles euclid reference](https://userbase.tidalcycles.org/euclidFull.html)
- [TidalCycles Mini Notation](https://tidalcycles.org/docs/reference/mini_notation/)
- [Sonic Pi A.09 Randomisation tutorial](https://github.com/sonic-pi-net/sonic-pi/blob/stable/etc/doc/tutorial/A.09-randomisation.md)
- [Sonic Pi 04-Randomisation tutorial](https://github.com/sonic-pi-net/sonic-pi/blob/stable/etc/doc/tutorial/04-Randomisation.md)
- [Sonic Pi 08.4-Rings tutorial](https://github.com/sonic-pi-net/sonic-pi/blob/stable/etc/doc/tutorial/08.4-Rings.md)
- [Sonic Pi official tutorial](https://sonic-pi.net/tutorial.html)
- [hundredrabbits/Orca (GitHub)](https://github.com/hundredrabbits/Orca)
- [100R — orca overview](https://100r.co/site/orca.html)
- [FoxDot (GitHub, Qirky)](https://github.com/Qirky/FoxDot)
- [FoxDot Pattern Methods docs](https://foxdot681713046.wordpress.com/docs/pattern-methods/)
- [FoxDot Pattern Functions docs](https://foxdot681713046.wordpress.com/docs/pattern-functions/)
- [Overtone (GitHub)](https://github.com/overtone/overtone)
- [Overtone README](https://github.com/overtone/overtone/blob/master/README.md)
- [Gibber (Charlie Roberts)](https://charlie-roberts.com/gibber/)
- [Gibber Live Coding Blog overview](https://blog.livecoding.nyuadim.com/2024/02/16/gibber/)
- [Sonic Pi Figurate Numbers project context (GitHub topics search)](https://github.com/topics/algorithmic-composition)
- [Strudel REPL](https://strudel.cc/)
- [strudel-tutorial: Strudel vs Tidal](https://github.com/nikolasgioannou/strudel-tutorial/blob/main/notes/16-strudel-vs-tidal.md)
- [Euterpea homepage](https://www.euterpea.com/)
- [Euterpea on Hackage](https://hackage.haskell.org/package/Euterpea)
- [Euclidean rhythm — Wikipedia](https://en.wikipedia.org/wiki/Euclidean_rhythm)
- [Markov Chains for Computer Music Generation (ResearchGate)](https://www.researchgate.net/publication/353985934_Markov_Chains_for_Computer_Music_Generation)
- [Using Linear Algebra and Markov Chains to Algorithmically Generate Music (Medium)](https://medium.com/@vanessaseto1/using-linear-algebra-and-markov-chains-to-algorithmically-generate-music-compositions-7dc88edda642)
- [emlyn/sonic-pi-tool (GitHub)](https://github.com/emlyn/sonic-pi-tool)
- [Sonic Pi 12.1-Receiving-OSC tutorial](https://github.com/sonic-pi-net/sonic-pi/blob/dev/etc/doc/tutorial/12.1-Receiving-OSC.md)
- [Brian Eno's Endless Music Machines (Gorillasun)](https://www.gorillasun.de/blog/brian-enos-endless-music-machines/)
- [Twenty Techniques For Generative Music, Inspired By Brian Eno (Synthtopia)](https://www.synthtopia.com/content/2019/04/24/twenty-techniques-for-generative-music-inspired-by-brian-eno/)
- [Introduction to Generative Music (Alex Bainter, Medium)](https://medium.com/@alexbainter/introduction-to-generative-music-91e00e4dba11)
- [Generative music — Wikipedia](https://en.wikipedia.org/wiki/Generative_music)
- [Interacting with Generative Music through Live Coding (ResearchGate)](https://www.researchgate.net/publication/27478005_Interacting_with_Generative_Music_through_Live_Coding)

---
*Feature research for: algorithmic/generative music tooling (Ruby layer over Sonic Pi)*
*Researched: 2026-09-15*
