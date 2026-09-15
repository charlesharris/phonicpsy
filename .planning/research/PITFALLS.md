# Pitfalls Research

**Domain:** Ruby layer generating/driving Sonic Pi code (algorithmic/generative music tooling)
**Researched:** 2026-09-15
**Confidence:** MEDIUM-HIGH (Sonic Pi integration facts are HIGH — verified against sonic-pi-net/sonic-pi source, issues, and official docs; generative-music-craft and Ruby-determinism findings are MEDIUM — cross-checked community/practitioner consensus, not project-specific)

## Critical Pitfalls

### Pitfall 1: The OSC handshake is a moving target, and the "obvious" port doesn't run code

**What goes wrong:**
Sonic Pi has *two different OSC surfaces* that get conflated. Port 4560 (or whatever `--osc-in-udp-port` reports) is Sonic Pi's **cue-listening port** — external OSC messages arrive there and show up in the cue log, but nothing executes unless a `live_loop`/`sync` inside a running buffer is explicitly listening and `eval`-ing the payload. The `/run-code` message that actually executes arbitrary code is a *separate, GUI-internal* channel: since Sonic Pi 4.x it requires a random server port **and** a random auth token, both written to `spider.log` (or the daemon's stdout) at boot and rotated every launch. A maintainer comment in `daemon.rb` states outright that `/run-code` "should really only be run from the GUI" — external OSC clients are second-class citizens on that path. Multiple users report `/run-code` working from inside the GUI's own send-to-self test but silently doing nothing from an external Python/Ruby process on the exact same port.

**Why it happens:**
The token+port scraping requirement is a security response to earlier versions where any process on the network (or even localhost) could execute arbitrary Ruby via OSC. It was added without a stable public API or version-locked doc, so every external tool (sonic-pi-cli, python-sonic, sonic-pi-tool) has broken across major Sonic Pi versions and had to re-reverse-engineer log parsing.

**How to avoid:**
- Don't build a custom token/port scraper as the primary path. Use the **cue-listener pattern explicitly recommended by Sonic Pi's own author** (Sam Aaron): ship a small, static bootstrap buffer once (`live_loop :remote { use_real_time; c = sync "/osc*/trigger/code"; eval(c[0]) }` — reload from an OSC cue) run manually or via `init.rb`, and drive it thereafter with plain cues to the well-known 4560 port. This trades "purely programmatic from cold boot" for "one-time manual buffer run" but is far more stable across versions.
- If `/run-code` is wanted anyway, treat token/port discovery as an isolated, version-pinned adapter module with a single integration test that fails loudly on log-format changes — not something generators depend on directly.
- Look at `spx` (RubyGems, released Nov 2025, uses `osc-ruby`) — a newer Ruby CLI built specifically to `play`/`record` Sonic Pi code via its OSC server; study or vendor its port/token handling rather than re-deriving it, since it's already solved this for a recent version.

**Warning signs:** OSC messages show green/received in the Sonic Pi cue log but nothing plays; behavior differs between "works when Sonic Pi GUI is focused" and "works headless"; works on one Sonic Pi point release and silently stops on the next.

**Phase to address:** Phase covering live OSC push (the "push into a running instance" requirement) — spike the integration path first, before building generators on top of it. This is exactly the kind of gotcha the project's own `PROJECT.md` context flags as an integration seam, and it deserves a throwaway spike, not being discovered mid-generator-work.

**Sources:** [Clarifying usage of OSC /run-code in latest versions (4+) — issue #3330](https://github.com/sonic-pi-net/sonic-pi/issues/3330); [Revisiting Calling Sonic Pi From Ruby — Kevin Murphy](https://kevinjmurphy.com/posts/revisiting-calling-sonic-pi-from-ruby/); [spx on RubyGems](https://rubygems.org/gems/spx)

---

### Pitfall 2: Errors in pushed code vanish into the void

**What goes wrong:**
When code is pushed via OSC (either `/run-code` or a cue-driven `eval`), a Ruby/Sonic-Pi syntax or runtime error does **not** propagate back over the wire to the caller as a response message — Sonic Pi has no OSC error-reply protocol. The error surfaces only in the Sonic Pi GUI's log pane (if the GUI is even open) or, in headless mode, in server-side logs the external process isn't watching. Reports of `/run-code not playing anything, other methods work fine` and of headless daemons that appear to receive OSC but do nothing are consistent with this: the client has no way to distinguish "silently failed to parse" from "silently succeeded but produced no sound."

**Why it happens:**
Sonic Pi's OSC surface was designed for live-coding performers watching their own GUI, not for programmatic clients that need a synchronous execute/ack/error contract. There is no `/run-code-error` reply message in the protocol.

**How to avoid:**
- Never trust "the OSC send succeeded" as "the code ran." Validate generated code *before* sending it: run it through `RubyVM::InstructionSequence.compile` (or a plain `Ripper.sexp`) locally to catch syntax errors pre-flight, since that's the failure category most likely with generated code.
</br>- For headless/unattended operation, run Sonic Pi's server with logging enabled and tail the log file from the driving process so failures are at least visible somewhere, even without a structured reply channel.
- Treat "I pushed code and heard nothing" as an expected, frequent event in the design, not an edge case — build a fast manual verification habit (a short confirmatory beep/tone at the start of generated code) rather than trying to solve the general error-reporting problem.

**Warning signs:** Silence after a push with no exception raised anywhere in the Ruby driver; behavior differences between GUI-open and headless runs of the identical push.

**Phase to address:** Live push phase — bake pre-flight syntax validation into the emission path from day one; it is cheap and directly buys back iteration-loop confidence (Core Value).

**Sources:** [/run-code not playing anything — issue #767](https://github.com/sonic-pi-net/sonic-pi/issues/767); [Run-code not working in headless mode — in_thread](https://in-thread.sonic-pi.net/t/run-code-not-working-in-headless-mode/7635); [Sonic Pi Headless Doesn't receive OSC cues — in_thread](https://in-thread.sonic-pi.net/t/sonic-pi-headless-doesnt-receive-osc-cues/2171)

---

### Pitfall 3: Generated code silently gets truncated past ~8-9KB over OSC

**What goes wrong:**
Sonic Pi's OSC transport is UDP by default, and UDP has a hard practical payload ceiling. A long-standing, still-open issue reports code failing to run once it crosses roughly 8,846 characters — even when the overflow is *comments*, i.e. it's a raw byte-length cutoff, not a "too many notes" limit. Since this project algorithmically *generates* code (which tends to be more verbose per musical idea than hand-written code — unrolled patterns, generated comments, long `play_pattern_timed` arrays), it is easy to walk straight into this ceiling on any moderately dense generator without realizing it, because the failure looks identical to Pitfall 2 (silence, no error).

**Why it happens:**
The GUI/server/scsynth architecture communicates over UDP internally; UDP datagrams have OS/network MTU-driven size limits well below what a generative-music buffer full of unrolled note data can produce.

**How to avoid:**
- Keep a hard character-count assertion in the code-emission path (e.g., refuse/warn above ~8KB) so oversized output fails loudly in the Ruby layer, not silently on the wire.
- Prefer terse code-generation idioms over unrolled note lists: emit loops with parameterized data (arrays passed via `cue`/`osc` payload data messages, or looked-up-by-index patterns) rather than long inline literals, keeping the *code* small even when the *musical content* is large.
- If large payloads are unavoidable, split across multiple buffers linked by `cue`/`sync`, or drive via TCP OSC instead of UDP (reported as a workaround, with a materially higher size ceiling).

**Warning signs:** A generator works fine at short lengths/low densities and mysteriously "does nothing" only once note count, layers, or verbosity grow — looks identical to a hang or silent failure.

**Phase to address:** Live push phase — add the size guard as part of the same pre-flight validation work as Pitfall 2, before generators get complex enough to trip it.

**Sources:** [Maximum limit on file size — issue #146](https://github.com/sonic-pi-net/sonic-pi/issues/146) (still open, re-bumped as [#741](https://github.com/samaaron/sonic-pi/issues/741)); [Maximum characters limit per buffer — in_thread](https://in-thread.sonic-pi.net/t/maximum-characters-limit-per-buffer/229)

---

### Pitfall 4: Pushing new code doesn't cleanly replace what's running

**What goes wrong:**
`run-code`/buffer-run semantics in Sonic Pi are **additive by job, not replace-by-name**: each run spawns a new "job" thread. A `live_loop` with the same name redefined *while it's mid-`sleep`* picks up the new definition on its *next* iteration (this is the celebrated live-coding feature — "redefine on the fly"), but non-loop top-level code and effects wrapped *around* a `live_loop` do not get re-applied on a re-run — a documented, still-relevant confusion (effects around a live loop not reacting to changes when re-running a buffer). If a generator's pushed snippet declares a *new* `live_loop` name each time (e.g., a randomly-generated identifier) instead of a stable one, loops accumulate rather than replace, and the mix gets denser and denser with orphaned, un-stoppable loops from earlier pushes.

**Why it happens:**
Sonic Pi's job/thread model was designed around a human hitting one Run button on one buffer repeatedly with stable loop names — not around a script pushing a stream of distinct code blobs.

**How to avoid:**
- Every generator that emits a `live_loop` must derive the loop name **deterministically from a stable generator identity** (e.g., `:phonicpsy_bass`, not `:bass_#{rand(9999)}`), so re-pushes actually replace rather than stack.
- Keep an explicit registry (even a simple constant list) of loop names the tool has ever pushed into the current session, exposed via a `stop-all`/`panic` command that emits `live_loop :x { stop }` for every known name — this is the practical antidote to Pitfall 5 below too.
- Put effect wrapping *inside* the `live_loop` body (as Sonic Pi's own docs recommend) rather than wrapped around it at top level, since that's the form that reliably re-applies on redefinition.

**Warning signs:** Sound gets progressively busier/denser across iterations with no code change explaining it; stopping "the loop" via the GUI doesn't silence everything; CPU/voice count climbs session over session.

**Phase to address:** Live push / streaming phase. Also directly relevant to the pluggable-generator phase: the generator interface should *require* a stable loop-name contract, not leave it to each generator author to remember.

**Sources:** [A.02 Live Coding tutorial](https://github.com/sonic-pi-net/sonic-pi/blob/dev/etc/doc/tutorial/A.02-live-coding.md); [Effects around live loops not reacting to changes when re-running — issue #794](https://github.com/sonic-pi-net/sonic-pi/issues/794); [09.3 Multiple Live Loops tutorial](https://github.com/sonic-pi-net/sonic-pi/blob/dev/etc/doc/tutorial/09.3-Multiple-Live-Loops.md)

---

### Pitfall 5: A runaway `live_loop` you can't stop without restarting Sonic Pi

**What goes wrong:**
There is no OSC "kill this loop from outside" message beyond sending code that redefines the loop with a `stop`. If a generator emits a loop whose `sleep` value can evaluate to `0` (e.g., via integer division of two generated integers, `steps / bpm` where both are ints), Sonic Pi doesn't necessarily hang — small-magnitude or zero sleeps typically manifest as a **"Thread got far behind the time"** runtime exception once the audio thread can't keep pace, which can cascade to a genuinely unresponsive/crashing session on constrained hardware. Negative sleep values raise a distinct Timing Exception outright. Either way, once a loop is in this state, the only reliably reported fix in the community is stopping via the GUI stop-button or, failing that, killing and restarting the whole Sonic Pi process — losing all other currently-running (good) loops in the same session.
</br>Separately, a `live_loop` **must** contain a `sleep` at all — Sonic Pi's own safety net raises "did not sleep" if a generator's control-flow (e.g., an early-return branch, or a conditional that skips the sleep call) produces a loop body that can execute a full cycle without hitting one.

**Why it happens:**
Sleep durations in generated code are frequently *computed* (from tempo, subdivision, randomly-chosen note duration, etc.) rather than literal constants, so the "always positive, always present" invariant that's trivially true in hand-written code is not automatically true in generated code — a stray `/` between two integers, an empty pattern-length array, or a generator branch that forgets the sleep are all one-line bugs with this specific, ecosystem-specific failure mode.

**How to avoid:**
- Centralize all `sleep` emission through one helper in the Ruby generation layer that clamps to a floor (e.g., `[computed_sleep, 0.05].max`) and always forces float division (`duration.to_f / steps`), so no generator author can accidentally emit an integer-zero sleep.
- Statically check every emitted `live_loop` body contains at least one syntactic `sleep`/`sync` call on every code path before it's ever pushed (a simple string/AST check is enough — full control-flow analysis is not required for a personal tool).
- Build the "stop everything" panic command (Pitfall 4) early and treat it as a first-class, frequently-exercised part of the workflow, not a rarely-used escape hatch — for a project whose Core Value is fast iteration, the ability to instantly silence a bad push matters as much as the ability to push at all.

**Warning signs:** Audio starts glitching/stuttering shortly after a push; Sonic Pi log shows "Thread got far behind the time"; a generator with computed durations occasionally (not always) fails, correlating with specific random seeds/parameter combinations that produce near-zero durations.

**Phase to address:** Core generation phase (sleep-safety belongs in the shared code-emission layer, not per-generator) with the panic/stop command landing alongside the live-push phase.

**Sources:** [Thread got far behind the time — ERROR — in_thread](https://in-thread.sonic-pi.net/t/thread-got-far-behind-the-time-error/1698); [live_loop syncing issue? — issue #1730](https://github.com/samaaron/sonic-pi/issues/1730); [01.1 Live Coding tutorial (live_loop sleep requirement)](https://github.com/sonic-pi-net/sonic-pi/blob/dev/etc/doc/tutorial/01.1-Live-Coding.md)

---

### Pitfall 6: The generated music is technically correct and completely unlistenable ("it's all noise")

**What goes wrong:**
This is the signature failure of algorithmic composition, and it happens *even when every engineering piece works perfectly*: syntactically valid code, correct OSC delivery, stable seeds, no crashes — and the output is an undifferentiated wash of notes with no perceivable structure. Concretely, this comes from a small, well-known set of missing constraints, roughly in order of "musicality gained per unit of effort":
1. **No scale/pitch quantization** — picking raw MIDI note numbers or unconstrained `rrand` frequencies instead of choosing scale degrees from `scale(:c, :minor_pentatonic)` and mapping through `.choose`/weighted index. Un-quantized pitch is the single biggest source of "noise."
2. **No rhythmic grid alignment** — sleeps computed as raw continuous randoms (`rrand(0.1, 0.9)`) instead of chosen from a discrete, musically-related set (`[0.25, 0.5, 1.0].choose` or a subdivision-of-a-bar grid). Ungridded timing reads as noise even when pitches are fine.
3. **No register separation between parts** — a bass generator and a lead generator both drawing from the same pitch range collide and mask each other; parts need disjoint or minimally-overlapping octave ranges to read as separate voices at all.
4. **No density/rest limits** — every step firing a note (no rests, no silence) produces a wall of sound; real musical lines have empty beats. A "probability of playing this step" parameter (commonly 40-70% for a rhythm part, lower for a lead) does more for perceived musicality than almost any other single knob.
5. **Uniform random instead of weighted/structured choice** — `array.sample`/uniform `.choose` across a scale treats the leading tone and the tonic as equally likely on every beat, producing melodies with no tonal center. Weighting scale-degree choice (tonic/fifth/third heavily favored, tension tones rare) or using a simple first-order Markov transition table over scale degrees buys a large, cheap improvement over uniform choice.
6. **No voice-leading / continuity constraint between successive notes** — large random leaps every note read as chaotic; constraining consecutive-note interval size (e.g., bias toward steps of 0-4 scale-degrees, occasional larger leap) is a classic, cheap fix from algorithmic-composition practice.

**Why it happens:**
These constraints are invisible in the code — a generator that ignores all six still "works" in every engineering sense (valid Ruby, runs, makes sound), so nothing forces the author to notice until they actually listen. It's the exact trap a personal, greenfield playground project is most likely to fall into first, because the natural build order is "get code emission working" before "make it sound good," and the former is easy to verify (does it run?) while the latter is not (do I like it?).

**How to avoid:**
- Bake scale-quantization and grid-alignment into the *shared* generator toolkit (helpers every generator uses), not left to each generator to reimplement — this is the highest-leverage single decision in the whole project for avoiding "sounds like noise," because it fixes pitfalls 1 and 2 above for every generator at once.
- Require every rhythm-producing generator to support a density/rest parameter from the start, not as a later enhancement — silence is a first-class musical value, not a missing feature.
- Prefer weighted `.choose` (a simple `{value => weight}` table, or Sonic Pi's own `.choose` combined with a manually duplicated-weighted array) over uniform `.sample`/`.choose` as the *default* idiom in generator code, and treat literal `rand`/`rrand` over a full range as a smell worth a second look in code review.
- Treat "does this actually sound good, not just run" as an explicit UAT gate for any generator work — the project's Core Value (hear it, twist a knob) makes this checkable in seconds, so there's no excuse for shipping a generator that's never actually been listened to.

**Warning signs:** A generator "passes" (runs without error, produces audio) but the author finds themselves not wanting to actually listen to it a second time; every generator sounds similar regardless of its supposed musical identity (rock loop vs. ambient wash sound the same); increasing note count/density makes it *worse*, not more interesting — a strong signal there's no structural constraint underneath.

**Phase to address:** This deserves to be explicit in the roadmap, not implicit — either as the closing concern of the core-generation phase (shared scale/grid/weighted-choice helpers ship alongside the first generator, not after) or as its own short "musicality pass" phase before pluggability work. Given the project's Core Value, this pitfall is arguably the most important one to sequence early: a plugin framework for generators that all sound like noise is wasted effort (see Pitfall 10).

**Sources:** Sonic Pi's own algorithmic-composition idioms (`scale`, `.choose`, weighted-array patterns) as documented in [A.09 Randomisation](https://github.com/sonic-pi-net/sonic-pi/blob/stable/etc/doc/tutorial/A.09-randomisation.md); cross-checked general generative-music practice (density/rest, register separation, weighted scale-degree choice, voice-leading interval constraints) — MEDIUM confidence, established community practice rather than a single authoritative source.

---

### Pitfall 7: "Same seed, same output" quietly breaks in several specific places

**What goes wrong:**
The determinism requirement is explicit in PROJECT.md, and it's more fragile than it looks, for reasons specific to this stack:
- **Two independent random streams.** Sonic Pi's *own* `rand`/`rrand`/`.choose` inside emitted code draw from a separate, pre-generated 441,000-value stream that's reset and re-seeked via `use_random_seed` — this is **completely independent** of Ruby's `Kernel.rand`/`Random` used in the *generating* Ruby layer. Seeding one does nothing to the other. A generator that seeds the Ruby-side composer but also calls Sonic Pi's own `rand` inside a self-contained `live_loop` (the "generative code Sonic Pi runs forever unattended" requirement) has two seeds to manage, and forgetting to emit `use_random_seed` inside the generated code means that half of the "same seed, same output" contract is silently unfulfilled — the Ruby-side structure will replay identically, but the in-buffer randomization will differ every run.
- **`Kernel.rand`/top-level `rand` is global, mutable state.** If any generator (or a shared helper, or a library dependency) calls unscoped `rand`/`srand` anywhere during a run, it perturbs the global stream that *other* generators may also be implicitly relying on via unscoped `rand`, producing cross-generator seed leakage — generator A's output changes depending on whether generator B ran first in the same process. Using `Random.new(seed)` instances passed explicitly through each generator's call chain avoids this entirely; relying on bare `rand`/`Kernel.rand` anywhere in the generation path does not.
- **Hash and Set iteration order.** Ruby Hashes preserve insertion order (not a hazard by themselves), but code that builds a `Hash` or `Set` from another unordered source (e.g., `Dir.glob`, `Hash#group_by` on data with duplicate keys inserted in varying order, or symbol-keyed hashes merged from multiple partial hashes whose *build* order depends on iteration over something unordered) can end up iterating in an order that isn't as fixed as assumed. This mostly bites when a generator does something like `notes.group_by(&:pitch_class).each` and then `.choose`s from group order — reorder the grouping and the seeded RNG consumes values in a different sequence, changing the output even though the seed didn't change.
- **Floating-point accumulation in cumulative timing.** If a generator computes each event's absolute time by *summing* successive float sleeps (`t += duration`) rather than deriving from an integer step count, float rounding error accumulates across a long generative piece; it won't break determinism (same seed still sums to the same float trajectory on the same machine/Ruby version), but it can make output subtly non-reproducible **across Ruby versions or platforms** if floating-point summation order or FPU behavior differs — a real risk if a generated `.rb` file is opened on a different machine later.
- **Time-dependent values sneaking in.** Anything that touches `Time.now`, file mtimes, `Process.pid`, or object `#hash`/`#object_id` for tie-breaking (`.sort_by` with a non-unique key falling back to insertion-dependent stable-sort behavior, or `.min_by`/`.max_by` ties) introduces run-to-run variance that has nothing to do with the seed and is easy to introduce accidentally (e.g., using current time as a "unique loop name suffix," which also collides with Pitfall 4).

**Why it happens:**
Ruby's convenience — global `rand`, insertion-ordered but not always deterministically-*constructed* Hashes, implicit float math — is exactly the kind of ambient state that's invisible until two runs with the same seed diverge, and the two-RNG-stream problem is specific to embedding one language's generative code inside another's runtime, which is precisely this project's architecture.

**How to avoid:**
- Thread an explicit `Random.new(seed)` instance through every generator's public interface; ban bare `rand`/`srand`/`Kernel.rand` calls in generator code via a code-review habit (or a simple grep-based check) rather than relying on discipline alone.
- Whenever a generator emits self-contained generative code (a `live_loop` with its own randomness, per the "runs forever unattended" requirement), always emit an explicit `use_random_seed <derived-seed>` as the first line, derived deterministically from the same top-level seed (e.g., a hash of `[seed, generator_name]`) — make this part of the shared emission helper, not something each generator remembers to do.
- Never build ordering-sensitive structures (`Hash`, `Set`, `group_by` results) from sources whose own order isn't already deterministic; when in doubt, `.sort` before `.each`/`.choose`.
- Derive absolute event times from `step_index * fixed_duration` rather than accumulating sleeps, when the generator computes its own schedule outside of Sonic Pi's own `sleep`.
- Treat "does re-running with the same seed twice, in the same process and in two separate processes, produce byte-identical generated code" as a concrete, automatable check (diff two `.rb` exports) rather than a vague goal — cheap to build, and it will catch every one of the above the first time it's violated.

**Warning signs:** A generator's output "usually" matches on the same seed but occasionally doesn't; output differs between running one generator alone vs. running it after another generator in the same process; a `.rb` file exported today sounds subtly different from the same seed's export next week with no code changes (time-dependent leakage).

**Phase to address:** Core generation phase — the determinism requirement is listed as Active in PROJECT.md and should have its verification (seed-diff check) built alongside the first generator, not bolted on later once several generators already have ambient `rand` calls to hunt down.

**Sources:** [A.09 Randomisation — Sonic Pi's pre-generated random stream and `use_random_seed`](https://github.com/sonic-pi-net/sonic-pi/blob/stable/etc/doc/tutorial/A.09-randomisation.md); [Sonic Pi: Randomization — Joey Reyes](https://joeyreyes.dev/blog/sonic-pi/randomization); [Ruby `Random` class docs](https://docs.ruby-lang.org/en/master/Random.html) — confirms `Random.new` instances are independent of the global `Kernel.rand` PRNG state.

---

### Pitfall 8: The iteration loop dies by a thousand small frictions, not one big one

**What goes wrong:**
The project's stated Core Value is edit→hear in seconds. Given the research above, the realistic threats to that loop are concrete and specific, not generic "slow build" complaints:
- **Every knob-twist round-trips through the fragile OSC/token path** (Pitfall 1) instead of a cheap local mechanism — if twisting a knob means re-discovering the port/token or re-running a multi-second bootstrap, the *inner* loop (knob-twist) inherits the friction that should only apply to the rare *outer* loop (generator-edit).
- **Silent failures (Pitfall 2/3) masquerade as "nothing changed"** — the single most iteration-corrosive event in a live-coding tool is a push that does nothing and gives no feedback, because the natural response is to re-push, tweak something unrelated, and burn several cycles debugging the wrong layer.
- **Accumulating orphaned live_loops (Pitfall 4/5)** mean the "sound I'm hearing" stops corresponding to "the code I last edited" — once several stale loops are stacked, the feedback loop between edit and heard-result is broken even though nothing crashed.
- **Requiring Sonic Pi to be manually restarted** after certain changes (e.g., after an unrecoverable "Thread got far behind the time" cascade) reintroduces multi-second-to-tens-of-seconds startup latency into what should be a sub-second loop — and headless startup specifically has a reported ~6-second warm-up before it reliably accepts OSC at all, which is directly relevant if the project's live path uses a headless server rather than the GUI.
- **File-path/buffer-juggling for the export path** — if using the `.rb` export requires remembering where files land, manually opening them in the Sonic Pi editor, and manually hitting Run, that's ceremony sitting directly in the "keep what works" path the project explicitly wants (both live-push and file-export are Active requirements).

**Why it happens:**
Each of these is individually minor; the risk for a personal project is that none of them feels worth fixing in isolation ("I'll just restart Sonic Pi, it's fine"), and they compound until the tool feels like a chore rather than disappearing, which directly contradicts the Core Value.

**How to avoid:**
- Design the knob-twist path to avoid the OSC handshake entirely where possible — e.g., knob state lives in the Ruby process and re-emits+re-pushes only the affected `live_loop`'s body via the already-open, already-authenticated cue-listener connection (Pitfall 1's recommended pattern), not a fresh handshake per twist.
- Build the "stop everything and get back to a known-good state" panic command (Pitfall 4/5) as early tooling, since it is the fastest recovery path when the loop breaks, and treat "how long does it take to get from a broken state back to hearing something" as a metric worth caring about, not just "how long does the happy path take."
- Keep a single, unambiguous export location (e.g., always `./exports/<generator>-<seed>.rb`) so file-path thinking never enters the loop.
- If headless startup latency turns out to matter (multi-second warm-up before OSC is live), keep a long-running Sonic Pi *session* open across an entire work session rather than starting/stopping it per iteration — treat "Sonic Pi is already running" as an assumed precondition of the tool, not something it manages for you, consistent with PROJECT.md's constraint that a locally running Sonic Pi is a dependency.

**Warning signs:** Reaching for the GUI Stop button becomes a frequent, reflexive action; noticing you've stopped trusting that what you hear reflects what you last edited; catching yourself avoiding trying an idea because "it's annoying to restart."

**Phase to address:** Live push phase for the mechanical fixes; but this pitfall is really a design lens to apply across every phase — each phase's plan should explicitly ask "does this add a step to the edit→hear loop?"

**Sources:** Synthesizes Pitfalls 1-5 above plus [Run-code not working in headless mode — in_thread](https://in-thread.sonic-pi.net/t/run-code-not-working-in-headless-mode/7635) (headless startup latency).

---

### Pitfall 9: Building the framework before any generator sounds good

**What goes wrong:**
The most common way solo generative-music/live-coding projects die is investing in structure — a plugin/generator framework with a registration system, an intermediate representation (IR) between "music description" and "emitted code," a configuration DSL for describing generators, a parameter-knob abstraction layer — before a *single* generator actually produces something the author wants to listen to twice. The abstraction gets built against imagined future generators rather than real ones, so when the second and third real generators arrive, the framework is wrong in ways that weren't visible from generator #0, and it gets reworked anyway — meaning the up-front investment bought nothing except delay before the first sound.

**Why it happens:**
Framework-building is legible, low-risk-feeling work with a clear "done" state (the interface compiles, the registry works), whereas "make this sound good" is open-ended, uncomfortable, and hard to know when to stop — so it's the natural thing to defer, especially for a personal project with no external deadline forcing the question. This is compounded here because the project's Out of Scope list already explicitly excludes a "polished GUI," acknowledging the general risk — but the same instinct applies just as easily to an over-general generator-plugin API or an over-designed IR, which PROJECT.md doesn't call out.

**How to avoid:**
- Sequence one concrete, ugly-but-working generator (hardcoded, no plugin interface, no shared IR) all the way through to "sounds good and is fun to twist" *before* extracting any shared interface. Let the second generator be built by literally copy-pasting the first and diverging — only extract the plugin contract once there are two-to-three real generators to generalize *from*, not to.
- Resist designing the parameter/knob system in the abstract; let it fall out of what the first generator's actual, specific knobs turn out to be (tempo, density, scale, register spread — from Pitfall 6's constraint list) rather than a generic "any generator can expose any typed parameter" system built speculatively.
- Explicitly timebox or scope-cap "framework" work relative to "generator" work per phase — if a phase's plan spends more effort on the plugin mechanism than on making one generator sound good, that's the warning sign in real time, not in hindsight.
- Keep the IR as thin as possible for as long as possible — for a personal tool with one consumer (Sonic Pi's Ruby DSL) and one producer (Ruby generator code in the same language), a shared intermediate representation may not be needed at all; string-building helpers (the scale/grid/weighted-choice toolkit from Pitfall 6) may be sufficient without a formal IR layer, and "perfecting" one is a classic sink for a project whose real payoff is musical, not architectural.

**Warning signs:** Time spent on registration/plugin-loading code exceeds time spent listening to actual output; the phrase "once the framework is in place, generators will be easy to add" is doing a lot of unverified load-bearing work; there's a generator interface but fewer than two generators implementing it; excitement is coming from the elegance of the code rather than from the sound.

**Phase to address:** This is a sequencing concern for the roadmap as a whole — the coarse phase order should put "one generator sounds good" ahead of "generators are pluggable" (both are Active requirements in PROJECT.md, and the pluggability requirement should be satisfied by *generalizing from* a working generator, not preceding one).

**Sources:** Synthesizes general solo/greenfield generative-tooling failure patterns; cross-referenced against PROJECT.md's own Out of Scope reasoning (the "polished GUI" exclusion applies the same logic this pitfall generalizes). MEDIUM confidence — pattern-level claim, not tied to a single citable incident.

---

### Pitfall 10: Chasing "real" music theory depth that never gets heard

**What goes wrong:**
A close cousin of Pitfall 9, specific to this domain: investing in deep, general music-theoretic machinery — full negative-harmony/modal-interchange support, a general voice-leading solver, arbitrary time-signature/polymeter support, a complete chord-substitution engine — because it feels like "doing the music theory properly," when a much smaller, targeted subset of constraints (Pitfall 6's list: scale quantization, grid alignment, register separation, density, weighted choice, bounded intervals) would produce most of the audible improvement for a fraction of the effort. This is a particularly tempting trap here because PROJECT.md explicitly frames the generators as "built from music theory," which can be read as license to build theory-first rather than ear-first.

**Why it happens:**
Music theory is a large, genuinely interesting, well-documented field, and it's easy to confuse "implementing more of it" with "making better music" — but the marginal audible return on theoretical completeness drops sharply after the basics (a pentatonic scale with weighted tonic/fifth emphasis and a density parameter already sounds intentional; full jazz reharmonization machinery on top of a generator nobody's listened to yet buys little).

**How to avoid:**
- Apply the constraint list in Pitfall 6 as the *default*, minimum bar for "sounds like music, not noise" — and treat anything beyond it (chord substitution, modal interchange, full voice-leading solvers) as optional depth added *after* a generator already sounds good with the basics, driven by "I specifically want to hear X" rather than "the theory would be more complete with X."
- Let generator *variety* (the explicitly-welcomed "16-bit FF loops," "noise walls," "weirdest version of a vibe" flavor from PROJECT.md) be the outlet for ambition, rather than depth within any single generator's theoretical sophistication — this fits the project's own stated framing better and produces more listenable output per hour spent.
- If a theory feature is tempting, ask "would a simpler version of this (e.g., a fixed weighted table instead of a computed harmonic function) sound different enough to matter" before building the general version.

**Warning signs:** Time spent reading/implementing music theory concepts outpaces time spent listening to what's already built; a generator has more configuration surface than musical personality; the question "does this sound good" gets deferred in favor of "is this theoretically correct."

**Phase to address:** Same sequencing concern as Pitfall 9 — a "musicality pass" phase (if the roadmap has one) should scope itself explicitly to the cheap-constraint list, not open-ended theory depth, and later milestones are the right place for anything beyond that if it's ever wanted at all.

**Sources:** Synthesizes PROJECT.md's own framing ("built from music theory... flavor rather than commitment," novelty/surprise welcomed but not the success bar) with the general algorithmic-composition-craft pattern in Pitfall 6. MEDIUM confidence.

---

## Technical Debt Patterns

| Shortcut | Immediate Benefit | Long-term Cost | When Acceptable |
|----------|--------------------|-----------------|------------------|
| Hardcode the first 1-2 generators with no plugin interface | Faster to a listenable result | Some rework when extracting the shared interface | Always acceptable, in fact recommended (see Pitfall 9) |
| Use `init.rb`-based code injection instead of solving OSC push | Gets *something* audible without solving the OSC handshake | Can't hear the song without restarting Sonic Pi every time; unusable for the fast-iteration Core Value | Only for a first end-to-end spike/demo, never as the live-push mechanism |
| Skip pre-flight syntax validation on emitted code | Simpler emission path | Every syntax bug becomes a silent, undebuggable no-op push (Pitfall 2) | Never — this is cheap and high-leverage, build it early |
| Let each generator manage its own `Random` seeding ad hoc | Faster to write a first generator | Determinism silently breaks the moment a second generator or a shared helper touches `rand` (Pitfall 7) | Never past the very first generator — centralize before generator #2 |
| Build knob-twisting by re-running the whole generator + full re-push each time | Simple to implement | Reintroduces OSC-handshake/job-stacking overhead into the fast inner loop (Pitfall 8) | Acceptable for an initial spike; revisit once knob-twisting is a core Requirement being exercised often |

## Integration Gotchas

| Integration | Common Mistake | Correct Approach |
|-------------|------------------|-------------------|
| Sonic Pi OSC `/run-code` | Assuming any external OSC client can call `/run-code` like the GUI does | Use the cue-listener + `eval` pattern for external control; treat `/run-code` token/port scraping as a fragile, isolated adapter if used at all |
| Sonic Pi headless server | Sending OSC immediately after spawning the daemon process | Wait for the daemon to be ready (reported ~6s on constrained hardware) or poll a readiness signal before sending |
| `sonic-pi-tool` naming confusion | Assuming "sonic-pi-tool" is a Ruby library to `require` | The well-known `sonic-pi-tool` (lpil/emlyn) is a standalone Rust/Python CLI shelled out to, not a Ruby gem; the Ruby-native option is the newer `spx` gem or driving `osc-ruby` directly |
| Large generated buffers over OSC | Sending long unrolled note-array code as one `/run-code`/cue payload | Keep emitted code terse (parameterized loops, not unrolled literals); guard with a size check; split across buffers/cues if needed |
| `live_loop` re-push | Emitting a fresh randomly-named loop per push | Always emit stable, generator-derived loop names so re-pushes replace rather than stack |

## Performance Traps

| Trap | Symptoms | Prevention | When It Breaks |
|------|----------|------------|-----------------|
| Computed `sleep` durations from integer math | "Thread got far behind the time" errors, audio stutter | Force float division and clamp to a minimum floor in a shared helper | As soon as any generator computes a sleep from a ratio of two integers that can be 0 |
| Many simultaneous `live_loop`s / dense polyphony | Voices dropped or synth crackle under load; SuperCollider engine has practical simultaneous-voice ceilings | Cap concurrent parts/voices in the layering model; treat "how many parts at once" as a deliberate parameter, not unbounded | Layering several generative parts (bass+drums+lead+pad) simultaneously without a density/voice budget |
| Orphaned `live_loop`s from non-deterministic loop names | Session gets progressively denser and CPU climbs across iterations with no code change | Stable loop-name contract (Pitfall 4) + panic/stop-all command | Any session with more than a handful of pushes without a restart |

## "Looks Done But Isn't" Checklist

- [ ] **Deterministic generation:** Often missing the *emitted-code* seed (`use_random_seed` inside self-contained `live_loop`s) — verify by diffing two `.rb` exports from the same seed, including generators that emit their own in-buffer randomness, not just the Ruby-side structure.
- [ ] **Live push:** Often missing pre-flight syntax validation and a size guard — verify by deliberately pushing a broken/oversized generator and confirming the *Ruby driver* reports the failure, not just silence.
- [ ] **`live_loop` re-push:** Often missing stable loop-naming — verify by pushing the same generator twice in a row and confirming voice count/CPU doesn't climb.
- [ ] **"Runs forever unattended" generative code:** Often missing a stop mechanism reachable without restarting Sonic Pi — verify a panic/stop-all command actually silences a currently-running unattended `live_loop`.
- [ ] **A generator "works":** Often means "runs without error," not "sounds good" — verify by actually listening twice, not just checking for exceptions.

## Recovery Strategies

| Pitfall | Recovery Cost | Recovery Steps |
|---------|-----------------|------------------|
| Orphaned/stacked `live_loop`s (Pitfall 4/5) | LOW, if a panic command exists | Push a `stop`-everything cue for all known loop names; if the command doesn't exist yet, manually hit Stop in the GUI |
| Silent push failure (Pitfall 2/3) | LOW | Re-run the emitted code locally with `Ripper`/compile-check first to isolate whether it's a syntax error or a delivery/size failure |
| Determinism broken by ambient `rand` (Pitfall 7) | MEDIUM | Grep generator code for bare `rand`/`srand`/`Kernel.rand`, replace with threaded `Random.new(seed)` instances; add the seed-diff regression check going forward |
| Framework built before a generator sounds good (Pitfall 9) | MEDIUM-HIGH | Freeze framework work; spend a dedicated session making one existing generator sound good with no interface changes; only then resume generalizing |

## Pitfall-to-Phase Mapping

| Pitfall | Prevention Phase | Verification |
|---------|--------------------|----------------|
| OSC handshake fragility (1) | Live push / streaming phase | Spike survives a Sonic Pi restart and a headless-vs-GUI comparison |
| Silent error reporting (2) | Live push / streaming phase | Deliberately push broken code; confirm the Ruby driver surfaces a failure |
| OSC size limit (3) | Live push / streaming phase | Push a deliberately oversized generator and confirm a guard trips before silent truncation |
| Job stacking / loop-name collisions (4) | Live push phase; contract enforced in pluggable-generator phase | Push the same generator twice; voice/CPU count unchanged |
| Runaway loops / bad sleeps (5) | Core generation phase (shared sleep helper); panic command in live push phase | Fuzz a generator's parameters toward extremes; confirm no zero/negative sleep escapes and stop-all silences it |
| "Sounds like noise" (6) | Core generation phase (shared scale/grid/weighted-choice helpers), reinforced by an explicit musicality pass before/alongside pluggability | Listening-based UAT on every generator, not just "does it run" |
| Determinism traps (7) | Core generation phase | Automated same-seed diff check on generated `.rb` output, including emitted in-buffer seeding |
| Iteration-loop friction (8) | Cross-cutting; mechanics land in live push phase | Time/step-count from "bad state" back to "hearing something correct" |
| Framework-before-sound (9) | Roadmap sequencing (pluggable-generator phase ordered after a working generator) | At least one generator is demonstrably good before the plugin interface is generalized |
| Theory depth over ear (10) | Any musicality-focused phase | Constraint list from Pitfall 6 is the scoped bar; anything beyond is explicitly opt-in, not default |

## Sources

- [sonic-pi-net/sonic-pi — Clarifying usage of OSC /run-code in latest versions (4+), issue #3330](https://github.com/sonic-pi-net/sonic-pi/issues/3330)
- [sonic-pi-net/sonic-pi — /run-code not playing anything, issue #767](https://github.com/sonic-pi-net/sonic-pi/issues/767)
- [sonic-pi-net/sonic-pi — Maximum limit on file size, issue #146](https://github.com/sonic-pi-net/sonic-pi/issues/146) / [re-bump #741](https://github.com/samaaron/sonic-pi/issues/741)
- [sonic-pi-net/sonic-pi — Effects around live loops not reacting to changes when re-running, issue #794](https://github.com/sonic-pi-net/sonic-pi/issues/794)
- [samaaron/sonic-pi — live_loop syncing issue?, issue #1730](https://github.com/samaaron/sonic-pi/issues/1730)
- [Sonic Pi tutorial: A.02 Live Coding](https://github.com/sonic-pi-net/sonic-pi/blob/dev/etc/doc/tutorial/A.02-live-coding.md)
- [Sonic Pi tutorial: A.09 Randomisation](https://github.com/sonic-pi-net/sonic-pi/blob/stable/etc/doc/tutorial/A.09-randomisation.md)
- [Sonic Pi tutorial: 01.1 Live Coding (live_loop sleep requirement)](https://github.com/sonic-pi-net/sonic-pi/blob/dev/etc/doc/tutorial/01.1-Live-Coding.md)
- [Sonic Pi tutorial: 09.3 Multiple Live Loops](https://github.com/sonic-pi-net/sonic-pi/blob/dev/etc/doc/tutorial/09.3-Multiple-Live-Loops.md)
- [in_thread: Run-code not working in headless mode](https://in-thread.sonic-pi.net/t/run-code-not-working-in-headless-mode/7635)
- [in_thread: Sonic Pi Headless Doesn't receive OSC cues](https://in-thread.sonic-pi.net/t/sonic-pi-headless-doesnt-receive-osc-cues/2171)
- [in_thread: Thread got far behind the time - ERROR](https://in-thread.sonic-pi.net/t/thread-got-far-behind-the-time-error/1698)
- [in_thread: Maximum characters limit per buffer?](https://in-thread.sonic-pi.net/t/maximum-characters-limit-per-buffer/229)
- [Kevin Murphy — Revisiting Calling Sonic Pi From Ruby](https://kevinjmurphy.com/posts/revisiting-calling-sonic-pi-from-ruby/)
- [Joey Reyes — Sonic Pi: Randomization](https://joeyreyes.dev/blog/sonic-pi/randomization)
- [spx gem on RubyGems](https://rubygems.org/gems/spx)
- [lpil/sonic-pi-tool (Rust CLI)](https://github.com/lpil/sonic-pi-tool) and [emlyn/sonic-pi-tool (Python rewrite)](https://github.com/emlyn/sonic-pi-tool)
- [Ruby core docs: class Random](https://docs.ruby-lang.org/en/master/Random.html)
- General generative/algorithmic-composition craft (scale quantization, rhythmic grid, register separation, density/rest, weighted choice, voice-leading interval bounds) — synthesized from cross-checked community/practitioner consensus; no single authoritative citation, held at MEDIUM confidence.

---
*Pitfalls research for: Ruby layer generating/driving Sonic Pi code (generative music tooling)*
*Researched: 2026-09-15*
