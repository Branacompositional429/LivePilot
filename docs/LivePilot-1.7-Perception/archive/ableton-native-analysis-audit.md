# What Ableton Live Already Gives You (Before Writing a Line of Python)

## The Question

Before building 30 new Python-based MCP tools, what can we build using features
Ableton Live 12 already has — its built-in devices, Max/MSP objects, FluCoMa package,
and the Live Object Model API that LivePilot already talks to?

---

## Layer 1: Ableton Live 12 Built-In Features

### Audio-to-MIDI Conversion (BUILT-IN)

Live has three Convert commands in the Create menu, available on any audio clip:

**Convert Melody** — Monophonic pitch detection → MIDI notes on a new track.
Works well on isolated vocals, solo instruments, whistling. Uses transient markers
to determine note boundaries.

**Convert Harmony** — Polyphonic chord detection → MIDI chords on a new track.
Works on piano, guitar, pads. Accuracy drops with complex voicings.

**Convert Drums** — Onset detection → Kick/snare/hihat pattern on a new Drum Rack.
Best on isolated drum breaks. Falls apart with other instruments present.

**Can LivePilot trigger this via LOM?** NO — and this is the critical finding.
The Convert commands are UI-only actions. There is no `clip.convert_to_midi()` or
equivalent in the Live Object Model. You can't trigger them programmatically from
M4L or a Remote Script. They require user right-click → menu selection.

**What LivePilot CAN do:** After the user manually converts, LivePilot can read the
resulting MIDI clip via LOM — access every note, pitch, velocity, duration, position.
The analysis of the MIDI data is trivial; it's the conversion step that's manual.

**Verdict vs basic-pitch:**
- Live's Convert Melody: Monophonic only, no pitch bends, can't be automated
- Live's Convert Harmony: Polyphonic but error-prone on complex chords, can't be automated
- basic-pitch: Polyphonic, detects pitch bends, runs headless from MCP tool
- **Winner: basic-pitch** — unless you're OK with the manual conversion step, in which
  case Live's conversion + LivePilot's MIDI reading is a zero-dependency alternative
  for simple cases.

---

### Transient Detection (BUILT-IN)

Live automatically detects transients when loading audio clips. These appear as small
gray markers in the Sample Editor. LivePilot already has access to warp markers via LOM.

**What this gives you for free:**
- Onset/beat positions without any Python library
- Transient markers that drive Live's own warping engine
- Can be read and manipulated via LivePilot's existing warp tools

**What it doesn't give you:** Onset *strength*, onset *type* classification (percussive
vs tonal), or tempo stability analysis. For those, you need librosa or FluCoMa.

---

### Metering (BUILT-IN)

Live's track meters show peak and RMS in dB. That's it. No LUFS. No true peak.
No loudness range. No integrated loudness over time.

**Can LivePilot read meter values via LOM?** YES — `track.output_meter_left` and
`track.output_meter_right` give real-time peak levels. LivePilot's existing analyzer
already captures this (RMS/Peak metering is one of the 20 current tools).

**What's missing:** Everything that matters for professional loudness work — LUFS
(ITU-R BS.1770), LRA, true peak (dBTP). Live simply doesn't compute these.
You still need pyloudnorm for any serious loudness analysis.

---

### Spectrum Device (BUILT-IN)

Live's Spectrum audio effect shows a real-time FFT display. But it's a visual tool —
there's no LOM property that exposes the FFT data. You can see it, but code can't read it.

LivePilot already works around this with its own M4L-based 8-band spectrum analyzer.
The question is whether to upgrade that.

---

### Key/Scale Detection (BUILT-IN — SORT OF)

Live 12 has a Scale Mode in the transport bar (`song.root_note`, `song.scale_name` in
LOM). But this is a *user-set* scale, not audio-detected. Live does NOT automatically
detect the key of audio clips from their content.

LivePilot's existing Krumhansl-Schmuckler key detection in the M4L analyzer already
surpasses what Live offers natively. No change needed here.

---

### Tuner Device (BUILT-IN)

Live's Tuner audio effect shows the detected pitch of a monophonic signal. But like
Spectrum, it's visual-only — no LOM access to the detected pitch value.

However, you CAN build the same thing in M4L using Max's `pitch~` object and expose
the data. Which brings us to...

---

## Layer 2: Max/MSP Built-In Objects (Available in M4L)

These ship with Max and work inside any M4L device. No external packages needed.

| Object | What It Does | Quality |
|--------|-------------|---------|
| `fiddle~` | Polyphonic pitch detection (up to ~4 voices) | Decent but pre-deep-learning era. Misses complex polyphony. |
| `bonk~` | Onset/transient detection with attack classification | Good for percussion. The original onset detector. |
| `pitch~` | Monophonic pitch tracking | Reliable for single voices/instruments. |
| `loudness~` | Loudness estimation (Stevens/Zwicker model) | NOT LUFS. Perceptual loudness in sone, not broadcast-standard. |
| `sigmund~` | Sinusoidal decomposition + pitch tracking | More accurate than fiddle~ for monophonic material. |
| `analyzer~` | Basic spectral data + pitch | Jack of all trades, master of none. |
| `centroid~` | Spectral centroid | Single number, works fine. |
| `brightness~` | Spectral brightness metric | Similar to centroid but with configurable cutoff. |

**What you could build with these (zero new dependencies):**

A beefed-up M4L analyzer that adds spectral centroid, brightness, onset detection,
and basic pitch tracking to the existing 8-band spectrum + RMS/Peak. This would give
LivePilot ~5-7 more real-time descriptors without touching Python.

**What you CAN'T build:** Anything requiring deep learning, long-window analysis,
or offline computation. These objects are all real-time, single-frame, CPU-constrained.

---

## Layer 3: FluCoMa Package (Free, Installable in Max)

This is where it gets interesting. FluCoMa (Fluid Corpus Manipulation) is a free,
open-source package from the University of Huddersfield that runs inside Max/M4L.
It's essentially a mini-librosa for real-time use.

### What FluCoMa gives you inside M4L:

**Spectral Descriptors (real-time):**
- `fluid.spectralshape~` — centroid, spread, skewness, kurtosis, rolloff, flatness, crest
  (That's 7 descriptors in one object — covers most of what librosa's spectral analysis does)
- `fluid.melbands~` — 40 mel-frequency bands (vs librosa's 128, but 40 is plenty for
  real-time production monitoring)
- `fluid.mfcc~` — 13 MFCCs in real-time (timbral fingerprint)
- `fluid.chroma~` — 12 pitch classes (harmonic content)

**Loudness:**
- `fluid.loudness~` — EBU R128 momentary loudness + true peak
  BUT: Only momentary. Not integrated LUFS. Not LRA. Not the full BS.1770 suite.

**Pitch & Onsets:**
- `fluid.pitch~` — YIN algorithm, monophonic, better than fiddle~
- `fluid.onsetslice~` / `fluid.onsetfeature~` — onset detection with feature extraction
- `fluid.transientslice~` — transient-specific segmentation

**Source Separation (real-time, limited):**
- `fluid.hpss~` — Harmonic/Percussive Source Separation
  Splits audio into harmonic (tonal) and percussive (transient) components in real-time.
  NOT the same as Demucs — this is 2-way (harmonic/percussive), not 4-way (vocals/drums/
  bass/other). But it's something.
- `fluid.sines~` — Sinusoidal modeling (sines vs noise decomposition)
- `fluid.transients~` — Isolate transient layer

**Novelty / Structure (partial):**
- `fluid.noveltyslice~` — Detects local novelty (sudden changes in spectral content)
  This is a primitive form of structure detection — it finds boundaries where the
  sound character changes. It won't label sections as "verse" or "chorus" but it
  will find where transitions happen.

**Machine Learning (inside Max!):**
- `fluid.mlpclassifier~` — Neural network classifier
- `fluid.kdtree~` — Nearest-neighbor search for similarity
- `fluid.umap~` — Dimensionality reduction

**Buffer Analysis (offline-in-Max):**
- `fluid.bufstats~` — Statistical summaries of any descriptor over time
- `fluid.bufnmf~` — Non-negative Matrix Factorization on buffers

### Also: zsa.descriptors Package (free Max external)

- `zsa.centroid~`, `zsa.flatness~`, `zsa.flux~`, `zsa.rolloff~`
- `zsa.bark~`, `zsa.mel~` — Bark/Mel band energies
- `zsa.roughness~` — Perceptual roughness (psychoacoustic metric, inside Max!)
- `zsa.kurtosis~`, `zsa.skewness~` — Spectral shape

The roughness metric from zsa is notable — it partially covers MOSQITO's territory
without leaving Max.

---

## The Overlap Map: What Ableton/M4L Covers vs. What Needs Python

| Proposed Tool | Can M4L/FluCoMa Do It? | Quality vs Python | Verdict |
|--------------|:---:|---|---|
| **Loudness (LUFS)** | Partial — `fluid.loudness~` gives momentary only | 30% of pyloudnorm | **Still need Python** — no integrated LUFS, no LRA |
| **Spectral analysis** | YES — `fluid.spectralshape~` + `fluid.melbands~` + `fluid.mfcc~` | 70% of librosa for real-time | **M4L can cover this** — fewer mel bands but good enough for monitoring |
| **Harmonic/Chroma** | YES — `fluid.chroma~` | 60% of librosa chroma | **M4L is adequate** for real-time chord tracking |
| **Audio-to-MIDI** | Live's Convert (manual) + `fiddle~` (monophonic) | 30% of basic-pitch | **Still need Python** — no polyphonic, no pitch bends, no automation |
| **Source separation** | Partial — `fluid.hpss~` (harmonic/percussive only) | 15% of Demucs | **Still need Python** — 2-way vs 4-way, no vocal isolation |
| **Structure analysis** | Partial — `fluid.noveltyslice~` (boundaries only) | 20% of MSAF | **Still need Python** — no section labeling, no form detection |
| **Onset detection** | YES — `bonk~` + `fluid.onsetslice~` | 80% of librosa onsets | **M4L is fine** for this |
| **Pitch tracking** | YES — `fluid.pitch~` (monophonic YIN) | 50% of CREPE | **M4L is adequate** for monophonic; Python only if polyphonic needed |
| **Psychoacoustics** | Partial — `zsa.roughness~` | 25% of MOSQITO | **Supporting signal only** — one metric vs full Zwicker model |
| **Timbral descriptors** | YES — FluCoMa spectral shape covers most | 60% of Essentia | **M4L is fine** — reinforces the "skip Essentia" decision |
| **Fingerprinting** | NO — nothing in Max does perceptual hashing | 0% | **Need Python if wanted** |
| **Reference comparison** | NO — requires loading external file + full analysis | 0% | **Need Python** |

---

## The Revised Strategy: Three Layers Instead of Two

Instead of the original plan (M4L real-time + Python offline), there's a smarter
three-layer approach:

### Layer A: Enhanced M4L Analyzer (FluCoMa + zsa — zero Python)

Upgrade the existing LivePilot M4L device to include FluCoMa and zsa objects.
This gives you real-time monitoring that's dramatically better than the current
8-band spectrum, at zero Python cost:

**New real-time tools from M4L alone:**
1. `get_spectral_shape` — centroid, spread, rolloff, flatness, crest (from fluid.spectralshape~)
2. `get_timbral_profile` — 13 MFCCs (from fluid.mfcc~)
3. `get_mel_spectrum` — 40 mel bands (from fluid.melbands~)
4. `get_chroma` — 12 pitch classes for chord tracking (from fluid.chroma~)
5. `get_onsets` — transient detection with features (from fluid.onsetslice~)
6. `get_harmonic_percussive` — H/P separation ratio (from fluid.hpss~)
7. `get_roughness` — perceptual roughness (from zsa.roughness~)
8. `get_novelty` — spectral change detection (from fluid.noveltyslice~)
9. `get_momentary_loudness` — EBU R128 momentary (from fluid.loudness~)

That's **9 new real-time MCP tools** with no Python, no pip install, no GPU.
Combined with the existing 20 tools, LivePilot goes from 20 to 29 perception
tools just by upgrading the M4L device.

**Installation:** User installs FluCoMa package in Max's Package Manager (one click)
and zsa.descriptors (one download). That's it.

### Layer B: Lightweight Python Analysis (no GPU, no torch)

For things M4L genuinely can't do but that don't need deep learning:

**Tools that only need lightweight Python libs:**
1. `analyze_loudness` — Full LUFS suite (pyloudnorm — 50KB library, pure Python)
2. `analyze_dynamic_range` — LRA, crest factor (pyloudnorm + scipy)
3. `compare_loudness` — A/B bounces (pyloudnorm)
4. `analyze_structure` — Section boundaries + labels (msaf — uses librosa internally)
5. `analyze_spectral_evolution` — How spectral character changes over full track (librosa)
6. `compare_to_reference` — A/B against reference track (librosa + pyloudnorm)

**Dependencies:** librosa, pyloudnorm, scipy, soundfile, msaf
**Total size:** ~100 MB. No GPU. No torch. No tensorflow. Installs in 30 seconds.

These 6 tools fill the biggest gaps that M4L can't cover: integrated loudness, LRA,
full-track spectral evolution, and structure detection with labeling.

### Layer C: Heavy Python Analysis (GPU, torch — only if you want it)

The neural network tools that are impossible any other way:

1. `separate_stems` — 4-way source separation (Demucs — requires PyTorch)
2. `transcribe_to_midi` — Polyphonic audio → MIDI with pitch bends (basic-pitch)
3. `diagnose_mix` — Compound pipeline: separate → analyze → diagnose

**Dependencies:** Everything from Layer B + torch, torchaudio, demucs, basic-pitch
**Total size:** ~3 GB (dominated by PyTorch)

This is the "heavy artillery" that justifies the big install.

---

## What This Means: The Reduced Build

**Original plan:** 30-35 new Python tools, all requiring pip install, some needing GPU.

**Revised plan:**

| Layer | Tools | Dependencies | Install Effort |
|-------|:---:|---|---|
| A: Enhanced M4L | 9 new real-time | FluCoMa + zsa (Max packages) | One-click in Max Package Manager |
| B: Light Python | 6 offline tools | librosa, pyloudnorm, msaf (~100 MB) | `livepilot setup --light` |
| C: Heavy Python | 3 offline tools | + torch, demucs, basic-pitch (~3 GB) | `livepilot setup --all` |
| **Total** | **18 new tools** | | |

That's 18 tools instead of 35, covering 90% of the value at half the complexity.
The ones we dropped are the ones the honest assessment rated lowest anyway
(Essentia, CREPE, MOSQITO full suite, most fingerprinting).

---

## What Got Eliminated (And Why That's Fine)

**Dropped from Python → covered by M4L:**
- Real-time spectral analysis → `fluid.spectralshape~` (better than running Python for live monitoring)
- Real-time MFCCs/timbral → `fluid.mfcc~` (no need for Essentia)
- Onset detection → `fluid.onsetslice~` + `bonk~` (mature, battle-tested)
- Basic chroma/chord tracking → `fluid.chroma~` (good enough for real-time)
- Roughness/psychoacoustic → `zsa.roughness~` (one metric beats none)

**Dropped entirely:**
- Essentia — librosa + FluCoMa cover the same ground
- CREPE — basic-pitch handles transcription; `fluid.pitch~` handles monophonic tracking
- MOSQITO full suite — `zsa.roughness~` gives the most useful psychoacoustic metric
- Chromaprint/fingerprinting — niche, can add later if needed

**Kept in Python (can't be replaced):**
- Integrated LUFS / LRA / true peak (pyloudnorm) — M4L only does momentary
- Structure detection with labeling (MSAF) — M4L's noveltyslice~ finds boundaries but can't label them
- Full-track spectral evolution (librosa) — needs offline multi-pass analysis
- Source separation into 4 stems (Demucs) — nothing in Max comes close
- Polyphonic audio-to-MIDI with pitch bends (basic-pitch) — Live's converter is manual and limited
- Reference track comparison — requires loading external audio, impossible in M4L

---

## The Build Order (Final, Revised)

**Phase 1 — M4L Enhancement (no Python at all):**
1. Install FluCoMa + zsa.descriptors in Max
2. Add fluid.spectralshape~ to the LivePilot analyzer → `get_spectral_shape` tool
3. Add fluid.mfcc~ → `get_timbral_profile` tool
4. Add fluid.melbands~ → `get_mel_spectrum` tool
5. Add fluid.chroma~ → `get_chroma` tool
6. Add fluid.onsetslice~ → `get_onsets` tool
7. Add fluid.hpss~ → `get_harmonic_percussive` tool
8. Add zsa.roughness~ → `get_roughness` tool
9. Add fluid.noveltyslice~ → `get_novelty` tool
10. Add fluid.loudness~ → `get_momentary_loudness` tool

**Result:** 29 perception tools (20 existing + 9 new). Zero Python. Test this layer
thoroughly before moving on. This alone is a massive upgrade.

**Phase 2 — Lightweight Python (no GPU):**
11. Add pyloudnorm → `analyze_loudness`, `analyze_dynamic_range`, `compare_loudness`
12. Add librosa → `analyze_spectral_evolution`, `compare_to_reference`
13. Add msaf → `analyze_structure`

**Result:** 35 tools (29 M4L + 6 Python). ~100 MB install. Covers loudness compliance,
structure detection, reference comparison.

**Phase 3 — Heavy Python (GPU optional):**
14. Add demucs → `separate_stems`
15. Add basic-pitch → `transcribe_to_midi`
16. Build compound → `diagnose_mix`

**Result:** 38 tools total. Full perception layer. ~3 GB install.

---

## The Honest Bottom Line

About 40% of what we planned to build in Python can be done inside M4L with FluCoMa
and zsa, with zero additional Python dependencies. These are the real-time monitoring
tools — spectral shape, timbral profile, onset detection, chroma, roughness.

The remaining 60% genuinely needs Python because it requires either:
- Standards compliance (LUFS/LRA — specific ITU-R algorithms)
- Deep neural networks (Demucs, basic-pitch)
- Full-track offline analysis (structure, spectral evolution over minutes)
- External file loading (reference comparison)

The three-layer approach means you can ship Phase 1 fast, get real value, and add
Python layers only when you actually need what they provide. No big-bang install.
No "everything or nothing." Each layer works independently and adds value on its own.
