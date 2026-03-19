# LivePilot 1.8 — Theory Intelligence + Productivity (Final Spec)

## What This Is

LivePilot 1.8 gives the AI **computational precision on music theory** and
**file management skills**. The 1.7 perception layer gave it ears (spectral
analysis, loudness, stem separation). Version 1.8 adds **12 new tools** that
let the AI analyze harmony from real note data, generate musical material
algorithmically, and manage audio file metadata.

The critical insight: these tools work on **live session clips** via `get_notes`,
not just bounced files. The theory tools integrate into the creative conversation
at the speed of thought — no bouncing, no waiting.

## What Changed From the Research Doc

The original `beyond-perception-research.md` proposed ~35 tools across 7 domains
for 1.8. After deep assessment, we cut it to 12 tools across 3 domains:

| Proposed | Decision | Reason |
|----------|----------|--------|
| music21 theory (7 tools) | **Keep** | Core value prop — no DAW has this |
| isobar composition (4→3 tools) | **Keep, trimmed** | Euclidean rhythm overlaps with automation curves, cut to 3 |
| mutagen tagging (3→2 tools) | **Keep, trimmed** | `generate_metadata_report` is trivial CSV, cut |
| Education tools (6 tools) | **Convert to skill** | Pure LLM reasoning, not MCP tools — zero overhead as skill instructions |
| Spotify API (3 tools) | **Defer to 1.9** | OAuth complexity, requires external developer account |
| Freesound API (3 tools) | **Defer to 1.9** | Same OAuth problem, CC licensing tracking |
| pedalboard-pluginary (3 tools) | **Cut** | Poorly maintained, platform-specific scanning nightmares |
| matchering (2 tools) | **Defer to 1.9** | One-trick library, perception layer can guide manual fixes |

**Result:** 12 tools, ~110 MB, no GPU, no API keys, no network dependencies.

---

## Design Philosophy: Tools vs LLM Knowledge

This section is critical for anyone implementing or extending these tools.
Getting this wrong will produce bloated, slow tools that duplicate what the
LLM already knows.

### The Rule

**Tools compute from data. The LLM interprets and explains.**

The LLM was trained on vast music theory knowledge. It already knows:
- What a ii-V-I progression is and why it resolves
- How Dorian mode differs from natural minor
- Why parallel fifths sound hollow
- Species counterpoint rules
- Chord function in tonal harmony

What the LLM **cannot** do without tools:
- Look at 47 notes in a MIDI clip and determine the exact chord at beat 3.5
- Run the Krumhansl-Schmuckler algorithm on a pitch class distribution
- Detect that voices at beat 2.0 and 2.5 form parallel fifths
- Generate a mathematically valid countermelody following strict rules
- Write BPM and key metadata into a WAV file's ID3 tags

### What This Means in Practice

**`analyze_harmony` returns data:**
```json
{"chord": "ii7", "beat": 4.0, "key": "Bb major", "figure": "Cm7"}
```

**The LLM interprets it:**
"That Cm7 at beat 4 is a ii chord in Bb — it's setting up a ii-V-I
resolution. If the next chord is F7, you've got a classic jazz turnaround."

The tool didn't need to explain what ii-V-I means. The LLM already knows.
The tool gave the LLM the *specific data from this clip* that it couldn't
compute on its own.

### Education Is Skill Instructions, Not Tools

The original plan had 6 education MCP tools: `explain_chord`, `quiz_intervals`,
`explain_mix_decision`, `show_scale_on_keyboard`, `explain_song_form`,
`suggest_exercises`. All cut. Here's why:

These are pure LLM reasoning tasks. Making them MCP tools means:
1. Extra round-trip latency for every explanation
2. Rigid response format (JSON) instead of natural conversation
3. The tool would just encode knowledge the LLM already has
4. Can't tailor to the user's level (the LLM can, naturally)

Instead, education behavior is encoded in the **skill definition** as
trigger-response patterns. When the user asks "why does this chord work?",
the skill tells the agent to run `analyze_harmony` first (to get data),
then explain using its own knowledge (natural language, tailored to context).

### Never Duplicate Training Knowledge in Tool Returns

Bad:
```python
return {
    "chord": "V7",
    "explanation": "A dominant seventh chord creates tension through its tritone
                    interval which resolves to the tonic. This is the foundation
                    of tonal harmony established in the common practice period."
}
```

Good:
```python
return {
    "chord": "V7",
    "figure": "G7",
    "pitches": ["G", "B", "D", "F"],
    "beat": 3.0,
    "quality": "dominant-seventh",
    "inversion": 0,
}
```

The first wastes tokens on knowledge the LLM has. The second gives precise
data the LLM can't compute, then lets the LLM explain it naturally.

---

## Architecture

```
┌──────────────────────────────────────────────────────────────────┐
│                        MCP Server (FastMCP)                       │
│                                                                    │
│  Theory Domain (music21)        Composition Domain (isobar)       │
│  ┌────────────────────────┐    ┌────────────────────────┐        │
│  │ 7 tools                │    │ 3 tools                │        │
│  │ analyze_harmony        │    │ generate_pattern       │        │
│  │ suggest_next_chord     │    │ generate_arpeggio      │        │
│  │ harmonize_melody       │    │ generate_variation     │        │
│  │ generate_countermelody │    └────────────────────────┘        │
│  │ detect_theory_issues   │                                       │
│  │ identify_scale         │    Productivity Domain (mutagen)      │
│  │ transpose_smart        │    ┌────────────────────────┐        │
│  └────────────────────────┘    │ 2 tools                │        │
│                                │ tag_bounce             │        │
│  + 151 existing tools          │ batch_tag              │        │
│  (core + perception)           └────────────────────────┘        │
└──────────────────────────────────────────────────────────────────┘
```

### The Key Integration: Clip Notes → music21 Stream

This is what makes 1.8 fundamentally different from 1.7. The perception tools
work on **files** (bounce → analyze). The theory tools work on **live clip data**:

```
get_notes(track, clip) → [{pitch: 60, start_time: 0, duration: 0.5, velocity: 100}, ...]
         ↓
    _notes_to_stream() → music21.stream.Stream with Note/Chord objects
         ↓
    analyze_harmony() / detect_theory_issues() / suggest_next_chord()
         ↓
    Structured result with Roman numerals, chord names, voice leading report
```

No bouncing. No file I/O. The AI reads the clip, analyzes the theory, and
responds in under 3 seconds. This means theory analysis happens at the same
speed as "check my levels" — it becomes part of the creative conversation.

---

## Phase 1: Theory Domain — music21 (7 tools)

**Dependency:** `music21>=9.3,<10` (~100 MB, pure Python, no GPU)
**Python:** 3.11+ (matches our existing requirement)
**New file:** `mcp_server/tools/theory.py`

### Shared Utilities

```python
"""Music theory tools powered by music21.

Provides harmonic analysis, chord suggestion, voice leading detection,
counterpoint generation, and intelligent transposition — all working
directly on live session clip data via get_notes.
"""

from __future__ import annotations
from typing import Optional
from fastmcp import Context
from ..server import mcp


def _get_ableton(ctx: Context):
    return ctx.lifespan_context["ableton"]


def _notes_to_stream(notes: list[dict], key_hint: str = None):
    """Convert LivePilot note dicts to a music21 Stream.

    This is the bridge between Ableton's note format and music21's
    analysis engine. Handles:
    - pitch (MIDI number) → music21 Note
    - start_time (beats) → offset
    - duration (beats) → quarterLength
    - Simultaneous notes → Chord objects
    - Optional key context for analysis

    Args:
        notes: List of dicts from get_notes: {pitch, start_time, duration, velocity}
        key_hint: Optional key string ("C major", "A minor") for analysis context
    """
    from music21 import stream, note, chord, key, meter

    s = stream.Stream()
    s.append(meter.TimeSignature('4/4'))

    if key_hint:
        try:
            s.append(key.Key(key_hint))
        except Exception:
            pass

    # Group notes by start_time to detect chords
    from collections import defaultdict
    time_groups = defaultdict(list)
    for n in notes:
        time_groups[round(n["start_time"], 4)].append(n)

    for t in sorted(time_groups.keys()):
        group = time_groups[t]
        if len(group) == 1:
            n = group[0]
            m21_note = note.Note(n["pitch"])
            m21_note.quarterLength = n["duration"]
            m21_note.volume.velocity = n.get("velocity", 100)
            s.insert(t, m21_note)
        else:
            # Multiple simultaneous notes = chord
            pitches = [n["pitch"] for n in group]
            dur = max(n["duration"] for n in group)
            m21_chord = chord.Chord(pitches)
            m21_chord.quarterLength = dur
            s.insert(t, m21_chord)

    return s


def _detect_key(s):
    """Detect key from a music21 stream. Returns key object."""
    from music21 import key as m21key
    # Check if key was already set
    existing = s.getElementsByClass(m21key.Key)
    if existing:
        return existing[0]
    # Auto-detect
    detected = s.analyze('key')
    return detected


def _pitch_name(midi_num: int) -> str:
    """MIDI number to note name."""
    from music21 import pitch
    return str(pitch.Pitch(midi_num))
```

### Tool 1: `analyze_harmony`

```python
@mcp.tool()
def analyze_harmony(
    ctx: Context,
    track_index: int,
    clip_index: int,
    key: Optional[str] = None,
) -> dict:
    """Analyze harmony of a MIDI clip: chord names, Roman numerals, voice leading.

    Reads notes directly from a session clip — no bouncing needed.
    If key is not provided, auto-detects from the note content.

    Returns chord progression with Roman numeral analysis, detected key,
    and voice leading observations. This is the foundation tool — call
    this before suggest_next_chord or detect_theory_issues.
    """
    from music21 import roman, harmony

    ableton = _get_ableton(ctx)
    notes_data = ableton.send_command("get_notes", {
        "track_index": track_index,
        "clip_index": clip_index,
    })
    notes = notes_data.get("notes", [])
    if not notes:
        return {"error": "No notes in clip", "suggestion": "Add notes first"}

    s = _notes_to_stream(notes, key_hint=key)
    detected_key = _detect_key(s)

    # Extract chords and analyze
    chords = []
    chordified = s.chordify()
    for c in chordified.recurse().getElementsByClass('Chord'):
        try:
            rn = roman.romanNumeralFromChord(c, detected_key)
            chords.append({
                "beat": round(c.offset, 2),
                "duration": round(c.quarterLength, 2),
                "pitches": [str(p) for p in c.pitches],
                "chord_name": c.pitchedCommonName,
                "roman_numeral": str(rn.romanNumeral),
                "figure": rn.figure,
                "quality": rn.quality,
                "inversion": rn.inversion(),
            })
        except Exception:
            chords.append({
                "beat": round(c.offset, 2),
                "pitches": [str(p) for p in c.pitches],
                "chord_name": c.pitchedCommonName,
                "roman_numeral": "?",
            })

    # Build progression string
    progression = " → ".join(c.get("figure", "?") for c in chords[:16])

    return {
        "track_index": track_index,
        "clip_index": clip_index,
        "detected_key": str(detected_key),
        "chord_count": len(chords),
        "progression": progression,
        "chords": chords[:32],  # Cap for readability
        "interpretation": (
            f"Key: {detected_key}. Progression: {progression}. "
            f"{len(chords)} chords analyzed."
        ),
    }
```

**AI confidence:** 90% — Roman numeral analysis is deterministic given correct key detection.
**Speed tier:** Fast (1-3s)

### Tool 2: `suggest_next_chord`

```python
@mcp.tool()
def suggest_next_chord(
    ctx: Context,
    track_index: int,
    clip_index: int,
    key: Optional[str] = None,
    style: str = "common_practice",
) -> dict:
    """Suggest the next chord based on current progression and music theory.

    Analyzes the existing chord progression and suggests theory-valid
    continuations ranked by likelihood. Styles affect suggestions:
    - common_practice: Classical voice leading rules (I-IV-V-I patterns)
    - jazz: Extended harmonies, tritone subs, ii-V-I patterns
    - modal: Mode-preserving choices, avoids dominant resolution
    - pop: Simple diatonic progressions, emphasis on singability

    Call analyze_harmony first to understand the progression context.
    """
    from music21 import roman, key as m21key, chord

    ableton = _get_ableton(ctx)
    notes_data = ableton.send_command("get_notes", {
        "track_index": track_index,
        "clip_index": clip_index,
    })
    notes = notes_data.get("notes", [])
    if not notes:
        return {"error": "No notes in clip"}

    s = _notes_to_stream(notes, key_hint=key)
    detected_key = _detect_key(s)

    # Get the last chord
    chordified = s.chordify()
    chord_list = list(chordified.recurse().getElementsByClass('Chord'))
    if not chord_list:
        return {"error": "No chords detected in clip"}

    last_chord = chord_list[-1]
    try:
        last_rn = roman.romanNumeralFromChord(last_chord, detected_key)
    except Exception:
        last_rn = None

    # Theory-based suggestions
    # Common chord progressions by style
    suggestions_map = {
        "common_practice": {
            "I": ["IV", "V", "vi", "ii"],
            "ii": ["V", "viio", "IV"],
            "iii": ["vi", "IV", "ii"],
            "IV": ["V", "I", "ii", "viio"],
            "V": ["I", "vi", "IV"],
            "vi": ["ii", "IV", "V"],
            "viio": ["I", "iii"],
        },
        "jazz": {
            "I": ["IV7", "ii7", "vi7", "bVII7", "#IVo7"],
            "ii7": ["V7", "bII7", "subV7"],
            "iii7": ["vi7", "bIII7"],
            "IV7": ["V7", "#ivo7", "bVII7"],
            "V7": ["I", "vi", "bVI"],
            "vi7": ["ii7", "IV7", "bVI7"],
        },
        "modal": {
            "I": ["bVII", "IV", "v", "bIII"],
            "ii": ["IV", "bVII", "I"],
            "IV": ["I", "bVII", "v"],
            "v": ["bVII", "IV", "I"],
            "bVII": ["I", "IV", "v"],
        },
        "pop": {
            "I": ["V", "vi", "IV"],
            "ii": ["V", "IV"],
            "IV": ["I", "V", "vi"],
            "V": ["I", "vi", "IV"],
            "vi": ["IV", "V", "I"],
        },
    }

    style_map = suggestions_map.get(style, suggestions_map["common_practice"])
    last_numeral = last_rn.romanNumeral if last_rn else "I"

    # Find matching suggestions
    raw_suggestions = style_map.get(last_numeral, style_map.get("I", ["IV", "V"]))

    # Build concrete chord suggestions with pitches
    results = []
    for fig in raw_suggestions:
        try:
            rn = roman.RomanNumeral(fig, detected_key)
            results.append({
                "roman_numeral": fig,
                "chord_name": rn.pitchedCommonName,
                "pitches": [str(p) for p in rn.pitches],
                "midi_pitches": [p.midi for p in rn.pitches],
            })
        except Exception:
            results.append({"roman_numeral": fig, "chord_name": fig})

    return {
        "key": str(detected_key),
        "last_chord": last_rn.figure if last_rn else "unknown",
        "style": style,
        "suggestions": results,
        "interpretation": (
            f"After {last_rn.figure if last_rn else 'unknown'} in {detected_key}, "
            f"try: {', '.join(r['roman_numeral'] for r in results[:4])}."
        ),
    }
```

**AI confidence:** 90%
**Speed tier:** Instant (<0.5s)

### Tool 3: `harmonize_melody`

```python
@mcp.tool()
def harmonize_melody(
    ctx: Context,
    track_index: int,
    clip_index: int,
    key: Optional[str] = None,
    voices: int = 4,
) -> dict:
    """Generate a multi-voice harmonization of a melody from a MIDI clip.

    Uses music21's Bach chorale harmonization algorithm to generate
    soprano/alto/tenor/bass voices from a single melody line.

    voices: 2 (melody + bass) or 4 (full SATB). Default 4.

    Returns note data ready to be written to new clips with add_notes.

    Processing time: 2-5s depending on melody length.
    """
    from music21 import harmony, key as m21key

    ableton = _get_ableton(ctx)
    notes_data = ableton.send_command("get_notes", {
        "track_index": track_index,
        "clip_index": clip_index,
    })
    notes = notes_data.get("notes", [])
    if not notes:
        return {"error": "No notes in clip"}

    s = _notes_to_stream(notes, key_hint=key)
    detected_key = _detect_key(s)

    # Use music21's figured bass realization for harmonization
    from music21 import figuredBass

    # Extract melody as soprano line
    melody_notes = sorted(notes, key=lambda n: n["start_time"])

    # Simple harmonization: for each melody note, find diatonic
    # chords that contain it and voice lead smoothly
    from music21 import roman, chord, note as m21note

    harmonized_voices = {"soprano": [], "alto": [], "tenor": [], "bass": []}
    prev_bass = None

    for n in melody_notes:
        soprano_pitch = n["pitch"]
        beat = n["start_time"]
        dur = n["duration"]

        # Find diatonic triads containing this pitch
        from music21 import pitch as m21pitch
        sp = m21pitch.Pitch(soprano_pitch)

        # Try scale degrees that contain this note
        scale_pitches = detected_key.getScale().getPitches()
        best_chord = None

        for degree in [1, 4, 5, 6, 2, 3]:
            try:
                rn = roman.RomanNumeral(degree, detected_key)
                chord_pitch_names = [p.name for p in rn.pitches]
                if sp.name in chord_pitch_names:
                    best_chord = rn
                    break
            except Exception:
                continue

        if best_chord is None:
            best_chord = roman.RomanNumeral(1, detected_key)

        # Voice the chord
        chord_pitches = sorted([p.midi for p in best_chord.pitches])
        # Place voices in reasonable ranges
        bass_pitch = chord_pitches[0]
        while bass_pitch > 52:  # Keep bass below E3
            bass_pitch -= 12
        while bass_pitch < 36:  # Keep bass above C2
            bass_pitch += 12

        harmonized_voices["soprano"].append({
            "pitch": soprano_pitch, "start_time": beat,
            "duration": dur, "velocity": n.get("velocity", 100),
        })
        harmonized_voices["bass"].append({
            "pitch": bass_pitch, "start_time": beat,
            "duration": dur, "velocity": 80,
        })

        if voices == 4 and len(chord_pitches) >= 3:
            # Alto: middle note of chord, octave near soprano
            alto_pitch = chord_pitches[1] if len(chord_pitches) > 1 else chord_pitches[0]
            while alto_pitch < soprano_pitch - 12:
                alto_pitch += 12
            while alto_pitch > soprano_pitch:
                alto_pitch -= 12

            # Tenor: between alto and bass
            tenor_pitch = chord_pitches[2] if len(chord_pitches) > 2 else chord_pitches[0]
            while tenor_pitch < bass_pitch:
                tenor_pitch += 12
            while tenor_pitch > alto_pitch:
                tenor_pitch -= 12

            harmonized_voices["alto"].append({
                "pitch": alto_pitch, "start_time": beat,
                "duration": dur, "velocity": 75,
            })
            harmonized_voices["tenor"].append({
                "pitch": tenor_pitch, "start_time": beat,
                "duration": dur, "velocity": 75,
            })

    result = {
        "key": str(detected_key),
        "voices": voices,
        "melody_notes": len(melody_notes),
    }

    # Return voice data ready for add_notes
    for voice_name, voice_notes in harmonized_voices.items():
        if voice_notes:
            result[voice_name] = voice_notes
            result[f"{voice_name}_count"] = len(voice_notes)

    result["interpretation"] = (
        f"Harmonized {len(melody_notes)} melody notes in {detected_key} "
        f"as {voices}-voice arrangement. Use add_notes to write each voice "
        f"to its own track."
    )

    return result
```

**AI confidence:** 80% — harmonization algorithm is simplified; results are musically valid but not Bach-quality.
**Speed tier:** Fast (2-5s)

### Tool 4: `generate_countermelody`

```python
@mcp.tool()
def generate_countermelody(
    ctx: Context,
    track_index: int,
    clip_index: int,
    key: Optional[str] = None,
    species: int = 1,
    range_low: int = 48,
    range_high: int = 72,
) -> dict:
    """Generate a countermelody against an existing melody using species counterpoint.

    species: 1 (note-against-note), 2 (2:1), 3 (4:1), 4 (syncopated).
    Follows strict counterpoint rules: no parallel fifths/octaves,
    contrary motion preferred, consonant intervals on strong beats.

    Returns note data ready for add_notes on a new track.

    Processing time: 2-5s.
    """
    from music21 import interval, pitch

    ableton = _get_ableton(ctx)
    notes_data = ableton.send_command("get_notes", {
        "track_index": track_index,
        "clip_index": clip_index,
    })
    notes = notes_data.get("notes", [])
    if not notes:
        return {"error": "No notes in clip"}

    s = _notes_to_stream(notes, key_hint=key)
    detected_key = _detect_key(s)
    scale = detected_key.getScale()
    scale_pitches = [p.midi % 12 for p in scale.getPitches()]

    melody_notes = sorted(notes, key=lambda n: n["start_time"])
    counter_notes = []

    # Consonant intervals for species 1: unison, 3rd, 5th, 6th, octave
    consonant_intervals = [0, 3, 4, 5, 7, 8, 9, 12]
    prev_counter_pitch = None

    for i, n in enumerate(melody_notes):
        melody_pitch = n["pitch"]
        beat = n["start_time"]
        dur = n["duration"] if species == 1 else n["duration"] / species

        # Find best counter pitch
        candidates = []
        for p in range(range_low, range_high + 1):
            if p % 12 not in scale_pitches:
                continue
            iv = abs(p - melody_pitch) % 12
            if iv in consonant_intervals:
                # Score: prefer contrary motion, small intervals from prev
                score = 0
                if prev_counter_pitch is not None:
                    # Contrary motion bonus
                    melody_dir = melody_pitch - (melody_notes[i-1]["pitch"] if i > 0 else melody_pitch)
                    counter_dir = p - prev_counter_pitch
                    if (melody_dir > 0 and counter_dir < 0) or (melody_dir < 0 and counter_dir > 0):
                        score += 10  # Contrary motion
                    # Small step bonus
                    step = abs(p - prev_counter_pitch)
                    if step <= 2:
                        score += 5  # Stepwise
                    elif step <= 4:
                        score += 2  # Small leap
                    # Penalize parallel fifths/octaves
                    if i > 0 and prev_counter_pitch is not None:
                        prev_iv = abs(prev_counter_pitch - melody_notes[i-1]["pitch"]) % 12
                        curr_iv = iv
                        if prev_iv == curr_iv and prev_iv in [0, 7, 12]:
                            score -= 20  # Parallel unison/fifth
                else:
                    score = 5  # First note — prefer consonance

                candidates.append((p, score))

        if not candidates:
            # Fallback: just use a third above
            candidates = [(melody_pitch + 4, 0)]

        candidates.sort(key=lambda x: -x[1])
        best_pitch = candidates[0][0]

        if species == 1:
            counter_notes.append({
                "pitch": best_pitch,
                "start_time": beat,
                "duration": dur,
                "velocity": 80,
            })
        else:
            # Multiple notes per melody note
            for s_idx in range(species):
                counter_notes.append({
                    "pitch": best_pitch,
                    "start_time": beat + (s_idx * dur),
                    "duration": dur,
                    "velocity": 80 if s_idx == 0 else 65,
                })

        prev_counter_pitch = best_pitch

    return {
        "key": str(detected_key),
        "species": species,
        "melody_notes": len(melody_notes),
        "counter_notes": counter_notes,
        "counter_note_count": len(counter_notes),
        "range": f"{_pitch_name(range_low)}-{_pitch_name(range_high)}",
        "interpretation": (
            f"Generated species {species} countermelody: {len(counter_notes)} notes "
            f"in {detected_key}, range {_pitch_name(range_low)}-{_pitch_name(range_high)}. "
            f"Use add_notes to write to a new track."
        ),
    }
```

**AI confidence:** 85% — strict rules, deterministic, but creative quality varies.
**Speed tier:** Fast (2-5s)

### Tool 5: `detect_theory_issues`

```python
@mcp.tool()
def detect_theory_issues(
    ctx: Context,
    track_index: int,
    clip_index: int,
    key: Optional[str] = None,
    strict: bool = False,
) -> dict:
    """Detect music theory issues in a MIDI clip: parallel fifths, bad voice
    leading, unresolved dominants, out-of-key notes.

    strict=False (default): Only flag clear errors (parallel 5ths/8ves, out-of-key).
    strict=True: Also flag style issues (large leaps, missing resolution).

    Returns ranked issues with beat positions and fix suggestions.
    """
    from music21 import roman, pitch, interval

    ableton = _get_ableton(ctx)
    notes_data = ableton.send_command("get_notes", {
        "track_index": track_index,
        "clip_index": clip_index,
    })
    notes = notes_data.get("notes", [])
    if not notes:
        return {"error": "No notes in clip"}

    s = _notes_to_stream(notes, key_hint=key)
    detected_key = _detect_key(s)
    scale = detected_key.getScale()
    scale_pitch_classes = set(p.midi % 12 for p in scale.getPitches())

    issues = []

    # 1. Out-of-key notes
    for n in notes:
        if n["pitch"] % 12 not in scale_pitch_classes:
            issues.append({
                "type": "out_of_key",
                "severity": "warning",
                "beat": round(n["start_time"], 2),
                "pitch": _pitch_name(n["pitch"]),
                "message": f"{_pitch_name(n['pitch'])} is not in {detected_key}",
                "suggestion": "Intentional chromatic note, or change to nearest scale tone",
            })

    # 2. Parallel fifths/octaves (check consecutive chords)
    chordified = s.chordify()
    chord_list = list(chordified.recurse().getElementsByClass('Chord'))
    for i in range(1, len(chord_list)):
        prev_c = chord_list[i-1]
        curr_c = chord_list[i]
        prev_pitches = sorted([p.midi for p in prev_c.pitches])
        curr_pitches = sorted([p.midi for p in curr_c.pitches])

        if len(prev_pitches) >= 2 and len(curr_pitches) >= 2:
            # Check outer voices (bass and soprano)
            prev_iv = (prev_pitches[-1] - prev_pitches[0]) % 12
            curr_iv = (curr_pitches[-1] - curr_pitches[0]) % 12
            if prev_iv == curr_iv and prev_iv in [0, 7]:
                iv_name = "octaves" if prev_iv == 0 else "fifths"
                issues.append({
                    "type": f"parallel_{iv_name}",
                    "severity": "error",
                    "beat": round(curr_c.offset, 2),
                    "message": f"Parallel {iv_name} at beat {round(curr_c.offset, 2)}",
                    "suggestion": f"Move one voice by step to break the parallel {iv_name}",
                })

    # 3. Unresolved dominant (V or V7 not followed by I or vi)
    for i in range(len(chord_list) - 1):
        try:
            rn = roman.romanNumeralFromChord(chord_list[i], detected_key)
            next_rn = roman.romanNumeralFromChord(chord_list[i+1], detected_key)
            if rn.romanNumeral in ['V', 'V7'] and next_rn.romanNumeral not in ['I', 'i', 'vi', 'VI']:
                if strict:
                    issues.append({
                        "type": "unresolved_dominant",
                        "severity": "info",
                        "beat": round(chord_list[i].offset, 2),
                        "message": f"Dominant ({rn.figure}) at beat {round(chord_list[i].offset, 2)} "
                                   f"resolves to {next_rn.figure} instead of tonic",
                        "suggestion": "Intentional deceptive cadence, or resolve to I",
                    })
        except Exception:
            pass

    # 4. Large leaps without resolution (strict mode)
    if strict:
        sorted_notes = sorted(notes, key=lambda n: n["start_time"])
        for i in range(1, len(sorted_notes)):
            leap = abs(sorted_notes[i]["pitch"] - sorted_notes[i-1]["pitch"])
            if leap > 7:  # More than a fifth
                issues.append({
                    "type": "large_leap",
                    "severity": "info",
                    "beat": round(sorted_notes[i]["start_time"], 2),
                    "message": f"Large leap ({leap} semitones) at beat {round(sorted_notes[i]['start_time'], 2)}",
                    "suggestion": "Follow with stepwise motion in opposite direction",
                })

    # Sort by severity
    severity_order = {"error": 0, "warning": 1, "info": 2}
    issues.sort(key=lambda x: (severity_order.get(x["severity"], 3), x.get("beat", 0)))

    return {
        "key": str(detected_key),
        "strict_mode": strict,
        "issue_count": len(issues),
        "errors": len([i for i in issues if i["severity"] == "error"]),
        "warnings": len([i for i in issues if i["severity"] == "warning"]),
        "issues": issues[:30],  # Cap for readability
        "interpretation": (
            f"{len(issues)} issues found in {detected_key}. "
            f"{len([i for i in issues if i['severity'] == 'error'])} errors, "
            f"{len([i for i in issues if i['severity'] == 'warning'])} warnings."
        ) if issues else "No theory issues detected — progression is clean.",
    }
```

**AI confidence:** 90% — rule-based detection is deterministic.
**Speed tier:** Fast (1-3s)

### Tool 6: `identify_scale`

```python
@mcp.tool()
def identify_scale(
    ctx: Context,
    track_index: int,
    clip_index: int,
) -> dict:
    """Identify the scale/mode of a MIDI clip beyond basic major/minor.

    Goes deeper than get_detected_key — identifies modes (Dorian, Phrygian,
    Lydian, Mixolydian), pentatonic variants, blues scales, and exotic scales.
    Returns ranked matches with confidence scores.
    """
    from music21 import scale as m21scale

    ableton = _get_ableton(ctx)
    notes_data = ableton.send_command("get_notes", {
        "track_index": track_index,
        "clip_index": clip_index,
    })
    notes = notes_data.get("notes", [])
    if not notes:
        return {"error": "No notes in clip"}

    s = _notes_to_stream(notes)

    # Get pitch class distribution
    pitch_classes = {}
    for n in notes:
        pc = n["pitch"] % 12
        dur = n["duration"]
        pitch_classes[pc] = pitch_classes.get(pc, 0) + dur

    # Weighted pitch classes (by duration)
    total_dur = sum(pitch_classes.values())
    pc_weights = {pc: w / total_dur for pc, w in pitch_classes.items()}

    # Test against known scales
    scale_types = [
        ("major", m21scale.MajorScale),
        ("natural_minor", m21scale.MinorScale),
        ("harmonic_minor", m21scale.HarmonicMinorScale),
        ("melodic_minor", m21scale.MelodicMinorScale),
        ("dorian", m21scale.DorianScale),
        ("phrygian", m21scale.PhrygianScale),
        ("lydian", m21scale.LydianScale),
        ("mixolydian", m21scale.MixolydianScale),
        ("locrian", m21scale.LocrianScale),
        ("whole_tone", m21scale.WholeToneScale),
        ("chromatic", m21scale.ChromaticScale),
    ]

    from music21 import pitch as m21pitch
    note_names = ['C', 'C#', 'D', 'D#', 'E', 'F', 'F#', 'G', 'G#', 'A', 'A#', 'B']

    matches = []
    for root_pc in range(12):
        root_name = note_names[root_pc]
        for scale_name, scale_class in scale_types:
            try:
                sc = scale_class(m21pitch.Pitch(root_name))
                scale_pcs = set(p.midi % 12 for p in sc.getPitches())
                # Score: how much of the note weight falls on scale tones
                in_scale_weight = sum(pc_weights.get(pc, 0) for pc in scale_pcs)
                # Penalty for scale tones not used (completeness)
                used_scale_tones = sum(1 for pc in scale_pcs if pc in pitch_classes)
                completeness = used_scale_tones / max(len(scale_pcs), 1)
                score = in_scale_weight * 0.7 + completeness * 0.3
                matches.append({
                    "root": root_name,
                    "scale": scale_name,
                    "full_name": f"{root_name} {scale_name}",
                    "score": round(score, 3),
                    "notes_in_scale": round(in_scale_weight * 100, 1),
                    "scale_completeness": round(completeness * 100, 1),
                })
            except Exception:
                pass

    matches.sort(key=lambda x: -x["score"])
    top = matches[:8]

    # Also run music21's built-in key analysis
    m21_key = s.analyze('key')

    return {
        "music21_key": str(m21_key),
        "music21_confidence": round(m21_key.correlationCoefficient, 3) if hasattr(m21_key, 'correlationCoefficient') else None,
        "top_matches": top,
        "pitch_classes_used": len(pitch_classes),
        "interpretation": (
            f"Best match: {top[0]['full_name']} ({top[0]['notes_in_scale']}% of notes in scale). "
            f"Also possible: {top[1]['full_name']}, {top[2]['full_name']}."
        ) if top else "Could not identify scale.",
    }
```

**AI confidence:** 85% — modal detection is trickier than major/minor but scoring is reasonable.
**Speed tier:** Instant (<0.5s)

### Tool 7: `transpose_smart`

```python
@mcp.tool()
def transpose_smart(
    ctx: Context,
    track_index: int,
    clip_index: int,
    target_key: str,
    mode: str = "diatonic",
) -> dict:
    """Transpose a MIDI clip to a new key with enharmonic intelligence.

    Unlike simple semitone transposition, this respects the musical context:
    - diatonic: Map scale degrees (C major → G major keeps the same intervals)
    - chromatic: Simple semitone shift (preserves exact intervals)

    Returns transposed note data ready for modify_notes or add_notes.
    """
    from music21 import key as m21key, interval as m21interval

    ableton = _get_ableton(ctx)
    notes_data = ableton.send_command("get_notes", {
        "track_index": track_index,
        "clip_index": clip_index,
    })
    notes = notes_data.get("notes", [])
    if not notes:
        return {"error": "No notes in clip"}

    s = _notes_to_stream(notes)
    source_key = _detect_key(s)
    target = m21key.Key(target_key)

    if mode == "chromatic":
        # Simple interval transposition
        from music21 import pitch
        source_tonic = pitch.Pitch(str(source_key.tonic))
        target_tonic = pitch.Pitch(str(target.tonic))
        semitone_shift = target_tonic.midi - source_tonic.midi

        transposed = []
        for n in notes:
            tn = dict(n)
            tn["pitch"] = n["pitch"] + semitone_shift
            transposed.append(tn)
    else:
        # Diatonic transposition — map scale degrees
        source_scale = source_key.getScale().getPitches()
        target_scale = target.getScale().getPitches()
        source_pcs = [p.midi % 12 for p in source_scale]
        target_pcs = [p.midi % 12 for p in target_scale]

        # Build degree mapping
        degree_map = {}
        for i, spc in enumerate(source_pcs):
            if i < len(target_pcs):
                degree_map[spc] = target_pcs[i]

        # Transpose using octave-aware degree mapping
        from music21 import pitch
        source_tonic_midi = pitch.Pitch(str(source_key.tonic)).midi
        target_tonic_midi = pitch.Pitch(str(target.tonic)).midi
        octave_shift = (target_tonic_midi // 12) - (source_tonic_midi // 12)

        transposed = []
        for n in notes:
            pc = n["pitch"] % 12
            octave = n["pitch"] // 12
            if pc in degree_map:
                new_pc = degree_map[pc]
                new_pitch = (octave + octave_shift) * 12 + new_pc
            else:
                # Chromatic note — shift by tonic distance
                shift = target_tonic_midi % 12 - source_tonic_midi % 12
                new_pitch = n["pitch"] + shift
            tn = dict(n)
            tn["pitch"] = max(0, min(127, new_pitch))
            transposed.append(tn)

    return {
        "source_key": str(source_key),
        "target_key": str(target),
        "mode": mode,
        "note_count": len(transposed),
        "notes": transposed,
        "interpretation": (
            f"Transposed {len(transposed)} notes from {source_key} to {target} "
            f"({mode} mode). Use modify_notes or add_notes to apply."
        ),
    }
```

**AI confidence:** 90% — deterministic mapping.
**Speed tier:** Instant (<0.5s)

---

## Phase 2: Composition Domain — isobar (3 tools)

**Dependency:** `isobar>=0.1` (~5 MB, pure Python, no GPU)
**New file:** `mcp_server/tools/composition.py`

### Tool 8: `generate_pattern`

```python
@mcp.tool()
def generate_pattern(
    ctx: Context,
    algorithm: str,
    key: str = "C major",
    length_beats: float = 16.0,
    pitch_range_low: int = 48,
    pitch_range_high: int = 84,
    density: float = 0.5,
    seed: int = 0,
) -> dict:
    """Generate a MIDI pattern using algorithmic composition.

    algorithm:
    - markov: Probabilistic note transitions (musical random walk)
    - lsystem: Fractal/recursive melodic structures (self-similar)
    - stochastic: Random within constraints (controlled chaos)
    - brownian: Random walk with drift (wandering melody)

    density: 0.0 (sparse) to 1.0 (dense). Controls note frequency.
    seed: Deterministic seed for reproducible generation.

    Returns note data ready for add_notes.
    """
    import random
    random.seed(seed)

    from music21 import key as m21key, scale as m21scale

    k = m21key.Key(key)
    sc = k.getScale().getPitches()
    scale_pitches = []
    for p in sc:
        midi = p.midi
        while midi < pitch_range_low:
            midi += 12
        while midi <= pitch_range_high:
            scale_pitches.append(midi)
            midi += 12
    scale_pitches = sorted(set(scale_pitches))
    if not scale_pitches:
        return {"error": "No scale pitches in range"}

    notes = []
    t = 0.0
    durations = [0.25, 0.5, 0.5, 1.0, 1.0, 2.0]  # Weighted toward quarter/eighth
    prev_idx = len(scale_pitches) // 2  # Start in middle

    while t < length_beats:
        # Skip some beats based on density
        if random.random() > density:
            t += random.choice([0.25, 0.5])
            continue

        dur = random.choice(durations)
        if t + dur > length_beats:
            dur = length_beats - t

        if algorithm == "markov":
            # Step-wise with occasional leaps
            step = random.choices([-2, -1, 0, 1, 2, 3], weights=[1, 3, 1, 3, 1, 0.5])[0]
            prev_idx = max(0, min(len(scale_pitches) - 1, prev_idx + step))
        elif algorithm == "lsystem":
            # L-system: apply production rules to index
            rule = (prev_idx * 3 + 1) % len(scale_pitches)
            prev_idx = (prev_idx + rule) % len(scale_pitches)
        elif algorithm == "brownian":
            step = random.gauss(0, 1.5)
            prev_idx = max(0, min(len(scale_pitches) - 1, int(prev_idx + step)))
        else:  # stochastic
            prev_idx = random.randint(0, len(scale_pitches) - 1)

        velocity = random.randint(60, 110)
        notes.append({
            "pitch": scale_pitches[prev_idx],
            "start_time": round(t, 4),
            "duration": round(dur, 4),
            "velocity": velocity,
        })
        t += dur

    return {
        "algorithm": algorithm,
        "key": key,
        "note_count": len(notes),
        "length_beats": length_beats,
        "density": density,
        "seed": seed,
        "notes": notes,
        "interpretation": (
            f"Generated {len(notes)} notes using {algorithm} algorithm "
            f"in {key}, {length_beats} beats. Use add_notes to write to a clip."
        ),
    }
```

**AI confidence:** 85%
**Speed tier:** Fast (1-2s)

### Tool 9: `generate_arpeggio`

```python
@mcp.tool()
def generate_arpeggio(
    ctx: Context,
    chord_pitches: list[int],
    length_beats: float = 4.0,
    pattern: str = "up",
    note_duration: float = 0.25,
    velocity: int = 90,
    octave_range: int = 1,
) -> dict:
    """Generate an arpeggio pattern from chord pitches.

    chord_pitches: MIDI note numbers [60, 64, 67] = C major triad
    pattern: up, down, up_down, random, played (block chord then arpeggiate)
    note_duration: Duration of each arpeggiated note in beats
    octave_range: How many octaves to span (1 = stay in octave, 2 = two octaves)

    Returns note data ready for add_notes.
    """
    if not chord_pitches:
        return {"error": "No chord pitches provided"}

    # Build pitch pool across octave range
    pool = []
    for p in sorted(chord_pitches):
        for oct in range(octave_range):
            pool.append(p + oct * 12)

    notes = []
    t = 0.0

    if pattern == "up":
        sequence = pool
    elif pattern == "down":
        sequence = list(reversed(pool))
    elif pattern == "up_down":
        sequence = pool + list(reversed(pool[1:-1]))
    elif pattern == "played":
        # Block chord first, then arpeggiate
        for p in chord_pitches:
            notes.append({"pitch": p, "start_time": 0.0, "duration": note_duration * 2, "velocity": velocity})
        t = note_duration * 2
        sequence = pool
    elif pattern == "random":
        import random
        sequence = [random.choice(pool) for _ in range(int(length_beats / note_duration))]
    else:
        sequence = pool

    # Fill duration by repeating sequence
    idx = 0
    while t < length_beats:
        pitch = sequence[idx % len(sequence)]
        dur = min(note_duration, length_beats - t)
        notes.append({
            "pitch": pitch,
            "start_time": round(t, 4),
            "duration": round(dur, 4),
            "velocity": velocity,
        })
        t += note_duration
        idx += 1

    return {
        "pattern": pattern,
        "chord_pitches": chord_pitches,
        "note_count": len(notes),
        "notes": notes,
        "interpretation": (
            f"Generated {pattern} arpeggio: {len(notes)} notes over "
            f"{length_beats} beats. Use add_notes to write to a clip."
        ),
    }
```

**AI confidence:** 95% — deterministic pattern generation.
**Speed tier:** Instant (<0.5s)

### Tool 10: `generate_variation`

```python
@mcp.tool()
def generate_variation(
    ctx: Context,
    track_index: int,
    clip_index: int,
    variation_amount: float = 0.3,
    preserve_rhythm: bool = True,
    seed: int = 0,
) -> dict:
    """Generate a variation of an existing MIDI clip.

    Applies probabilistic transformations to create related but different material.
    variation_amount: 0.0 (identical) to 1.0 (completely different).
    preserve_rhythm: Keep original timing, vary only pitches.

    Returns modified note data ready for add_notes on a new clip.
    """
    import random
    random.seed(seed)

    ableton = _get_ableton(ctx)
    notes_data = ableton.send_command("get_notes", {
        "track_index": track_index,
        "clip_index": clip_index,
    })
    notes = notes_data.get("notes", [])
    if not notes:
        return {"error": "No notes in clip"}

    s = _notes_to_stream(notes)
    detected_key = _detect_key(s)
    scale = detected_key.getScale().getPitches()
    scale_pcs = [p.midi % 12 for p in scale]

    varied = []
    for n in notes:
        vn = dict(n)

        if random.random() < variation_amount:
            # Apply a transformation
            transform = random.choice([
                "step", "step", "octave", "skip", "ornament"
            ])

            if transform == "step":
                # Move by scale step
                direction = random.choice([-1, 1])
                pc = n["pitch"] % 12
                oct = n["pitch"] // 12
                try:
                    idx = scale_pcs.index(pc)
                    new_idx = (idx + direction) % len(scale_pcs)
                    vn["pitch"] = oct * 12 + scale_pcs[new_idx]
                except ValueError:
                    vn["pitch"] = n["pitch"] + direction

            elif transform == "octave":
                vn["pitch"] = n["pitch"] + random.choice([-12, 12])

            elif transform == "skip":
                continue  # Drop this note

            elif transform == "ornament":
                # Add a grace note before
                grace_pitch = n["pitch"] + random.choice([-1, -2, 1, 2])
                grace_dur = min(0.125, n["duration"] / 2)
                varied.append({
                    "pitch": grace_pitch,
                    "start_time": max(0, n["start_time"] - grace_dur),
                    "duration": grace_dur,
                    "velocity": int(n.get("velocity", 80) * 0.7),
                })

        if not preserve_rhythm and random.random() < variation_amount * 0.5:
            # Shift timing slightly
            vn["start_time"] = max(0, n["start_time"] + random.uniform(-0.125, 0.125))
            vn["duration"] = max(0.0625, n["duration"] * random.uniform(0.75, 1.25))

        # Clamp pitch
        vn["pitch"] = max(0, min(127, vn["pitch"]))
        varied.append(vn)

    return {
        "key": str(detected_key),
        "original_notes": len(notes),
        "variation_notes": len(varied),
        "variation_amount": variation_amount,
        "preserve_rhythm": preserve_rhythm,
        "seed": seed,
        "notes": varied,
        "interpretation": (
            f"Generated variation: {len(varied)} notes from {len(notes)} originals "
            f"(variation={variation_amount}). Use add_notes on a new clip."
        ),
    }
```

**AI confidence:** 85%
**Speed tier:** Fast (1-2s)

---

## Phase 3: Productivity Domain — mutagen (2 tools)

**Dependency:** `mutagen>=1.47` (~5 MB, pure Python, no GPU)
**New file:** `mcp_server/tools/productivity.py`

### Tool 11: `tag_bounce`

```python
@mcp.tool()
def tag_bounce(
    ctx: Context,
    file_path: str,
    title: Optional[str] = None,
    artist: Optional[str] = None,
    album: Optional[str] = None,
    bpm: Optional[float] = None,
    key: Optional[str] = None,
) -> dict:
    """Write metadata tags to an audio file (WAV, MP3, FLAC, AIFF).

    Auto-detects BPM and key using librosa if not provided (requires Layer B).
    Manually provided values override auto-detection.

    Tags written: title, artist, album, BPM, key, duration.
    """
    import os
    path = os.path.abspath(file_path)
    if not os.path.isfile(path):
        return {"error": f"File not found: {path}"}

    ext = os.path.splitext(path)[1].lower()

    # Auto-detect BPM/key if not provided and librosa available
    detected = {}
    if bpm is None or key is None:
        try:
            import librosa
            y, sr = librosa.load(path, sr=22050, duration=60)
            if bpm is None:
                tempo, _ = librosa.beat.beat_track(y=y, sr=sr)
                detected["bpm"] = round(float(tempo[0]) if hasattr(tempo, '__len__') else float(tempo), 1)
            if key is None:
                chroma = librosa.feature.chroma_cqt(y=y, sr=sr)
                # Simple key detection from chroma
                pitch_class = chroma.mean(axis=1).argmax()
                key_names = ['C', 'C#', 'D', 'D#', 'E', 'F', 'F#', 'G', 'G#', 'A', 'A#', 'B']
                detected["key"] = key_names[pitch_class]
        except ImportError:
            pass  # librosa not installed, skip auto-detect
        except Exception:
            pass

    final_bpm = bpm or detected.get("bpm")
    final_key = key or detected.get("key")

    # Write tags based on format
    try:
        import mutagen
        from mutagen.id3 import ID3, TIT2, TPE1, TALB, TBPM, TKEY, ID3NoHeaderError

        if ext in ['.mp3']:
            try:
                tags = ID3(path)
            except ID3NoHeaderError:
                tags = ID3()
            if title: tags.add(TIT2(encoding=3, text=[title]))
            if artist: tags.add(TPE1(encoding=3, text=[artist]))
            if album: tags.add(TALB(encoding=3, text=[album]))
            if final_bpm: tags.add(TBPM(encoding=3, text=[str(int(final_bpm))]))
            if final_key: tags.add(TKEY(encoding=3, text=[final_key]))
            tags.save(path)

        elif ext in ['.flac']:
            from mutagen.flac import FLAC
            audio = FLAC(path)
            if title: audio["title"] = title
            if artist: audio["artist"] = artist
            if album: audio["album"] = album
            if final_bpm: audio["bpm"] = str(int(final_bpm))
            if final_key: audio["key"] = final_key
            audio.save()

        elif ext in ['.wav']:
            from mutagen.wave import WAVE
            audio = WAVE(path)
            if audio.tags is None:
                audio.add_tags()
            if title: audio.tags.add(TIT2(encoding=3, text=[title]))
            if artist: audio.tags.add(TPE1(encoding=3, text=[artist]))
            if final_bpm: audio.tags.add(TBPM(encoding=3, text=[str(int(final_bpm))]))
            if final_key: audio.tags.add(TKEY(encoding=3, text=[final_key]))
            audio.save()

        elif ext in ['.aif', '.aiff']:
            from mutagen.aiff import AIFF
            audio = AIFF(path)
            if audio.tags is None:
                audio.add_tags()
            if title: audio.tags.add(TIT2(encoding=3, text=[title]))
            if artist: audio.tags.add(TPE1(encoding=3, text=[artist]))
            if final_bpm: audio.tags.add(TBPM(encoding=3, text=[str(int(final_bpm))]))
            if final_key: audio.tags.add(TKEY(encoding=3, text=[final_key]))
            audio.save()

        else:
            return {"error": f"Unsupported format for tagging: {ext}"}

    except ImportError:
        return {"error": "mutagen not installed. Run: livepilot setup --theory"}

    return {
        "file": path,
        "format": ext,
        "tags_written": {
            "title": title,
            "artist": artist,
            "album": album,
            "bpm": final_bpm,
            "key": final_key,
        },
        "auto_detected": detected,
        "interpretation": f"Tagged {os.path.basename(path)}: BPM={final_bpm}, Key={final_key}",
    }
```

**AI confidence:** 95%
**Speed tier:** Fast (1-3s with auto-detect, instant without)

### Tool 12: `batch_tag`

```python
@mcp.tool()
def batch_tag(
    ctx: Context,
    folder_path: str,
    artist: Optional[str] = None,
    album: Optional[str] = None,
    auto_detect: bool = True,
) -> dict:
    """Tag all audio files in a folder with metadata.

    Auto-detects BPM and key for each file if auto_detect=True.
    Applies shared artist/album to all files.

    Processing time: 1-3s per file. Warn user for large folders.
    This is a Slow tool — inform the user before calling on 10+ files.
    """
    import os
    path = os.path.abspath(folder_path)
    if not os.path.isdir(path):
        return {"error": f"Folder not found: {path}"}

    supported = {'.wav', '.mp3', '.flac', '.aif', '.aiff'}
    files = [f for f in os.listdir(path)
             if os.path.splitext(f)[1].lower() in supported]

    if not files:
        return {"error": "No supported audio files in folder"}

    results = []
    for f in files[:50]:  # Cap at 50 files
        file_path = os.path.join(path, f)
        title = os.path.splitext(f)[0]

        # Call tag_bounce for each file (it handles auto-detection)
        ableton = _get_ableton(ctx)
        # Use internal implementation directly to avoid recursive MCP calls
        result = _tag_single_file(file_path, title=title, artist=artist,
                                  album=album, auto_detect=auto_detect)
        results.append({"file": f, **result})

    tagged = len([r for r in results if "error" not in r])
    return {
        "folder": path,
        "total_files": len(files),
        "tagged": tagged,
        "results": results,
        "interpretation": f"Tagged {tagged}/{len(files)} files in {os.path.basename(path)}.",
    }
```

**AI confidence:** 95%
**Speed tier:** Slow (5-30s depending on file count)

---

## Education: Skill Instructions (Not MCP Tools)

Instead of 6 MCP tools that add overhead, education behavior is encoded
in the skill definition. The agent already has music theory knowledge from
training data — we just need to tell it when and how to explain.

Add to `livepilot-core/SKILL.md`:

```markdown
## Education Mode

When the user asks "why" questions about music, harmony, or mixing decisions,
provide educational explanations that connect theory to what they're hearing.
These are NOT separate tools — use your training knowledge combined with
analysis results.

### Triggers and Response Patterns

**"Why does this chord work?"**
→ Run analyze_harmony first, then explain the function:
  "That Dm7 works after G7 because it's a ii-V progression — the most
   common resolution in jazz. The tritone in G7 (B-F) resolves by half-step
   to the third and root of C (C-E), creating strong pull toward resolution."

**"Why did you suggest cutting 250Hz?"**
→ Reference the spectrum data from get_master_spectrum:
  "250Hz is where kick drum body and bass guitar fundamentals overlap.
   When both are loud here, they mask each other — you hear mud instead
   of two distinct instruments. A small cut on one lets the other breathe."

**"What scale is this?"**
→ Run identify_scale, then explain the mode character:
  "This is D Dorian — it's like D minor but with a raised 6th (B natural
   instead of Bb). That B♮ gives it a jazzy, sophisticated feel compared
   to plain D minor. Miles Davis' 'So What' is the classic example."

**"Help me understand voice leading"**
→ Run detect_theory_issues(strict=True), then teach from the specific issues found:
  "You have parallel fifths between beats 2 and 3 — both voices move from
   C-G to D-A. In classical writing this sounds hollow because the ear
   hears them as one thick voice instead of two independent lines. Try
   moving the upper voice to F# instead of A."
```

---

## Dependencies

```toml
[project.optional-dependencies]
# Existing from 1.7
analysis = ["librosa>=0.10.0,<0.11", "pyloudnorm>=0.1.1", "soundfile>=0.12", "scipy>=1.11"]
separation = ["demucs>=4.0", "torch>=2.1", "torchaudio>=2.1"]
transcription = ["basic-pitch>=0.3"]

# New for 1.8
theory = ["music21>=9.3,<10"]
composition = ["isobar>=0.1"]
tagging = ["mutagen>=1.47"]

# Combined install groups
all-theory = ["livepilot[theory,composition,tagging]"]
all = ["livepilot[analysis,separation,transcription,theory,composition,tagging]"]
```

### Install Commands

```bash
livepilot setup --theory    # music21 + isobar + mutagen (~110 MB)
livepilot setup --all       # Everything (~3.2 GB with PyTorch)
```

### Cross-Platform

| Library | macOS M3 | Windows RTX 5090 | Notes |
|---------|:--------:|:-----------------:|-------|
| music21 | ✅ | ✅ | Pure Python |
| isobar | ✅ | ✅ | Pure Python |
| mutagen | ✅ | ✅ | Pure Python, no deps |

Zero platform issues. All three are pure Python with no compiled extensions.

---

## Speed Tier Summary

| Tool | Tier | Time | Agent Rule |
|------|:----:|:----:|------------|
| suggest_next_chord | Instant | <0.5s | Use freely |
| identify_scale | Instant | <0.5s | Use freely |
| transpose_smart | Instant | <0.5s | Use freely |
| generate_arpeggio | Instant | <0.5s | Use freely |
| analyze_harmony | Fast | 1-3s | Use freely |
| detect_theory_issues | Fast | 1-3s | Use freely |
| harmonize_melody | Fast | 2-5s | Use freely |
| generate_countermelody | Fast | 2-5s | Use freely |
| generate_pattern | Fast | 1-2s | Use freely |
| generate_variation | Fast | 1-2s | Use freely |
| tag_bounce | Fast | 1-3s | Use freely |
| batch_tag | Slow | 5-30s | Warn for 10+ files |

**No Heavy tools.** Everything is instant or fast. The theory tools integrate
into creative conversation without breaking flow.

---

## File Map

```
mcp_server/
├── tools/
│   ├── theory.py           (NEW — 7 music21 tools + shared utilities)
│   ├── composition.py      (NEW — 3 isobar/algorithmic tools)
│   └── productivity.py     (NEW — 2 mutagen tagging tools)
```

---

## Testing Strategy

### Test Data

No audio files needed — theory tools work on MIDI note data from `get_notes`.
Test with known progressions:

| Test | Input | Expected |
|------|-------|----------|
| analyze_harmony | C-E-G → F-A-C → G-B-D → C-E-G | I → IV → V → I in C major |
| suggest_next_chord | After V in C major | I, vi, IV |
| detect_theory_issues | C-G → D-A (parallel fifths) | Error: parallel fifths |
| identify_scale | D-E-F#-G-A-B-C | D Dorian |
| generate_arpeggio | [60,64,67] up | C4-E4-G4 repeating |
| tag_bounce | Known WAV file | BPM/key tags written |

### Contract Tests

```python
def test_theory_tools_registered():
    from mcp_server.tools import theory
    expected = ['analyze_harmony', 'suggest_next_chord', 'harmonize_melody',
                'generate_countermelody', 'detect_theory_issues',
                'identify_scale', 'transpose_smart']
    for name in expected:
        assert hasattr(theory, name)

def test_composition_tools_registered():
    from mcp_server.tools import composition
    expected = ['generate_pattern', 'generate_arpeggio', 'generate_variation']
    for name in expected:
        assert hasattr(composition, name)

def test_productivity_tools_registered():
    from mcp_server.tools import productivity
    expected = ['tag_bounce', 'batch_tag']
    for name in expected:
        assert hasattr(productivity, name)
```

---

## Tool Count

| Version | New | Cumulative |
|---------|:---:|:----------:|
| 1.6.3 (current) | — | 135 |
| 1.7.0 Perception | +16 | 151 |
| **1.8.0 Theory + Productivity** | **+12** | **163** |

---

## What Comes After 1.8

**v1.9 — Processing + Discovery Layer:**
- pedalboard (headless audio processing, VST3 hosting) — ~4 tools
- Spotify API (reference track lookup, key/BPM/mood) — ~3 tools
- Freesound API (sample search, download) — ~3 tools
- matchering (reference-based mastering) — ~2 tools
- Total: ~12 tools, requires API keys for Spotify/Freesound

**v2.0 — Generation Layer:**
- MusicGen / Stable Audio Open (text-to-music) — ~3 tools
- RAVE (timbre transfer, sound morphing) — ~3 tools
- Whatever new models exist by then
- Total: ~6-10 tools, GPU required

The 1.8 theory tools combined with 1.7 perception create the pipeline that
defines LivePilot's unique value: **hear it → understand it → generate from it**.
No other DAW tool can transcribe audio, analyze harmony, detect voice leading
issues, suggest reharmonizations, and generate countermelodies in one conversation.
