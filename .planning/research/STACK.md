# Stack Research

**Domain:** Algorithmic-composition layer that emits and drives Sonic Pi code from Ruby
**Researched:** 2026-09-15
**Confidence:** HIGH for the Sonic Pi integration architecture and RubyGems maintenance data (verified directly against primary sources: a full clone of `sonic-pi-net/sonic-pi` at tag `v5.0.0` and current `dev` HEAD, plus live `rubygems.org` API and `gh api` GitHub metadata). MEDIUM/LOW for anything sourced only from web-search summaries, flagged inline.

## Headline Finding — Read This Before Anything Else

**`sonic-pi-tool` and every fork/rewrite of it (Rust, Python, and the Ruby `sonic-pi-cli`) are broken against current Sonic Pi.** They all hardcode UDP port `4557` and send `/run-code` with no auth token. That was correct for Sonic Pi ≤3.x. Sonic Pi 4.x introduced a security fix — confirmed by reading it directly in the `sonic-pi-net/sonic-pi` source — that gates `/run-code`, `/stop-all-jobs`, and friends behind a **dynamically-allocated port plus a random 32-bit token**, generated fresh on every boot and printed exactly once to a process's stdout, with no file on disk that exposes it while a normally-launched Sonic Pi.app is running. PROJECT.md's stated assumption ("`sonic-pi-tool` is the established CLI for driving a running instance") does not hold for the version you will actually install today (v5.0.0, released 2026-08-07). See "Sonic Pi Integration Seams" below for the verified, working alternative.

## Recommended Stack

### Core Technologies

| Technology | Version | Purpose | Why Recommended |
|------------|---------|---------|-----------------|
| Ruby | 4.0.x (current stable; floor 3.2+) | Generator language, runtime for the whole tool | Matches Sonic Pi's DSL syntax per PROJECT.md's own constraint; no need to match Sonic Pi's *internal* vendored Ruby version since the two processes only ever talk over UDP/OSC, never share a process |
| `osc-ruby` gem | 1.1.5 | Send OSC messages to Sonic Pi | Actively maintained (GitHub repo pushed 2025-11-16, gem released 2024-07-21, 62,552 downloads), tiny synchronous API, no forced dependency on EventMachine for the client path |
| Hand-rolled music-theory core | n/a (your code) | Scales, chords, rhythm primitives | No maintained Ruby equivalent to Python's music21/mingus exists (verified below) — this is a "write 100–150 lines yourself" category, not a library gap to paper over |

### Supporting Libraries

| Library | Version | Purpose | When to Use |
|---------|---------|---------|-------------|
| `listen` | 3.10.0 | Watch `generators/` for edits and trigger reload | Automatic hot-reload path; pulls in `rb-fsevent` on macOS for native, near-instant FSEvents notification |
| `pry` | 0.16.0 | Long-lived interactive session, manual `load` reload | The actual outer loop of the whole tool — keep one session open, hold the live OSC client, twist knobs and re-invoke generators from the prompt |
| `anthropic` (official SDK) | 1.71.0 | Optional prose → structured parameter set | Only loaded behind the optional LLM feature flag; never touches the code-emission path |

### Development Tools

| Tool | Purpose | Notes |
|------|---------|-------|
| `ripper` (Ruby stdlib) | Syntax-validate emitted Sonic Pi/Ruby source | `require "ripper"`; no gem, no subprocess, sub-millisecond |
| Minitest (Ruby stdlib) | Golden-file snapshot tests, seeded-determinism tests | Ships with Ruby; no reason to add RSpec or a snapshot-testing gem for comparing plain strings |

## Sonic Pi Integration Seams (verified against primary source)

Verified by cloning `sonic-pi-net/sonic-pi` (both the `v5.0.0` release tag and current `dev` HEAD, commit `cba6a37`, dated 2026-09-12 — the two match on every file inspected, so this is the architecture of the version you'll install today) and reading the actual implementation, not docs summaries.

**Two separate OSC surfaces exist, with very different stability and auth characteristics:**

1. **The internal control channel** (`/run-code`, `/stop-all-jobs`, `/save-and-run-buffer`, `/stop-job`, `/load-buffer`, `/save-buffer`, `/buffer-newline-and-indent`) — implemented in `app/api/src/sonicpi_api.cpp` (the GUI's C++ API layer) and driven by `app/server/ruby/bin/daemon.rb`. Every message on this channel **requires a 32-bit signed integer token as its first argument**. Confirmed directly in source, e.g.:
   ```cpp
   void SonicPiAPI::RunCode(const std::string& code, const std::string& workspace, bool silent) {
       oscpkt::Message msg("/run-code");
       msg.pushInt32(m_token);
       msg.pushStr(code);
       ...
   }
   ```
   The port this channel listens on, and the token itself, are **allocated fresh on every boot** by a "Boot Daemon" process and printed exactly once to that daemon's stdout (`app/server/ruby/bin/daemon.rb` line ~266: `puts "#{daemon_port} #{gui_listen} #{gui_send} #{scsynth_port} #{cues_port} #{token}"`). Whoever spawns the daemon (normally the Qt GUI app) captures that line over a pipe. It is **not** written to any log file at boot time — the one `Util.log` call that would record it is explicitly commented out in the source, and the only place the token is logged to `~/.sonic-pi/log/debug.log` is in the shutdown handler, after the fact. **There is no supported, stable way for an external process to discover the token of an already-running, normally-launched Sonic Pi.app.** This channel is, in practice, private to the GUI process that booted it.

2. **The external cue-in server** (documented in `etc/doc/tutorial/12.2-Sending-OSC.md`, surfaced in Preferences → IO → OSC) — listens on a fixed, user-visible, **unauthenticated** UDP port (default **4560**), togglable and loopback-restrictable from the GUI, and re-broadcasts incoming OSC as cue events that a running `live_loop` picks up via `sync "/osc*/..."` or `get[...]`. This surface has not changed shape across versions and is the one thing in this whole system that qualifies as a genuinely stable "public interface."

**The working pattern — verified against a real, currently-maintained project** (`abhishekjairath/sonic-pi-mcp`, GitHub repo pushed 2026-03-22): don't fight the token. Paste a small, self-contained "runner" `live_loop` into Sonic Pi once, which listens on the untokenized cue-in port and `eval`s whatever code string arrives:

```ruby
# runner.rb — paste into a Sonic Pi buffer once, hit Run, leave it running.
live_loop :phonicpsy_runner do
  code = sync "/osc*/phonicpsy-run"
  begin
    eval(code)
  rescue Exception => e
    puts "phonicpsy runner error: #{e.message}"
  end
end
```

Then your Ruby generator process just does:

```ruby
require "osc-ruby"
client = OSC::Client.new("localhost", 4560)
client.send(OSC::Message.new("/phonicpsy-run", generated_code_string))
```

No token, no dynamic-port discovery, no fragile dependency on Sonic Pi's internal process layout — only the documented cue-server surface. This satisfies PROJECT.md's constraint ("driven only through public interfaces... no forking or patching Sonic Pi itself") more literally than trying to reverse-engineer the token channel would. It also happens to double as a working example of the "self-contained generative `live_loop`" artifact the generator layer is required to be able to emit — the runner buffer isn't a special case bolted onto the architecture, it's the same shape of thing the tool produces anyway.

**For a hard stop** (`/stop-all-jobs`) you have two options: (a) route it through the same runner pattern — send an OSC cue the runner interprets as `stop_all_jobs` inside its own thread — or (b) accept that a true global stop is a GUI-side action (click Stop, or `Cmd+.`) since the token-gated `/stop-all-jobs` is unreachable from outside the GUI process. Recommend (a): extend the runner to listen on a second cue path and call Sonic Pi's own `live_loop`/`in_thread` `.kill`-equivalent (`stop` inside each named loop, gated on a shared `set`/`get` flag), the same technique the reference runner buffer above already uses for crossfades.

**Escape hatch for advanced/unattended use:** Sonic Pi's own source ships a first-party headless boot harness — `app/server/ruby/bin/headless_boot.rb` (present in the `v5.0.0` tag, used by Sonic Pi's own CI via `headless-run.rb`/`headless-record.rb`) — that spawns the Boot Daemon itself and reads the token straight off its own stdout pipe, with zero GUI involved. This sidesteps the discovery problem entirely because *you* are the process that spawned the daemon. It's real, currently-shipped, first-party code, not a hack — but it ties you to Sonic Pi's internal installed file layout (an implementation detail, not a documented API, and it can move between releases), and it means no visible Sonic Pi window at all. Given PROJECT.md explicitly deprioritizes GUI and wants the tool to "disappear," this is worth knowing about, but the cue-server runner pattern above is the lower-risk default: it only depends on a genuinely documented, version-stable interface.

**Version stability note:** the token requirement is a deliberate security fix (closing what was, in Sonic Pi ≤3.x, an unauthenticated arbitrary-code-execution-over-UDP hole). It is very unlikely to be reverted. Build for it as a permanent fact of the current architecture, not a transient inconvenience.

## `sonic-pi-tool` Maintenance Status (verified)

| Project | Language | Last activity | Verdict |
|---|---|---|---|
| `lpil/sonic-pi-tool` | Rust | pushed 2024-06-16, no GitHub releases/tags, 234★ | Source hardcodes port 4557, no token → broken against current Sonic Pi |
| `emlyn/sonic-pi-tool` | Python | pushed 2021-03-07, 69★ | Stale copy of the same pre-token protocol |
| `Koumarin/py-sonic-pi-tool` | Python | pushed 2022-08-17 | Same |
| `Widdershin/sonic-pi-cli` (gem `sonic-pi-cli` 0.2.0) | Ruby | pushed 2020-12-02 | Depends on `osc-ruby ~> 1`, same pre-token protocol, also broken |

**Recommendation: do not shell out to any of these.** They add a process-management dependency for no benefit, and none of them work against a normally-launched current Sonic Pi. Speak OSC directly from Ruby via `osc-ruby` and the cue-server runner pattern above — it's simultaneously more current, lower-ceremony (one gem, no subprocess), and gives you direct control over error handling instead of parsing a CLI's stdout.

## Ruby OSC Libraries (verified against rubygems.org + GitHub)

| Gem | Version | Released | Repo last pushed | Downloads | Verdict |
|---|---|---|---|---|---|
| `osc-ruby` (aberant fork) | 1.1.5 | 2024-07-21 | 2025-11-16 | 62,552 | **Recommended.** Simple `OSC::Client.new(host, port).send(OSC::Message.new(path, *args))`. EventMachine only required for its server class (`OSC::EMServer`), which you don't need — you're only sending. |
| `ruby-osc` (maca) | 1.0.0 | 2020-03-29 | 2020-03-29 | 43,001 | Not recommended. Built entirely on an EventMachine reactor for both client and server — a heavier dependency for a fire-and-forget send use case, and 5+ years untouched. |

Hand-rolling an OSC encoder was considered and rejected: the format itself is simple (address string + type-tag string + args, 4-byte aligned), but `osc-ruby` is a single small dependency, actively maintained, and removes a whole category of binary-encoding bugs from your own code for free. This is the rare case where "boring gem" beats "write it yourself" — unlike the music-theory question below.

## Ruby Music-Theory / Algorithmic-Composition Libraries — the Ecosystem Really Is This Thin

Every candidate gem name was checked live against `rubygems.org`'s API (not memory):

| Gem | Version | Last release | Verdict |
|---|---|---|---|
| `music_theory` | 0.3.11 | 2016-03-24 (10 years) | Exists, has real download count (43,855), but abandoned; depends on `activesupport` + `wavefile` for what should be pure math; no `source_code_uri` published |
| `scale` | 0.1.3 | 2015-04-12 (11 years) | Exists (11,244 downloads), long abandoned |
| `midilib` | 4.0.2 | 2023-09-12 | Real MIDI file/device library, actively-ish maintained, but solves MIDI I/O — a different problem, since this project never touches MIDI |
| `unimidi` | 0.5.1 | 2022-02-18 | Same category, MIDI device I/O, not relevant |
| `midi-eye` / `midi-nibbler` (arirusso) | 0.3.10 / 0.2.4 | 2017 | Abandoned, MIDI-focused, not relevant |
| `euclidean` | 0.2.0 | 2014-03-27 | Generic 2D geometry primitives, not music-specific despite the tempting name; abandoned |
| `chord`, `scales`, `rhythm`, `harmony`, `melody` | — | — | All exist on rubygems.org but are unrelated libraries that happen to share the name (an ActiveRecord-style query gem, a caching framework, a CSS typography gem, a JS-in-Ruby gem, and an empty placeholder, respectively) |

**Honest recommendation: hand-roll scale/chord/rhythm primitives.** There is no Ruby equivalent to Python's `music21`/`mingus`, full stop — PROJECT.md already accepted this tradeoff knowingly, and the research confirms it's the right read. This is genuinely a "write ~100–150 lines yourself" category, not a gap to be papered over with an abandoned 2016 gem:

- **Scales**: a scale is an array of semitone-offset formulas indexed off a root note (e.g. major = `[0,2,4,5,7,9,11]`); Sonic Pi's own `scale` function already does the heavy lifting for playback, so your Ruby layer only needs formulas for *choosing* pitches during generation, not a full theory engine.
- **Chords**: a small hash of interval-formula tables (major, minor, dom7, etc.) — again, Sonic Pi's own `chord` function does the DSP-facing part; your layer only needs the formula to pick voicings/progressions.
- **Rhythm**: Euclidean rhythms (the Bresenham-line algorithm) are ~10 lines of Ruby; weighted-choice / Markov-ish sequencing over a small state table is similarly small.

This is also the *better* design choice here, not just the cheaper one: PROJECT.md requires deterministic, seedable output ("same seed, same output"). An abandoned gem's own internal randomness/state (if any) is one more thing to audit and pin; code you own and pass an explicit `Random.new(seed)` into is trivially auditable and testable.

## Hot-Reload / Fast-Iteration Tooling

| Tool | Version | Ceremony | Verdict |
|---|---|---|---|
| Ruby's `load` (stdlib, no gem) | — | None | The actual reload primitive. Unlike `require` (a no-op on a second call for the same path), `load` re-executes the file every time. This is what any of the options below ultimately call. |
| `pry` | 0.16.0 | None | Recommended as the interactive shell: keep one long-lived session open holding your `OSC::Client`, call `load "generators/acid_bass.rb"` after each edit, re-invoke the generator, hear it. Zero setup beyond `gem install pry`. |
| `listen` | 3.10.0 | Low | Recommended *addition* once manually typing `load` gets old: a ~15-line watcher on `generators/` that calls `load` on the changed file and re-runs the current generator automatically, so editing and hearing collapses to "save the file." On macOS it uses `rb-fsevent` (mature, 512M+ downloads, native FSEvents — sub-second reaction). |
| `guard` | 2.20.2 | Medium | Actively maintained (released 2026-07-16) but adds a `Guardfile` + plugin DSL (`guard-shell`, etc.) that buys you nothing here beyond what a 15-line `listen` block already does. Only reach for this if the loop grows more actions than "reload and replay" (e.g., you also want it auto-running a linter). |
| `zeitwerk` | 2.8.3 | Medium-High | Solves constant-to-file autoloading for larger, Rails-shaped code trees, and has its own `Loader#reload`. Overkill for a handful of directly-invoked generator scripts with no autoload conventions to enforce. Revisit only if `generators/` grows into dozens of interdependent classes and manual `require`/`load` bookkeeping starts to hurt. |

**Recommended combo, lowest ceremony that still hits "under a second or two":** a long-lived Pry session as the outer loop, plus a small `listen`-based watcher that auto-reloads on save. If even that feels like premature tooling on day one, the true zero-dependency floor is just typing `load "generators/whatever.rb"` in Pry by hand — legitimate, and worth doing first before reaching for `listen` at all.

## Testing Generated Code

No gem is needed for any of this — it's all Ruby stdlib:

- **Syntax validity** — `require "ripper"; Ripper.sexp(code)` returns `nil` for syntactically invalid Ruby and a parse tree otherwise. In-process, sub-millisecond, no subprocess spawn. This is strictly better than shelling out to `ruby -c` (spawns a process per check, and only gives you a stderr string rather than a tree you can introspect).
- **Ceiling on what this catches** — syntactic validity is necessary but not sufficient for "valid Sonic Pi." Sonic Pi's DSL (`play`, `sample`, `live_loop`, `use_synth`, etc.) is just Ruby method calls, so a syntactically valid file can still reference a synth/sample name that doesn't exist and will only fail at Sonic Pi runtime. That gap is fine to leave open — PROJECT.md's requirement is "syntactically valid, runnable," and catching a typo'd synth name is exactly what the fast OSC-push-and-listen loop is for.
- **Golden-file / snapshot tests** — plain Minitest comparing generator output against a committed fixture string. No need for RSpec or a snapshot-testing plugin (e.g. `rspec-snapshot`) just to diff two strings.
- **Seeded-determinism tests** — pass an explicit `Random.new(42)` into the generator (not global `Kernel.srand`, so tests stay order-independent) and assert two runs with the same seed produce identical output strings.

## Optional LLM Front-End (prose → parameters, never → code)

| Gem | Version | Released | Verdict |
|---|---|---|---|
| `anthropic` (official SDK) | 1.71.0 | 2026-09-15 | **Recommended default.** This is now Anthropic's own official Ruby SDK (`github.com/anthropics/anthropic-sdk-ruby`), replacing the older community `ruby-anthropic` gem. Supports tool-use / JSON-schema-constrained structured output — define your parameter struct as a tool schema and force the model to call it, which maps directly onto "prose in, parameter set out" without ever asking the model to write Sonic Pi code. |
| `ruby-openai` (alexrudall) | 8.3.0 | 2025-08-29 | Fine alternative if you prefer an OpenAI model — long-standing, ~46.7M downloads, supports `response_format: {type: "json_schema"}` the same way. |

**Keep this gem behind the optional feature, not a hard dependency.** `bundle install` for the core generator/OSC path should never need an API key or network access. Whichever SDK you pick, use its schema-constrained/tool-use mode rather than parsing free-form text — that's the one place in this project where "just regex the JSON out of the response" would reintroduce exactly the kind of flakiness the deterministic core is designed to avoid.

## Installation

```bash
# Core (generator + OSC push)
gem install osc-ruby

# Fast-iteration loop
gem install pry listen

# Optional LLM front-end (only if using the prose-to-params feature)
gem install anthropic
# or, for OpenAI instead:
# gem install ruby-openai
```

No gem needed for: music theory (hand-rolled), file export (`File.write`, stdlib), syntax checking (`Ripper`, stdlib), or golden-file/determinism testing (Minitest, stdlib).

## Alternatives Considered

| Recommended | Alternative | When to Use Alternative |
|-------------|-------------|--------------------------|
| Cue-server runner + `osc-ruby` | Sonic Pi's shipped `headless_boot.rb` harness | You want a fully unattended, no-GUI-at-all session and are comfortable depending on Sonic Pi's internal installed file layout rather than its documented cue-server interface |
| `osc-ruby` | `ruby-osc` (maca) | You already depend on EventMachine elsewhere in your process and want one reactor for everything |
| Hand-rolled music theory | `music_theory` gem | Never — it's a decade abandoned and pulls in `activesupport` for no reason |
| Pry + `listen` | `guard` | Your reload loop grows beyond "reload and replay" into multiple orchestrated actions |
| `anthropic` gem | `ruby-openai` | You prefer OpenAI models over Claude for the prose-to-parameters step |
| Ruby `Ripper` | shelling to `ruby -c` | Never for this project — no reason to pay subprocess overhead for a check stdlib does in-process |

## What NOT to Use

| Avoid | Why | Use Instead |
|-------|-----|--------------|
| `sonic-pi-tool` (any fork/rewrite/language) | Hardcodes the pre-v4 fixed port + no-token protocol; verified broken against current Sonic Pi | `osc-ruby` + the cue-server runner pattern above |
| `sonic-pi-cli` gem (Widdershin) | Same broken pre-token protocol, also 5+ years stale | Same |
| `music_theory` gem | 10 years abandoned, unnecessary `activesupport` dependency | Hand-rolled scale/chord/rhythm formulas |
| `ruby-osc` (maca) as OSC client | EventMachine reactor dependency for a use case that's just fire-and-forget sends | `osc-ruby` |
| Trying to reach Sonic Pi's token-gated `/run-code` channel from outside the GUI process | No supported discovery mechanism for the token when Sonic Pi is launched normally; you'll be reverse-engineering an internal security boundary | The cue-server runner pattern (untokenized, documented, stable) |
| `zeitwerk` on day one | Solves a problem (const-to-file autoloading at scale) this project doesn't have yet | Plain `require`/`load`; adopt `zeitwerk` later if `generators/` grows large |
| Parsing free-form LLM text for the parameter set | Reintroduces nondeterminism/flakiness into the one path that's supposed to be a thin, optional adapter | The chosen SDK's schema-constrained/tool-use structured output mode |

## Stack Patterns by Variant

**If you want the simplest possible "hear it now" loop (single self-contained buffer, no live OSC streaming):**
- Skip OSC entirely for that generator run. Write the generated code to a `.rb` file and open/run it in Sonic Pi's own editor, or paste it in.
- Still use `Ripper` for pre-flight syntax checking and Minitest for golden-file tests — those don't depend on the delivery mechanism.

**If you want the "stream fresh code while playback continues" requirement:**
- Use the cue-server runner pattern (`osc-ruby` → port 4560 → your pasted-once runner buffer's `eval`). This is the only verified-working path for this requirement against current Sonic Pi.

**If/when the generator layer grows past a handful of files:**
- Reach for `zeitwerk` then, not now, to replace manual `require`/`load` bookkeeping — and note it ships its own reload support, so it can subsume part of `listen`'s job at that point too.

## Version Compatibility

| Package A | Compatible With | Notes |
|-----------|-----------------|-------|
| `osc-ruby` 1.1.5 | Ruby 1.9+ (per gemspec lineage), verified working on modern Ruby via active 2025-11 repo activity | No known incompatibility with current Ruby 4.0.x |
| Cue-server runner pattern | Sonic Pi v5.0.0 (verified) | Also expected to work on Sonic Pi 4.x — the cue-server (`/osc` receiving on port 4560) predates the v5 Boot Daemon rewrite and is documented as stable across the 4.x → 5.x transition; the internal token-gated channel is what changed, not the cue-server |
| `anthropic` gem 1.71.0 | Ruby 3.2+ typically required by modern SDKs | Verify against the gem's own `required_ruby_version` at install time if pinning an older Ruby |
| Your generator-layer Ruby version | Sonic Pi's internal vendored Ruby | No linkage — the two processes communicate only via UDP/OSC text, never share a Ruby VM |

## Sources

- `sonic-pi-net/sonic-pi` GitHub repo, cloned directly: tag `v5.0.0` (released 2026-08-07, current stable) and `dev` HEAD (commit `cba6a37`, 2026-09-12) — `app/api/src/sonicpi_api.cpp`, `app/server/ruby/bin/daemon.rb`, `app/server/ruby/bin/headless_boot.rb`, `app/server/ruby/lib/sonicpi/osc_api.rb`, `app/server/ruby/paths.rb`, `app/server/ruby/lib/sonicpi/util.rb`, `etc/doc/tutorial/12.2-Sending-OSC.md` — HIGH confidence, direct primary-source read
- `sonic-pi-net/sonic-pi` GitHub issue #3330 ("Clarifying usage of OSC \run-code in latest versions (4+)") — MEDIUM confidence, community discussion corroborating the token requirement
- `abhishekjairath/sonic-pi-mcp` GitHub repo (pushed 2026-03-22), `sonic-pi-queue.rb` read directly — HIGH confidence, working reference implementation of the cue-server runner pattern
- `lpil/sonic-pi-tool` GitHub repo, `src/server.rs` read directly — HIGH confidence, confirms the stale fixed-port/no-token protocol
- `rubygems.org` API (`/api/v1/gems/<name>.json`, `/api/v1/versions/<name>.json`) queried live for: `osc-ruby`, `ruby-osc`, `midilib`, `unimidi`, `scale`, `music_theory`, `sonic-pi-cli`, `midi-eye`, `midi-nibbler`, `euclidean`, `wavefile`, `listen`, `guard`, `guard-shell`, `rb-inotify`, `rb-fsevent`, `zeitwerk`, `pry`, `pry-remote`, `ripper-tags`, `ruby-openai`, `anthropic`, `openai`, `gemini-ai`, `tomlrb` — HIGH confidence, live registry data
- GitHub repo metadata via `gh api` for `aberant/osc-ruby`, `maca/ruby-osc`, `lpil/sonic-pi-tool`, `emlyn/sonic-pi-tool`, `Koumarin/py-sonic-pi-tool`, `Widdershin/sonic-pi-cli`, `abhishekjairath/sonic-pi-mcp` — HIGH confidence, live API data
- Ruby release schedule via web search (ruby-lang.org news) — MEDIUM confidence

---
*Stack research for: algorithmic-composition Ruby layer driving Sonic Pi*
*Researched: 2026-09-15*
