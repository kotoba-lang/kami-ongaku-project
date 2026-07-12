# kami-ongaku-project

Portable `.cljc` DAW-session data model — the L3 SSoT of the `ongaku`
(music production) engine stack defined in
[ADR-2607121400](https://github.com/com-junkawasaki/root/blob/main/90-docs/adr/2607121400-kami-ongaku-eizo-commercial-grade-cljs-stack.md).
This is the layer that actually ties
[`kami-ongaku-notation`](https://github.com/kotoba-lang/kami-ongaku-notation)
(score IR) and
[`kami-ongaku-sequencer`](https://github.com/kotoba-lang/kami-ongaku-sequencer)
(MIDI-equivalent event IR) into one session, the way a Logic Pro/Cubase/
Ableton project file actually does — tracks, a real bus **graph** (buses
can feed other buses, not just a flat list), tick-placed clips, parameter
automation, and a tick-keyed tempo map.

## Model

- **Track** — `:midi` / `:audio` / `:notation`, with mute/solo/armed
  flags and an `:output-bus` reference.
- **Bus** — name, `:inputs` (a set of track-or-bus ids — a real signal
  graph), and an ordered `:plugin-chain` of opaque plugin-instance refs
  (bypass + wet/dry). The plugin *contract* itself is out of scope here —
  that's `kami-ongaku-plugin-host`, Wave 3 of the ADR.
- **Clip placement** — a `{:track-id :start-tick :length-ticks :content}`
  reference on the project's tick timeline (matching
  `kami-ongaku-sequencer`'s integer-tick convention, no floats). Content
  shape depends on the referenced track's type: a `kami.ongaku.sequencer`
  pattern for `:midi`, a `kami.ongaku.notation` part for `:notation`, an
  opaque `{:audio/uri ...}` ref for `:audio`.
- **Automation** — tick-keyed breakpoint lanes targeting a
  `(track-or-bus-id, param)` pair, with per-segment interpolation
  (`:linear`/`:hold`/`:curve`).
- **Tempo map** — a tick-keyed sequence of `{:bpm :time-sig}` points
  (tempo/time-signature can change mid-project, not just a single global
  BPM), mirroring what `kami-ongaku-sequencer`'s SMF meta events already
  capture.

## Validation

`validate-project` checks referential integrity across the whole graph,
not just per-record shape (the constructors only check their own record's
shape and return `nil` on invalid input, same convention as
`kami-ongaku-notation`):

- dangling `track/output-bus` / `bus/inputs` / automation-lane target
  references
- **bus-graph cycles** — a real DFS cycle detector over the `bus/inputs`
  edges, since a bus receiving (directly or transitively) from itself is
  a genuine signal-flow bug
- clip content shape vs. the referenced track's declared type
- tempo-map / automation-lane tick ordering

```clojure
(require '[kami.ongaku.project :as project])

(def proj
  (project/project
   {:tempo-map [(project/tempo-point {:tick 0 :bpm 120 :time-sig [4 4]})]
    :tracks [(project/track {:id "t1" :type :midi :name "Lead" :output-bus "b1"})]
    :buses [(project/bus {:id "b1" :name "Instruments"})]
    :clips [(project/clip-placement
             {:id "c1" :track-id "t1" :start-tick 0 :length-ticks 960
              :content {:name "verse" :length-ticks 960 :loop? false :events []}})]}))

(project/valid-project? proj)     ;=> true
(project/validate-project proj)   ;=> []
```

## Not in v0

- No audio rendering, no plugin execution — pure session/graph data model.
- No mixing/gain math — automation *targets* a param key, it does not
  itself define what that param does.
- `kami-ongaku-plugin-host` (Wave 3) owns the actual plugin contract;
  `:bus/plugin-chain` here only holds opaque instance refs.
- No `:audio` clip content validation beyond "has a URI" — decoding/probing
  audio files is out of scope for a pure data-model repo.

## Test

```bash
clojure -M:test
clojure -M:lint
```
