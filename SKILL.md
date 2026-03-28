---
name: yt-hardcoded-subtitle-extractor
description: >
  Extract hardcoded (burned-in) subtitles from YouTube videos as text and save the
  corresponding audio clips as files. Use this skill whenever the user wants to:
  extract subtitles that are baked into the video frames (not soft CC/caption tracks),
  capture dialogue audio clips paired with their subtitle text, process Hokkien, Taiwanese,
  Cantonese, or other Chinese-language drama videos where subtitles are hardcoded in
  Traditional Chinese characters, or build a subtitle+audio dataset from YouTube content.
  Trigger on phrases like "extract subtitles from video", "get subtitles and audio", "subtitle
  audio extract", "hardcoded subtitles", "burned-in captions", or any mention of saving
  audio clips paired with what characters say on screen.
---

# YouTube Hardcoded Subtitle + Audio Extractor

This skill extracts hardcoded (burned-in) subtitles from YouTube videos using the Chrome browser
connector and vision OCR, then records the corresponding audio clips for each subtitle using the
Web Audio API. Validated on Hokkien/Taiwanese drama content (Traditional Chinese characters).

---

## Architecture Overview

**Why this approach (and not yt-dlp):**
The Cowork VM sandbox routes internet through a managed proxy — Python/yt-dlp can't resolve
YouTube hostnames directly. Chrome CAN stream YouTube through that proxy. So the correct
architecture is: **Chrome streams → Web Audio API records segments → browser downloads to
~/Downloads → copy to workspace**.

```
YouTube Video (in Chrome)
       │
       ├─── Video frames ──► zoom() screenshot ──► vision OCR ──► subtitle text + timestamp
       │
       └─── Audio stream ──► Web Audio API ──────► MediaRecorder ──► .webm clips ──► workspace
```

---

## Step-by-Step Workflow

### Phase 1: Open and Buffer the Video

1. Get (or create) a Chrome tab:
   ```
   tabs_context_mcp(createIfEmpty=true)
   navigate(url="https://www.youtube.com/watch?v=VIDEO_ID")
   ```

2. Click the play button once (center of video area) to initialise the player, wait 4–5 seconds.

3. Use JS to verify the video is ready — seek to a mid-point to trigger buffering:
   ```js
   const video = document.querySelector('video');
   video.currentTime = 120;
   // Then check: video.readyState should reach 4 (HAVE_ENOUGH_DATA)
   ```
   Wait 3–5 more seconds after seeking.

4. Pause the video:
   ```js
   video.pause();
   ```

---

### Phase 2: Scan for Subtitles (Vision OCR)

Subtitles in Taiwanese/Hokkien dramas appear in the **bottom 20–25% of the video frame**,
centred, with large white text and black outline.

**Scanning loop — for each candidate timestamp:**

1. Seek via JS:
   ```js
   const video = document.querySelector('video');
   video.currentTime = TARGET_SECONDS;
   video.pause();
   ```
   Wait 2 seconds for the frame to render.

2. Zoom into the subtitle region (adjust pixel bounds to match the video player position on screen):
   ```
   zoom(region=[video_left, video_bottom_30pct, video_right, video_bottom])
   ```
   Typical values when the YT player fills the left ~70% of a 1440px-wide screen:
   `region=[118, 580, 1000, 735]`

3. Read the subtitle from the zoomed image using vision. If text is present, record:
   `{timestamp_s: 195, text: "你害我爸把土地賣掉"}`
   If no text is visible, note "silent/action" and skip.

**Recommended scan cadence:** every 3–4 seconds for dialogue-heavy scenes.
Skip timestamps where previous frames showed action (motion blur) or silence.

**Example subtitle region zoom coordinates** (1461px-wide Chrome window, YT in inline player):
- Video frame occupies approximately x: 118–1000, y: 63–735
- Subtitle zone: `region=[118, 580, 1000, 735]` (bottom ~20% of player)

---

### Phase 3: Set Up Web Audio Capture

Run this **once** before recording any clips. It wires the video element into the Web Audio API:

```js
// Run once after buffering
const video = document.querySelector('video');
const audioCtx = new AudioContext();
const source = audioCtx.createMediaElementSource(video);
const dest = audioCtx.createMediaStreamDestination();
source.connect(dest);
source.connect(audioCtx.destination); // keep speakers active so audio plays during recording
window._audioCapture = { video, dest, audioCtx, clips: [] };
// Verify: audioCtx.state should be "running"
```

---

### Phase 4: Record Audio Clips (Batch)

Use this self-contained async function to record all subtitle clips sequentially and
auto-download each one to the user's browser Downloads folder:

```js
// Paste this entire block into javascript_exec
window._audioCapture.allDone = false;
window._audioCapture.allClips = [];

// Define the subtitle list — [{time: seconds, label: "safe_filename_text"}, ...]
const subtitles = [
  {time: 195, label: '你害我爸把土地賣掉'},
  {time: 198, label: '你先聽我解釋好嗎'},
  // ... add all collected subtitles here
];

async function recordClip(startTime, duration, label) {
  return new Promise((resolve) => {
    const {video, dest} = window._audioCapture;
    video.currentTime = startTime;
    const recorder = new MediaRecorder(dest.stream, {mimeType: 'audio/webm;codecs=opus'});
    const chunks = [];
    recorder.ondataavailable = e => chunks.push(e.data);
    recorder.onstop = () => {
      const blob = new Blob(chunks, {type: 'audio/webm'});
      // Trigger browser download
      const url = URL.createObjectURL(blob);
      const a = document.createElement('a');
      a.href = url;
      a.download = `subtitle_${startTime}s_${label}.webm`;
      document.body.appendChild(a);
      a.click();
      document.body.removeChild(a);
      setTimeout(() => URL.revokeObjectURL(url), 1000);
      resolve({label, size: blob.size, time: startTime});
    };
    video.addEventListener('seeked', function onSeeked() {
      video.removeEventListener('seeked', onSeeked);
      recorder.start();
      video.play();
      setTimeout(() => { video.pause(); recorder.stop(); }, duration * 1000);
    }, {once: true});
  });
}

async function recordAll() {
  for (const s of subtitles) {
    const result = await recordClip(s.time, 3, s.label);  // 3 seconds per clip
    window._audioCapture.allClips.push(result);
    await new Promise(r => setTimeout(r, 500)); // brief pause between clips
  }
  window._audioCapture.allDone = true;
}

recordAll();
'Recording started — check window._audioCapture.allDone to monitor progress'
```

**Timing:** each clip takes ~3.5s to record. For N clips, wait N × 4 seconds.
Poll with: `window._audioCapture.allDone` and `window._audioCapture.allClips.length`

---

### Phase 5: Move Files to Workspace

The .webm files land in the user's `~/Downloads`. Mount it and copy:

```python
# 1. Request Downloads folder access
request_cowork_directory(path="~/Downloads")

# 2. Copy to workspace
import shutil, glob
clips = glob.glob("/sessions/.../mnt/Downloads/subtitle_*.webm")
for clip in clips:
    shutil.copy(clip, "/sessions/.../mnt/Subtitle Audio Extract/audio_clips/")
```

Or via Bash:
```bash
mkdir -p "/path/to/workspace/audio_clips"
cp ~/Downloads/subtitle_*.webm "/path/to/workspace/audio_clips/"
```

---

### Phase 6: Save the Subtitle Index

Write a CSV alongside the audio files:

```
timestamp_s,subtitle_text,audio_file,notes
195,你害我爸把土地賣掉,subtitle_195s_你害我爸把土地賣掉.webm,You caused my father to sell the land
198,你先聽我解釋，好嗎,subtitle_198s_你先聽我解釋好嗎.webm,Please listen first okay?
...
```

---

## Output Format

```
Subtitle Audio Extract/
├── subtitles_ep<N>_<scene>.csv       ← subtitle index with timestamps
└── audio_clips/
    ├── subtitle_195s_你害我爸把土地賣掉.webm
    ├── subtitle_198s_你先聽我解釋好嗎.webm
    └── ...
```

Each .webm file is ~48KB for a 3-second Opus-encoded clip (excellent quality, small size).

---

## Key Technical Constraints & Gotchas

| Issue | Cause | Solution |
|-------|-------|----------|
| yt-dlp fails with proxy/DNS errors | VM sandbox routes only browser traffic | Use Chrome + Web Audio API instead |
| `video.duration` is `NaN` at first | Video not yet buffered | Seek to mid-point, wait for `readyState == 4` |
| `video.src` is a `blob:` URL | YouTube uses MediaSource Extensions | Can't download directly; use Web Audio API |
| Base64 data blocked in JS return | Security sandboxing in Chrome extension | Use browser download (anchor click) instead |
| Audio Context state `suspended` | Browser autoplay policy | Click or interact with page first |
| Subtitles not visible at some timestamps | Silent/action scene, or mid-scene cut | Skip and try ±2s offsets |
| Duplicate downloads (e.g. `clip (1).webm`) | Test clip + batch clip for same timestamp | Keep the one without ` (1)` suffix |

---

## Tips for Hokkien/Taiwanese Dramas

- Subtitle font: large, bold, white with black outline — highly readable via vision OCR
- Typical subtitle duration: 2–4 seconds per line
- Subtitle position: always bottom-centre of frame, ~15–20% from bottom
- Dialogue-heavy scenes: scan every 3s; action/music scenes: scan every 8–10s
- Character names are often shouted as 1-2 character subtitles (e.g. "世傑") — these are valid entries
- The "訂閱" (subscribe) badge in the corner is NOT a subtitle — ignore it
- Some episodes have a brief title card at the start with no dialogue — skip the first 60–90s
