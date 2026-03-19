# Beyond Perception: What Else Can LivePilot Do?

## The Question

The 1.7 perception layer gives LivePilot ears. But ears are just one sense. What other
Python libraries, APIs, and AI models can LivePilot integrate to become a genuinely
comprehensive music production assistant — across composition, sound design, productivity,
inspiration, and education?

Everything below is something **Ableton Live and Max4Live cannot do natively**.

---

## Category 1: Music Theory & Composition Intelligence

### music21 (MIT) — The Music Theory Brain
**What it is:** The most comprehensive music theory library in any language. 20+ years
of development at MIT. v10 released 2026. Actively maintained.

**What it gives LivePilot that nothing else can:**
- **Voice leading analysis** — detect parallel fifths, hidden octaves, crossed voices
- **Chord progression generation** — suggest next chords based on theory rules
- **Species counterpoint** — generate countermelodies (levels 1-5, strict rules)
- **Bach chorale harmonization** — given a melody, generate 4-part harmony
- **Roman numeral analysis** — "this progression is I-vi-IV-V in C major"
- **Mode/scale identification** — beyond key detection, identify modes and exotic scales
- **Sheet music generation** — export analysis as MusicXML or LilyPond notation

**Critical finding:** An MCP server for music21 ALREADY EXISTS on GitHub
(brightlikethelight/music21-mcp-server) with 13 tools. LivePilot could fork or integrate.

**MCP tools this enables:**
- `suggest_next_chord` — given current progression + key, suggest theory-valid options
- `harmonize_melody` — input melody MIDI → output 4-part harmonization
- `analyze_harmony` — input MIDI clip → Roman numeral analysis + voice leading report
- `generate_countermelody` — species counterpoint over existing melody
- `detect_theory_issues` — flag parallel fifths, bad voice leading, unresolved dominants
- `transpose_smart` — transpose respecting enharmonic spelling and key context
- `generate_lead_sheet` — export chord chart as PDF/MusicXML from MIDI analysis

**Install:** `pip install music21` (~100 MB). Pure Python. No GPU.
**AI confidence:** 90%+ — music theory is structured rules, exactly what AI excels at.

---

### isobar — Algorithmic Composition Engine
**What it is:** Python library for creating musical patterns using algorithms — Markov
chains, Euclidean rhythms, L-systems, probability distributions.

**What it gives LivePilot:**
- **Markov chain composition** — analyze a corpus of MIDI → generate new material in that style
- **Euclidean rhythm generation** — mathematically optimal beat distributions
- **Arpeggiator patterns** — up, down, random, with probability weighting
- **L-system melodies** — fractal/recursive musical structures
- **Stochastic generation** — controlled randomness with musical constraints

**MCP tools this enables:**
- `generate_pattern` — create MIDI pattern from algorithm (markov/euclidean/lsystem)
- `generate_arpeggio` — input chord + rhythm style → output MIDI arpeggio
- `generate_euclidean_rhythm` — input pulses + steps → rhythmic pattern
- `generate_variation` — take existing MIDI → create probabilistic variation

**Install:** `pip install isobar` (~5 MB). Tiny. No GPU.
**AI confidence:** 85% — algorithmic rules are deterministic, I can tune parameters well.

---

## Category 2: Sound Design & Audio Generation

### Pedalboard (Spotify) — Headless Audio Processing
**What it is:** Spotify's open-source audio effects library. Wraps JUCE C++ framework.
Can load VST3/AU plugins headlessly. **300x faster than pySoX.**

**What it gives LivePilot:**
- **Batch audio processing** — apply effects chains to files without opening a DAW
- **VST3/AU hosting** — load any installed plugin in Python, render audio through it
- **Built-in effects** — reverb, delay, compression, EQ, chorus, distortion, limiter
- **Sample pre-processing** — normalize, trim, fade, resample before loading into Live

**MCP tools this enables:**
- `process_audio` — apply effect chain to a file (e.g., "add reverb + compress")
- `render_through_plugin` — run audio through a specific VST3 headlessly
- `batch_normalize` — normalize a folder of samples to consistent loudness
- `create_stems_mix` — mix separated stems with custom processing per stem

**Install:** `pip install pedalboard` (~20 MB). CPU only. Very fast.
**AI confidence:** 85% — I know what effects do, can suggest processing chains.

---

### RAVE — Real-Time Sound Morphing & Timbre Transfer
**What it is:** Neural audio synthesis that runs in real-time on CPU. From IRCAM Paris.
Can morph between two sounds, transfer timbre, and explore a latent sonic space.

**What it gives LivePilot:**
- **Timbre transfer** — make a drum loop sound like it's played on piano
- **Sound interpolation** — smoothly morph between two samples
- **Texture generation** — create evolving sonic textures from seed sounds
- **Latent space exploration** — navigate a continuous space of sounds

**MCP tools this enables:**
- `morph_sounds` — input two audio files + blend ratio → output morphed audio
- `transfer_timbre` — apply timbre of source A to content of source B
- `generate_texture` — create ambient texture from seed sample + parameters

**Install:** Requires PyTorch (already in Layer C). Pre-trained models available.
Runs real-time on CPU. RTX 5090 makes it instantaneous.
**AI confidence:** 60% — I can control parameters but sonic results are unpredictable.

---

### MusicGen / Stable Audio Open — Text-to-Music Generation
**What it is:** Neural models that generate music from text descriptions.

**MusicGen (Meta):** 300M-3.3B params. Generates ~30 sec of audio.
**Stable Audio Open:** 1.21B params. Slightly better quality. Faster inference.
**ACE-Step v1.5 (2026):** Newest. 4 minutes of music in 20 seconds on A100.

**What it gives LivePilot:**
- Generate background tracks from text ("chill lo-fi hip hop, 85 BPM, rainy mood")
- Create scratch audio for arrangement prototyping
- Generate sound effects and ambient beds

**MCP tools this enables:**
- `generate_audio` — text prompt → audio file
- `generate_loop` — text prompt + BPM + bars → tempo-synced loop
- `generate_variation` — input audio + text guidance → modified version

**Install:** Requires PyTorch. ~2-4 GB model weights. 16GB+ VRAM recommended.
RTX 5090's 32GB VRAM runs the largest models easily.
**AI confidence:** 70% — I can craft good prompts, but output quality varies.

---

### Freesound API — Sample Search & Download
**What it is:** API to search and download from 600K+ Creative Commons samples on
Freesound.org. Python client: `freesound-python`.

**MCP tools this enables:**
- `search_samples` — "punchy kick 808" → returns matching samples with previews
- `download_sample` — fetch sample directly to project folder
- `find_similar` — upload a sample → find perceptually similar sounds

**Install:** `pip install freesound-python`. Requires free API key.
**AI confidence:** 80% — I can describe sounds well, search is text-based.

---

### Matchering — Reference-Based Mastering
**What it is:** Open-source audio matching tool. Takes your track + a reference →
matches RMS, frequency response, stereo width, and peak levels.

**MCP tools this enables:**
- `master_to_reference` — input track + reference → mastered output
- `match_eq` — match frequency balance to reference without full mastering

**Install:** `pip install matchering`. No GPU.
**AI confidence:** 75% — it's algorithmic, not AI. Results are predictable.

---

## Category 3: Productivity & Workflow

### Spotify API — Reference Track Intelligence
**What it is:** Spotify's Web API returns audio features for any track:
BPM, key, mode, time signature, danceability, energy, valence (happiness).

**What it gives LivePilot:**
- Instant BPM/key lookup for any reference track
- Mood/energy metrics for creative direction matching
- Compatible key suggestions for harmonic mixing/mashups

**MCP tools this enables:**
- `lookup_reference` — "Daft Punk Get Lucky" → BPM:116, Key:F#m, Energy:0.81
- `find_tracks_in_key` — find Spotify tracks matching your session's key + BPM range
- `suggest_compatible_keys` — for DJ/mashup work, find harmonically compatible keys

**Install:** `pip install spotipy`. Requires Spotify developer API key (free).
**AI confidence:** 95% — structured data lookup, zero ambiguity.

---

### mutagen + librosa — Auto-Tag Bounces
**What it is:** Read/write audio file metadata (ID3 tags, Vorbis comments).
Combined with librosa for BPM/key detection.

**MCP tools this enables:**
- `tag_bounce` — analyze a bounce → write BPM, key, LUFS, duration to file tags
- `batch_tag` — tag all files in a folder with detected metadata
- `generate_metadata_report` — CSV export of all bounce metadata

**Install:** `pip install mutagen` (~5 MB). Already have librosa from Layer B.
**AI confidence:** 95% — reading/writing tags is deterministic.

---

### pedalboard-pluginary — VST/AU Plugin Scanner
**What it is:** Scans your system for installed VST3/AU plugins, builds a searchable
database with metadata (name, vendor, category, I/O config).

**MCP tools this enables:**
- `scan_plugins` — catalog all installed VST3/AU plugins
- `find_plugin` — search by name, category, vendor ("find all reverb plugins")
- `suggest_plugin` — "I need a compressor for vocals" → returns matching installed plugins

**Install:** `pip install pedalboard-pluginary`. Uses SQLite for caching.
**AI confidence:** 80% — can match descriptions to plugin categories well.

---

## Category 4: AI-Powered Creative Tools

### Groove2Groove — MIDI Style Transfer
**What it is:** Neural network that transfers the style of one MIDI accompaniment to
another. Input: your chord progression + a reference style → output: your chords played
in the reference's style (rhythm, voicing, density).

**MCP tools this enables:**
- `transfer_style` — input MIDI + style reference MIDI → restyled MIDI output
- `generate_accompaniment` — chord progression + genre → styled accompaniment

**Install:** PyTorch + pre-trained checkpoints (~500 MB).
**AI confidence:** 70% — style transfer results vary. Best on 8-bar sections.

---

### RVC (Retrieval-based Voice Conversion) — Voice Cloning
**What it is:** Real-time voice conversion. Train a model on a voice → convert any
vocal to sound like that voice. The current standard for open-source voice cloning.

**MCP tools this enables:**
- `clone_voice` — input vocal + target voice model → converted vocal
- `create_voice_model` — train a new voice model from audio samples

**Install:** PyTorch + pretrained models. 6-12 GB VRAM for inference.
**AI confidence:** 50% — I can run the pipeline but judging vocal quality is subjective.
**Ethics note:** Only use on your own voice or with explicit permission.

---

### Drum Pattern Generation (MidiDrumiGen)
**What it is:** AI drum pattern generator that outputs MIDI. Designed specifically
for Ableton Live integration. PyTorch + FastAPI.

**MCP tools this enables:**
- `generate_drums` — style/genre + BPM → MIDI drum pattern
- `humanize_drums` — add velocity variation and timing drift to quantized drums

**Install:** PyTorch + FastAPI. Moderate size.
**AI confidence:** 75% — pattern quality depends on training data, but I can
suggest appropriate styles and tweak parameters effectively.

---

## Category 5: Education & Self-Teaching

This one is special: LivePilot doesn't just need tools, it needs to **teach**. The AI
already knows music theory from training data. With music21 and the perception layer,
it can combine analysis with explanation.

**MCP tools this enables (no additional libraries needed):**
- `explain_chord` — "Why does this Dm7 work after G7?" → circle of fifths, ii-V-I
- `explain_mix_decision` — "Why did you suggest cutting 250Hz?" → spectral masking explanation
- `quiz_intervals` — play two notes → ask user to identify the interval
- `show_scale_on_keyboard` — generate visual of scale degrees on piano keyboard
- `explain_song_form` — after structure analysis, explain why the form works
- `suggest_exercises` — based on detected theory weaknesses, suggest practice material

**AI confidence:** 90% — music theory explanation is pure knowledge work.
This is one of the highest-value, lowest-effort additions.

---

## The Priority Matrix: What to Build and When

### Tier 1 — Build Now (Huge Value, Low Complexity)

| Tool Domain | Library | New MCP Tools | Install Size | GPU? |
|------------|---------|:---:|---|:---:|
| **Music Theory** | music21 | ~7 tools | ~100 MB | No |
| **Auto-Tagging** | mutagen | ~3 tools | ~5 MB | No |
| **Reference Lookup** | spotipy | ~3 tools | ~2 MB | No |
| **Education** | (uses music21 + existing) | ~6 tools | 0 | No |

These four add ~19 tools with minimal dependencies. music21 is the star — it's the
music theory equivalent of what librosa is for audio analysis. And education tools
cost nothing extra because they just combine existing analysis with explanation.

### Tier 2 — Build Next (High Value, Medium Complexity)

| Tool Domain | Library | New MCP Tools | Install Size | GPU? |
|------------|---------|:---:|---|:---:|
| **Algorithmic Composition** | isobar | ~4 tools | ~5 MB | No |
| **Audio Processing** | pedalboard | ~4 tools | ~20 MB | No |
| **Sample Search** | freesound-python | ~3 tools | ~1 MB | No |
| **Plugin Scanning** | pedalboard-pluginary | ~3 tools | ~5 MB | No |
| **Reference Mastering** | matchering | ~2 tools | ~10 MB | No |

These add ~16 tools. Still no GPU. Pedalboard is the standout — headless VST hosting
is a genuine superpower that no DAW exposes programmatically.

### Tier 3 — Build Later (High Value, High Complexity)

| Tool Domain | Library | New MCP Tools | Install Size | GPU? |
|------------|---------|:---:|---|:---:|
| **Text-to-Music** | MusicGen/Stable Audio | ~3 tools | ~4 GB | Yes |
| **Sound Morphing** | RAVE | ~3 tools | ~500 MB | Optional |
| **Style Transfer** | Groove2Groove | ~2 tools | ~500 MB | Yes |
| **Drum Generation** | MidiDrumiGen | ~2 tools | ~200 MB | Yes |
| **Voice Cloning** | RVC | ~2 tools | ~1 GB | Yes |

These add ~12 tools but require GPU. Text-to-music is the headline feature —
generating audio from descriptions is genuinely transformative. RAVE is unique
because it runs on CPU in real-time.

---

## ~~The Full Picture: LivePilot Tool Count Projection~~ (SUPERSEDED)

> **This table is from the original research phase and is no longer accurate.**
> See the Decision Log above and the CLAUDE.md for the corrected projection.

| Version | Domain | Tools | Cumulative | Status |
|---------|--------|:---:|:---:|:---:|
| ~~1.5 (current)~~ 1.6.3 | Core + Automation | — | **135** | Shipped |
| 1.7.0 | Perception (FluCoMa + librosa + Demucs) | +16 | 151 | Spec complete |
| 1.8.0 | Theory + Composition + Tagging | +12 | 163 | Spec complete |
| 1.9 | Processing + Discovery | ~12 | ~175 | Research phase |
| 2.0 | Generation | ~8 | ~183 | Research phase |

From 135 → ~183 tools. hear it (1.7), understand it (1.8), process it (1.9),
generate it (2.0). Education is skill instructions, not tools.

---

## What's Actually Impossible Without These

Let me be specific about what these tools enable that you **literally cannot do**
in Ableton Live, Max4Live, or any combination of native features:

**music21:** Generate species counterpoint. Produce Roman numeral analysis. Detect
voice leading errors in a four-part arrangement. Create Bach-style harmonization
of a melody. None of this exists in Live or any plugin.

**isobar:** Generate Euclidean rhythms with probability weighting. Create Markov
chain compositions from a corpus. Generate L-system melodies. Live has Follow
Actions but they're manual and limited.

**Pedalboard:** Host VST3 plugins headlessly in Python. Render audio through a
plugin without opening the plugin GUI. Process 100 files through the same effect
chain in seconds. Impossible in Live without manually rendering each one.

**MusicGen/Stable Audio:** Generate audio from a text description. Create a
"chill lo-fi beat, 85 BPM, vinyl crackle, Rhodes piano" from nothing. No
equivalent exists in any DAW.

**RAVE:** Morph between two sounds continuously. Transfer the timbre of a violin
to a synth pad. Explore a latent space of sounds. Nothing in Live does neural
audio synthesis.

**Spotify API:** Look up the exact BPM, key, and mood metrics of any published
track in seconds. Live has no internet connectivity or metadata lookup.

**Matchering:** Match your mix's frequency balance, stereo width, and loudness to
a reference track algorithmically. Live has no automated reference matching.

**Freesound:** Search 600K+ samples by description and download directly into your
project folder. Live's browser only searches local files.

---

## The Honest Assessment: What's Worth It

Not everything above should be built. Here's the filter:

**Build (clear value, proven libraries, I can use the output well):**
- music21 theory tools — 90% confidence, massive unique value
- Education/explanation tools — 90% confidence, zero extra dependencies
- Auto-tagging (mutagen) — 95% confidence, pure productivity win
- Spotify reference lookup — 95% confidence, instant value
- isobar algorithmic generation — 85% confidence, creative inspiration
- Pedalboard audio processing — 85% confidence, genuine workflow saver
- Freesound search — 80% confidence, sample discovery
- Matchering — 75% confidence, reference mastering

**Build cautiously (high ceiling, unpredictable results):**
- MusicGen/Stable Audio — output quality varies, but RTX 5090 handles it easily
- RAVE sound morphing — creative but results are hard to predict
- Groove2Groove style transfer — works well on simple material, falls apart on complex

**Skip or defer (complexity not worth it yet):**
- RVC voice cloning — ethical complexity, niche use case for production
- .als file generation — format is fragile, breaks easily with naive XML editing
- Lyrics generation — no good open-source model, Genius API is read-only
- Plugin preset analysis — Serum format is reverse-engineered, fragile

---

## Decision Log (Updated March 2026)

This research document was written before implementation planning. The final
specs (`livepilot-1.7-spec.md`, `livepilot-1.8-spec.md`) supersede the
roadmap below. Here's what changed and why:

### Items kept for 1.8 (see livepilot-1.8-spec.md)
- **music21** (7 tools) — core value, works on live clips via get_notes
- **isobar** (3 tools, trimmed from 4) — Euclidean rhythm overlaps with automation curves
- **mutagen** (2 tools, trimmed from 3) — metadata report too trivial for a tool

### Items converted from tools to skill instructions
- **Education tools** (6 tools → 0 tools) — `explain_chord`, `quiz_intervals`,
  `explain_mix_decision`, `show_scale_on_keyboard`, `explain_song_form`,
  `suggest_exercises`. All pure LLM reasoning. The LLM already knows music theory
  from training data — wrapping explanations in MCP tools adds latency for zero
  capability gain. Education behavior is encoded as trigger-response patterns in
  the skill definition instead.

### Items deferred to 1.9
- **Spotify API** — OAuth complexity, requires external developer account setup
- **Freesound API** — Same OAuth problem, plus CC licensing tracking
- **matchering** — One-trick library, perception layer can guide manual fixes
- **pedalboard** — headless audio processing, still valuable, moved to 1.9

### Items cut permanently
- **pedalboard-pluginary** — Poorly maintained (last release 2023), platform-specific
  plugin scanning is fragile across macOS sandbox and Windows VST paths
- **RVC voice cloning** — Ethical minefield, niche use case, massive model size
- **Groove2Groove** — Research code, unmaintained since 2022, deferred indefinitely
- **MidiDrumiGen** — Requires FastAPI sidecar, too fragile for production
- **generate_lead_sheet** — Requires PDF generation (abjad/lilypond), heavy dependency
  for niche feature, deferred

### Design principle established
**Tools compute from data. The LLM interprets and explains.** Never duplicate
training knowledge in tool returns. Tools return precise data the LLM can't
compute (Roman numerals from 47 MIDI notes); the LLM explains what it means
musically. See the full design philosophy in `livepilot-1.8-spec.md`.

---

## Original Roadmap (for reference — superseded by decision log above)

These tools slot into the existing three-phase approach:

**1.7 (Perception Layer)** — 16 tools. See `livepilot-1.7-spec.md`.

**1.8 (Theory + Productivity)** — 12 tools. See `livepilot-1.8-spec.md`.

**1.9 (Processing + Discovery):**
- Pedalboard (headless audio processing)
- Spotify API (reference track lookup — requires API key)
- Freesound API (sample search — requires API key)
- Matchering (reference mastering)

**2.0 (Generation):**
- MusicGen or Stable Audio Open (text-to-music)
- RAVE (sound morphing, timbre transfer)
- Whatever new models exist by then

---

## Key Dependencies Summary (Updated)

| Library | Version | Size | GPU? | Status |
|---------|---------|------|:---:|--------|
| music21 | 1.8 | ~100 MB | No | **Confirmed** |
| isobar | 1.8 | ~5 MB | No | **Confirmed** |
| mutagen | 1.8 | ~5 MB | No | **Confirmed** |
| pedalboard | 1.9 | ~20 MB | No | Deferred |
| spotipy | 1.9 | ~2 MB | No | Deferred (OAuth) |
| freesound-python | 1.9 | ~1 MB | No | Deferred (OAuth) |
| matchering | 1.9 | ~10 MB | No | Deferred |
| MusicGen | 2.0 | ~4 GB | Yes | Deferred |
| RAVE | 2.0 | ~500 MB | Optional | Deferred |
| ~~pedalboard-pluginary~~ | — | — | — | **Cut** |
| ~~Groove2Groove~~ | — | — | — | **Cut** |
| ~~MidiDrumiGen~~ | — | — | — | **Cut** |
| ~~RVC~~ | — | — | — | **Cut** |

---

## The Bottom Line

The perception layer (1.7) gives LivePilot ears. music21 (1.8) gives it
**computational precision on theory** — not a music theory degree (the LLM
already has that from training), but the ability to compute from actual
note data. Combined, the pipeline becomes:

1. Transcribe audio to MIDI (basic-pitch, 1.7)
2. Analyze the harmony from real notes (music21, 1.8)
3. Detect voice leading issues with precision (music21, 1.8)
4. Suggest reharmonizations using theory rules (music21, 1.8)
5. Generate countermelody algorithmically (music21 + isobar, 1.8)
6. LLM explains every step in natural language (skill instructions, free)

The tools give data. The LLM gives meaning. Neither alone is enough.
Together, LivePilot understands music at both the signal level and the
theory level simultaneously — and can explain what it finds.
