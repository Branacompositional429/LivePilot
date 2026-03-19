# LivePilot Perception Layer: Deep Research & Arsenal Expansion

## What We Have Now

The LivePilot Analyzer is a Max for Live device sitting on the master bus. It gives the AI **20 tools** providing:

- **8-band spectrum** (real-time FFT split into octave bands)
- **RMS / Peak metering** (overall loudness snapshot)
- **Key detection** via Krumhansl-Schmuckler algorithm
- **Sample manipulation** (Simpler slicing, cropping, reversing, warping)
- **Warp markers** (add, move, remove)
- **Device introspection** (hidden parameters, automation state, device tree walking)

This is a **real-time, lightweight, master-bus-only** perception layer. It answers: "what is happening right now on the output?" But it can't answer deeper questions like: "why does this mix sound muddy?", "what's the harmonic progression over the last 8 bars?", "where does the verse end and the chorus begin?", or "how does the kick's transient interact with the bass's fundamental?"

---

## The Bounce-to-Disk Paradigm: Why It Changes Everything

Real-time M4L analysis is constrained by:
1. **CPU budget** — must complete within the audio buffer callback
2. **Single point of observation** — master bus only, post-mix
3. **No look-ahead** — can only see the current frame
4. **Limited algorithm complexity** — no deep learning inference, no multi-pass analysis

**Bouncing to disk** removes all four constraints. Once audio is a WAV file on disk, the MCP server (Python side) can run arbitrarily expensive analysis — multi-pass spectral decomposition, neural network inference, structural segmentation — and return results as rich JSON that the AI can reason over.

The workflow becomes:
```
Ableton (bounce track/stem) → WAV on disk → Python analysis pipeline → Rich JSON → AI reasoning
```

This is not "non-real-time" as a limitation — it's **non-real-time as a superpower**. The analysis only needs to happen once per bounce, and the insights it produces are permanent, searchable, and composable.

---

## The Arsenal: What We Can Add

### Tier 1 — Foundational (High Impact, Proven Libraries)

#### 1. Loudness Intelligence (pyloudnorm + custom)
**What it reveals:** Professional loudness compliance, dynamic range health, compression artifacts

| Metric | What It Tells The AI |
|--------|---------------------|
| Integrated LUFS | Overall loudness — is this -14 LUFS for Spotify or -8 for club? |
| Short-term LUFS (3s windows) | Loudness contour over time — where are the energy dips/spikes? |
| Momentary LUFS (400ms) | Micro-dynamics — are transients breathing or crushed? |
| Loudness Range (LRA) | Dynamic variety — is the track fatiguing or alive? |
| True Peak (dBTP) | Inter-sample peaks — will this clip on conversion? |
| Peak-to-Loudness Ratio (PLR) | Headroom health — how much transient clarity remains? |
| Crest Factor per band | Which frequency ranges are over-compressed? |

**Implementation:** `pyloudnorm` for ITU-R BS.1770-4 compliant LUFS. Custom windowed analysis for LRA curves. ~50 lines of Python wrapping the library.

**MCP Tools:**
- `analyze_loudness` — full LUFS report for a bounced file
- `analyze_dynamic_range` — LRA, crest factor, compression detection
- `compare_loudness` — A/B two bounces (before/after mastering chain)

---

#### 2. Deep Spectral Analysis (librosa)
**What it reveals:** Timbral DNA, harmonic content, spectral evolution over time

| Feature | Dimension | What It Tells The AI |
|---------|-----------|---------------------|
| Mel Spectrogram | 128 bands × frames | Full spectral texture map (way beyond 8 bands) |
| MFCCs | 20 coefficients | Timbral fingerprint — what does this *sound like*? |
| Spectral Centroid | 1D curve | Brightness trajectory — is the mix getting duller over time? |
| Spectral Bandwidth | 1D curve | Spectral spread — narrow (focused) vs wide (washy) |
| Spectral Contrast | 7 sub-bands | Peak-valley energy ratio — how "defined" is each frequency range? |
| Spectral Rolloff | 1D curve | Where 85% of energy lives — detecting low-pass buildup or harsh high-end |
| Spectral Flatness | 1D curve | Noise vs tone ratio — detecting excessive noise, distortion, or artifacts |
| Chroma (STFT + CQT) | 12 pitch classes | Harmonic content collapsed to pitch — chord detection, key tracking |
| Tonnetz | 6D tonal space | Harmonic relationships (fifths, thirds) — detecting tonal tension/resolution |
| Tempogram | BPM × time | Rhythmic stability map — tempo drift, polyrhythmic sections |
| Onset Strength | 1D curve | Transient density — where the rhythmic energy lives |
| Zero Crossing Rate | 1D curve | Percussive vs tonal content ratio |

**Implementation:** librosa does all of this. The key insight is **not to dump raw features** but to build **interpretive summaries**: "The spectral centroid rises 400Hz between bars 16-24, suggesting the filter sweep is opening the mix for the chorus" — that's what the AI should receive.

**MCP Tools:**
- `analyze_timbre` — MFCC profile + spectral shape summary
- `analyze_harmonic_content` — chroma, tonnetz, chord progression estimation
- `analyze_spectral_evolution` — how the spectral character changes over time
- `analyze_rhythm_stability` — tempogram + onset analysis

---

#### 3. Source Separation (Demucs / HTDemucs)
**What it reveals:** Individual stem quality in a mixed-down bounce — the holy grail for mix diagnostics

This is the most transformative addition. Instead of analyzing only the master bus mix, we can **decompose any bounce into stems** and analyze each one independently:

- **Vocals** — is the vocal sitting right? Frequency masking with other elements?
- **Drums** — transient integrity, kick/snare balance, room sound
- **Bass** — low-end clarity, fundamental vs harmonics ratio
- **Other** (keys, synths, guitars) — mid-range congestion diagnosis

HTDemucs v4 achieves 9.2 dB SDR on MUSDB18-HQ. With GPU (CUDA), processing takes <10 seconds. Even on CPU, a 3-minute track processes in ~2-3 minutes — totally acceptable for a bounce-and-analyze workflow.

**The compound insight:** Run Demucs separation → then run librosa/essentia analysis on each stem → the AI gets a per-instrument spectral report. It can say: "Your bass fundamental at 60Hz is being masked by the kick's sub. The bass's first harmonic at 120Hz is clear, but the guitar's low-mid energy at 200-400Hz is creating mud. Suggestion: high-pass the guitar at 180Hz and cut 2dB at 250Hz on the bass."

**MCP Tools:**
- `separate_stems` — run Demucs on a bounce, return paths to stem files
- `analyze_stem` — deep analysis of an individual separated stem
- `diagnose_mix` — full pipeline: separate → analyze each → identify masking/conflicts

---

#### 4. Music Structure Analysis (MSAF + All-In-One)
**What it reveals:** Song form, section boundaries, repetition patterns

| Analysis | Output |
|----------|--------|
| Boundary detection | Timestamps where sections change |
| Segment labeling | intro / verse / chorus / bridge / outro / inst |
| Repetition structure | Which sections repeat and how they vary |
| Self-similarity matrix | Visual map of structural relationships |

**Why this matters for production:** The AI currently has no concept of song structure from audio. It can read arrangement clips and their names, but if the user hasn't labeled them, or if analyzing a reference track, structure analysis closes that gap. The AI can say: "Your chorus hits at 0:48 but the energy only peaks at 0:52 — consider front-loading the chorus entry with a cymbal crash or filter sweep 2 beats earlier."

**MCP Tools:**
- `analyze_structure` — section boundaries + labels
- `compare_structure` — compare your track's form to a reference track

---

#### 5. Audio-to-MIDI Transcription (Spotify basic-pitch)
**What it reveals:** Polyphonic note content from audio, with pitch bend detection

This closes a massive gap: the AI can currently read MIDI clips it created, but it **cannot hear what notes are actually playing in audio**. With basic-pitch:

- Transcribe a vocal melody to MIDI for harmonization
- Extract the chord progression from a reference track
- Reverse-engineer a synth line from a bounced stem
- Detect pitch drift or detuning issues

basic-pitch is lightweight (<20MB, <17K parameters), instrument-agnostic, and handles polyphony. It also detects pitch bends (vibrato, glissando), which most transcription tools lose.

**MCP Tools:**
- `transcribe_to_midi` — audio → MIDI file with pitch bends
- `extract_melody` — predominant melody line from a mix
- `detect_pitch_issues` — flag notes that are out of tune

---

### Tier 2 — Advanced (Deep Insight, More Complex)

#### 6. Psychoacoustic Analysis (MOSQITO + custom)
**What it reveals:** How the sound *feels* to a human listener, beyond what frequencies are present

| Metric | Unit | What It Tells The AI |
|--------|------|---------------------|
| Perceived Loudness | sone | Psychoacoustic loudness (not just dB) — accounts for frequency weighting |
| Sharpness | acum | High-frequency harshness perception — "this mix is fatiguing" |
| Roughness | asper | Amplitude modulation dissonance — beating between close frequencies |
| Sensory Consonance | — | Combined tonalness + roughness — harmonic pleasantness |
| Fluctuation Strength | vacil | Slow modulation perception (tremolo, slow LFO artifacts) |

**Why this matters:** Two mixes can have identical spectra but feel completely different. Psychoacoustic metrics capture what a trained ear hears but can't articulate: "this sounds harsh", "this is fatiguing", "this feels smooth". The AI gets vocabulary for subjective qualities.

**MCP Tools:**
- `analyze_psychoacoustics` — full perceptual quality report
- `detect_harshness` — flag specific time ranges where sharpness/roughness spike

---

#### 7. Timbral & High-Level Descriptors (Essentia)
**What it reveals:** Music-specific semantic descriptors that bridge DSP and human language

Essentia goes beyond librosa by providing **music-production-relevant descriptors:**

- **Dissonance** — harmonic clash quantification
- **Inharmonicity** — how "metallic" or "bell-like" a sound is
- **Spectral Complexity** — number of peaks in the spectrum (busy vs clean)
- **HFC (High Frequency Content)** — brightness energy metric
- **Pitch Salience** — how clearly pitched the overall signal is (vs noise/percussion)
- **HPCP (Harmonic Pitch Class Profile)** — more robust chroma than librosa for chord detection
- **Beat loudness** — loudness profile synchronized to detected beats
- **Rhythm descriptors** — onset rate, rhythm regularity, beat strength

**MCP Tools:**
- `analyze_timbral_quality` — inharmonicity, dissonance, complexity profile
- `analyze_rhythm_character` — beat strength, onset patterns, groove feel
- `get_high_level_descriptors` — semantic labels: "bright", "dark", "aggressive", "smooth"

---

#### 8. Advanced Pitch Analysis (CREPE + aubio)
**What it reveals:** Precise pitch tracking for vocals, leads, and bass — down to cent-level accuracy

CREPE (deep CNN) achieves >90% accuracy within 10 cents on polyphonic material. Combined with aubio's transient detection:

- **Vocal pitch accuracy** — is the singer hitting the notes? Where are they flat/sharp?
- **Bass intonation** — is the bass drifting from the root?
- **Pitch drift over time** — are oscillators detuning?
- **Vibrato analysis** — rate and depth characterization
- **Transient/steady-state separation** — isolate attacks from sustains for independent analysis

**MCP Tools:**
- `analyze_pitch_accuracy` — cent-level pitch deviation report
- `analyze_vibrato` — rate, depth, regularity characterization
- `detect_transients` — precise attack timestamps and strengths

---

### Tier 3 — Intelligence Layer (Compound Analysis)

#### 9. Mix Diagnostics Engine (compound tool)
**What it reveals:** Actionable mix problems with specific fix suggestions

This isn't a library — it's a **pipeline** that combines multiple analyses:

```
Bounce → Demucs separation → Per-stem analysis → Cross-stem comparison → Diagnostic report
```

Diagnostics it can produce:

| Problem | Detection Method | AI Can Suggest |
|---------|-----------------|----------------|
| Frequency masking | Spectral overlap between separated stems | EQ cuts, frequency slotting |
| Phase issues | Cross-correlation between stems | Polarity flip, phase alignment |
| Muddy low-mids | Energy accumulation at 200-500Hz across stems | Per-track HPF, mid-side EQ |
| Harsh high-end | Sharpness spikes in psychoacoustic analysis | De-essing, high-shelf reduction |
| Dynamic crush | Low crest factor + low LRA | Reduce compression ratio, increase attack |
| Stereo problems | Left-right correlation analysis | Mid-side processing, panning adjustments |
| Loudness imbalance | Per-section LUFS comparison | Automation rides, gain staging |
| Timing drift | Cross-correlation of transients between stems | Quantize, manual alignment |

**MCP Tools:**
- `diagnose_mix_problems` — full diagnostic report with ranked issues
- `suggest_mix_fixes` — specific parameter suggestions tied to detected problems

---

#### 10. Reference Track Comparison
**What it reveals:** How your mix compares to a professional reference on every measurable dimension

| Comparison | Method |
|-----------|--------|
| Loudness profile | LUFS curves overlaid |
| Spectral balance | Average spectrum comparison |
| Dynamic range | LRA / crest factor comparison |
| Stereo width | Correlation coefficient comparison |
| Structural pacing | Section energy curves compared |
| Timbral similarity | MFCC distance |
| Harmonic complexity | Chroma / tonnetz comparison |

**MCP Tools:**
- `compare_to_reference` — comprehensive A/B analysis against a reference track
- `match_loudness_profile` — suggest EQ/compression to approach reference's spectral balance

---

#### 11. Audio Fingerprinting & Similarity (Chromaprint)
**What it reveals:** Perceptual similarity between sounds, sections, or tracks

Beyond identifying tracks (AcoustID), the fingerprinting approach enables:

- **Finding similar sections** within a track (detect copy-paste arrangements)
- **Comparing iterations** of a mix (what actually changed between bounce v1 and v2?)
- **Building a sample similarity index** — "find sounds in my library that are perceptually close to this one"

**MCP Tools:**
- `fingerprint_audio` — generate perceptual fingerprint
- `compare_audio_similarity` — similarity score between two audio files
- `diff_bounces` — what changed between two versions of a mix?

---

## Implementation Architecture

```
┌─────────────────────────────────────────────────────┐
│                    MCP Server                        │
│                                                      │
│  ┌─────────────┐  ┌──────────────┐  ┌────────────┐ │
│  │ Real-Time   │  │ Bounce-to-   │  │ Technique  │ │
│  │ Perception  │  │ Disk Deep    │  │ Memory     │ │
│  │ (M4L)       │  │ Analysis     │  │            │ │
│  │             │  │ (Python)     │  │            │ │
│  │ • 8-band    │  │              │  │ • Store    │ │
│  │   spectrum  │  │ • Loudness   │  │   analysis │ │
│  │ • RMS/Peak  │  │ • Deep       │  │   results  │ │
│  │ • Key       │  │   spectral   │  │ • Compare  │ │
│  │             │  │ • Stems      │  │   across   │ │
│  │             │  │ • Structure  │  │   sessions │ │
│  │             │  │ • Transcribe │  │ • Learn    │ │
│  │             │  │ • Psycho-    │  │   taste    │ │
│  │             │  │   acoustic   │  │            │ │
│  │             │  │ • Timbral    │  │            │ │
│  │             │  │ • Pitch      │  │            │ │
│  │             │  │ • Mix diag   │  │            │ │
│  │             │  │ • Reference  │  │            │ │
│  └─────────────┘  └──────────────┘  └────────────┘ │
│        ▲                 ▲                 ▲        │
│        │                 │                 │        │
│   Real-time         File-based         Persistent   │
│   (master bus)      (any WAV)          (JSON DB)    │
└─────────────────────────────────────────────────────┘
```

### Dependency Stack

```
Core:       librosa, numpy, scipy, soundfile
Loudness:   pyloudnorm
Separation: demucs (torch)
Transcribe: basic-pitch (tensorflow-lite)
Structure:  msaf or all-in-one
Pitch:      crepe (tensorflow)
Psycho:     mosqito
Timbral:    essentia
Fingerprint: pyacoustid + chromaprint
```

### Suggested MCP Tool Grouping

| Domain | Tools | Requires |
|--------|-------|----------|
| `deep_spectrum` | analyze_timbre, analyze_harmonic_content, analyze_spectral_evolution | librosa |
| `loudness` | analyze_loudness, analyze_dynamic_range, compare_loudness | pyloudnorm |
| `stems` | separate_stems, analyze_stem, diagnose_mix | demucs + librosa |
| `structure` | analyze_structure, compare_structure | msaf |
| `transcribe` | transcribe_to_midi, extract_melody, detect_pitch_issues | basic-pitch |
| `perception` | analyze_psychoacoustics, detect_harshness | mosqito |
| `timbral` | analyze_timbral_quality, analyze_rhythm_character | essentia |
| `pitch` | analyze_pitch_accuracy, analyze_vibrato, detect_transients | crepe + aubio |
| `mix_intel` | diagnose_mix_problems, suggest_mix_fixes, compare_to_reference | compound |
| `fingerprint` | fingerprint_audio, compare_audio_similarity, diff_bounces | chromaprint |

This would add roughly **30-35 new MCP tools**, bringing the total from 127 to ~160, with the new tools representing a qualitative leap — from "what parameter is set" to "what does this actually sound like and how can it be better."

---

## Priority Recommendation

If I had to pick the order of implementation for maximum impact:

1. **Loudness Intelligence** — smallest lift, immediate professional value, every producer needs this
2. **Deep Spectral (librosa)** — the foundation everything else builds on
3. **Source Separation (Demucs)** — the game-changer, enables per-stem diagnostics
4. **Audio-to-MIDI (basic-pitch)** — closes the "AI can't hear notes" gap
5. **Music Structure (MSAF)** — enables arrangement-aware reasoning
6. **Mix Diagnostics Engine** — the compound tool that ties 1-3 together
7. **Psychoacoustic + Timbral** — the "taste" layer
8. **Reference Comparison** — the "how good is this" benchmark
9. **Pitch Analysis (CREPE)** — vocal/instrument accuracy
10. **Fingerprinting** — version comparison and similarity search

The first three alone would make LivePilot's perception layer arguably the deepest audio intelligence system in any MCP server.

---

## Part 5: Installation & Long-Term Maintenance Strategy

### Philosophy: One Command, Everything Works

Since this is a personal project — not a public release — the strategy is dead simple: extend the existing `livepilot.js` installer to handle everything in one shot. No Docker, no conda environments, no separate installers. Just `livepilot setup` and go.

### The Setup: What Actually Needs Installing

The full analysis stack breaks down into three weight classes:

**Lightweight (< 5 MB total, pip install in seconds):**
pyloudnorm, soundfile, scipy, numpy, mosqito, pyacoustid, aubio

**Medium (~ 50-100 MB, pip install in under a minute):**
librosa, essentia, msaf, basic-pitch (uses TensorFlow Lite — small)

**Heavyweight (1-3 GB, the PyTorch ecosystem):**
demucs (requires torch), crepe (requires tensorflow)

The heavyweights are what make installation non-trivial. PyTorch alone is ~2 GB and needs the right build for your GPU (CUDA on Windows, MPS on Mac).

### The Approach: Extended `livepilot.js` with `uv`

[uv](https://github.com/astral-sh/uv) is the fastest Python package manager available (written in Rust, 10-100x faster than pip). It handles virtual environments, dependency resolution, and — critically — **automatic PyTorch backend selection** via `--torch-backend=auto`, which detects whether to install CUDA wheels (Windows RTX 5090) or CPU/MPS wheels (Mac M3 Max).

**Step 1 — Install uv itself** (one-liner, no dependencies):
```bash
# Windows (PowerShell)
powershell -ExecutionPolicy ByPass -c "irm https://astral.sh/uv/install.ps1 | iex"

# macOS
curl -LsSf https://astral.sh/uv/install.sh | sh
```

**Step 2 — Replace `requirements.txt` with `pyproject.toml`:**
```toml
[project]
name = "livepilot"
version = "1.5.0"
requires-python = ">=3.10,<3.13"

dependencies = [
    "fastmcp>=3.0.0,<4.0.0",
]

[project.optional-dependencies]
analysis = [
    "librosa>=0.10.0,<0.11",
    "pyloudnorm>=0.1.1",
    "soundfile>=0.12",
    "scipy>=1.11",
    "mosqito>=1.2",
    "pyacoustid>=1.3",
    "aubio>=0.4.9",
]
separation = [
    "demucs>=4.0",
    "torch>=2.1",
    "torchaudio>=2.1",
]
transcription = [
    "basic-pitch>=0.3",
]
pitch = [
    "crepe>=0.0.16",
    "tensorflow>=2.14,<3.0",
]
all = [
    "livepilot[analysis,separation,transcription,pitch]",
]
```

**Step 3 — The actual install command in `livepilot.js`:**
```javascript
// Add to bin/livepilot.js
async function setupAnalysis() {
    console.log("Installing analysis dependencies...");

    // uv creates the venv and installs everything in one command
    // --torch-backend=auto picks CUDA on Windows, MPS on Mac
    await exec("uv pip install -e '.[all]' --torch-backend=auto", {
        cwd: ROOT,
        env: { ...process.env, UV_PROJECT_ENVIRONMENT: VENV_DIR }
    });

    console.log("Pre-downloading AI models...");
    // Pre-download so first analysis doesn't hang
    await exec(`${pythonBin} -c "
import demucs.pretrained; demucs.pretrained.get_model('htdemucs')
from basic_pitch.inference import predict  # triggers model download
print('Models ready.')
    "`);
}
```

**What the user runs:**
```bash
livepilot setup          # base MCP server (as today)
livepilot setup --all    # base + full analysis stack
livepilot setup --light  # base + analysis (no GPU models)
```

That's it. One command. uv handles the rest.

### Cross-Platform Reality Check

| Library | Windows (RTX 5090) | macOS (M3 Max) | Notes |
|---------|-------------------|----------------|-------|
| librosa | ✅ native | ✅ native | Pure Python + numpy |
| pyloudnorm | ✅ native | ✅ native | Pure Python |
| demucs | ✅ CUDA | ✅ MPS | `--torch-backend=auto` handles this |
| basic-pitch | ✅ native | ✅ native | TF Lite, CPU only, fast enough |
| crepe | ✅ CUDA (TF) | ⚠️ CPU only | TF GPU on Mac is dead; CPU is fine for offline |
| mosqito | ✅ native | ✅ native | Pure Python + scipy |
| essentia | ⚠️ needs WSL | ✅ native | Windows: use `essentia` via pip, may need VS Build Tools |
| msaf | ✅ native | ✅ native | Python + librosa |
| aubio | ✅ native | ✅ native | C library with Python bindings |
| chromaprint | ✅ native | ✅ native | Needs `fpcalc` binary installed separately |

**Essentia on Windows note:** As of early 2026, `pip install essentia` works on Windows with VS Build Tools installed. If that's too annoying, skip Essentia — librosa covers 90% of what Essentia does for this use case (timbral descriptors, rhythm analysis). Essentia's unique value is its pre-trained music classification models, which are nice-to-have, not essential.

### Long-Term Maintenance: Keeping the Stack Healthy

This is the part most projects ignore and then regret. Here's the realistic picture:

#### What Will Break (and When)

**PyTorch / TensorFlow version bumps (every 6-12 months):**
These are the biggest source of breakage. Demucs pins to specific torch versions, CREPE pins to specific TF versions. When PyTorch 2.5 drops and Demucs hasn't updated yet, `pip install` starts throwing resolution errors.

**Fix:** Pin major versions in `pyproject.toml` (already done above) and only update when you actively need something. The `>=2.1,<3.0` range gives breathing room. Don't chase latest unless you need a specific feature.

**CUDA toolkit changes (every 12-18 months):**
New GPU drivers sometimes need new CUDA versions. PyTorch wheels are built for specific CUDA versions (11.8, 12.1, 12.4...).

**Fix:** uv's `--torch-backend=auto` handles this — it picks the right wheel for your installed CUDA toolkit. As long as you keep your NVIDIA drivers reasonably current, this just works.

**Python version compatibility (every 12 months):**
Python 3.13 just dropped. Some libraries lag 3-6 months behind. The `requires-python = ">=3.10,<3.13"` pin prevents accidentally running on a Python too new for the stack.

**Fix:** When you want to move to a new Python, test the venv creation: `uv venv --python 3.13 .venv-test && uv pip install -e '.[all]'`. If it works, bump the pin. If not, wait.

**Library abandonment (unpredictable):**
MSAF hasn't been super actively maintained. CREPE is stable but quiet. Essentia is well-funded by MTG Barcelona.

**Fix:** The architecture already protects you. Each library is isolated behind its own MCP tool function with lazy imports. If MSAF dies, you delete `analyze_structure()` and nothing else breaks. The modular design means dead libraries can be swapped or removed without cascading failures.

#### The Maintenance Routine (Quarterly, 30 Minutes)

```bash
# 1. Check what's outdated
uv pip list --outdated

# 2. Check if anything has security issues
uv pip audit

# 3. Try updating in a test venv
uv venv .venv-test
uv pip install -e '.[all]' --torch-backend=auto --upgrade
# Run a quick smoke test
python -c "from mcp_server.tools.deep_analysis import *; print('OK')"

# 4. If everything works, update the real venv
# If not, keep the current pinned versions
```

#### Version Pinning Strategy

For a personal project, there are two good options:

**Option A — Range pins (recommended, what's in the pyproject.toml above):**
Use `>=X.Y,<X+1.0` ranges. This lets patch/minor updates through (bug fixes, performance) while blocking major version changes (API breaks). You get automatic improvements when you reinstall, but nothing should break catastrophically.

**Option B — Exact lock file:**
Run `uv pip freeze > requirements-lock.txt` after a working install. This captures the exact versions of everything. If something breaks in the future, you can always `uv pip install -r requirements-lock.txt` to get back to the last known-good state.

**Recommendation:** Use Option A for day-to-day, but keep a lock file as a snapshot after each time you verify everything works. Best of both worlds.

#### The "Is It Worth Bundling Everything?" Question

**Yes, bundle everything via `livepilot setup --all`.** Here's why:

The total disk footprint of the full stack is ~3-4 GB (dominated by PyTorch). On a machine with an RTX 5090, that's nothing. The install takes 2-5 minutes with uv. And the value is enormous — every tool in the arsenal is available the moment you need it, with no "oh wait, I need to install X first" friction during a creative session.

The only library that's debatable is **Essentia** (Windows build complexity) and **CREPE** (pulls in TensorFlow alongside PyTorch, adding ~1.5 GB). If disk space or install time matters:

- **Skip Essentia** — librosa covers the same ground for this use case
- **Skip CREPE** — basic-pitch already handles pitch transcription; CREPE's continuous pitch tracking is a luxury feature
- This drops the install from ~5 GB to ~3 GB and removes the TensorFlow dependency entirely

#### A `livepilot doctor` Command

Add a diagnostic command that checks everything is healthy:

```bash
livepilot doctor
```

Output:
```
LivePilot Health Check
━━━━━━━━━━━━━━━━━━━━━
Python:      3.11.8      ✅
venv:        .venv       ✅
FastMCP:     3.2.1       ✅
Ableton:     connected   ✅
M4L Bridge:  receiving   ✅

Analysis Stack:
  librosa:     0.10.2    ✅
  pyloudnorm:  0.1.1     ✅
  demucs:      4.0.1     ✅  (HTDemucs model cached)
  basic-pitch: 0.3.5     ✅  (model cached)
  mosqito:     1.2.0     ✅
  torch:       2.3.0     ✅  (CUDA 12.4, RTX 5090)
  crepe:       —         ⏭️  (not installed, optional)
  essentia:    —         ⏭️  (not installed, optional)

GPU: NVIDIA RTX 5090 (CUDA 12.4)
Demucs inference: ~8s per track (GPU)
```

This gives you instant visibility into what's installed, what's working, and what's using GPU acceleration.

### Summary: The Pragmatic Path

| Decision | Choice | Why |
|----------|--------|-----|
| Package manager | **uv** | 10-100x faster than pip, handles PyTorch GPU auto-detection |
| Install method | `livepilot setup --all` | One command, extends existing infrastructure |
| Dependency format | `pyproject.toml` with optional groups | Clean, standard, lets you install light or full |
| Version strategy | Range pins + frozen lock file backup | Flexible but recoverable |
| GPU handling | `--torch-backend=auto` | Zero config, works on both machines |
| Model pre-download | Yes, during install | No surprises during sessions |
| Maintenance | Quarterly 30-min check | Realistic for a personal project |
| Skip candidates | Essentia, CREPE | 90% of value with 60% of the complexity |
| Health check | `livepilot doctor` | Always know what's working |

The entire setup adds maybe 50-80 lines to `livepilot.js` and a `pyproject.toml` file. That's the whole installation story.
