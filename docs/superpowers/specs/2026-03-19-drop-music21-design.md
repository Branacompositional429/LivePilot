# Drop music21 — Replace with Pure Python Theory Engine

**Date:** 2026-03-19
**Status:** Approved
**Scope:** Replace 100MB music21 dependency with ~300 lines of pure Python

## Problem

LivePilot v1.6.4 shipped 7 music theory tools powered by music21. But:
- music21 is ~100MB with heavy transitive deps (numpy, matplotlib)
- We use <1% of its API surface — just key detection, chord naming, interval math
- Claude already knows music theory from training — music21 is a calculator for things the LLM can reason about
- The optional dependency creates friction: users must `pip install music21` separately
- music21's first import takes 2-3s, slowing cold start
- music21 has a classical bias that doesn't match production music contexts

## Solution

New file `mcp_server/tools/_theory_engine.py` — zero-dependency pure Python module implementing the 5 primitives our 7 tools actually need. The MCP tools in `theory.py` keep identical APIs and output formats.

## Architecture

```
theory.py (7 MCP tools — unchanged API)
    └── _theory_engine.py (pure Python music theory math)
            ├── detect_key()         — Krumhansl-Schmuckler with 7 mode profiles
            ├── pitch_name()         — MIDI ↔ note name conversion
            ├── parse_key()          — "A minor" → {tonic: 9, mode: "minor"}
            ├── get_scale_pitches()  — tonic + mode → pitch class set
            ├── build_chord()        — scale degree + key → concrete pitches
            ├── roman_numeral()      — chord pitch classes → Roman figure
            ├── chord_name()         — pitch classes → "C major triad"
            ├── check_voice_leading() — parallel 5ths/8ves, crossing, hidden 5th
            └── chordify()           — group notes by beat → list of chord dicts
```

## Engine Specification

### Constants

```python
NOTE_NAMES = ['C', 'C#', 'D', 'D#', 'E', 'F', 'F#', 'G', 'G#', 'A', 'A#', 'B']

# Krumhansl-Schmuckler key profiles (Krumhansl & Kessler, 1982)
MAJOR_PROFILE = [6.35, 2.23, 3.48, 2.33, 4.38, 4.09, 2.52, 5.19, 2.39, 3.66, 2.29, 2.88]
MINOR_PROFILE = [6.33, 2.68, 3.52, 5.38, 2.60, 3.53, 2.54, 4.75, 3.98, 2.69, 3.34, 3.17]

SCALES = {
    'major':      [0,2,4,5,7,9,11],
    'minor':      [0,2,3,5,7,8,10],
    'dorian':     [0,2,3,5,7,9,10],
    'phrygian':   [0,1,3,5,7,8,10],
    'lydian':     [0,2,4,6,7,9,11],
    'mixolydian': [0,2,4,5,7,9,10],
    'locrian':    [0,1,3,5,6,8,10],
}

# Triad quality by scale degree (0-indexed)
TRIAD_QUALITIES = {
    'major': ['major','minor','minor','major','major','minor','diminished'],
    'minor': ['minor','diminished','major','minor','minor','major','major'],
}
```

### Functions

#### `detect_key(notes, mode_detection=True) → dict`
- Build pitch-class histogram (12 bins) weighted by note duration
- For each of 12 possible tonics × 7 modes: rotate profile, compute Pearson correlation
- If `mode_detection=False`: only test major + minor (14 candidates instead of 84)
- Return: `{tonic: int, tonic_name: str, mode: str, confidence: float, alternatives: [{...}]}`
- Pearson correlation: `r = Σ((x-x̄)(y-ȳ)) / √(Σ(x-x̄)² × Σ(y-ȳ)²)` — stdlib `math` only

#### `pitch_name(midi) → str`
- `NOTE_NAMES[midi % 12] + str(midi // 12 - 1)`
- `pitch_name(60)` → `"C4"`, `pitch_name(69)` → `"A4"`

#### `parse_key(key_str) → dict`
- Parse "C", "C major", "A minor", "D dorian", "f# minor"
- Handle enharmonics: Bb→A#, Db→C#, etc.
- Return: `{tonic: int, mode: str}`

#### `get_scale_pitches(tonic, mode) → list[int]`
- Return pitch classes (0-11) for the scale
- `get_scale_pitches(0, 'major')` → `[0, 2, 4, 5, 7, 9, 11]`

#### `build_chord(degree, tonic, mode) → dict`
- Build triad from scale degree (0-indexed)
- Return: `{root_pc: int, pitches: [int], quality: str}`
- Handles triads (3 notes) — root, third, fifth from the scale

#### `roman_numeral(chord_pcs, tonic, mode) → dict`
- Match chord pitch classes against all 7 scale-degree triads
- Detect inversions: root position, 1st, 2nd inversion
- Format: uppercase = major ("I", "IV", "V"), lowercase = minor ("ii", "iii", "vi"), ° = dim
- Return: `{figure: str, quality: str, degree: int, inversion: int, root_name: str}`

#### `chord_name(pitches_or_pcs) → str`
- Identify chord from pitch classes using interval pattern matching
- Patterns: major [0,4,7], minor [0,3,7], dim [0,3,6], aug [0,4,8], sus2 [0,2,7], sus4 [0,5,7]
- 7ths: maj7 [0,4,7,11], min7 [0,3,7,10], dom7 [0,4,7,10], dim7 [0,3,6,9], half-dim [0,3,6,10]
- Return: `"{root} {quality}"` e.g. "C major triad", "A minor seventh"

#### `check_voice_leading(prev_pitches, curr_pitches) → list[dict]`
- Input: two sorted pitch lists (bass to soprano), from consecutive chords
- Checks outer voices (bass + soprano):
  - Parallel 5th: prev interval ≡ 7 (mod 12) AND curr interval ≡ 7 AND both voices moved
  - Parallel 8ve: same but interval ≡ 0 (mod 12)
  - Voice crossing: curr soprano < curr bass
  - Hidden 5th: both voices moved same direction AND land on interval 7

#### `chordify(notes, quant=0.125) → list[dict]`
- Group notes by quantized beat position (1/32 note grid)
- Return: `[{beat: float, duration: float, pitches: [int], pitch_classes: [int]}]`
- Replaces music21's `stream.chordify()` — same quantization logic already in `_notes_to_stream`

## Changes to `theory.py`

1. Remove ALL `from music21 import X` statements
2. Remove `_notes_to_stream()`, `_detect_key()`, `_pitch_name()`, `_parse_key_string()`, `_require_music21()`
3. Replace with imports from `._theory_engine`
4. Tools work directly on note dicts from `get_notes` → pass to engine functions
5. Same 7 tools, same parameters, same return format — API is frozen

### Per-tool changes (internal only):

| Tool | Before (music21) | After (engine) |
|------|------------------|----------------|
| `analyze_harmony` | `s.chordify()` → `romanNumeralFromChord()` | `chordify(notes)` → `roman_numeral()` |
| `suggest_next_chord` | `RomanNumeral(fig, key).pitches` | `build_chord(degree, tonic, mode)` |
| `detect_theory_issues` | `VoiceLeadingQuartet` | `check_voice_leading()` |
| `identify_scale` | `s.analyze('key')` | `detect_key(notes, mode_detection=True)` |
| `harmonize_melody` | `RomanNumeral(degree, key)` | `build_chord(degree, tonic, mode)` |
| `generate_countermelody` | `key.getScale().getPitches()` | `get_scale_pitches(tonic, mode)` |
| `transpose_smart` | `key.getScale().getPitches()` + pitch math | `get_scale_pitches()` + pitch math |

## Test Changes

### `tests/test_theory.py`
- Remove all `from music21 import` statements
- Update `TestNotesToStream` → `TestChordify` (test the engine's chordify)
- Update `TestKeyDetection` to use engine directly
- Update `TestRomanNumerals` to use engine directly
- Add `TestTheoryEngine`:
  - `test_pitch_name`: 60→"C4", 69→"A4"
  - `test_parse_key`: "A minor"→{tonic:9, mode:"minor"}
  - `test_scale_pitches`: C major→[0,2,4,5,7,9,11]
  - `test_voice_leading_parallel_fifths`: detects known parallel 5ths
  - `test_chord_name`: [0,4,7]→"C major triad"
- `test_tools_contract.py` — zero changes (142 tools, 13 domains)

### New: `tests/test_theory_engine.py`
- Standalone tests for the engine module
- No MCP server needed — pure unit tests
- Tests all 8 engine functions independently

## Documentation Updates

- `requirements.txt` — remove `# pip install music21>=9.3` comment
- All references to "requires music21" → "built-in, zero dependencies"
- SKILL.md theory section — remove music21 install note
- overview.md theory section — remove "Requires: pip install music21" line
- README.md theory row — remove "(requires music21)"
- CHANGELOG.md — entry for v1.6.5

## What We Lose

1. **Exotic chord names** — music21's `pitchedCommonName` handles hundreds of chord types. Our `chord_name()` handles ~12 common patterns. Edge case: a French augmented sixth won't get named. Producers won't notice.
2. **30+ scale modes** — music21 tests against Hungarian minor, Japanese pentatonic, etc. We test 7 Western modes. Sufficient for 99.9% of Ableton production.
3. **Automatic voice separation** — music21's `chordify` uses pitch proximity heuristics. Ours uses simple beat quantization. For MIDI clips (not sheet music), this is actually more accurate.

## What We Gain

1. **Zero dependencies** — theory tools work on every install
2. **~100MB smaller** — no music21/numpy/matplotlib in .venv
3. **Instant cold start** — no 2-3s first-import delay
4. **Full control** — no classical music bias
5. **Debuggable** — 300 lines vs 500K lines
6. **Cross-platform** — no compiled C extensions to worry about
