# Plan: DAW translator that sounds the same and proves it

**Status:** saved for a future build. Nothing has been built yet.

## Context

Carlo and Emilio want a "universal language for DAWs": load an FL Studio project with its samples and get a project that opens in another DAW with everything in place.

Converters already exist (RysUpConvert, Doseedo, WG DAW Converter, Jukeblocks), so they chose a different angle that targets two gaps:

1. **Sound match:** existing tools leave FL stock plugins as placeholders or approximations, so the converted project can sound different.
2. **Proof:** the vendor pages reviewed don't mention checking the result against the original mix. The exported MP3/WAV can serve as the reference.

Intended outcome: drop in an `.flp`, samples, stems and a reference mix, then get an Ableton project that sounds like the original, stays editable where possible, and comes with a report showing what matches and what doesn't.

**Scope:** FL Studio to Ableton Live 12 first. The internal format is built so other DAWs can be added, but no other DAW is added until this pair works on real projects.

## What the user gets

- **Generated Live Set (`.als`), with two groups:**
  - `Sound Match`: active audio stems, one per FL mixer track, so playback matches the original.
  - `Editable`: muted rebuilt tracks with samples, MIDI, third-party plugins and mixer settings.
- **Match report (HTML):** a song timeline with differing sections marked, plus a per-track list marked `exact`, `approximate` or `stem only`, with reasons.

Stem export is manual at first: the user exports from FL with "Split mixer tracks". This avoids depending on automated FL rendering.

## Architecture

Python core, run locally, so projects and samples never leave the computer. CLI first, drag-and-drop UI later.

```
.flp + samples ─> importers/flp ─> hub model ─> exporters/ableton ─> .als + copied audio
                                      │
stems + reference mix ────────────────┴─> verify ─> report.html
```

Planned layout (the project directory is currently empty):

- `src/dawbridge/model.py`: hub model, using DAWproject concepts where they fit (tracks, clips, notes, automation, routing, plugin references), plus a `Loss` record for anything not carried over
- `src/dawbridge/importers/flp.py`: FLP to hub model
- `src/dawbridge/exporters/ableton.py`: hub model to `.als`
- `src/dawbridge/verify/`: alignment, comparison and report generation
- `src/dawbridge/cli.py`: `verify` and `convert` commands
- `templates/`: blank Live 12 set saved from Ableton, used as the base for generated files
- `tests/`: synthetic audio and small project fixtures
- `corpus/`: real test projects, git-ignored because of sample rights

## Build order

### 1. Verifier and competitor scorecard
Build this first because it works without a converter and tests the idea straight away.

- `dawbridge verify <reference> <candidate>`
- Decode both files and match sample rates, align them using cross-correlation (which also absorbs MP3 encoder delay), then match loudness.
- Compare by section on a bar or short-window grid: log-mel spectral difference, onset timing and level.
- **Calibrate thresholds from real noise:** compare repeated FL exports of the same project, and WAV against MP3. Randomized plugins and MP3 encoding differ without being errors, so they must not be flagged.
- Output an HTML timeline of differing sections.
- Run 5 real FL projects through the existing converters and score them.

**Decision point:** continue only if the scorecard shows audible failures that matter. If the existing tools already score well, stop and rethink the angle before building a converter.

### 2. FLP importer
- Prototype with PyFLP and test it against the FL version actually in use. The FLP format is undocumented and new versions can break parsers.
- Extract tempo, time signature, channels, sample paths, patterns and notes, playlist clips, mixer tracks and routing, plugin names, and automation clips.
- Record anything unsupported as a `Loss`. Never drop it silently.

### 3. Ableton exporter, Sound Match layer
- Generate the `.als` from `templates/`, adding one audio track per stem, with tempo and time signature set.
- Copy audio into the project folder with relative paths.
- This milestone alone should give a project that sounds right. Verify it with step 1.

### 4. Ableton exporter, Editable layer
Add in this order, testing on the corpus after each step:

1. Audio clips and sample positions
2. MIDI notes and Simpler instruments for sample channels
3. Volume, pan, groups, sends and routing
4. Third-party plugins (plugin state may not transfer; test before promising it)
5. Volume and pan automation

Map FL's playlist, which isn't tied to tracks, onto Ableton tracks by splitting overlapping clips across lanes. FL stock plugins stay `stem only`, named in the report.

### 5. Per-track verification
The user exports tracks from Ableton with the `Editable` group active. Compare each one against its FL stem and set its `exact` / `approximate` label from the result. Until this step, labels describe only what was transferred, not how it sounds.

### 6. Drag-and-drop app
A local UI over the CLI: drop the folder, get the Ableton project and report. Build it only after the core works, since the CLI is enough for testing.

### Later, not in this build
- Ableton to FL Studio
- Logic, which is closed and version-sensitive, and needs a terms-of-use review first
- Automated rendering from FL and Ableton
- MP3-only reconstruction

## Risks and open items

| Risk | Handling |
|---|---|
| **PyFLP is GPL-3.0** | Fine for prototyping. Before any closed-source release, either release under GPL, write an independent parser, or get legal advice. |
| **FLP or `.als` changes with new versions** | Pin tested versions, fail with a clear message on unknown ones, and keep corpus regression tests. |
| **`.als` format is not officially documented** | Start from a real Live-saved template and change as little as possible. |
| **Reference mismatch** (project edited after the MP3 was exported) | Report low overall match and ask for a fresh export. |
| **Master effects** (a limiter changes everything downstream) | Decide during step 3, based on how FL's stem export treats the master in testing. |
| **Reverse-engineering and resale terms** | Review FL Studio and Ableton terms before any commercial release. |

## Needed from Carlo and Emilio

- 5 real FL projects, ranging from simple to messy, each with samples, stems and a reference export
- FL Studio and Ableton Live 12 installed, for exports and for opening generated sets
- Trial access to the competitor tools for the scorecard

## Verification

**Automated (`pytest`)**
- Verifier: identical audio passes. Injected faults (a removed drum hit, a 3 dB level change, a 20 ms shift) are flagged at the correct time. A WAV and its MP3 encode are not flagged.
- Importer: parsed tempo, notes and clip positions match values noted by hand from small FL test projects.
- Exporter: generated `.als` decompresses to valid XML, all audio paths resolve, and values round-trip through the parser.

**End-to-end, on each corpus project**
1. Run `dawbridge convert` on the project.
2. Open the `.als` in Ableton: no errors or missing files.
3. Export from Ableton and run `dawbridge verify` against the reference: `Sound Match` should fall within the calibrated noise threshold.
4. Check the report by ear, confirming that flagged sections really differ and unflagged sections sound the same.
5. Unmute `Editable`, and confirm that every difference is either listed in the report or logged as a bug.
