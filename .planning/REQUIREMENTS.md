# Requirements: phonicpsy

**Defined:** 2026-09-15
**Core Value:** Twist a knob or edit a generator and hear the result in seconds — the tool disappears and you're just playing.

## v1 Requirements

Requirements for initial release. Each maps to roadmap phases.

### Generation Core

- [ ] **GEN-01**: Same seed and same parameters produce byte-identical emitted code across runs
- [ ] **GEN-02**: Generator-side randomness uses a seeded `Random` instance derived from the master seed, never global `Kernel.rand`
- [ ] **GEN-03**: Self-contained emitted code carries a `use_random_seed` derived from the master seed as the first statement inside the `live_loop` body
- [ ] **GEN-04**: Generators compose from scale and chord primitives that emit Sonic Pi's own `scale`/`chord` rather than reimplementing them in Ruby
- [ ] **GEN-05**: Generators compose from rhythm primitives, including Euclidean patterns emitted as Sonic Pi's `spread`
- [ ] **GEN-06**: Generated pitches are quantized to the generator's declared scale
- [ ] **GEN-07**: Generated note onsets align to a declared rhythmic grid
- [ ] **GEN-08**: Random selection is weighted rather than uniform, and a density limit caps notes per bar so output contains rests

### Intermediate Representation

- [ ] **IR-01**: Generators build a structured representation of musical events; they never emit Sonic Pi source strings directly
- [ ] **IR-02**: The representation holds randomness symbolically (choice, range, weighted) as data rather than as already-resolved values
- [ ] **IR-03**: One generator implementation can drive both self-contained emission and future streamed emission without being rewritten

### Emission and Transport

- [ ] **EMIT-01**: An emitter renders the intermediate representation into Sonic Pi source
- [ ] **EMIT-02**: Self-contained mode renders symbolic randomness as Sonic Pi randomness calls inside the emitted `live_loop`
- [ ] **EMIT-03**: Emitted source is syntax-validated before it is pushed or written, and invalid source fails loudly rather than silently
- [ ] **EMIT-04**: `live_loop` names are stable across re-pushes, so re-pushing replaces the running loop instead of stacking a second one
- [ ] **EMIT-05**: Generated code can be pushed into a running Sonic Pi over the OSC cue server and is audible without any copy-paste
- [ ] **EMIT-06**: The repo ships the Sonic Pi runner buffer that receives cued code, as a versioned artifact rather than pasted-in setup lore
- [ ] **EMIT-07**: Payload size is checked against the OSC datagram ceiling before sending; an oversized push reports an error instead of failing silently
- [ ] **EMIT-08**: Generated code can be written to a `.rb` file that opens and runs in the Sonic Pi editor, with no Sonic Pi process required

### Iteration Loop

- [ ] **LOOP-01**: A single command stops every running loop
- [ ] **LOOP-02**: Generators declare their parameters with names and value ranges
- [ ] **LOOP-03**: A generator's parameters can be listed and inspected at runtime
- [ ] **LOOP-04**: Parameter values can be changed and re-heard without editing generator source
- [ ] **LOOP-05**: One command re-rolls the seed and plays the resulting variant
- [ ] **LOOP-06**: Editing a generator file and re-running it requires no restart of the Ruby session
- [ ] **LOOP-07**: Edit-to-sound completes in a couple of seconds, with audio continuing between iterations

### Generator Library

- [ ] **LIB-01**: At least three distinct generators exist that produce output worth listening to
- [ ] **LIB-02**: A generator can produce a finite repeating loop
- [ ] **LIB-03**: A generator can produce an endless generative piece that does not repeat identically
- [ ] **LIB-04**: Single parts (bass, drums, lead) can be generated independently of one another
- [ ] **LIB-05**: Independently generated parts play together on a shared BPM and stay in sync
- [ ] **LIB-06**: Layered parts occupy separated registers so individual layers stay distinguishable
- [ ] **LIB-07**: A seed and parameter set can be saved as a favorite and later recalled to reproduce that output exactly

## v2 Requirements

Deferred to future release. Tracked but not in current roadmap.

### Streamed Emission

- **STRM-01**: A resolver concretizes symbolic randomness in the intermediate representation using the seeded Ruby RNG
- **STRM-02**: The Ruby process stays running and pushes freshly composed code into Sonic Pi mid-playback
- **STRM-03**: A v1 generator runs in streamed mode without modification
- **STRM-04**: Streamed chunks arrive early enough that playback never gaps

### Plugin System

- **PLUG-01**: Generators self-register via an `inherited` hook and a directory convention
- **PLUG-02**: Adding a generator requires no edit to any core file
- **PLUG-03**: The plugin contract is extracted from the working v1 generators rather than designed ahead of them

### Prose Front-End

- **PROSE-01**: An optional LLM path translates a prose description into a structured parameter set
- **PROSE-02**: The prose path is strictly optional — every generator remains fully usable without it
- **PROSE-03**: The LLM never produces emitted code, only parameter values

### Exploration

- **EXPL-01**: Parameters can be interpolated to morph between two saved parameter sets
- **EXPL-02**: Generative grammars and Markov-style weighted transition tables are available as generator building blocks

## Out of Scope

Explicitly excluded. Documented to prevent scope creep.

| Feature | Reason |
|---------|--------|
| LLM writing Sonic Pi Ruby directly | Musical intelligence stays deterministic; an LLM only ever produces a parameter set. Keeps output valid and reproducible |
| Structured song forms (intro/verse/chorus/bridge) | The unit of generation is loops, parts, and endless pieces. Arrangement over minutes is a different problem |
| Multi-user anything (auth, accounts, sharing, deploy) | Single-person playground running locally |
| Audio rendering or DSP | Sonic Pi and SuperCollider make the sound; this layer only ever produces code |
| Replacing, forking, or patching Sonic Pi | Used as-is through public interfaces only |
| A polished GUI | Nothing that slows the iteration loop. Interface stays as thin as it can be |
| Reimplementing `scale`, `chord`, `ring`, `spread` in Ruby | Sonic Pi's DSL already provides these well. Emit them, don't rebuild them |
| A custom pattern-algebra DSL (Tidal-style) | Multi-year effort that duplicates Sonic Pi's own `ring` and Enumerator idioms |
| A competing scheduler or quantization system | Fights Sonic Pi's timing model, which already owns scheduling |
| Training an ML model from a music corpus | Reintroduces exactly the non-determinism the project rejected |
| Driving Sonic Pi via `sonic-pi-tool` or the `/run-code` OSC channel | Verified broken against Sonic Pi v5.x — token-gated behind a dynamically-allocated port with no stable discovery path |
| Deep music-theory completeness (voice leading, counterpoint, functional harmony) | The cheap constraint list buys most of the musicality; theory depth is the classic way these projects stall before anything is heard |
| Windows and Linux support | Development target is macOS. Revisit only if it ever matters |

## Traceability

Which phases cover which requirements. Updated during roadmap creation.

| Requirement | Phase | Status |
|-------------|-------|--------|
| (pending roadmap creation) | — | — |

**Coverage:**
- v1 requirements: 33 total
- Mapped to phases: 0
- Unmapped: 33 ⚠️

---
*Requirements defined: 2026-09-15*
*Last updated: 2026-09-15 after initial definition*
