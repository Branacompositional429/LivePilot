# Installer Spec: `pyproject.toml` + `bin/livepilot.js` Changes

## Overview
Extend the existing `livepilot.js` Node installer to handle the full analysis stack
via `uv` (Rust-based Python package manager). Three commands:
- `livepilot setup` — base MCP server (unchanged)
- `livepilot setup --all` — base + full analysis stack (GPU models included)
- `livepilot setup --light` — base + analysis libs only (no GPU models)

---

## File 1: `pyproject.toml` (NEW — replaces requirements.txt)

```toml
[project]
name = "livepilot"
version = "1.7.0"
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

**Notes:**
- `analysis` group is lightweight (~100 MB), safe on both platforms
- `separation` pulls PyTorch (~2 GB) — `--torch-backend=auto` picks CUDA or MPS
- `pitch` pulls TensorFlow (~1.5 GB) — optional, skip to save space
- `all` installs everything

---
## File 2: `bin/livepilot.js` Changes

### New function: `ensureUv()`

```javascript
async function ensureUv() {
    try {
        await exec("uv --version");
        return;
    } catch {
        console.log("Installing uv (fast Python package manager)...");
        if (process.platform === "win32") {
            await exec('powershell -ExecutionPolicy ByPass -c "irm https://astral.sh/uv/install.ps1 | iex"');
        } else {
            await exec('curl -LsSf https://astral.sh/uv/install.sh | sh');
        }
    }
}
```

### New function: `setupAnalysis(flags)`

```javascript
async function setupAnalysis(flags = {}) {
    await ensureUv();

    // Determine install group based on flags
    const group = flags.light ? "analysis,transcription" : "all";
    console.log(`Installing [${group}] dependencies...`);

    // uv creates/reuses venv and installs in one command
    // --torch-backend=auto picks CUDA on Windows, MPS on Mac
    await exec(`uv pip install -e ".[${group}]" --torch-backend=auto`, {
        cwd: ROOT,
        env: { ...process.env, UV_PROJECT_ENVIRONMENT: VENV_DIR }
    });

    // Pre-download AI models so first analysis doesn't hang
    if (!flags.light) {
        console.log("Pre-downloading AI models...");
        await exec(`${pythonBin} -c "
import demucs.pretrained; demucs.pretrained.get_model('htdemucs')
from basic_pitch.inference import predict  # triggers model download
print('Models ready.')
        "`);
    }

    // Generate lock file for reproducibility
    console.log("Saving dependency snapshot...");
    await exec(`uv pip freeze > requirements-lock.txt`, { cwd: ROOT });

    console.log("Analysis stack ready.");
}
```

### Modified `setup` command (flag parsing)

```javascript
// In the main CLI argument parser:
case "setup":
    const flags = {
        all: args.includes("--all"),
        light: args.includes("--light"),
    };

    // Existing base setup
    await setupBase();

    // If --all or --light, also install analysis stack
    if (flags.all || flags.light) {
        await setupAnalysis(flags);
    }
    break;
```

---

### New command: `livepilot doctor`

```javascript
case "doctor":
    await runDoctor();
    break;

// ...

async function runDoctor() {
    const checks = [
        { name: "Python", cmd: `${pythonBin} --version` },
        { name: "venv", cmd: `test -d ${VENV_DIR}` },
        { name: "FastMCP", cmd: `${pythonBin} -c "import fastmcp; print(fastmcp.__version__)"` },
    ];

    const analysisLibs = [
        { name: "librosa", pkg: "librosa" },
        { name: "pyloudnorm", pkg: "pyloudnorm" },
        { name: "demucs", pkg: "demucs" },
        { name: "basic-pitch", pkg: "basic_pitch" },
        { name: "mosqito", pkg: "mosqito" },
        { name: "torch", pkg: "torch" },
        { name: "crepe", pkg: "crepe", optional: true },
        { name: "essentia", pkg: "essentia", optional: true },
    ];

    console.log("LivePilot Health Check");
    console.log("━━━━━━━━━━━━━━━━━━━━━");

    // Check core
    for (const check of checks) {
        try {
            const result = await exec(check.cmd);
            console.log(`${check.name}: ${result.trim()}  ✅`);
        } catch {
            console.log(`${check.name}: NOT FOUND  ❌`);
        }
    }

    // Check analysis libs
    console.log("\nAnalysis Stack:");
    for (const lib of analysisLibs) {
        try {
            const ver = await exec(
                `${pythonBin} -c "import ${lib.pkg}; print(getattr(${lib.pkg}, '__version__', 'installed'))"`
            );
            console.log(`  ${lib.name}: ${ver.trim()}  ✅`);
        } catch {
            const icon = lib.optional ? "⏭️  (optional)" : "❌";
            console.log(`  ${lib.name}: —  ${icon}`);
        }
    }

    // GPU check
    try {
        const gpu = await exec(
            `${pythonBin} -c "
import torch
if torch.cuda.is_available():
    print(f'GPU: {torch.cuda.get_device_name(0)} (CUDA {torch.version.cuda})')
elif hasattr(torch.backends, 'mps') and torch.backends.mps.is_available():
    print('GPU: Apple Silicon (MPS)')
else:
    print('GPU: None (CPU only)')
"`
        );
        console.log(`\n${gpu.trim()}`);
    } catch {
        console.log("\nGPU: Could not detect (torch not installed)");
    }

    // Check cached models
    try {
        await exec(`${pythonBin} -c "
import os, pathlib
torch_hub = pathlib.Path.home() / '.cache' / 'torch' / 'hub' / 'checkpoints'
demucs_cached = any(f.name.startswith('htdemucs') for f in torch_hub.glob('*')) if torch_hub.exists() else False
print(f'HTDemucs model: {\"cached ✅\" if demucs_cached else \"not cached ⚠️\"}')"
        `);
    } catch {
        // skip model check if torch not available
    }
}
```

---

## Expected `livepilot doctor` Output

```
LivePilot Health Check
━━━━━━━━━━━━━━━━━━━━━
Python: 3.11.8  ✅
venv: .venv  ✅
FastMCP: 3.2.1  ✅

Analysis Stack:
  librosa: 0.10.2  ✅
  pyloudnorm: 0.1.1  ✅
  demucs: 4.0.1  ✅
  basic-pitch: 0.3.5  ✅
  mosqito: 1.2.0  ✅
  torch: 2.3.0  ✅
  crepe: —  ⏭️  (optional)
  essentia: —  ⏭️  (optional)

GPU: NVIDIA RTX 5090 (CUDA 12.4)
HTDemucs model: cached ✅
```

---

## Maintenance Notes

### Quarterly Routine (30 min)
```bash
uv pip list --outdated              # Check what's stale
uv pip audit                        # Security check
uv venv .venv-test                  # Test in throwaway venv
uv pip install -e '.[all]' --torch-backend=auto --upgrade
python -c "from mcp_server.tools.deep_analysis import *; print('OK')"
# If OK → update real venv. If not → keep current versions.
```

### Version Pinning Strategy
- **Day-to-day:** Range pins in pyproject.toml (`>=X.Y,<X+1.0`)
- **Safety net:** `requirements-lock.txt` (exact freeze after verified install)
- **Recovery:** `uv pip install -r requirements-lock.txt` to restore last-known-good

### Skip Candidates (to reduce complexity)
- **Essentia** — Windows build complexity; librosa covers 90% of same ground
- **CREPE** — pulls TensorFlow (~1.5 GB); basic-pitch handles transcription already
- Dropping both saves ~2 GB and removes TensorFlow dependency entirely
