---
name: video-analysis
description: Use when the user gives you a video, screen recording, GIF, or screencast to watch, review, debug, or transcribe. Builds a contact sheet for instant overview, extracts deduplicated full-resolution frames, crops and upscales unreadable UI text, runs OCR, and transcribes audio.
---

# Video & Screen Recording Analysis

## Why this exists

You have no native video input. Any video you "watch" is really a set of still
frames plus (optionally) an audio transcript. Default handling samples too few
frames and downscales them, which destroys small UI text, terminal output, and
chart labels. This skill replaces that with an explicit, controlled pipeline.

**Never claim you watched a video. You read frames from it. Say that.**

The goal is to make a video feel as easy to work with as an attached photo. The
way you get there is the **three-tier read** in Step 3: one image for the whole
timeline, then drill in only where it matters.

## Requirements

**This skill needs a shell.** It works wherever you can run commands (Claude
Code, Cowork, any environment with code execution). In a chat surface with no
shell, none of this is available — say so plainly rather than half-attempting it.

`ffmpeg` and `ffprobe` are required. `tesseract` (OCR), `faster-whisper`
(transcription) and `opencv-python` (auto-crop) are optional but strongly
recommended. Check first:

```bash
which ffmpeg ffprobe tesseract
python3 -c "import cv2" 2>/dev/null && echo "opencv ok"
```

Install if missing (add `sudo` if not root):

```bash
apt-get install -y ffmpeg tesseract-ocr     # Debian/Ubuntu
brew install ffmpeg tesseract               # macOS
pip install opencv-python                   # for auto-crop
```

If `ffmpeg` is unavailable and cannot be installed, stop and tell the user. Do
not attempt to analyze a video without it.

## Step 0 — Get the file somewhere you can render frames

Frames must be written where your image-reading tool can render them. If the
video lives on a remote or mounted filesystem your reader cannot render from,
copy it into your working environment first. If it is not reachable at all, ask
the user to attach it rather than guessing at its contents.

Work in a scratch directory, never the user's project directory.

## Step 1 — Probe before anything else

```bash
ffprobe -v error -show_entries stream=codec_type,codec_name,width,height,r_frame_rate,nb_frames -show_entries format=duration,size -of default=nw=1 INPUT
```

Decide from this:

- **Duration** sets the frame budget (Step 2).
- **Resolution** — if height > 1440, text is likely legible; if < 900, warn the
  user that small text may be unreadable and ask them to re-record larger.
- **No audio stream** → skip Step 4 entirely. Do not announce a missing feature.
- **Over ~200MB** → ask for a trimmed clip before copying it anywhere slow.

## Step 2 — Extract frames

### Default: `mpdecimate` — drop duplicate frames

```bash
mkdir -p frames
ffmpeg -y -loglevel error -i INPUT -vf "mpdecimate" -fps_mode vfr frames/f_%03d.png
```

This keeps only frames that meaningfully differ from the one before. On screen
recordings — which are mostly static — it lands almost exactly on the moments
something changed.

### Do NOT use scene detection on screen recordings

`select='gt(scene,0.15)'` is the common advice and it is **wrong here**.
`scene` scores whole-frame change, so a recording where only a text line or a
panel changes against a static background scores near zero.

Measured on a 1080p screen recording with four completely different screens:

```
lavfi.scene_score = 0.001
```

That is ~150x below the usual 0.15 threshold. Scene detection returned **0
frames**; `mpdecimate` returned exactly **4** — one per screen. Scene detection
is for real video with cuts. For screen capture, use `mpdecimate`.

### Get the timestamps

```bash
ffmpeg -loglevel info -i INPUT -vf "mpdecimate,showinfo" -fps_mode vfr -f null - 2>&1 \
  | grep -oE "pts_time:[0-9.]+"
```

Frame N of the extraction corresponds to timestamp N in this list. **Always
report timestamps** so the user can jump to the moment in their own player.

### If `mpdecimate` returns too many or too few

- **Flood of frames** (video plays continuously, cursor always moving) → raise
  the threshold: `mpdecimate=hi=64*16:lo=64*8:frac=0.5`
- **Too few** → lower it: `mpdecimate=hi=64*4:lo=64*2:frac=0.01`
- **Continuous-motion video** (not screen capture) → fixed interval instead:
  `-vf "fps=1/N"` with `N` = duration_seconds / 25, minimum 1.

### Cap the count on long recordings

Budget roughly **15–40 frames**. Over ~50 burns context for little gain, and
`mpdecimate` does not respect a budget on its own — a 10-minute recording
produced **100 frames** in testing.

Count first, then thin to target in a second pass:

```bash
# 1. how many would we get?
N=$(ffmpeg -loglevel error -i INPUT -vf mpdecimate -fps_mode vfr -f null - 2>&1; \
    ffmpeg -y -loglevel error -i INPUT -vf mpdecimate -fps_mode vfr /tmp/_c_%04d.png && ls /tmp/_c_*.png | wc -l)

# 2. keep every Kth, where K = ceil(N / 30)
ffmpeg -y -loglevel error -i INPUT \
  -vf "mpdecimate,select='not(mod(n\,K))'" -fps_mode vfr frames/f_%03d.png
```

Verified: 100 raw frames with `K=4` yields 25 — inside budget, still evenly
covering the whole recording.

### Notes

- **Do not pass `-q:v` with PNG output.** PNG is lossless; the flag is a no-op.
  (Verified: identical byte size with and without it.)
- Use `-fps_mode vfr`, not the deprecated `-vsync vfr`.
- **Never downscale during extraction.** Keep full-resolution frames on disk;
  downscaling for overview happens in Step 3, where you control it.

## Step 3 — The three-tier read

This is the part that makes video feel like a photo. **Do not read 30 frames
one at a time.**

### Tier 1 — Contact sheet (one image, whole timeline)

Tile the frames into a single grid and read *that* first:

```bash
ffmpeg -y -loglevel error -i INPUT \
  -vf "mpdecimate,scale=375:-1,tile=4x3:margin=8:padding=6:color=0x333333" \
  -fps_mode vfr frames/sheet_%02d.png
```

One read now gives you the entire arc of the recording — what screens appear,
in what order, roughly when. Use it to decide where to look closely.

**Size the cells so the finished sheet lands near 1500px wide.** Your reader
downscales anything larger anyway, so a bigger sheet costs file size and buys
nothing:

| Grid | `scale=` per cell |
|---|---|
| `tile=2x2` | 750 |
| `tile=3x2` | 500 |
| `tile=4x3` | 375 |
| `tile=5x4` | 300 |

Pick the grid from the frame count: `tile=3x2` for ~6, `tile=4x3` for ~12,
`tile=5x4` for ~20. Emit multiple sheets if it overflows.

**Whether text survives depends on grid density.** At `2x2` with 750px cells,
16px UI text is often still readable. At `4x3` and denser it is not, and OCR on
the sheet returns nothing usable. Treat the sheet as **navigation**: use it to
find the moment, then confirm any text you intend to quote against the
full-resolution frame in Tier 2. Never quote from the sheet alone.

### Tier 2 — Full-resolution single frame

Once the contact sheet tells you which moment matters, read that frame at full
resolution:

```
frames/f_007.png
```

At 1080p this is often enough on its own — full-res frames OCR cleanly even at
16px text.

### Tier 3 — Crop and upscale

If the text is still too small (common at 4K, where downscaling to ~1568px on
the long edge shrinks 14px text to about 5px), crop the region and upscale
before reading:

**Crop the extracted frame file, not the video.** You already know which frame
you want, so re-running the whole video to crop every frame is wasted work:

```bash
# crop=W:H:X:Y  then 2x lanczos upscale
ffmpeg -y -loglevel error -i frames/f_007.png \
  -vf "crop=1100:50:50:285,scale=iw*2:ih*2:flags=lanczos" \
  frames/crop_007.png
```

To find coordinates: read one full frame, estimate the region, then crop. A
second pass is cheaper than guessing at a blurry screenshot.

### Better: let the frames tell you where to crop

Don't eyeball the coordinates — compute them. The region that **changed**
between two consecutive frames is almost always the region that matters, and
`opencv-python` will hand you its bounding box:

```python
import cv2, numpy as np, subprocess

a = cv2.imread('frames/f_001.png', 0)
b = cv2.imread('frames/f_002.png', 0)

diff = cv2.absdiff(a, b)
_, th = cv2.threshold(diff, 25, 255, cv2.THRESH_BINARY)
th = cv2.dilate(th, np.ones((15, 15), np.uint8), iterations=2)

cnts, _ = cv2.findContours(th, cv2.RETR_EXTERNAL, cv2.CHAIN_APPROX_SIMPLE)
if cnts:
    x, y, w, h = cv2.boundingRect(np.vstack(cnts))
    subprocess.run(['ffmpeg', '-y', '-loglevel', 'error', '-i', 'frames/f_002.png',
                    '-vf', f'crop={w}:{h}:{x}:{y},scale=iw*2:ih*2:flags=lanczos',
                    'frames/auto_002.png'])
    print(f'cropped changed region: {w}x{h} at ({x},{y})')
else:
    print('no change detected between these frames')
```

This turns "I think the error is somewhere in the upper left" into an exact
crop. Verified: on a 1080p frame pair it located a 595x260 region at (51,66),
and OCR of the auto-crop returned the full error string cleanly.

Tune the `25` threshold up if a noisy video or video player background produces
a bbox covering the whole frame; raise the dilation kernel to merge text that
comes back as many small boxes.

### Narrow the window when the user names a moment

"the error at the end", "around 1:20" — extract densely from just that range.
This is the single biggest quality win available:

```bash
ffmpeg -y -loglevel error -ss 75 -t 15 -i INPUT -vf "fps=2" frames/zoom_%03d.png
```

## Step 3b — OCR as a cross-check

Cheap, and it often reads text you cannot:

```bash
for f in frames/f_*.png; do echo "--- $f"; tesseract "$f" stdout 2>/dev/null; done
```

OCR is a hint, not ground truth. It mangles code punctuation, `l`/`1`/`I`, and
`0`/`O`. Cross-check anything load-bearing against the frame itself. If OCR and
the frame disagree, say so rather than silently picking one.

## Step 4 — Audio (only if an audio stream exists)

```bash
ffmpeg -y -loglevel error -i INPUT -vn -ac 1 -ar 16000 audio.wav
pip install faster-whisper -q || pip install faster-whisper --break-system-packages -q
python3 -c "
from faster_whisper import WhisperModel
m = WhisperModel('base', device='cpu', compute_type='int8')
segs, info = m.transcribe('audio.wav', vad_filter=True)
for s in segs: print(f'[{s.start:.1f}-{s.end:.1f}] {s.text.strip()}')
"
```

Use `base` by default; `small` if the audio is noisy or accented. Weights
download on first use — about a minute. Do not do this for a silent screen
recording.

## Step 5 — Report

1. **What the recording shows**, start to finish, with timestamps.
2. **The specific thing the user asked about**, quoted from a full-resolution
   frame or OCR — never from the contact sheet.
3. **What you could not make out**, named explicitly. Never paper over a blurry
   frame with a plausible guess.

If the frames were too low-resolution to answer, say so and tell the user how to
re-record it rather than guessing.

## Recording advice to give the user

- Record the window, not the whole desktop.
- Zoom the app to 150% before recording (browser: `Ctrl`/`Cmd` + `+`).
- 1080p is the sweet spot — 4K is *worse*, because it downscales harder.
- Pause 2–3 seconds on anything important.
- For code and terminal output, paste the text. A screenshot of a stack trace is
  always worse than the stack trace.

## Failure modes to avoid

- Using scene detection on a screen recording and silently getting zero frames.
- Reading 30 frames individually instead of building a contact sheet first.
- Quoting text off a contact sheet without confirming it at full resolution.
- Letting `mpdecimate` return 100+ frames on a long recording instead of capping.
- Guessing crop coordinates by eye when frame differencing will compute them.
- Reading a downscaled frame, guessing at the text, and stating it confidently.
- Running whisper on a video with no audio track.
- Extracting 200 frames and blowing the context window.
