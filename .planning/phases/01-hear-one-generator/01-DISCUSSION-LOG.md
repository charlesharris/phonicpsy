# Phase 1: Hear One Generator - Discussion Log

> **Audit trail only.** Do not use as input to planning, research, or execution agents.
> Decisions are captured in CONTEXT.md — this log preserves the alternatives considered.

**Date:** 2026-09-15
**Phase:** 1-Hear One Generator
**Areas discussed:** The first generator's identity, Command surface, Sound defaults, Transports & failure feedback

---

## Area Selection

| Option | Description | Selected |
|--------|-------------|----------|
| The first generator's identity | What musical idea is generator #1 | ✓ |
| Command surface | What "one command" looks like, and where stop-all lives | ✓ |
| Sound defaults | Scale, root, BPM, grid, density, synth | ✓ |
| Transports & failure feedback | Push acks, runner buffer delivery, export file shape | ✓ |

**User's choice:** All four areas.

---

## The First Generator's Identity

### Q1 — What kind of musical idea should generator #1 be?

| Option | Description | Selected |
|--------|-------------|----------|
| Arpeggiator over a chord | Fastest path to listenable sound; exercises scale + chord + grid + weighted choice; diverges cleanly in Phase 3; already ARCHITECTURE.md's worked example | ✓ |
| Acid bassline | Strong groove identity but narrower — one register, one voice; Phase 4 wants bass as its own part anyway | |
| 16-bit JRPG loop | Highest ceiling on "I'd put this on" but needs progression logic beyond a single chord | |
| You decide | Claude picks | |

**User's choice:** Arpeggiator over a chord.

### Q2 — One chord held, or a progression?

| Option | Description | Selected |
|--------|-------------|----------|
| Short fixed progression | 2–4 chords cycling; the difference between a demo and something you'd leave running | |
| One chord, held | Simplest thing satisfying every Phase 1 requirement; risk of going static fast | |
| Progression as a parameter | Build the machinery, default it to something; edges into Phase 2 territory | ✓ |

**User's choice:** Progression as a parameter.
**Notes:** Claude flagged that this risks absorbing Phase 2's LOOP-02 work and asked a scoping follow-up rather than accepting it as-is.

### Q3 — How far should "as a parameter" go in Phase 1?

| Option | Description | Selected |
|--------|-------------|----------|
| Plain keyword arg with a default | Ordinary Ruby keyword argument; no param DSL, no runtime introspection; Phase 2's LOOP-02/03/04 keep real work | ✓ |
| Full param DSL, pulled forward | Build Param descriptors now; framework-before-sound, against Pitfall 9 and the roadmap's sequencing rationale | |
| You decide | Claude picks the lightest swappable thing | |

**User's choice:** Plain keyword arg with a default.
**Notes:** This became the general pattern for the phase — later reused for `--scale` / `--root`.

### Q4 — Where should seeded randomness live?

| Option | Description | Selected |
|--------|-------------|----------|
| Note order + rests | Weighted Choice per step, Weighted rest/note for gaps; rhythm stays on a fixed Euclidean grid | ✓ |
| Note order, rests, and octave | Adds weighted octave jumps; more life per seed, pre-answers Phase 4's register concern | |
| Everything including rhythm | Onsets also vary; most variation but highest Pitfall 6 risk and makes GEN-07 harder to demonstrate | |

**User's choice:** Note order + rests.

---

## Command Surface

### Q1 — Primary way to invoke a generator?

| Option | Description | Selected |
|--------|-------------|----------|
| CLI binary, subcommands | `bin/phonicpsy play/export/stop`; matches "run one command" literally; already in ARCHITECTURE.md's structure | ✓ |
| Pry session only | STACK.md calls pry "the actual outer loop", but makes criterion 1 awkward and leaves no scriptable entry point | |
| Rake tasks | Zero parsing code, but awkward bracket syntax in zsh and startup cost in the inner loop | |

**User's choice:** CLI binary, subcommands.

### Q2 — Argument parsing?

| Option | Description | Selected |
|--------|-------------|----------|
| optparse (stdlib) | Zero gems, fine at this size, keeps core deps at osc-ruby only | ✓ |
| Hand-rolled ARGV parsing | ~20 lines, minimal, but gets rewritten when Phase 2 adds param flags | |
| A CLI gem (thor / dry-cli) | Nicer DSL, but adds a dependency and STACK.md evaluated no CLI gem | |

**User's choice:** optparse (stdlib).

### Q3 — Seed behavior with no `--seed`?

| Option | Description | Selected |
|--------|-------------|----------|
| Random seed, always printed | Fresh variant each run, recoverable by passing the seed back; what LOOP-05 builds on | ✓ |
| Fixed default seed | Most predictable, trivial criterion-3 demo, but re-running explores nothing | |
| Random, printed, and saved to a file | Adds a `--last` replay affordance, which duplicates Phase 2's LIB-07 favorites | |

**User's choice:** Random seed, always printed.

### Q4 — Shape of the panic / stop-all command?

| Option | Description | Selected |
|--------|-------------|----------|
| `phonicpsy stop`, via the runner | Distinct OSC cue the runner handles; same transport, nothing extra to install | ✓ |
| Separate tiny binary | Fastest to reach in a panic, but a second entry point duplicating OSC setup | |
| Stop cue plus Sonic Pi's own hotkey | Honest about the hard kill, but relying on the GUI conflicts with criterion 1's spirit | |

**User's choice:** `phonicpsy stop`, via the runner.

---

## Sound Defaults

### Q1 — Default scale and root?

| Option | Description | Selected |
|--------|-------------|----------|
| Minor pentatonic, A | No semitone clashes; hardest to make sound wrong | |
| Natural minor (aeolian), A | More melodic range, real dissonance risk against chord tones | |
| Dorian, D | Distinctive modal color, between the other two on range and risk | |

**User's choice:** *Free text* — "can we make this something we can change? e.g. implement all three scale/root setups and let them change via the UI?"
**Notes:** Claude confirmed this fits the D-03 pattern already established (plain keyword arg exposed as a CLI flag) and noted it is nearly free, since GEN-04 already requires emitting Sonic Pi's own `scale`. Split into two follow-up questions rather than treated as a rejection of the question.

### Q1a — Default when neither flag is passed?

| Option | Description | Selected |
|--------|-------------|----------|
| Minor pentatonic, A | Safest default specifically so the first sound the tool ever makes isn't the unlistenable one | ✓ |
| Dorian, D | More characterful first impression, more ear-tuning to guarantee criterion 4 | |
| Natural minor, A | Most conventional and legible, clearest harmonic movement | |

**User's choice:** Minor pentatonic, A.

### Q1b — Which scales does `--scale` accept?

| Option | Description | Selected |
|--------|-------------|----------|
| Pass any name through to Sonic Pi | ~40 names, validated in the theory core; unknown names fail loudly at the CLI | ✓ |
| Just the three discussed | Curated, vetted-by-ear set; smaller surface to test | |
| You decide | Claude picks | |

**User's choice:** Pass any name through to Sonic Pi.

### Q2 — Default tempo and grid?

| Option | Description | Selected |
|--------|-------------|----------|
| 110 BPM, 16th-note grid | Mid-tempo; enough spread slots to be interesting without blurring; room for rests | ✓ |
| 90 BPM, 8th-note grid | Sparser and clearer, but fewer slots to distinguish Euclidean patterns | |
| 128 BPM, 16th-note grid | Driving and energetic; fast enough that pitch mistakes blur past, cutting both ways | |

**User's choice:** 110 BPM, 16th-note grid.

### Q3 — How should density / rests default (GEN-08)?

| Option | Description | Selected |
|--------|-------------|----------|
| Euclidean onsets set the density | Empty spread slots are the rests; one mechanism serves GEN-05/07/08 together | ✓ |
| Euclidean grid plus a rest probability | More variation per seed, but two overlapping mechanisms that can thin the groove | |
| Hard notes-per-bar ceiling | Most literal reading of GEN-08, but a blunt post-filter cutting at arbitrary points | |

**User's choice:** Euclidean onsets set the density.

### Q4 — Default synth?

| Option | Description | Selected |
|--------|-------------|----------|
| `:prophet` | Warm, forgiving of short releases, very hard to make harsh | ✓ |
| `:pluck` | Clear note separation so rhythm and rests read distinctly; drier, can sound skeletal | |
| `:blade` | Closest to the JRPG flavor, but foregrounded and less pleasant left running | |

**User's choice:** `:prophet`.

---

## Transports & Failure Feedback

### Q1 — How do you learn a push actually landed (Pitfall 2)?

| Option | Description | Selected |
|--------|-------------|----------|
| Runner acks back over OSC | begin/rescue around eval, cue success or the error back to a port the CLI listens on | ✓ |
| Fire-and-forget, pre-flight only | Simplest, but leaves exactly Pitfall 2's hole — a runtime error looks like a dead runner | |
| Read Sonic Pi's log file | No runner changes, but couples to Sonic Pi's internal file layout and log format | |

**User's choice:** Runner acks back over OSC.

### Q2 — What happens when the runner buffer isn't running at all?

| Option | Description | Selected |
|--------|-------------|----------|
| Ping first, fail with instructions | Cheap ping before sending; no ack prints the fix plus runner.rb's path | ✓ |
| Push, then warn on missing ack | One round trip, faster happy path, but can't distinguish a dead runner from failed code | |
| Ping only on failure | Fast happy path plus accurate diagnosis, at the cost of the most branching logic | |

**User's choice:** Ping first, fail with instructions.

### Q3 — Where does the runner buffer live and how does it get into Sonic Pi (EMIT-06)?

| Option | Description | Selected |
|--------|-------------|----------|
| `runner.rb` in repo root, manual Run | Versioned in git, depends on nothing inside Sonic Pi's install; failure message prints its path | ✓ |
| `phonicpsy runner --print` | Same file plus a print/clipboard subcommand; nicer for a step repeated each Sonic Pi restart | |
| Install into Sonic Pi's `init.rb` | Manual step disappears entirely, but writes into Sonic Pi's own config and is harder to undo | |

**User's choice:** `runner.rb` in repo root, manual Run.

### Q4 — What header should exported `.rb` files carry?

| Option | Description | Selected |
|--------|-------------|----------|
| Seed, generator, params only | Everything needed to regenerate, nothing that varies; protects criterion 3's byte-identical diff | ✓ |
| No header at all | Trivially deterministic and cleanest to open, but a saved file doesn't say which seed made it | |
| Full header including timestamp | Most useful as a record; breaks criterion 3 unless the test strips the header before diffing | |

**User's choice:** Seed, generator, params only.

### Q5 — Where do exports land and what are they named?

| Option | Description | Selected |
|--------|-------------|----------|
| `out/` with generator + seed name | Self-describing directory, no collisions, same-seed re-export overwrites identically | ✓ |
| Explicit `--out`, no default | No stray directories or surprise writes, but adds typing to every export | |
| `out/` with a name you pass | More meaningful filenames, but same-name exports silently overwrite different seeds | |

**User's choice:** `out/` with generator + seed name.

### Q6 — Where should the EMIT-07 payload cap be set?

| Option | Description | Selected |
|--------|-------------|----------|
| Conservative fixed cap (~8KB) | Below PITFALLS.md's ~8–9KB truncation point, fails loudly with the byte count | ✓ |
| Cap, with export as the fallback | More helpful, but blurs the loud failure EMIT-07 asks for | |
| Measure the real ceiling first | Evidence-based, could ride the scheduled OSC spike, but Phase 1 never approaches the limit | |

**User's choice:** Conservative fixed cap (~8KB).

### Q7 — How should `live_loop` names be derived (EMIT-04)?

| Option | Description | Selected |
|--------|-------------|----------|
| Generator name only | Stable across every re-push; exactly criterion 5's behavior | |
| Generator name plus seed | Different seeds become different loops — the exact stacking EMIT-04 prevents | ✓ (initially) |
| Generator name, with `--as` override | Satisfies EMIT-04 by default, layering available; a small Phase 4 pull-forward | |

**User's choice (initial):** Generator name plus seed.
**Notes:** Claude had flagged the conflict in the option description and, after the choice, surfaced it explicitly: EMIT-04 and Success Criterion 5 both require stable names, so seed-suffixed loops would leave every previous loop playing and the phase verifier would fail the requirement. A resolution question followed.

### Q7a — Resolving the EMIT-04 conflict

| Option | Description | Selected |
|--------|-------------|----------|
| Stable name, `--as` to layer | Satisfies EMIT-04; layering becomes something you ask for rather than clean up after | |
| Keep seed in the name, amend EMIT-04 | Coherent additive design, but requires editing REQUIREMENTS.md and ROADMAP.md | |
| Stable name, strictly | Generator name only, no layering affordance; deliberate layering waits for Phase 4 | ✓ |

**User's choice:** Stable name, strictly.
**Notes:** Resolved with no requirement changes. EMIT-04 and Criterion 5 stand as originally written.

---

## Claude's Discretion

The user made an explicit choice on every question — no "you decide" answers were given.
The following were never asked and are left to the researcher and planner:

- Ack timeout duration and the exact OSC paths for push / ping / stop / ack
- Note release and sustain values, octave range, and loop length in bars
- Whether any FX (e.g. reverb) appear in the emitted code
- The specific default chord progression and its length
- Euclidean onset count (e.g. 7-in-16) for the default pattern
- How the walk order over chord notes is structured
- How the seed-diff determinism check is run in plan 01-03, and what the listening pass consists of

---

## Deferred Ideas

- Full parameter DSL with declared names and ranges — Phase 2 (LOOP-02/03/04)
- `--as` flag for deliberately naming a loop so two can play at once — Phase 4 (LIB-04/05/06)
- Saving the last seed for a `--last` replay flag — Phase 2 (LIB-07 favorites)
- `phonicpsy runner --print` / clipboard copy of `runner.rb` — ergonomic nicety, not needed now
- Spiking the true OSC truncation ceiling — the conservative cap suffices until approached
- Per-seed rhythmic variation and weighted octave jumps — rhythm stays fixed so GEN-07 stays demonstrable; octave movement suits Phase 4's LIB-06
- 16-bit Final Fantasy loops, noise walls, "weirdest version of a described vibe" — PROJECT.md flavors not chosen for generator #1; live candidates for Phase 3
