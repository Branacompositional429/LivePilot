# LivePilot 1.7→2.0 — Full Development Roadmap

## What This Project Covers
LivePilot expansion from 135 tools (v1.6.3) to ~200 tools (v2.0), across four versions:
- **1.7 Perception Layer** — 16 new audio analysis tools (ears)
- **1.8 Theory + Productivity** — 12 tools (music theory brain + algorithmic composition + file tagging)
- **1.9 Processing + Discovery** — ~12 tools (audio processing, sample search, reference mastering, API integrations)
- **2.0 Generation Layer** — ~8 tools (text-to-music, sound morphing, style transfer)

## Design Principles (apply to ALL versions)

### Tools vs LLM Knowledge
MCP tools provide **computational precision** on real data. The LLM provides
**interpretive depth** from training. Skills tell the agent **when to engage which**.
These stack — they don't compete.

**Make it a tool when:** The task requires reading/writing real data (notes, files,
parameters), running algorithms (FFT, Krumhansl key detection, counterpoint rules),
or producing deterministic results the LLM can't compute reliably in context.

**Make it a skill instruction when:** The task is pure reasoning, explanation, or
creative judgment that the LLM already handles well from training data. Education,
style advice, "why does this chord work" questions — these are LLM strengths.
Wrapping them in MCP tools adds latency and overhead for zero capability gain.

**Never duplicate training knowledge in tools.** A tool that returns
"a ii-V-I is the most common jazz progression" wastes a round-trip. The LLM
already knows that. Tools should return *data* ("this clip contains a ii-V-I
at beat 4.0 in Bb major"), and the LLM interprets it.

### Speed Tier Discipline
Every tool has a speed tier (Instant/Fast/Slow/Heavy). Heavy tools require
user consent. The agent escalates from fast to heavy, never skips levels.
See the 1.7 spec for the full escalation protocol.

## Spec Documents
- **`docs/livepilot-1.7-spec.md`** — v1.7 implementation spec (architecture, 16 tool
  signatures, code, installation, AI confidence ratings, build order). Single source of truth for 1.7.
- **`docs/livepilot-1.8-spec.md`** — v1.8 implementation spec (music21 theory, isobar
  composition, mutagen tagging — 12 tools, ~110 MB, no GPU, no API keys).
- **`docs/beyond-perception-research.md`** — v1.9→2.0 roadmap research (audio processing,
  sample search, text-to-music, sound morphing). Reference material for future versions.

## Current Phase: v1.7 Perception Layer

### Architecture (Three Layers)
```
Layer A: M4L Real-Time (FluCoMa only)      → 8 tools, zero Python (zsa.roughness cut)
Layer B: Light Python (librosa, pyloudnorm) → 5 tools, ~80 MB, no GPU (msaf/structure cut)
Layer C: Heavy Python (Demucs, basic-pitch) → 3 tools, ~3 GB, GPU optional
```
**Deferred to 1.7.1:** LivePilot_Capture.amxd (bounce-to-analysis pipeline)
Each layer works independently. Build in order: A → B → C.

### Existing Codebase Structure
```
mcp_server/
├── tools/
│   ├── analyzer.py          ← Real-time perception (modify for Layer A)
│   ├── device_atlas.py      ← Device control tools
│   └── technique_memory.py  ← Technique storage
├── server.py                ← FastMCP server, TCP:9878
├── m4l_bridge.py            ← UDP:9880/9881 bridge to M4L
m4l/
├── LivePilot_Analyzer.amxd  ← Existing analyzer (modify for FluCoMa)
bin/
├── livepilot.js             ← Node installer (modify for setup --all/--light, doctor)
```

### New Files for 1.7.0
```
mcp_server/tools/deep_analysis.py   ← Layer B + C tools (all Python analysis)
pyproject.toml                       ← Replaces requirements.txt (NEW)
```

### New Files for 1.8.0
```
mcp_server/tools/theory.py          ← 7 music21 tools + _notes_to_stream bridge
mcp_server/tools/composition.py     ← 3 algorithmic composition tools
mcp_server/tools/productivity.py    ← 2 mutagen tagging tools
```

### Deferred to 1.7.1
```
m4l_device/LivePilot_Capture.amxd   ← Audio capture device
m4l_device/livepilot_capture.js     ← JS bridge for capture
```

### Tool Registration Pattern
```python
from ..server import mcp

@mcp.tool()
def tool_name(ctx: Context, file_path: str) -> dict:
    """Docstring becomes the tool description."""
    import librosa  # ALWAYS lazy import — never at module level
    path = _validate_audio(file_path)
    return {"key": "value", "interpretation": "Human-readable summary"}
```

## Key Constraints
- **Lazy imports:** Never import torch/librosa/demucs at module level
- **File validation:** Every tool must validate file exists and is .wav/.mp3/.flac
- **Return summaries:** Include an "interpretation" key, don't dump raw arrays
- **GPU awareness:** `_get_device()` checks CUDA → MPS → CPU fallback
- **Subprocess for Demucs:** Use CLI call, not direct import (GPU memory management)
- **Error handling:** Every tool returns `_tool_error()` on failure, never raw exceptions
- **Concurrency:** All Layer B/C tools run in `ThreadPoolExecutor(max_workers=2)`, GPU ops use `gpu_lock`
- **Output paths:** All outputs go to `~/Documents/LivePilot/outputs/{category}/`
- **Testing:** `livepilot test` validates tools against reference audio in `tests/fixtures/`

## Dependencies (v1.7 — No Essentia, No CREPE, No TensorFlow, No msaf, No zsa)
- **Layer A (M4L):** FluCoMa package only (Max Package Manager)
- **Layer B (--light):** librosa, pyloudnorm, soundfile, scipy
- **Layer C (--all):** + torch, torchaudio, demucs, basic-pitch
- **Package manager:** uv (Rust-based, `--torch-backend=auto` for GPU detection)

## Build Priority (v1.7)
1. Phase 1: FluCoMa tools in M4L device (zero Python risk, 8 tools)
2. Phase 2: pyloudnorm + librosa tools (~80 MB, no GPU, 5 tools)
3. Phase 3: Demucs + basic-pitch tools (~3 GB, GPU optional, 3 tools)
4. Phase 4 (1.7.1): Capture device + diagnose_mix pipeline

## Full Roadmap: v1.7 → v2.0

### v1.8.0 — Theory Intelligence + Productivity (12 tools, ~110 MB, no GPU)
**Libraries:** music21, isobar, mutagen
**Domains:**
- Music theory (harmony analysis, voice leading, counterpoint, harmonization, scale ID, smart transpose)
- Algorithmic composition (Markov chains, L-systems, arpeggiation, variation)
- Productivity (auto-tag bounces with BPM/key/metadata)
**Key insight:** Theory tools work on live session clips via get_notes — no bouncing needed.
Zero Heavy tools, everything Instant or Fast. No API keys, no OAuth, no network deps.
**Cut from original plan:** Spotify (OAuth), Freesound (OAuth), pedalboard-pluginary (fragile),
matchering (one-trick), education tools (converted to skill instructions instead).

### v1.9 — Processing + Discovery Layer (~12 tools, API keys needed for some)
**Libraries:** pedalboard, spotipy, freesound-python, matchering
**Domains:**
- Headless audio processing (batch effects, VST3 hosting without GUI) — pedalboard
- Reference track lookup (BPM, key, mood from Spotify) — spotipy (requires API key)
- Sample search & download (600K+ CC samples) — freesound-python (requires API key)
- Reference mastering (match EQ/loudness to reference) — matchering
**Note:** Spotify and Freesound require OAuth/API keys. This is the first version
with external service dependencies. Keep them optional — all other tools work without.
**Items moved here from 1.8:** Spotify API, Freesound API, matchering.

### v2.0 — Generation Layer (~8 tools, GPU required)
**Libraries:** MusicGen/Stable Audio, RAVE, potentially Groove2Groove
**Domains:**
- Text-to-music generation (MusicGen, Stable Audio Open)
- Sound morphing & timbre transfer (RAVE, real-time on CPU)
- Style transfer if Groove2Groove stabilizes
- Whatever new models exist by then
**Items cut permanently:** RVC voice cloning (ethical issues), MidiDrumiGen (FastAPI sidecar, fragile),
pedalboard-pluginary (poorly maintained plugin scanning).

### Tool Count Projection
| Version | New Tools | Cumulative | Dependencies |
|---------|:---------:|:----------:|-------------|
| 1.6.3 (current) | — | 135 | fastmcp |
| 1.7.0 Perception | +16 | 151 | + FluCoMa, librosa, pyloudnorm, demucs, basic-pitch |
| 1.7.1 Capture pipeline | +2 | 153 | + M4L capture device |
| 1.8.0 Theory + Productivity | +12 | 165 | + music21, isobar, mutagen |
| 1.9 Processing + Discovery | ~12 | ~177 | + pedalboard, spotipy, freesound, matchering |
| 2.0 Generation | ~8 | ~185 | + MusicGen/RAVE (GPU) |

## Cross-Platform
- Windows: RTX 5090 (CUDA) — `--torch-backend=auto` picks CUDA wheels
- macOS: M3 Max (MPS) — `--torch-backend=auto` picks MPS wheels
- Both: FluCoMa and all lightweight Python libs work natively

## Engineering Details (in the spec doc)
The spec includes six "Gap Fix" sections covering: testing strategy with reference
audio files, standardized error handling, capture→analysis server-side handler,
concurrency model (thread pool + GPU lock), output path conventions, and the .amxd
build approach (Max `js` scripting for FluCoMa patching).

## Archive Folder
`archive/` contains the earlier research documents from the analysis phase.
These are reference material showing the research journey. The final consolidated
specs in `docs/` supersede all archive documents.
