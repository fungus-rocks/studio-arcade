# Studio Arcade

Ear-training arcade games for music producers. Four mini-games each train one studio skill, and every sound is synthesized live in the browser with the Web Audio API. There are no audio files, build step or dependencies.

**Play it:** download `index.html` and open it in a browser. A hosted copy lives at https://claude.ai/artifact/H26m7xYfYYFMH937iD5ezN, but that link is private for now.

**Roadmap:** [Studio Arcade Roadmap](https://claude.ai/code/artifact/9bc7fb4f-8a23-4221-9211-19fdc00cd01c) (private for now) covers Pro level packs, Session mode, the daily challenge and options for saving progress.

## The games

| Game | Skill | What you do | Levels |
| --- | --- | --- | --- |
| Frequency Hunter | EQ | One band is boosted or cut. Switch the EQ in and out, then name the frequency. | 5 + Mastery |
| Beat Copy | Drum programming | Hear a genre groove (house, rock, boom bap, trap, dembow, Amen break) and program it on a 16-step grid. | 6 |
| Patch Match | Synth sound design | Pick the waveform and set cutoff, resonance, attack and release to match a hidden patch. | 5 |
| Mix Doctor | Mixing | One track is too loud, too quiet or panned hard. Compare against the reference and diagnose it. | 5 + Mastery |

Each level awards up to 3 stars, and one star unlocks the next level. Headphones or decent speakers help, because laptop speakers can't reproduce the 63 Hz band.

### Mastery mode

Mastery unlocks once every level in a game has at least one star. It's an endless run where difficulty adapts to the player:

- Two right answers in a row make it one step harder. One miss makes it one step easier.
- Until the first miss, each right answer steps up straight away, so experienced players reach their level quickly.
- The run ends after 8 changes of direction (from getting harder to getting easier, or back) or 30 rounds. Your rating is the average difficulty at the last 6 changes of direction.

This is the "2-down / 1-up" staircase used in hearing tests. It settles where you answer correctly about 71% of the time, so the rating measures what you can reliably hear and isn't skewed by lucky guesses.

| Game | Steps (easiest to hardest) |
| --- | --- |
| Frequency Hunter | 3 bands at +12 dB → octave bands from +12 down to +4 dB → half-octave bands (15 choices) from +9 down to +3 dB. 10 steps. |
| Mix Doctor | One track 12, 9, 6, 4.5, 3, 2, 1.5 or 1 dB too loud or too quiet, with the reference available. 8 steps. |

## Controls

| Key | Action |
| --- | --- |
| Space | Play / stop |
| 1–9 | Pick an answer (or a waveform in Patch Match) |
| Enter | Check, submit, or go to the next round |
| B | Frequency Hunter: switch the EQ in or out (Flat / EQ in) |
| T / Y / A | Beat Copy and Patch Match: hear the target, yours, or both back to back (A/B) |
| R | Mix Doctor: switch between the problem mix and the reference |

Knobs: drag up or down (hold Shift for fine control), scroll the mouse wheel, or use the arrow keys. Double-click a knob to reset it.

## Saving progress

Progress saves automatically in your browser. To move it to another device or browser, or to back it up before clearing browser data, use a **save code**.

### Using a save code

1. On the main menu, click **Save codes** (bottom right, next to Reset progress).
2. Under **Your save code**, click **Copy code**. Keep the code somewhere safe, such as a note or an email to yourself. It looks like this:
   ```
   SA1-eyJzIjp7ImVxLTAiOjMsImVxLTEiOjJ9LCJtIjp7ImVxIjo2LjV9fQ.2et8
   ```
3. On the other device, open Studio Arcade, click **Save codes**, paste the code under **Load a save code**, and click **Load code**.

Loading **merges** with the progress already on that device. Each level keeps whichever star count is higher, and each Mastery best keeps the higher rating, so loading an old code never erases newer progress. Spaces and line breaks picked up while copying are ignored.

A code shows your progress at the moment you copied it. After you earn more stars, copy a new one.

### How it works

| Part | Example | Purpose |
| --- | --- | --- |
| `SA1-` | `SA1-` | Studio Arcade, save format version 1. Lets future versions still read old codes. |
| Body | `eyJzIjp7...` | Your stars and Mastery bests as compact JSON, encoded in URL-safe base64 |
| `.` + 4 characters | `.2et8` | Checksum (FNV-1a hash of the body). A mistyped or cut-off code is rejected instead of loading bad data. |

No server or database is involved: the code *is* your save. Inside the browser, progress is stored in `localStorage` under the key `studio-arcade-v1`:

```json
{ "stars": { "eq-0": 3, "beat-2": 1 }, "mastery": { "eq": { "best": 6.5, "runs": [{ "t": 1790916473000, "r": 6.5 }] } } }
```

Save codes carry only the Mastery best, not the run history.

## Development

Everything lives in `index.html`. It has no `<!doctype>`, `<html>` or `<body>` tags, because the artifact publisher wraps the page in its own document. Browsers still open it directly.

To jump straight to a screen, add one of these to the end of the URL:

| URL ending | Opens |
| --- | --- |
| `#eq`, `#beat`, `#synth`, `#mix` | That game, at your highest unlocked level |
| `#mix.2` | A specific level (numbered from 0), if it's unlocked |
| `#eq.mastery` | Mastery mode, even while it's locked (handy for testing) |

### How the code is organized

The `<script>` is split into sections marked `/* ===== ... ===== */`:

1. **Helpers and save system:** the `h()` DOM builder and the `save` object.
2. **Audio engine:** `ac()` creates the AudioContext, pink and white noise buffers, `DRUMS.*` synthesized drum voices, and `synth()`, a subtractive synth voice (oscillator → low-pass filter → amp envelope).
3. **Playback:** `newSession()` / `stopAll()` make sure only one thing plays at a time. `seqStart()` is a lookahead step sequencer that schedules notes ~120 ms ahead on the audio clock.
4. **Backing track:** `SONG` is a 4-bar A-minor loop with drums, bass, keys and lead. `songGraph()` gives each track its own gain and pan.
5. **UI framework:** the `knob()` component, hub, level picker, results modal, the adaptive-difficulty tracker (`staircase()`), and `finishMastery()`.
6. **One section per game,** each a `GAMES.push({ ... })`.
7. **Save codes:** `makeSaveCode()` and `loadSaveCode()`, plus the panel's button handlers.
8. **Boot:** keyboard handling, deep links, and the start-up code that runs again when the published page is updated.

### Adding a level

Each game reads its difficulty from a plain object, so a new level is usually one line in that game's `levels` array:

```js
// Frequency Hunter
{ name: 'Air Check', desc: '4 top bands, +6 dB', bands: [2000, 4000, 8000, 16000], gain: 6 },
// Mix Doctor
{ name: 'Fine Balance', desc: 'One track is 3 dB off', probs: ['loud', 'quiet'], amt: 3, ref: true },
```

A new frequency also needs an `EQ_INFO` entry (short tag plus explanation). A new Beat Copy groove is `{ name, desc, bpm, rows: { kick: [...], snare: [...] }, tip }`, using step numbers 0–15.

### Adding a game

Push an object with `id`, `name`, `skill`, `hue`, `blurb`, `levels`, a `deco(ctx, w, h)` function that draws the hub-card decoration, and `mount(stage, level, done)`:

- `mount` builds the game inside `stage`. It returns `{ key(e), unmount() }`. `key` returns `true` when it handles a key press.
- At the end of a level, call `done(stars, title, text)`.
- For Mastery support, add `mastery: { unit, start, steps }` and handle `level === 'M'`: create a `staircase()`, call `sc.record(ok)` after each answer, and call `done(sc)` when `sc.done` is true.
