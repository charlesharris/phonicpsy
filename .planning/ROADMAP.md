# Roadmap: phonicpsy

## Overview

Four phases, each a vertical slice that ends in something you can hear. Phase 1 gets one
hand-built generator all the way to audible sound — theory primitives, a minimal IR, the
emitter, the shipped Sonic Pi runner buffer, live OSC push, file export, and the panic
command — because a silent first phase would be a failed first phase. Phase 2 wraps that
working path in knobs and a hot-reload loop so twisting a parameter or editing a generator
is a couple of seconds, not a restart. Phase 3 stocks the library with three distinct
generators in loop and endless forms. Phase 4 layers independently generated bass, drums,
and lead into one synced, register-separated track. Plugin registry, streamed emission, and
the prose front-end are deliberately v2 — the point of v1 is that one generator sounds good
before any framework exists to make more of them.

## Phases

**Phase Numbering:**
- Integer phases (1, 2, 3): Planned milestone work
- Decimal phases (2.1, 2.2): Urgent insertions (marked with INSERTED)

Decimal phases appear between their surrounding integers in numeric order.

- [ ] **Phase 1: Hear One Generator** - One hand-built generator, in key and on the grid, audible via live OSC push and exported file
- [ ] **Phase 2: Knobs and the Fast Loop** - Parameters you can list, twist, and re-roll; edit-to-sound in seconds with no restart
- [ ] **Phase 3: A Library Worth Listening To** - Three distinct generators, in finite-loop and endless-piece forms
- [ ] **Phase 4: Layered Parts** - Bass, drums, and lead generated independently and playing together in sync

## Phase Details

### Phase 1: Hear One Generator
**Goal**: Run one command and hear a generative loop in Sonic Pi — scale-locked, grid-aligned, reproducible from its seed — reaching the speakers both by live OSC push and by an exported `.rb` file
**Mode:** mvp
**Depends on**: Nothing (first phase)
**Requirements**: GEN-01, GEN-02, GEN-03, GEN-04, GEN-05, GEN-06, GEN-07, GEN-08, IR-01, IR-02, IR-03, EMIT-01, EMIT-02, EMIT-03, EMIT-04, EMIT-05, EMIT-06, EMIT-07, EMIT-08, LOOP-01
**Success Criteria** (what must be TRUE):
  1. One command pushes a generator into a running Sonic Pi and you hear it, with no copy-paste and no manual step beyond having the shipped runner buffer running.
  2. The same generator can be written to a `.rb` file that opens in the Sonic Pi editor and plays, with no Sonic Pi process required to produce the file.
  3. Running the same generator twice with the same seed and parameters produces byte-identical code and sounds identical — a diff of the two emitted files is empty.
  4. What you hear is listenable, not noise: every pitch is in the declared scale, every onset lands on the declared grid, and there are audible rests rather than a wall of notes.
  5. The loop is controllable: re-pushing the same generator replaces the running loop instead of stacking a second one, one command silences everything, and code that is syntactically invalid or too large for an OSC datagram fails loudly before anything is sent.
**Plans**: 3 plans
**Research spike**: Verify the OSC cue-server seam against the installed Sonic Pi v5.x — port 4560, unauthenticated cue, resident runner buffer that `sync`s and `eval`s the payload. The token-gated `/run-code` channel is out of scope and must not be attempted.

Plans:
- [ ] 01-01-PLAN.md — Walking Skeleton tracer: `export --seed N` end to end (theory, IR, arpeggiator, emitter, syntax gate, atomic file write), then determinism, sleep-safety and symbolic-randomness proofs
- [ ] 01-02-PLAN.md — In-key knobs: `--scale` / `--root` / `--out` and a rolled, printed seed, validated loudly at the CLI against Sonic Pi's full 157-name scale list, with the scale-lock proven structurally
- [ ] 01-03-PLAN.md — Live slice: the shipped `runner.rb` buffer, OSC ping/push/ack, `play` with pre-flight syntax and size gates, a stable loop name, and `stop` via the per-step stop flag

### Phase 2: Knobs and the Fast Loop
**Goal**: Twist a parameter, re-roll a seed, or edit a generator file and hear the difference within a couple of seconds, without editing emission code or restarting the Ruby session
**Mode:** mvp
**Depends on**: Phase 1
**Requirements**: LOOP-02, LOOP-03, LOOP-04, LOOP-05, LOOP-06, LOOP-07, LIB-07
**Success Criteria** (what must be TRUE):
  1. You can ask a generator what its parameters are at runtime and see each name with its legal value range, without opening its source file.
  2. You can set a parameter to a new value and hear the result, without editing any generator or emitter code.
  3. One command re-rolls the seed and plays the resulting variant — recognisably the same idea, different in detail.
  4. Saving an edit to a generator file and re-playing it requires no restart of the Ruby session, and edit-to-sound completes in a couple of seconds with audio continuing between iterations.
  5. A seed plus its parameter set can be saved as a favorite and recalled later to reproduce that exact output.
**Plans**: 2 plans

Plans:
- [ ] 02-01: Parameter declarations, runtime inspection, setting values, seed re-roll
- [ ] 02-02: Hot-reload session and favorites — edit-to-sound loop under a couple of seconds

### Phase 3: A Library Worth Listening To
**Goal**: Three distinct generators you would actually choose to put on, each available as a finite repeating loop or as an endless piece that keeps evolving
**Mode:** mvp
**Depends on**: Phase 2
**Requirements**: LIB-01, LIB-02, LIB-03
**Success Criteria** (what must be TRUE):
  1. Three distinct generators each play something worth listening to for more than a minute — judged by ear, not by "it runs without erroring".
  2. A generator can produce a finite repeating loop that comes back around cleanly rather than drifting or clipping at the seam.
  3. A generator can produce an endless piece that is still changing two minutes in and has not fallen into identical repetition.
  4. Every generator in the library honors the Phase 1 guarantees — in key, on the grid, with rests, and byte-identical output for a given seed.
**Plans**: 2 plans

Plans:
- [ ] 03-01: Generators two and three, built by divergence from the first, each with a listening pass
- [ ] 03-02: Finite-loop and endless-piece forms across the library

### Phase 4: Layered Parts
**Goal**: Generate bass, drums, and lead independently and hear them play together as one coherent track, locked to a shared tempo with each layer still distinguishable
**Mode:** mvp
**Depends on**: Phase 3
**Requirements**: LIB-04, LIB-05, LIB-06
**Success Criteria** (what must be TRUE):
  1. Bass, drums, and lead can each be generated and heard on their own, without the others present.
  2. Parts generated independently and played together stay locked to one shared BPM, with no audible drift after several minutes of playback.
  3. In the layered mix each part stays distinguishable by ear — the registers are separated rather than collapsing into mud.
**Plans**: 1 plan

Plans:
- [ ] 04-01: Part generators, shared tempo/sync contract, register separation, layered listening pass

## Progress

**Execution Order:**
Phases execute in numeric order: 1 → 2 → 3 → 4

| Phase | Plans Complete | Status | Completed |
|-------|----------------|--------|-----------|
| 1. Hear One Generator | 0/3 | Not started | - |
| 2. Knobs and the Fast Loop | 0/2 | Not started | - |
| 3. A Library Worth Listening To | 0/2 | Not started | - |
| 4. Layered Parts | 0/1 | Not started | - |

## Notes

**Sequencing rationale (from research):**
- Phase 1 must produce audible sound. Framework-before-sound is the characteristic way projects of this shape die, so the plugin registry is v2 and generators two and three (Phase 3) arrive by copy-paste divergence from a generator that already sounds good.
- Musicality constraints (GEN-06/07/08) and the determinism seed-diff check ship *with* the first generator in Phase 1, not as a later polish pass. "Every engineering piece works and the output is unlistenable" is the characteristic failure of this genre.
- The stop-all/panic command (LOOP-01) is in Phase 1 because runaway `live_loop`s are the most likely day-one failure and the fix is cheap.
- IR-03 is a design property, not a feature: the IR shape introduced in Phase 1 must let one generator drive both self-contained and (v2) streamed emission without being rewritten. It is satisfied by how the IR is shaped, and verified by review rather than by a runtime behavior.

**Deferred to v2 (not in this roadmap):** streamed emission (STRM-01..04), the plugin registry (PLUG-01..03), the prose front-end (PROSE-01..03), and parameter interpolation / grammars (EXPL-01..02).
