# ison

An instrument said in JSON. **I**nstrument **S**ound **O**bject **N**otation,
the way [jmon](https://github.com/jmonlabs) is JSON Music Object Notation. An
ison is also the note held under the melody in Byzantine chant: a drone that
carries.

A SoundFont or an SFZ says what there is to play: recordings, zones,
envelopes, key switches. It does not say how the instrument is played. An ison
adds that layer, and only that: it wraps sounds a player already has, it
contains none.

```json
{
  "format": "ison",
  "version": "1.0",
  "name": "Violin",
  "description": "A solo violin recorded in every way of playing. No recording holds more than 4 to 5 s: a long note is bowed in strokes.",
  "family": "bowed strings",
  "range": [55, 100],
  "sources": {
    "ldk": { "sf2": "/samples/soundfonts/Violin_LDK1609.sf2" }
  },
  "modes": {
    "normal": { "source": "ldk", "program": 0, "holds": 5 },
    "vibrato": { "source": "ldk", "program": 11, "holds": 4, "vibrato": { "onset": 0.5, "depth": 9, "rate": 5 } },
    "tremolo": { "source": "ldk", "program": 4, "holds": 4 },
    "pizzicato": { "source": "ldk", "program": 15, "holds": 1.5, "oneShot": true }
  },
  "play": {
    "bow": { "max": 4, "gap": 0.15, "change": 0.85 },
    "vibrato": { "from": 2, "mode": "vibrato" },
    "stroke": { "attack": 0.2, "swell": 0.15, "peak": 0.5, "fade": 0.2 }
  },
  "level": { "db": 0, "velocity": "linear" },
  "controls": {
    "vibrato": { "cc": 1, "default": 0, "description": "a synthetic vibrato over the recorded one, 0 to 1" }
  }
}
```

In a piece, the instrument is the track's `synth`, and a note may choose its mode:

```js
{ label: "Violin", synth: violin, notes: [
  { pitch: 69, duration: 8, time: 0 },                       // two bow strokes, with the recorded vibrato
  { pitch: 76, duration: 1, time: 9, variant: "pizzicato" }, // a note that picks its mode
] }
```

## The fields

The schema, [`ison-schema.json`](ison-schema.json), has a sentence for every
field; this is the shape of it.

- `format`, `version`, `name`, `description`, `family`, `range`, `pieces`,
  `words`: who it is. `family` lets a host apply its defaults (bowed strings
  enter a little softer). `words` are what you find it by.
- `sources`: the sounds, by name. Each is a spec the player already reads: a
  SoundFont program (`{ sf2, program }`), an SFZ file (`{ sfz }`), a Faust DSP
  (`{ faust, params }`), a General MIDI program (`{ gm }`), a Tone.js synth
  (`{ type, options }`). An instrument may take its modes from several.
- `modes`: the ways of playing, by name. Each says its source and what to ask of
  it (a `program` and `bankSelect`, a `keyswitch` or `controls`, `params`,
  `options`), then what the sound does: `holds`, how many seconds a note lasts
  before the recording ends; `oneShot`; the `vibrato` the recording already
  has. The first mode is the default; a note chooses another with `variant`.
- `play`: how the instrument is played. Rules a host applies to a track's notes
  before scheduling them: `bow` (a note longer than `max` beats, or than what
  its mode holds, is played in equal strokes, `gap` beats apart, the next at
  `change` of the velocity), `vibrato` (a stroke of `from` beats or more plays in
  the mode named), `stroke` (the loudness shape of a stroke, written as the
  note's `dynamics`). More will come from other instruments: a breath, a pedal.
- `level`: a gain in dB, calibrated so the instrument sits with the others, and
  the velocity curve the source expects (`sqrt` for one that hears velocity as
  MIDI does).
- `envelope`: an attack and a release in seconds, replacing an SFZ source's.
  A SoundFont preset's envelope is scaled instead, relative to its own, with
  the source's `controllers`: 73 the attack, 72 the release (64 the preset's,
  0 about eight times shorter). A bowed string whose release is too long
  hides its bow changes.
- `controls`: the continuous controls by name. Each one *is* a controller
  number (`cc`), so a lane written by name is the same move for a player and
  for a MIDI file; `param` says where it lands in a Faust DSP. With a starting
  value and a description. A piece drives one with an automation lane whose
  target is the name:

  ```js
  automation: { tracks: { Wind: [{ target: "whistle", anchorPoints: [{ time: 0, value: 0.2 }, { time: 16, value: 0.9 }] }] } }
  ```

## Playing one

[jmon/sound](https://github.com/jmonlabs/sound) reads the format (`src/ison.js`,
from 1.4.0). Two halves, kept apart: `performIson(ison, notes, { tempo })` is
pure, the notes played the way the instrument says (strokes, variants,
dynamics), which a host calls before it schedules; `create(ison, Tone)` builds
the instrument, a SoundFont's modes as the variants of one instrument, any
other source a node per mode, all into one output at the instrument's level.
[jmon/show](https://github.com/jmonlabs/show) calls `performPiece` first, live
and when rendering.

A piece turns an instrument's settings with `sound.ison(base, changes)`, which
merges level by level: `ison(violin, { play: { bow: { gap: 0.05 } } })` or
`ison(whale, { mode: "slow" })`.

What is not done yet: `{ ison: "url" }` loaded by the player itself (a notebook
fetches the JSON and puts it in `synth`), `dynamics` on a Tone.js source.

## Examples

`examples/` holds the first instruments written in the format: a solo violin
with seven modes and its bowing, a viola, a cello, basses and a chamber
section of strings, a Faust wind with three modes and three controls, an FM
whale with three voices.
Their sample paths are those of the author's setup. `docs/ison.fr.md` is the
design note, in French.

## License

GPL-3.0-or-later, like the rest of jmon.
