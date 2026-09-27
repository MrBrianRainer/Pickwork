# Pickwork — Claude Code Handoff

## What this is
A single-file web app (`index.html`, no build step) where you play a real acoustic
guitar and the browser scores you: mic for pitch detection, camera (optional) for
picking-hand finger tracking. Styled like a gig-poster/Guitar-Hero mashup. Runs
locally via `python -m http.server`, deployed via GitHub Pages (needs https for
mic/camera permissions).

**This handoff covers two things:** (1) orientation to the existing code so you
don't have to reverse-engineer it from scratch, and (2) the next task — more
built-in songs, organized into defined categories and difficulty levels.

## How the developer works
- Phone-first, short bursts. Small, runnable, testable increments.
- After each chunk, say exactly how to verify it (which mode/song to click, what
  should happen on screen).
- Don't reduce scope silently — flag it and propose an alternative instead.
- Commit after each working increment.

## Critical constraint: copyright
**Never write out the actual notes/riffs/melody of a real, currently-copyrighted
song.** This came up already — the developer asked for Hotel California, Fast
Car, and a Disney song, and the answer was no, with the alternative being: import
a legally-obtained Guitar Pro file (Songsterr/Ultimate Guitar Pro purchase, or a
file the developer already owns) through the existing importer, which also pulls
in the real backing band from the file. All new built-in songs must be original
compositions. Public-domain classical/folk pieces (already used for a few) are
fine to arrange. When in doubt, don't transcribe — write something original in
the same style instead.

## Architecture tour (`index.html`)

### Song data model
A "song" object has: `id, title, artist, style, part, groups, chords, backing,
band, bars, tempo, endTick`. `groups` are note events (`{tick, notes: [{s, f,
finger, midi}]}`, string 1=low E..6=high e). `backing` is an optional array of
band events (`{tick, kind, notes|drum, durT, vel}`) for drums/bass/pad/organ/
dist/lead. Two ways a song gets built:

1. **Built-in, via the `built({...})` helper** (search for
   `/* ======================= Built-in songs`). You describe bars as beat-slot
   sequences using compact tokens like `p1:0` (thumb, string 1, fret 0) or
   `3.5` for a chord-style dyad; see `tok()`. A bar can carry `ch:` (chord name,
   must exist in `SHAPES`) and `harm:`/`harmNow` (harmony for the backing band —
   see `harmony()`). Pass `band: {...}` per bar to drive the backing generator
   (`genBacking`): `d` (drum pattern: rock/drive/four/half/boogie/end), `b`
   (bass pattern: eighths/boogie/pop/half/whole/hit), `k` (pad/organ), `g`
   (guitar: chug/ring/push/hit), plus `crash`/`fill` flags. Look at the 5
   existing band songs (`gasoline`, `rustbelt`, `static`, `lanterns`, `neon`) in
   `bandSongs()` for working examples of each genre's pattern choices, and the
   earlier fingerstyle-only songs (`warmup`, `ode`, `romance`, `campfire`,
   `travis`) for songs with no band.
2. **Imported, via alphaTab** (Guitar Pro/MusicXML/alphaTex) — `scoreToSong()`
   converts a parsed score, including other tracks as `backing` via
   `trackKind()`/`gmDrum()`. Don't touch this path for the song-adding task.

### Rendering
`render()` draws the 3D-perspective note highway on canvas each frame. Gems were
recently reworked to always be a light disc with a colored ring + dark text
(readability fix — string color alone was hard to read while scrolling). The
hit-line pads now preview the next fret per string. If you touch this code,
re-run the same kind of headless Playwright screenshot check used before
(render a demo run, inspect the PNG) rather than assuming a canvas change looks
right.

### Difficulty (per-song, already built)
`filterDifficulty(song, diff)` derives **Garage/Club/Arena** from a song's full
note set automatically (beat-aligned thinning). This already works for every
song, built-in or imported, and doesn't need new work.

### Levels and genres (setlist organization)
- **Genre** (`genre:` in `built()`, must be one of `GENRES`) is the browse category:
  Fingerstyle, Classic Rock, Modern Rock, Pop, Country & Folk, Ballad, Reggae, Punk.
  `built()` throws if it's missing. `style:` is now free-text flavor ("12-bar boogie"),
  shown after the artist on the poster.
- **Level 1–5** is computed, never hand-set: `songScore()` =
  notes/sec + 0.5×avg string jump + 0.5×chord changes per bar + 0.15×frets above 5,
  on the full Arena note set at 100% speed. `LEVEL_CUTS = [2, 3, 4, 5]`. Names live in
  `LEVELS` (Open Mic → Headliner). Imports get a level at import time; older imports
  fall back from their 1–3 `heat`.
- Setlist = genre chips + level chips (saved in `setupPrefs.genre/level`), then one
  shelf per level, songs sorted by score. The poster's old ▲ heat meter was replaced by
  the level stamp + level name.
- Backing patterns added for the new genres: `d: 'onedrop' | 'punk'`,
  `b: 'alt' | 'dub'`, `g: 'skank'`.

### Song structure and lead-guitar solos
All 22 built-in songs are now full arrangements (intro/verse/chorus/solo/outro-style),
not single 8-16 bar loops — 20-90s each instead of ~10-20s.
- A "solo" needs no new engine support: it's ordinary single-note `groups`, just placed
  high on the neck instead of in a chord shape. The helper for it lives right after
  `voicing()`: `leadTok(str, rootPc, deg, minor, minFret)` finds the fret for a
  pentatonic scale degree above a root pitch class on a string (wraps to stay >= 3rd
  fret); `lickBar()` places a set of `[slot, string, degree]` triples into a bar;
  `LICKS_4/8/16` are reusable lick shapes at three densities (quarter/eighth/sixteenth
  notes); `solo(count, slotsPerBar, pcs, minor, strings, band)` is the entry point —
  pass **`slotsPerBar` = that song's own `sub × ts[0]`** (this is the one thing to get
  right: solo bars must match the song's bar-grid or notes land at the wrong time) and
  a pitch-class per bar to follow the chord progression underneath.
- Every band song got: verse, chorus, verse, chorus, an 8-bar `solo()` (4 for shorter/
  easier songs), then a final chorus. Fingerstyle songs got a new B-section or reprise;
  the more advanced ones (Travis Walk, Porch Swing, Cascade Etude, Rail Yard Rag) also
  got an unaccompanied `solo()`-based break.
- `romance` and `ode` are the two PD-arrangement songs; per the copyright rule above,
  their new sections are ORIGINAL continuations in the same style, not more of the real
  piece transcribed from memory — don't replace them with an actual transcription later
  without checking the source is legally obtained.
- Adding solos pushed several songs' computed level up (more string-jumping/reach from
  the higher-fret lead lines) — level numbers shifted from the previous batch; this is
  expected, not a bug, since level is meant to track exactly that.

### Storage
`localStorage` under key `pickwork.v2`: settings, per-song/difficulty best
scores (`bests`), imported tabs (`imports`). Song list for the setlist UI comes
from `BUILTINS` (array) — built-ins with `band: true` render under "Tonight's
setlist," the rest under "Fingerstyle warm-ups" (see `renderSetlist()`).

## The task: more songs + defined categories/levels

The developer's feedback: current 10 built-in songs (5 fingerstyle, 5 full-band)
are "okay for now," but the setlist needs **more songs** and **defined
categories/levels** — right now everything is just two flat shelves with no
sense of progression.

### 1. More original songs
Add roughly 10–15 more built-in songs using the `built()` DSL, covering gaps:
- More fingerstyle pieces at a range of difficulty (a few even easier than
  `warmup`, a few harder than `romance`).
- More genres in the band engine: try country/folk-rock (`d: 'half'` or
  `'boogie'` fits), a ballad (slow, sparse, `k: 'pad'` heavy), reggae-ish
  (off-beat guitar hits — may need a new `g` pattern in `genBacking`), and a
  punk/fast one (very fast `drive` drums, short chug guitar).
- Keep using only `harmony()`'s existing chord qualities (maj/min/5/6/7) and
  `SHAPES` entries; add new `SHAPES` entries if a song needs a chord not
  already defined (follow the existing `{frets, fingers}` format, frets array
  is string 1→6, -1 = muted).
- Every new song needs a `style` tag (shown on its poster) and should sit at an
  identifiable difficulty — see next section.

### 2. Defined categories and levels
Design and implement an organizing structure instead of the current two flat
shelves. Suggested shape (adjust if you find something better, but explain
why):
- **Levels 1–5** (or similar), based on **inherent song difficulty** — not the
  Garage/Club/Arena setting (that's a per-song note-density knob the player
  picks at setup; a level is a property of the song itself: tempo, chord
  complexity, whether it's fingerstyle vs. strummed/band, string-jumping
  difficulty). Come up with a simple, defensible scoring heuristic (e.g. tempo +
  chord-change frequency + note rate) rather than hand-guessing each one, so it
  stays consistent as more songs get added later.
- **Genre/style categories** for browsing (Fingerstyle, Classic Rock, Modern
  Rock, Pop, plus whatever new genres you add) — this can coexist with levels;
  think of levels as difficulty and categories as genre, both shown on each
  song's poster and usable as filters in the setlist UI.
- Update `renderSetlist()` and the flyer/poster markup to reflect the new
  grouping (e.g. shelves per level, or a level badge + genre tag per poster,
  with a filter control). Keep the gig-poster visual style — this is a design
  extension, not a redesign; check with the developer on layout specifics if
  it's ambiguous, since they've mentioned separate design/UI feedback is
  coming.
- Existing `heat()` (1–3 "crowd heat" indicator) is a rough proxy for note
  density/speed already — decide whether it's superseded by the new level
  system or kept as a secondary indicator. Don't remove player-facing features
  without flagging it.

### Acceptance criteria
- At least 10 new original built-in songs, spanning multiple genres and a
  visible difficulty spread, none reproducing a real copyrighted song.
- A defined, consistent level (difficulty tier) and category (genre) system,
  visible on the setlist screen, replacing the current two-shelf split.
- All existing songs (old and new) get a level and category — nothing
  unclassified.
- `node --check` on the extracted `<script type="module">` block passes (see
  how earlier changes were verified — extract the script to a `.mjs` file and
  run it through Node before treating a change as done).
- A quick headless run-through (Playwright or similar) of at least one song per
  new category, screenshotted, to confirm posters render correctly and the new
  grouping doesn't break the setlist layout on a 390px-wide mobile viewport.

### Out of scope for this task
- Camera coach cam, pitch-detection tuning, calibration — untouched, working.
- The kid-friendly no-camera sideload version — that's a separate future
  project/repo, not part of this file.
- Visual/UI redesign beyond what's needed to show categories/levels — the
  developer has separate design feedback coming later.

## Later additions (post-handoff)

### Community song batch (Gemini-drafted)
Added 5 more songs via a Gemini-drafted batch, reviewed and corrected here before
merging: Spanish Romance (PD classical), Bourbon Street Stroll (original swing/jazz
blues — added a new `Jazz & Blues` genre for it), Wipeout Pipeline (original surf
rock), Iron Anvil (original metal gallop), Kingston Skank (original reggae). Two
drafted songs ("House of the Rising Sun", "Scarborough Fair") were deliberately left
out: PD melody, but the famous version most people know is a specific 1960s
copyrighted arrangement (The Animals; Simon & Garfunkel) — too close to transcribe
safely. If asked to add either, write a clearly distinct arrangement, not a
recognizable cover of those recordings.
When reviewing an externally-drafted song before merging: check every `ch:` chord
name's fret tokens actually match that name's entry in `SHAPES` (a couple of the
drafted ones didn't — one chord was voiced at the wrong neck position entirely, one
had a stray extra note); check backing `d`/`b`/`g`/`k` values are ones `genBacking`
actually handles; check each bar's `seq` length equals that song's `sub × ts[0]`.

### Tuner
A standalone chromatic tuner, `<dialog id="tuner">`, opened via the round icon next to
Settings. Deliberately separate from the rhythm-game's pitch-matching code (which only
samples the spectrum once per note-onset — too infrequent for a continuously-updating
tuner needle): it runs its own analyser (`tunerAn`, a fresh `MediaStreamSource` off the
same `micStream`) and a classic autocorrelation pitch detector (`autoCorrelate()`) on
raw time-domain audio, polled every 80ms via `tunerTimer`. `closestString()` matches
the detected frequency to the nearest of the 6 open-string targets (`TUNER_HZ`,
derived from `TUNING`), folding octave errors by snapping to the nearest power-of-two
multiple first — verified against synthetic sine waves at each string's frequency
(including a doubled-frequency low-E case) before shipping. Tapping a string button
also plays its reference pitch via the existing `pluck()` synth. Reuses `ensureAudio()`/
`ensureMic()` so it shares (not duplicates) mic permission and the audio context with
the rest of the app.
