# M4L Capture Device — Implementation Spec
## Files: `m4l_device/LivePilot_Capture.amxd` + `m4l_device/livepilot_capture.js`

## Purpose
A Max for Live audio effect device that captures audio from any track to a WAV file on disk, then notifies the MCP server via UDP so it can run deep analysis. This is the bridge between Ableton's audio and the Python analysis pipeline.

## Signal Flow
```
Track Audio In → plugin~ (tap audio) → record~ → buffer~ → writewave → disk
                                                                         ↓
                                                              UDP notify → MCP server
```

## Device Layout (350px width, matching existing Analyzer)
```
┌─────────────────────────────────────┐
│  LivePilot Capture                  │
│                                     │
│  [REC]  [STOP]    ○ Armed          │
│                                     │
│  Duration: 00:00   Format: WAV      │
│  Track: [auto-detect]               │
│  Output: ~/Documents/ComfyUI/...    │
│                                     │
│  Last: kick_20260319_143022.wav     │
└─────────────────────────────────────┘
```

## Max Patcher Objects
```
plugin~ 2          — stereo audio tap (non-destructive, passes through)
record~ capture 2  — records into buffer named "capture"
buffer~ capture    — stereo buffer, dynamically sized
writewave          — exports buffer to WAV file
udpsend 127.0.0.1 9881  — notify MCP server (same port as existing bridge)
```

## JavaScript Bridge (`livepilot_capture.js`)

### Key Functions
```javascript
// Auto-detect track name via LiveAPI
function getTrackName() {
    var api = new LiveAPI("this_device canonical_parent");
    return api.get("name").toString().replace(/[^a-zA-Z0-9_-]/g, "_");
}

// Generate filename: trackname_YYYYMMDD_HHMMSS.wav
function generateFilename() {
    var d = new Date();
    var timestamp = d.getFullYear().toString() +
        ("0" + (d.getMonth()+1)).slice(-2) +
        ("0" + d.getDate()).slice(-2) + "_" +
        ("0" + d.getHours()).slice(-2) +
        ("0" + d.getMinutes()).slice(-2) +
        ("0" + d.getSeconds()).slice(-2);
    return getTrackName() + "_" + timestamp + ".wav";
}

// Notify MCP server that a capture is ready
function notifyCapture(filePath, trackName, durationMs) {
    var msg = JSON.stringify({
        "type": "capture_ready",
        "file_path": filePath,
        "track_name": trackName,
        "duration_ms": durationMs,
        "sample_rate": 44100,
        "channels": 2
    });
    // Send via udpsend object connected to port 9881
    outlet(0, msg);
}
```

## MCP Server Side — Handling Capture Notifications

The existing `SpectralReceiver` in `m4l_bridge.py` already listens on UDP:9880. We need to handle a new message type `capture_ready` on port 9881 (or extend the existing receiver).

```python
# In m4l_bridge.py or a new capture_receiver.py
def handle_capture_notification(data: dict):
    """Called when M4L capture device notifies a new WAV is ready."""
    file_path = data["file_path"]
    track_name = data["track_name"]
    # Store in capture registry for the AI to reference
    capture_cache[track_name] = {
        "file_path": file_path,
        "timestamp": time.time(),
        "duration_ms": data["duration_ms"]
    }
```

## MCP Tool for Capture Control

### `capture_audio(ctx, track_name: str, duration_seconds: float) -> dict`
Tell the M4L device to start/stop recording. Sends command via UDP:9881 to the capture device.

### `list_captures(ctx) -> dict`
Return all available captures with metadata (track, duration, timestamp, file path).

### `analyze_capture(ctx, track_name: str) -> dict`
Convenience: get latest capture for a track → run full analysis pipeline.

## Output Directory
Default: `~/Documents/LivePilot/captures/`
Configurable via M4L device UI or MCP tool parameter.

## Important Notes
- The device passes audio through unchanged (plugin~ → plugout~). It's non-destructive.
- Buffer size is dynamically allocated based on record duration.
- WAV format: 44100 Hz, 24-bit, stereo (matching Ableton's output).
- The device should work on any track (audio, return, master) — not just master bus.
- Multiple instances can run on different tracks simultaneously.
