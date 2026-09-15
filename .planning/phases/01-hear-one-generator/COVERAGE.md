# API Coverage — Sonic Pi v5.0.0 OSC surface

> Full coverage by default. Opt-outs are explicit, reasoned decisions.

The deterministic detector reported `detected: false` over this phase's ROADMAP scope, so this
matrix is not a required artifact. It is recorded anyway because the phase *does* drive an
external process (Sonic Pi) through a public interface, and the subtraction record is the useful
half: REQUIREMENTS.md already forbids one whole channel, and leaving that undecided-looking would
be the exact hole this gate exists to close.

**Capability surface:** Sonic Pi's externally reachable OSC interfaces, as read from the installed
v5.0.0 app bundle (`daemon.rb`, `spider-server.rb`, `osc_api.rb`, tutorials 12.1 / 12.2).

| capability | decision | reason |
|---|---|---|
| cue-server send (UDP 4560, unauthenticated, `/osc*/...` sync) | INTEGRATE | The verified working seam — carries `run`, `ping`, and `stop` for this project |
| cue-server receive in Sonic Pi (`sync "/osc*/<path>"`) | INTEGRATE | `runner.rb` is built on it (EMIT-06) |
| Sonic Pi → outbound OSC (`use_osc` + `osc` from a buffer) | INTEGRATE | Carries the ack/error reply back to the CLI (D-15, Pitfall 2 mitigation) |
| `/run-code` (token-gated GUI channel) | OPT-OUT | Verified broken against v5.x — token behind a dynamically-allocated port with no stable discovery path. Explicitly listed in REQUIREMENTS.md Out of Scope; the phase research spike forbids attempting it |
| `/stop-all-jobs` (token-gated) | OPT-OUT | Same token-gated channel as `/run-code`. LOOP-01 is met instead by a Time State stop flag the emitted loop checks every grid step (D-08) |
| `/save-and-run-buffer`, `/buffer-*` (token-gated GUI buffer control) | OPT-OUT | Same token gate; and writing into Sonic Pi's buffers would cross the "public interfaces only, never fork or patch Sonic Pi" project constraint |
| Networked OSC from non-loopback hosts (Preferences → IO → "Allow OSC from other computers") | OPT-OUT | Single-user local-only tool (PROJECT.md Scope). Left at Sonic Pi's verified-safe default; no code path in this project enables it. See threat `T-01-01` |
| MIDI in/out (`midi_*`, `/midi*` cues) | OPT-OUT | This project never touches MIDI — REQUIREMENTS.md Out of Scope ("Audio rendering or DSP"; the layer only ever produces code) |
| Ableton Link (`link_api.rb`) | OPT-OUT | Not needed yet — Phase 4's shared-BPM requirement (LIB-05) is met inside one Sonic Pi process by `use_bpm`; revisit only if syncing to an external clock ever matters |
| Gamepad API (`gamepad_api.rb`) | OPT-OUT | No controller input in scope for any v1 phase |
| Headless boot harness (`headless_boot.rb`) | OPT-OUT | Depends on Sonic Pi's internal installed file layout rather than its documented cue-server interface — STACK.md Alternatives Considered rejects it for exactly that reason |
