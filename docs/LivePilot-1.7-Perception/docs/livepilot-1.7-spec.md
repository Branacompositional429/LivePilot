# LivePilot 1.7 — Perception Layer Expansion (Final Spec)

## What This Is

LivePilot 1.7 adds deep audio analysis to the existing real-time perception layer.
The current M4L analyzer provides 20 tools (8-band spectrum, RMS/Peak, key detection,
warp markers, device introspection). Version 1.7 adds **16 new perception tools**
across three layers (roughness and structure detection cut — see exclusions),
bringing the total to **36 perception tools**.

## Architecture: Three Layers

```
┌──────────────────────────────────────────────────────────────┐
│                      MCP Server (FastMCP)                     │
│                                                               │
│  Layer A: M4L Real-Time        Layer B: Light Python          │
│  (FluCoMa + zsa)               (no GPU)                      │
│  ┌──────────────────┐          ┌──────────────────┐          │
│  │ 9 new tools      │          │ 6 new tools      │          │
│  │ spectral shape   │          │ LUFS/LRA         │          │
│  │ MFCCs/timbral    │          │ structure detect  │          │
│  │ mel bands        │          │ spectral evolve   │          │
│  │ chroma           │          │ reference compare │          │
│  │ onsets           │          │                   │          │
│  │ H/P separation   │          └──────────────────┘          │
│  │ roughness        │                                         │
│  │ novelty          │          Layer C: Heavy Python           │
│  │ momentary LUFS   │          (GPU, PyTorch)                 │
│  └──────────────────┘          ┌──────────────────┐          │
│                                │ 3 new tools      │          │
│  + 20 existing tools           │ stem separation   │          │
│  (spectrum, RMS, key,          │ audio-to-MIDI     │          │
│   warp, devices)               │ mix diagnostics   │          │
│                                └──────────────────┘          │
└──────────────────────────────────────────────────────────────┘
```

### Why Three Layers (Not Two)

FluCoMa and zsa.descriptors are free Max packages that provide real-time spectral
analysis, MFCCs, chroma, onset detection, roughness, and harmonic/percussive separation
inside M4L — covering ~40% of what was originally planned as Python tools. This means:

- Layer A ships fast with zero Python dependencies (just Max package installs)
- Layer B adds only lightweight Python libs (~100 MB, no GPU)
- Layer C is optional heavy artillery (~3 GB with PyTorch)

Each layer works independently. You can ship Phase 1 alone and it's already a
massive upgrade.

---

## Phase 1: Layer A — Enhanced M4L Analyzer

**Dependencies:** FluCoMa package (Max Package Manager, one click) + zsa.descriptors
**Install:** No pip, no Python, no command line
**Files to modify:** `LivePilot_Analyzer.amxd` (existing M4L device)

### New MCP Tools (8 tools — roughness cut, see below)

#### 1. `get_spectral_shape`
**Source:** `fluid.spectralshape~`
**Returns:** `{ centroid, spread, skewness, kurtosis, rolloff, flatness, crest }`
**What AI gets:** 7 spectral descriptors in one call. Centroid = brightness,
rolloff = where energy lives, flatness = noise vs tone, crest = peakiness.

#### 2. `get_timbral_profile`
**Source:** `fluid.mfcc~`
**Returns:** `{ mfcc: [13 coefficients] }`
**What AI gets:** Timbral fingerprint. MFCCs encode "what this sounds like" as a
compact vector. Useful for comparing timbral character across sections or tracks.

#### 3. `get_mel_spectrum`
**Source:** `fluid.melbands~`
**Returns:** `{ mel_bands: [40 values] }`
**What AI gets:** 40-band mel-frequency spectrum (vs current 8-band FFT). Huge
resolution upgrade for spectral monitoring.

#### 4. `get_chroma`
**Source:** `fluid.chroma~`
**Returns:** `{ chroma: [12 pitch classes, C through B] }`
**What AI gets:** Which notes/pitch classes are active right now. Enables real-time
chord detection and harmonic tracking.

#### 5. `get_onsets`
**Source:** `fluid.onsetslice~` + `fluid.onsetfeature~`
**Returns:** `{ onset_detected: bool, onset_strength: float, onset_type: string }`
**What AI gets:** Transient detection with strength and character. Knows when hits
happen and how hard.

#### 6. `get_harmonic_percussive`
**Source:** `fluid.hpss~`
**Returns:** `{ harmonic_energy: float, percussive_energy: float, ratio: float }`
**What AI gets:** Real-time H/P balance. "This section is 70% harmonic, 30%
percussive" — useful for arrangement character tracking.

#### ~~7. `get_roughness`~~ — CUT
~~**Source:** `zsa.roughness~`~~
**Reason:** Requires a second Max package (zsa.descriptors) separate from FluCoMa.
One tool doesn't justify a second install dependency. AI confidence was only 45%.
FluCoMa's `fluid.spectralshape~` crest factor is a reasonable proxy if needed later.

#### 7. `get_novelty`
**Source:** `fluid.noveltyslice~`
**Returns:** `{ novelty_score: float, boundary_detected: bool }`
**What AI gets:** Real-time change detection. Spikes when the spectral character
shifts suddenly — section transitions, drops, breakdowns.

#### 8. `get_momentary_loudness`
**Source:** `fluid.loudness~`
**Returns:** `{ momentary_lufs: float, true_peak: float }`
**What AI gets:** EBU R128 momentary loudness (400ms window) + true peak.
Better than RMS for loudness monitoring, but NOT integrated LUFS or LRA —
those need Python (Layer B).

### M4L Implementation Notes

- All 9 tools follow the existing pattern: M4L device sends data via UDP to the
  MCP server, which exposes it as MCP tools
- FluCoMa objects run in the M4L device's DSP chain — CPU cost is minimal
- Each fluid object outputs to a `[route]` → `[prepend]` → `[udpsend]` chain
- Data format matches existing analyzer protocol (JSON over UDP port 9881)
- Polling rate: Same as existing analyzer (~10-20Hz, configurable)

### What Layer A Replaces (No Longer Needed in Python)

These were originally planned as Python/librosa tools but FluCoMa covers them:
- Real-time spectral analysis → `fluid.spectralshape~`
- Real-time MFCCs → `fluid.mfcc~`
- Real-time chroma/chord tracking → `fluid.chroma~`
- Onset detection → `fluid.onsetslice~`
- Psychoacoustic roughness → `zsa.roughness~`
- Essentia timbral descriptors → FluCoMa covers the useful parts
- CREPE pitch tracking → `fluid.pitch~` for monophonic cases

---

## Phase 2: Layer B — Lightweight Python (No GPU)

**Dependencies:** librosa, pyloudnorm, scipy, soundfile
**Total install size:** ~80 MB
**Install command:** `livepilot setup --light`
**New file:** `mcp_server/tools/deep_analysis.py`

### Why These Can't Be Done in M4L

- Integrated LUFS requires analyzing an entire file (not real-time windowing)
- LRA requires statistical analysis across the full track duration
- Structure detection requires self-similarity matrices + trained models
- Spectral evolution needs multi-pass offline analysis
- Reference comparison needs to load external audio files

### New MCP Tools (5 tools — structure detection cut, see below)

#### 10. `analyze_loudness`
**Library:** pyloudnorm
**Input:** `file_path: str` (path to bounced WAV)
**Returns:**
```json
{
    "integrated_lufs": -12.3,
    "true_peak_dbtp": -0.8,
    "loudness_range_lu": 6.8,
    "short_term_lufs": [-14.1, -12.8, -11.2, ...],
    "interpretation": "Track is 1.7 dB above Spotify's -14 LUFS target. LRA of 6.8 LU is healthy. True peak at -0.8 dBTP — add a limiter ceiling at -1.0 to prevent codec clipping."
}
```
**AI confidence:** 95% — objective numbers with clear targets.

**Implementation:**
```python
@mcp.tool()
def analyze_loudness(ctx: Context, file_path: str) -> dict:
    """Full loudness report: integrated LUFS, LRA, true peak, short-term curve."""
    import pyloudnorm as pyln
    import soundfile as sf

    path = _validate_audio(file_path)
    data, rate = sf.read(path)
    meter = pyln.Meter(rate)
    integrated = meter.integrated_loudness(data)
    true_peak = float(20 * np.log10(np.max(np.abs(data)) + 1e-10))

    # Short-term LUFS (3-second windows)
    window = int(3.0 * rate)
    hop = int(0.1 * rate)
    short_term = []
    for i in range(0, len(data) - window, hop):
        chunk = data[i:i+window]
        try:
            st = meter.integrated_loudness(chunk)
            short_term.append(round(st, 1))
        except:
            short_term.append(None)

    # LRA (difference between 10th and 95th percentile of short-term)
    valid = [s for s in short_term if s is not None and s > -70]
    lra = np.percentile(valid, 95) - np.percentile(valid, 10) if valid else 0

    return {
        "integrated_lufs": round(integrated, 1),
        "true_peak_dbtp": round(true_peak, 1),
        "loudness_range_lu": round(lra, 1),
        "short_term_lufs": short_term[:50],  # cap for readability
        "interpretation": _interpret_loudness(integrated, true_peak, lra)
    }
```

#### 11. `analyze_dynamic_range`
**Library:** pyloudnorm + scipy
**Input:** `file_path: str`
**Returns:** Crest factor, compression detection, per-band dynamics.
**AI confidence:** 95%

#### 12. `compare_loudness`
**Library:** pyloudnorm
**Input:** `file_a: str, file_b: str`
**Returns:** Side-by-side LUFS/LRA/peak comparison with delta values.
**AI confidence:** 95% — pure number comparison.

#### ~~13. `analyze_structure`~~ — CUT
~~**Library:** msaf~~
**Reason:** msaf last updated 2021, pins `librosa<0.9`, conflicts with our
librosa 0.10+. Can revisit later using librosa's `novelty` + `recurrence_matrix`
directly without the msaf wrapper.

#### 13. `analyze_spectral_evolution`
**Library:** librosa
**Input:** `file_path: str`
**Returns:** Per-section spectral centroid, bandwidth, rolloff averages. How the
timbral character changes across the track over time.
**AI confidence:** 70% — good at trend detection, weaker at musical interpretation.

#### 14. `compare_to_reference`
**Library:** librosa + pyloudnorm
**Input:** `file_path: str, reference_path: str`
**Returns:** Side-by-side comparison of loudness, spectral balance, dynamic range,
timbral similarity (MFCC distance). Relative differences are objective.
**AI confidence:** 75% — comparative analysis dodges absolute judgment.

### Shared Utilities (for all Layer B/C tools)

```python
import os
import numpy as np

SUPPORTED_FORMATS = {'.wav', '.mp3', '.flac', '.aif', '.aiff', '.ogg'}

def _validate_audio(file_path: str) -> str:
    """Validate file exists and is a supported audio format."""
    path = os.path.abspath(file_path)
    if not os.path.isfile(path):
        raise FileNotFoundError(f"Audio file not found: {path}")
    ext = os.path.splitext(path)[1].lower()
    if ext not in SUPPORTED_FORMATS:
        raise ValueError(f"Unsupported format: {ext}")
    return path

def _load_audio(file_path: str, sr: int = 22050):
    """Load audio with librosa, lazy import."""
    import librosa
    return librosa.load(file_path, sr=sr)

def _get_device():
    """Detect best available compute device."""
    import torch
    if torch.cuda.is_available():
        return torch.device('cuda')
    elif hasattr(torch.backends, 'mps') and torch.backends.mps.is_available():
        return torch.device('mps')
    return torch.device('cpu')
```

### Tool Registration Pattern

All tools must follow the existing pattern in `analyzer.py`:
```python
from ..server import mcp

@mcp.tool()
def tool_name(ctx: Context, file_path: str) -> dict:
    """Docstring becomes the tool description."""
    import librosa  # Always lazy import heavy libs
    path = _validate_audio(file_path)
    # ... analysis ...
    return {"key": "value", "interpretation": "Human-readable summary"}
```

**Critical rules:**
- NEVER import torch/librosa/demucs at module level — always inside the function
- Every tool MUST validate file exists and format is supported
- Return interpretive summaries, not raw numpy arrays
- Include an "interpretation" key with a human-readable sentence

---

## Phase 3: Layer C — Heavy Python (GPU, Optional)

**Dependencies:** Everything from Layer B + torch, torchaudio, demucs, basic-pitch
**Total install size:** ~3 GB (dominated by PyTorch)
**Install command:** `livepilot setup --all`
**GPU:** CUDA on Windows (RTX 5090), MPS on Mac (M3 Max), falls back to CPU

### New MCP Tools (3 tools)

#### 15. `separate_stems`
**Library:** Demucs (HTDemucs v4)
**Input:** `file_path: str, model: str = "htdemucs"`
**Returns:**
```json
{
    "stems": {
        "vocals": "/path/to/output/vocals.wav",
        "drums": "/path/to/output/drums.wav",
        "bass": "/path/to/output/bass.wav",
        "other": "/path/to/output/other.wav"
    },
    "model": "htdemucs",
    "processing_time_sec": 8.2,
    "device": "cuda"
}
```
**AI confidence:** 65% — the separation itself is a pipeline step. My value is in
analyzing the separated stems with Layer B tools. Diagnosis accuracy depends on
Demucs separation quality (artifacts/bleed can mislead analysis).

**Implementation:** Use subprocess to call Demucs CLI rather than importing directly.
This avoids PyTorch GPU memory management issues:
```python
@mcp.tool()
def separate_stems(ctx: Context, file_path: str, model: str = "htdemucs") -> dict:
    """Separate audio into vocals, drums, bass, other stems using Demucs."""
    import subprocess, json, time
    path = _validate_audio(file_path)
    output_dir = os.path.join(os.path.dirname(path), "stems")
    os.makedirs(output_dir, exist_ok=True)

    start = time.time()
    result = subprocess.run(
        ["python", "-m", "demucs", "--two-stems=None", "-n", model, "-o", output_dir, path],
        capture_output=True, text=True, timeout=300
    )
    if result.returncode != 0:
        raise RuntimeError(f"Demucs failed: {result.stderr}")

    elapsed = round(time.time() - start, 1)
    stem_dir = os.path.join(output_dir, model, os.path.splitext(os.path.basename(path))[0])
    stems = {name: os.path.join(stem_dir, f"{name}.wav")
             for name in ["vocals", "drums", "bass", "other"]}
    return {"stems": stems, "model": model, "processing_time_sec": elapsed}
```

#### 16. `transcribe_to_midi`
**Library:** basic-pitch (Spotify)
**Input:** `file_path: str, output_midi: str = None`
**Returns:**
```json
{
    "midi_path": "/path/to/output.mid",
    "notes": [
        {"pitch": 60, "start": 0.0, "end": 0.5, "velocity": 80, "pitch_bend": 0.0},
        {"pitch": 64, "start": 0.25, "end": 0.75, "velocity": 72, "pitch_bend": -0.1},
        ...
    ],
    "note_count": 147,
    "pitch_range": {"low": "C3", "high": "G5"},
    "detected_key": "C major",
    "interpretation": "147 notes detected. Range C3-G5. Predominantly C major with
     occasional F# suggesting mixolydian inflections or secondary dominants."
}
```
**AI confidence:** 90% — MIDI is structured data, music theory is my strongest
audio-adjacent skill. Once notes become numbers I can identify chords, detect
out-of-key notes, suggest harmonizations.

**Implementation:**
```python
@mcp.tool()
def transcribe_to_midi(ctx: Context, file_path: str) -> dict:
    """Transcribe audio to MIDI with polyphonic pitch detection and pitch bends."""
    from basic_pitch.inference import predict
    from basic_pitch import ICASSP_2022_MODEL_PATH
    path = _validate_audio(file_path)

    model_output, midi_data, note_events = predict(path)
    midi_path = os.path.splitext(path)[0] + "_transcribed.mid"
    midi_data.write(midi_path)

    notes = [{"pitch": n.pitch, "start": round(n.start, 3),
              "end": round(n.end, 3), "velocity": n.velocity,
              "pitch_bend": round(n.pitch_bend, 2) if hasattr(n, 'pitch_bend') else 0}
             for n in note_events[:200]]  # cap for readability

    return {"midi_path": midi_path, "notes": notes, "note_count": len(note_events)}
```

#### 17. `diagnose_mix`
**Library:** Compound — uses separate_stems + analyze_loudness + librosa
**Input:** `file_path: str`
**Returns:** Full diagnostic report: separation → per-stem loudness + spectral
analysis → cross-stem masking detection → ranked issues with fix suggestions.
**AI confidence:** 65% — chains multiple uncertain analyses. Treat as a
well-read intern's second opinion, not a mastering engineer's verdict.

This tool calls the other tools internally — it's the compound pipeline:
```
Bounce → separate_stems → for each stem: analyze_loudness + spectral →
cross-correlate stems → detect masking/phase/balance issues → ranked report
```

---

## Installation

### pyproject.toml (NEW — replaces requirements.txt)

```toml
[project]
name = "livepilot"
version = "1.7.0"
requires-python = ">=3.10,<3.13"
dependencies = ["fastmcp>=3.0.0,<4.0.0"]

[project.optional-dependencies]
analysis = [
    "librosa>=0.10.0,<0.11",
    "pyloudnorm>=0.1.1",
    "soundfile>=0.12",
    "scipy>=1.11",
]
separation = [
    "demucs>=4.0",
    "torch>=2.1",
    "torchaudio>=2.1",
]
transcription = [
    "basic-pitch>=0.3",
]
all = [
    "livepilot[analysis,separation,transcription]",
]
```

Note: No Essentia, no CREPE, no TensorFlow, no MOSQITO. Dropped deliberately —
FluCoMa covers the real-time analysis, and the remaining Python tools don't need them.

### livepilot.js Changes

**`livepilot setup --light`** installs `analysis` group only (~100 MB, no GPU)
**`livepilot setup --all`** installs everything (~3 GB, auto-detects GPU)

Both use `uv` (Rust-based Python package manager, 10-100x faster than pip).
`--torch-backend=auto` picks CUDA wheels on Windows, MPS on Mac.

```javascript
async function ensureUv() {
    try { await exec("uv --version"); }
    catch {
        console.log("Installing uv...");
        if (process.platform === "win32")
            await exec('powershell -ExecutionPolicy ByPass -c "irm https://astral.sh/uv/install.ps1 | iex"');
        else
            await exec('curl -LsSf https://astral.sh/uv/install.sh | sh');
    }
}

async function setupAnalysis(flags) {
    await ensureUv();
    const group = flags.light ? "analysis" : "all";
    await exec(`uv pip install -e ".[${group}]" --torch-backend=auto`, {
        cwd: ROOT, env: { ...process.env, UV_PROJECT_ENVIRONMENT: VENV_DIR }
    });
    if (!flags.light) {
        console.log("Pre-downloading AI models...");
        await exec(`${pythonBin} -c "
import demucs.pretrained; demucs.pretrained.get_model('htdemucs')
from basic_pitch.inference import predict
print('Models ready.')"
        `);
    }
    await exec(`uv pip freeze > requirements-lock.txt`, { cwd: ROOT });
}
```

### `livepilot doctor` (NEW command)

Health check that reports what's installed and working:
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
  torch:       2.3.0     ✅  (CUDA 12.4, RTX 5090)

GPU: NVIDIA RTX 5090 (CUDA 12.4)
```

---

## M4L Capture Device (DEFERRED to 1.7.1 — LivePilot_Capture.amxd)

> **Note:** This device is deferred from 1.7.0 to 1.7.1. It's a whole new .amxd
> with buffer management, file writing, UDP notifications, and a server-side
> capture registry. The core perception tools don't need it — users can bounce
> to disk from Ableton and pass the file path to analysis tools manually.
> Ship the 16 perception tools first, add the capture pipeline after.

A new M4L device for capturing audio to disk, enabling the bounce-to-analysis pipeline.

### Signal Flow
```
Track audio in → [buffer~ capture_buf] → [writewave] → WAV on disk
                      ↓
              (pass-through to output — non-destructive)
                      ↓
              [udpsend] notify MCP server → file path + metadata
```

### Device Layout
- **Capture button** — starts/stops recording to buffer
- **Duration display** — shows capture length
- **Auto-analyze toggle** — if on, triggers `analyze_loudness` automatically after capture
- **Output directory** — configurable, defaults to `~/Documents/LivePilot/captures/`

### JS Bridge (`livepilot_capture.js` inside the M4L device)
```javascript
function getTrackName() {
    var track = new LiveAPI("canonical_parent");
    return track.get("name").toString();
}

function generateFilename() {
    var track = getTrackName();
    var date = new Date().toISOString().slice(0,19).replace(/:/g,"-");
    return track + "_" + date + ".wav";
}

function notifyCapture(filepath) {
    // Send capture notification to MCP server via UDP
    outlet(0, "udp", JSON.stringify({
        event: "capture_complete",
        file_path: filepath,
        track_name: getTrackName(),
        duration_sec: getDuration()
    }));
}
```

---

## Processing Time & Agent Behavior

### Speed Tier by Tool

Every tool has a processing time that determines how the agent should use it.
This table must be enforced in the skill definition.

| Tool | Speed Tier | Time (30s file) | Time (4min file) | Agent Rule |
|------|:----------:|:---------------:|:----------------:|------------|
| get_spectral_shape | Instant | <50ms | <50ms | Use freely |
| get_timbral_profile | Instant | <50ms | <50ms | Use freely |
| get_mel_spectrum | Instant | <50ms | <50ms | Use freely |
| get_chroma | Instant | <50ms | <50ms | Use freely |
| get_onsets | Instant | <50ms | <50ms | Use freely |
| get_harmonic_percussive | Instant | <50ms | <50ms | Use freely |
| get_novelty | Instant | <50ms | <50ms | Use freely |
| get_momentary_loudness | Instant | <50ms | <50ms | Use freely |
| analyze_loudness | Fast | ~0.5s | ~2s | Use freely |
| analyze_dynamic_range | Fast | ~0.5s | ~2s | Use freely |
| compare_loudness | Fast | ~1s | ~4s | Use freely |
| analyze_spectral_evolution | Slow | ~2s | ~8s | Warn user first |
| compare_to_reference | Slow | ~3s | ~12s | Warn user first |
| transcribe_to_midi | Slow | ~5s | ~15s | Warn user first |
| separate_stems | Heavy | ~15s (GPU) | ~60-90s (MPS) | Ask user consent |
| diagnose_mix | Heavy | ~30s (GPU) | ~120s (MPS) | Ask user consent |

### Agent Escalation Protocol

The agent must follow a strict escalation ladder — always start with the fastest
tools and only use heavier tools when explicitly needed:

```
Level 1 (instant):  get_master_spectrum + get_track_meters
Level 2 (fast):     analyze_loudness + analyze_dynamic_range
Level 3 (slow):     analyze_spectral_evolution + compare_to_reference
Level 4 (heavy):    separate_stems → per-stem analysis → diagnose_mix
```

Rules:
- Never skip levels. Start at Level 1, escalate with consent.
- Never call Heavy tools speculatively or during creative flow.
- Always warn the user before calling Slow tools.
- If the user says "check my mix", use Level 1-2. Offer Level 3-4 only if needed.
- When calling Heavy tools, report progress updates so the user knows it's working.

### Tool Docstring Convention

All Slow and Heavy tools must include timing in their docstring so the agent
sees it before calling:

```python
@mcp.tool()
def separate_stems(ctx: Context, file_path: str, model: str = "htdemucs") -> dict:
    """Separate audio into vocals, drums, bass, other stems using Demucs.

    Processing time: 15-25s (GPU) / 60-90s (CPU/MPS).
    Always inform the user before calling — this is a Heavy tool.
    """
```

### Quick Mode for Heavy Tools

Heavy tools should accept a `quick: bool = False` parameter:

- `quick=True` on `separate_stems`: process only first 30 seconds, use faster model
- `quick=True` on `diagnose_mix`: skip stem separation, use spectral analysis only

This gives the agent a fast escape hatch when it needs a rough answer without
the full processing cost.

---

## AI Capability: What I'm Good At, What I'm Not

### Confidence Ranking by Tool

| Tool | AI Confidence | Why |
|------|:---:|---|
| analyze_loudness | 95% | Objective numbers, standardized targets, zero ambiguity |
| analyze_dynamic_range | 95% | Same — pure metrics |
| compare_loudness | 95% | Number comparison |
| transcribe_to_midi | 90% | MIDI = structured data, music theory is my strength |
| compare_to_reference | 75% | Relative comparison dodges my biggest weakness |
| analyze_spectral_evolution | 70% | Good at trends, weaker at musical meaning |
| get_spectral_shape | 70% | Same — strong on numbers, shaky on sonic meaning |
| separate_stems | 65% | Value depends on Demucs quality |
| diagnose_mix | 65% | Compound uncertainty — well-read intern, not mastering engineer |
| get_chroma | 65% | Chord detection is reasonable but not perfect |
| get_timbral_profile | 60% | Can compare timbres, can't describe sonic character reliably |

### What I'm Great At
- Cross-referencing 12+ metrics simultaneously (no human does this)
- Tracking changes across bounce versions with precision
- Remembering loudness targets and checking every bounce automatically
- Consistency across album projects

### What I'll Never Do Well
- "Does this sound good?" — I have statistics, not taste
- "Is this creative choice working?" — I'll flag intentional choices as problems
- "The mix has soul" — no metric for soul, groove, or vibe
- Catching subtle phase issues that manifest as "hollowness"

### Best Workflow
Use me for tedious technical analysis so you can focus on creative decisions.
Don't let me make creative decisions. Let me say "your verse is 3dB quieter
than your chorus and the spectral centroid drops 500Hz" and then YOU decide
whether that's a problem or a feature.

---

## What Was Deliberately Excluded (And Why)

| Library | Reason for Exclusion |
|---------|---------------------|
| **Essentia** | Windows build complexity; FluCoMa + librosa cover the same ground |
| **CREPE** | Pulls TensorFlow (~1.5 GB); basic-pitch handles transcription, fluid.pitch~ handles monophonic |
| **MOSQITO** | Full Zwicker model overkill; spectral crest from FluCoMa is a reasonable proxy |
| **Chromaprint** | Niche use case (fingerprinting); can add later if needed |
| **TensorFlow** | Excluded entirely — only PyTorch as the one deep learning framework |
| **zsa.descriptors** | Second Max package dependency for one tool (roughness at 45% AI confidence). Not worth the install friction. |
| **msaf** | Last updated 2021, pins librosa<0.9, conflicts with modern librosa 0.10+. Structure detection can be done later with raw librosa. |
| **RVC** | Ethical minefield, niche use case, massive model size. Deferred indefinitely. |
| **Groove2Groove** | Research code, not maintained since 2022. Deferred to 2.0+. |
| **MidiDrumiGen** | Requires FastAPI sidecar, too fragile. Deferred to 2.0+. |

This keeps the stack minimal: one DL framework (PyTorch), one Max package (FluCoMa),
four lightweight Python libs (librosa, pyloudnorm, soundfile, scipy). Total: ~80 MB
for Layer B, ~3 GB for Layer C (dominated by PyTorch).

---

## Cross-Platform Compatibility

| Component | Windows (RTX 5090) | macOS (M3 Max) |
|-----------|:---:|:---:|
| FluCoMa (M4L) | ✅ | ✅ |
| librosa | ✅ | ✅ |
| pyloudnorm | ✅ | ✅ |
| soundfile | ✅ | ✅ |
| scipy | ✅ | ✅ |
| demucs | ✅ CUDA (15-25s) | ✅ MPS (60-90s) |
| basic-pitch | ✅ CPU | ✅ CPU |

`--torch-backend=auto` handles CUDA vs MPS automatically. No manual config needed.

---

## Maintenance

**Quarterly routine (30 min):**
```bash
uv pip list --outdated
uv pip audit
# Test in throwaway venv before upgrading real one
uv venv .venv-test && uv pip install -e '.[all]' --torch-backend=auto --upgrade
python -c "from mcp_server.tools.deep_analysis import *; print('OK')"
```

**Version strategy:** Range pins in pyproject.toml + `requirements-lock.txt`
snapshot after each verified install. Flexible but recoverable.

**If a library dies:** Each tool has lazy imports behind its own function.
Delete the function, nothing else breaks. Modular by design.

---

## File Map (What to Create for 1.7.0)

```
mcp_server/
├── tools/
│   ├── analyzer.py              (EXISTING — add Layer A data intake)
│   └── deep_analysis.py         (NEW — Layer B + C tools)
├── server.py                    (EXISTING — register new tools)
m4l_device/
├── LivePilot_Analyzer.amxd      (MODIFY — add FluCoMa objects)
├── livepilot_bridge.js          (EXISTING — add FluCoMa data handlers)
bin/
├── livepilot.js                 (MODIFY — add setup --light/--all, doctor)
pyproject.toml                   (NEW — replaces requirements.txt)

# Deferred to 1.7.1:
# m4l_device/LivePilot_Capture.amxd   (NEW — audio capture device)
# m4l_device/livepilot_capture.js     (NEW — JS bridge for capture)
```

---

## Implementation Priority

1. **Phase 1 first.** Get the 9 FluCoMa/zsa tools working in the M4L device.
   Test thoroughly. This alone is a massive perception upgrade with zero risk.
2. **Phase 2 next.** Add the 6 lightweight Python tools. Test loudness analysis
   against known reference values. Verify structure detection accuracy.
3. **Phase 3 last.** Only after Phases 1-2 are solid. Demucs + basic-pitch
   are the highest-value but highest-complexity additions.

---

## What Comes After 1.7

See **`livepilot-1.8-spec.md`** for the full v1.8 implementation spec, and
**`beyond-perception-research.md`** for v1.9→2.0 research.

| Version | Focus | New Tools | Key Libraries |
|---------|-------|:---------:|---------------|
| **1.8** | Theory + Productivity | 12 | music21, isobar, mutagen |
| **1.9** | Processing + Discovery | ~12 | pedalboard, spotipy, freesound, matchering |
| **2.0** | Generation | ~8 | MusicGen/RAVE |

**The biggest unlock after perception:** music21 gives LivePilot **computational
precision** on music theory (Roman numeral analysis, voice leading detection,
counterpoint generation). The LLM already knows theory from training — music21
lets it compute from actual note data instead of guessing. The tools provide data,
the LLM provides interpretation. Education behavior is encoded as skill instructions,
not as MCP tools — pure LLM reasoning shouldn't pay MCP round-trip overhead.

The pipeline: get_notes → music21 analysis → LLM interprets → suggests changes.
No bouncing, no files. Theory analysis at conversation speed.

Build 1.7 first. Then read `livepilot-1.8-spec.md` for the theory layer spec.

---

## Gap Fixes: Engineering Details

The following six sections close the remaining engineering gaps identified
during the final review. These are essential for a clean Claude Code handoff.

---

### Gap 1: Testing Strategy

Every tool needs a way to verify it works. Without test data and expected values,
Claude Code will build tools that "run" but may return garbage.

**Test audio files (create or source these):**

| File | Duration | Purpose | Known Values |
|------|----------|---------|-------------|
| `test_sine_440hz.wav` | 5 sec | Sanity check | Centroid ≈ 440 Hz, flat spectrum |
| `test_mastered_pop.wav` | 30 sec | Loudness calibration | LUFS ≈ -14, LRA ≈ 6-8 LU |
| `test_verse_chorus.wav` | 60 sec | Structure detection | 2 clear sections at ~30 sec boundary |
| `test_piano_cmaj.wav` | 10 sec | Transcription check | Known C major chord, 3 notes |
| `test_drum_loop.wav` | 4 bars | H/P separation | High percussive ratio expected |

**`livepilot test` command (add to livepilot.js):**
```javascript
async function runTests() {
    const testDir = path.join(ROOT, "tests", "fixtures");
    const results = [];

    // Test 1: Loudness accuracy
    const lufs = await callTool("analyze_loudness", {file_path: path.join(testDir, "test_mastered_pop.wav")});
    results.push({
        name: "loudness_accuracy",
        pass: Math.abs(lufs.integrated_lufs - (-14)) < 1.0,  // within 1 dB
        expected: -14, actual: lufs.integrated_lufs
    });

    // Test 2: Structure boundary detection
    const struct = await callTool("analyze_structure", {file_path: path.join(testDir, "test_verse_chorus.wav")});
    const boundary = struct.sections[1]?.start;
    results.push({
        name: "structure_boundary",
        pass: Math.abs(boundary - 30.0) < 2.0,  // within 2 seconds
        expected: 30, actual: boundary
    });

    // Test 3: Transcription note count
    const midi = await callTool("transcribe_to_midi", {file_path: path.join(testDir, "test_piano_cmaj.wav")});
    results.push({
        name: "transcription_notes",
        pass: midi.note_count >= 2 && midi.note_count <= 5,
        expected: "2-5 notes", actual: midi.note_count
    });

    console.log("\nLivePilot Test Results:");
    results.forEach(r => console.log(`  ${r.pass ? "✅" : "❌"} ${r.name}: expected ${r.expected}, got ${r.actual}`));
    return results.every(r => r.pass);
}
```

**Test file location:** `tests/fixtures/` — commit small WAV files to repo.
Keep them under 1 MB each (short duration, mono, 44.1kHz).

---

### Gap 2: Error Handling Patterns

Every tool must handle failures gracefully. Standardize error responses so the
AI always gets structured feedback, never raw stack traces.

**Standard error response format:**
```python
def _tool_error(error_type: str, message: str, suggestion: str = None) -> dict:
    """Standardized error return for all tools."""
    result = {
        "error": True,
        "error_type": error_type,
        "message": message,
    }
    if suggestion:
        result["suggestion"] = suggestion
    return result
```

**Error types and how to handle them:**

| Scenario | error_type | Example suggestion |
|----------|-----------|-------------------|
| File not found | `file_not_found` | "Check the file path. Use absolute paths." |
| Unsupported format | `unsupported_format` | "Supported: .wav, .mp3, .flac, .aif, .ogg" |
| Corrupt/unreadable audio | `corrupt_audio` | "File may be corrupted. Try re-exporting from Live." |
| GPU out of memory | `gpu_oom` | "File too long for GPU. Try a shorter clip or use --cpu." |
| Model not downloaded | `model_missing` | "Run 'livepilot setup --all' to download AI models." |
| Timeout (>5 min) | `timeout` | "File too long. Try analyzing a shorter section." |
| Library not installed | `missing_dependency` | "This tool requires Layer C. Run 'livepilot setup --all'." |

**Wrap every tool with try/except:**
```python
@mcp.tool()
def analyze_loudness(ctx: Context, file_path: str) -> dict:
    """Full loudness report."""
    try:
        path = _validate_audio(file_path)
        import pyloudnorm as pyln
        import soundfile as sf
        # ... analysis code ...
        return {"integrated_lufs": ..., "interpretation": ...}
    except FileNotFoundError:
        return _tool_error("file_not_found", f"File not found: {file_path}",
                          "Check the path. Use absolute paths.")
    except ImportError:
        return _tool_error("missing_dependency", "pyloudnorm not installed",
                          "Run 'livepilot setup --light' to install analysis tools.")
    except Exception as e:
        return _tool_error("unexpected", str(e),
                          "Report this error — it may be a bug.")
```

**File size guardrails:**
```python
MAX_FILE_SIZE_MB = {
    "layer_b": 200,    # 200 MB max for lightweight tools
    "layer_c": 500,    # 500 MB max for heavy tools (Demucs)
}

def _check_file_size(path: str, layer: str) -> None:
    size_mb = os.path.getsize(path) / (1024 * 1024)
    limit = MAX_FILE_SIZE_MB.get(layer, 200)
    if size_mb > limit:
        raise ValueError(f"File is {size_mb:.0f} MB (max {limit} MB for {layer} tools)")
```

**Demucs subprocess timeout:** Already in the spec at 300 seconds. Add a
`--timeout` parameter so users can extend for long tracks:
```python
def separate_stems(ctx, file_path, model="htdemucs", timeout_sec=300):
```

---

### Gap 3: Capture → Analysis Server-Side Handler

The spec defines the M4L capture device sending UDP notifications, but not what
receives them on the MCP server side.

**Add to `m4l_bridge.py` — capture event listener:**
```python
import json
import asyncio

# Existing UDP listener already runs on port 9881
# Add a handler for "capture_complete" events

async def handle_capture_event(data: dict):
    """Called when M4L capture device finishes recording."""
    event = data.get("event")
    if event != "capture_complete":
        return

    file_path = data["file_path"]
    track_name = data.get("track_name", "unknown")
    duration = data.get("duration_sec", 0)

    # Log the capture
    logger.info(f"Capture received: {track_name} ({duration:.1f}s) → {file_path}")

    # Store in capture registry (so AI can reference recent captures)
    capture_registry.append({
        "file_path": file_path,
        "track_name": track_name,
        "duration_sec": duration,
        "timestamp": datetime.now().isoformat(),
        "analyzed": False
    })

    # Auto-analyze if enabled (controlled by M4L device toggle)
    if data.get("auto_analyze", False):
        from .tools.deep_analysis import analyze_loudness
        result = await asyncio.to_thread(analyze_loudness, None, file_path)
        capture_registry[-1]["loudness"] = result
        capture_registry[-1]["analyzed"] = True
```

**New MCP tool to access captures:**
```python
@mcp.tool()
def list_recent_captures(ctx: Context, limit: int = 10) -> dict:
    """List recent audio captures from the M4L capture device."""
    return {"captures": capture_registry[-limit:]}
```

**The capture registry** is an in-memory list (not a database). It resets when
the server restarts. This is fine — captures are files on disk, the registry
is just a convenience index.

---

### Gap 4: Concurrency Model

FastMCP uses asyncio. Tool functions that do CPU-heavy work (librosa, Demucs)
will block the event loop unless handled correctly.

**Rule: All Layer B/C tools must run in a thread pool.**

```python
import asyncio
from concurrent.futures import ThreadPoolExecutor

# Shared thread pool — max 2 concurrent analysis jobs
# (prevents GPU OOM from parallel Demucs runs)
analysis_pool = ThreadPoolExecutor(max_workers=2, thread_name_prefix="analysis")

@mcp.tool()
async def analyze_loudness(ctx: Context, file_path: str) -> dict:
    """Full loudness report."""
    loop = asyncio.get_event_loop()
    return await loop.run_in_executor(analysis_pool, _analyze_loudness_sync, file_path)

def _analyze_loudness_sync(file_path: str) -> dict:
    """Synchronous implementation (runs in thread pool)."""
    import pyloudnorm as pyln
    import soundfile as sf
    path = _validate_audio(file_path)
    # ... actual analysis ...
    return {... }
```

**Why max_workers=2:**
- Layer B tools (librosa, pyloudnorm) are CPU-bound — 2 can run in parallel fine
- Layer C tools (Demucs) grab the GPU — only 1 should run at a time
- The thread pool naturally queues excess requests

**GPU lock for Layer C:**
```python
import threading
gpu_lock = threading.Lock()

def _analyze_with_gpu(func, *args):
    """Serialize GPU-bound operations."""
    with gpu_lock:
        return func(*args)
```

**What the user experiences:**
- Ask for loudness analysis → instant response (~2 sec)
- Ask for stem separation → "processing..." (~8 sec), but other tools still respond
- Ask for two stem separations at once → second one queues, finishes after the first

---

### Gap 5: Standardized Output Paths

All tool outputs go to one configurable root directory with clear organization.

**Default output root:** `~/Documents/LivePilot/outputs/`
(Overridable via config or `--output-dir` flag)

**Directory structure:**
```
~/Documents/LivePilot/outputs/
├── captures/              ← M4L capture device recordings
│   └── TrackName_2026-03-19T14-30-00.wav
├── stems/                 ← Demucs stem separation output
│   └── TrackName_2026-03-19/
│       ├── vocals.wav
│       ├── drums.wav
│       ├── bass.wav
│       └── other.wav
├── midi/                  ← basic-pitch transcriptions
│   └── TrackName_2026-03-19_transcribed.mid
└── reports/               ← JSON analysis reports (optional)
    └── TrackName_2026-03-19_loudness.json
```

**Shared path utility:**
```python
import os
from datetime import datetime

OUTPUT_ROOT = os.path.expanduser("~/Documents/LivePilot/outputs")

def _output_path(category: str, source_file: str, extension: str) -> str:
    """Generate standardized output path."""
    base = os.path.splitext(os.path.basename(source_file))[0]
    date = datetime.now().strftime("%Y-%m-%d")
    cat_dir = os.path.join(OUTPUT_ROOT, category)
    os.makedirs(cat_dir, exist_ok=True)
    return os.path.join(cat_dir, f"{base}_{date}{extension}")
```

**Update tool implementations to use it:**
```python
# In separate_stems:
output_dir = os.path.join(OUTPUT_ROOT, "stems")

# In transcribe_to_midi:
midi_path = _output_path("midi", file_path, "_transcribed.mid")
```

---

### Gap 6: The .amxd Build Problem

Claude Code cannot create .amxd files from scratch — they're binary Max patches.
Three options, in order of practicality:

**Option A (Recommended): Script-driven patching via `js` object**

Max's `js` object can create and connect objects programmatically. Write a
`livepilot_setup_flucom.js` script that runs inside an existing M4L device
and builds the FluCoMa signal chain:

```javascript
// livepilot_setup_flucoma.js — Run once inside Max to build the patch
var p = this.patcher;

// Create FluCoMa objects
var spectral = p.newdefault(100, 100, "fluid.spectralshape~");
var mfcc = p.newdefault(100, 200, "fluid.mfcc~");
var chroma = p.newdefault(100, 300, "fluid.chroma~");
var melbands = p.newdefault(100, 400, "fluid.melbands~");
var onset = p.newdefault(300, 100, "fluid.onsetslice~");
var hpss = p.newdefault(300, 200, "fluid.hpss~");
var roughness = p.newdefault(300, 300, "zsa.roughness~");
var novelty = p.newdefault(300, 400, "fluid.noveltyslice~");
var loudness = p.newdefault(500, 100, "fluid.loudness~");

// Create routing to UDP
var prepend = p.newdefault(500, 300, "prepend", "spectral");
var udpsend = p.newdefault(500, 400, "udpsend", "127.0.0.1", 9881);

// Connect: audio input → FluCoMa objects → prepend → udpsend
p.connect(spectral, 0, prepend, 0);
p.connect(prepend, 0, udpsend, 0);
// ... repeat for each object
```

**How Claude Code uses this:**
1. Claude Code writes the `.js` setup script
2. The user opens the existing LivePilot_Analyzer.amxd in Max
3. The user adds a `js livepilot_setup_flucoma.js` object and bangs it
4. The script creates all FluCoMa routing automatically
5. The user saves the device — done

**Option B: Provide a Max patch diff (human applies it)**
Claude Code writes a detailed step-by-step Max patching guide with screenshots
described in text. The user follows it manually. Slower but no scripting risk.

**Option C: Ship a pre-built .amxd in the repo**
Build the FluCoMa patch once by hand, commit the .amxd binary to the repo.
Claude Code can't modify it, but it doesn't need to — the M4L side is
"build once, use forever." All dynamic behavior comes from the Python MCP tools.

**Recommendation:** Use Option A for initial build, then commit the resulting
.amxd as Option C for future installs. The js script serves as both builder
and documentation.
