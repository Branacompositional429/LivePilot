# Deep Analysis Module — Implementation Spec
## File: `mcp_server/tools/deep_analysis.py`

## Overview
Single Python module registering ~30 MCP tools for offline audio analysis. Each tool takes a `file_path` to a bounced WAV and returns a JSON-serializable dict. All heavy libraries are lazy-imported inside tool functions.

## Shared Utilities (top of file)

```python
from ..server import mcp
from mcp.server.fastmcp import Context
import os

SUPPORTED_FORMATS = {'.wav', '.mp3', '.flac', '.aif', '.aiff', '.ogg'}

def _validate_audio(file_path: str) -> str:
    """Validate file exists and is a supported audio format. Returns absolute path."""
    path = os.path.abspath(file_path)
    if not os.path.isfile(path):
        raise FileNotFoundError(f"Audio file not found: {path}")
    ext = os.path.splitext(path)[1].lower()
    if ext not in SUPPORTED_FORMATS:
        raise ValueError(f"Unsupported format: {ext}. Use: {SUPPORTED_FORMATS}")
    return path

def _load_audio(file_path: str, sr: int = 22050):
    """Load audio with librosa. Returns (y, sr)."""
    import librosa
    return librosa.load(file_path, sr=sr, mono=False)

def _get_device():
    """Return best available torch device."""
    import torch
    if torch.cuda.is_available():
        return torch.device('cuda')
    elif hasattr(torch.backends, 'mps') and torch.backends.mps.is_available():
        return torch.device('mps')
    return torch.device('cpu')
```

## Tool Group 1: Loudness (pyloudnorm)

### `analyze_loudness(ctx, file_path: str) -> dict`
Full LUFS report for a bounced file.
```python
@mcp.tool()
def analyze_loudness(ctx: Context, file_path: str) -> dict:
    """Analyze loudness of an audio file. Returns integrated LUFS, true peak, LRA, and crest factor."""
    import soundfile as sf
    import pyloudnorm as pyln
    import numpy as np

    path = _validate_audio(file_path)
    data, rate = sf.read(path)
    meter = pyln.Meter(rate)

    # Integrated LUFS
    loudness = meter.integrated_loudness(data)

    # True peak (dBTP) via 4x oversampling
    from scipy.signal import resample
    oversampled = resample(data, len(data) * 4, axis=0)
    true_peak_linear = np.max(np.abs(oversampled))
    true_peak_dbtp = 20 * np.log10(true_peak_linear + 1e-10)

    # Peak-to-Loudness Ratio
    plr = true_peak_dbtp - loudness

    # Crest factor (RMS vs peak)
    rms = np.sqrt(np.mean(data ** 2))
    crest_factor = 20 * np.log10(np.max(np.abs(data)) / (rms + 1e-10))

    return {
        "integrated_lufs": round(loudness, 1),
        "true_peak_dbtp": round(true_peak_dbtp, 1),
        "peak_to_loudness_ratio": round(plr, 1),
        "crest_factor_db": round(crest_factor, 1),
        "sample_rate": rate,
        "duration_seconds": round(len(data) / rate, 2),
        "channels": data.shape[1] if data.ndim > 1 else 1
    }
```

### `analyze_dynamic_range(ctx, file_path: str) -> dict`
LRA, short-term LUFS curve, per-band crest factors.

### `compare_loudness(ctx, file_a: str, file_b: str) -> dict`
A/B two bounces — returns deltas for all loudness metrics.

## Tool Group 2: Deep Spectral (librosa)

### `analyze_timbre(ctx, file_path: str) -> dict`
Returns: 20 MFCCs (mean + std), spectral centroid/bandwidth/rolloff/flatness (mean + std), brightness assessment.

### `analyze_harmonic_content(ctx, file_path: str) -> dict`
Returns: chroma distribution (12 pitch classes), estimated key + confidence, tonnetz summary, chord change timestamps.

### `analyze_spectral_evolution(ctx, file_path: str) -> dict`
Returns: spectral centroid curve (downsampled to ~1 point/second), brightness trajectory description, spectral contrast per band.

### `analyze_rhythm_stability(ctx, file_path: str) -> dict`
Returns: estimated BPM + confidence, tempo stability score, onset density curve, beat strength profile.

## Tool Group 3: Source Separation (Demucs)

### `separate_stems(ctx, file_path: str, model: str = "htdemucs") -> dict`
Run Demucs separation via subprocess. Returns paths to 4 stem files.
```python
@mcp.tool()
def separate_stems(ctx: Context, file_path: str, model: str = "htdemucs") -> dict:
    """Separate audio into stems (vocals, drums, bass, other) using Demucs neural network."""
    import subprocess, tempfile, os

    path = _validate_audio(file_path)
    out_dir = tempfile.mkdtemp(prefix="livepilot_stems_")

    # Run demucs as subprocess to avoid memory issues with repeated calls
    cmd = ["python", "-m", "demucs", "--two-stems" if model == "htdemucs" else "",
           "-n", model, "-o", out_dir, path]
    cmd = [c for c in cmd if c]  # filter empty strings
    result = subprocess.run(cmd, capture_output=True, text=True, timeout=300)

    if result.returncode != 0:
        raise RuntimeError(f"Demucs failed: {result.stderr}")

    # Find output stems
    stem_dir = os.path.join(out_dir, model, os.path.splitext(os.path.basename(path))[0])
    stems = {}
    for stem_name in ["vocals", "drums", "bass", "other"]:
        stem_path = os.path.join(stem_dir, f"{stem_name}.wav")
        if os.path.isfile(stem_path):
            stems[stem_name] = stem_path

    return {
        "stems": stems,
        "model": model,
        "output_directory": stem_dir,
        "stem_count": len(stems)
    }
```

### `analyze_stem(ctx, file_path: str) -> dict`
Deep analysis of a single stem — calls analyze_timbre + analyze_loudness internally.

### `diagnose_mix(ctx, file_path: str) -> dict`
Compound tool: separate_stems → analyze each stem → cross-stem spectral overlap detection → frequency masking report → actionable suggestions. This is the highest-value tool.

## Tool Group 4: Transcription (basic-pitch)

### `transcribe_to_midi(ctx, file_path: str, output_path: str = None) -> dict`
Polyphonic audio-to-MIDI. Returns: MIDI file path, note count, pitch range, estimated key.
```python
@mcp.tool()
def transcribe_to_midi(ctx: Context, file_path: str, output_path: str = None) -> dict:
    """Transcribe audio to MIDI using basic-pitch. Handles polyphony and pitch bends."""
    from basic_pitch.inference import predict
    from basic_pitch import ICASSP_2022_MODEL_PATH
    import tempfile

    path = _validate_audio(file_path)
    if output_path is None:
        output_path = tempfile.mktemp(suffix=".mid", prefix="livepilot_midi_")

    model_output, midi_data, note_events = predict(path)

    midi_data.write(output_path)

    # Summarize
    notes = [n for inst in midi_data.instruments for n in inst.notes]
    pitches = [n.pitch for n in notes] if notes else [0]

    return {
        "midi_path": output_path,
        "note_count": len(notes),
        "pitch_range": {"low": min(pitches), "high": max(pitches)},
        "duration_seconds": midi_data.get_end_time(),
        "instruments": len(midi_data.instruments)
    }
```

### `extract_melody(ctx, file_path: str) -> dict`
Predominant melody line extraction. Returns: pitch curve, note sequence.

### `detect_pitch_issues(ctx, file_path: str) -> dict`
Flag notes that deviate >20 cents from nearest semitone.

## Tool Group 5: Structure (MSAF)

### `analyze_structure(ctx, file_path: str) -> dict`
Returns: section boundaries with timestamps, segment labels (intro/verse/chorus/bridge/outro), repetition structure.

### `compare_structure(ctx, file_a: str, file_b: str) -> dict`
Compare structural form of two tracks.

## Tool Group 6: Psychoacoustic (mosqito)

### `analyze_psychoacoustics(ctx, file_path: str) -> dict`
Returns: perceived loudness (sone), sharpness (acum), roughness (asper), fluctuation strength.

### `detect_harshness(ctx, file_path: str, threshold: float = 2.5) -> dict`
Returns: time ranges where sharpness exceeds threshold, with severity ranking.

## Tool Group 7: Mix Intelligence (compound)

### `diagnose_mix_problems(ctx, file_path: str) -> dict`
Full pipeline: Demucs → per-stem analysis → cross-stem comparison.
Returns ranked list of: frequency masking, phase issues, muddy low-mids, harsh highs, dynamic crush, stereo problems, loudness imbalance, timing drift. Each with severity score and specific fix suggestion.

### `suggest_mix_fixes(ctx, file_path: str) -> dict`
Builds on diagnose_mix_problems, returns actionable parameter suggestions tied to Ableton devices.

### `compare_to_reference(ctx, file_path: str, reference_path: str) -> dict`
Multi-dimensional A/B: loudness profile, spectral balance, dynamic range, stereo width, timbral similarity (MFCC distance), structural pacing.

## Tool Group 8: Fingerprinting (chromaprint)

### `fingerprint_audio(ctx, file_path: str) -> dict`
Generate perceptual fingerprint.

### `compare_audio_similarity(ctx, file_a: str, file_b: str) -> dict`
Similarity score (0-1) between two files.

### `diff_bounces(ctx, file_a: str, file_b: str) -> dict`
What changed between two mix versions — spectral diff, loudness diff, timbral diff.

## Registration

The module auto-registers with the MCP server by being imported in `server.py`:
```python
# Add to mcp_server/server.py imports:
from .tools import transport, tracks, clips, notes, devices, scenes
from .tools import mixing, browser, arrangement, memory, analyzer
from .tools import deep_analysis  # NEW — 1.7 perception tools
```

No other changes to server.py are needed. The `@mcp.tool()` decorator handles registration.

## Error Handling Pattern
Every tool should:
1. Validate the input file with `_validate_audio()`
2. Wrap the analysis in try/except
3. Return a `{"error": "message"}` dict on failure (don't raise — let the AI handle it)
4. Include timing info: `"processing_time_seconds": round(elapsed, 2)`

## Output Guidelines
- All numeric values: round to 1-2 decimal places
- Time values: always in seconds
- Frequency values: always in Hz
- Loudness values: always in dB or LUFS (specify which)
- Include a `"summary"` string field with a 1-2 sentence human-readable interpretation
- Example: `"summary": "Integrated loudness is -11.2 LUFS (loud for streaming, typical for club music). True peak at -0.3 dBTP risks inter-sample clipping."`
