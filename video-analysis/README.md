# video-analysis

Makes Claude actually read your screen recordings.

Part of [HunterLabsDev Skills](https://github.com/HunterLabsDev/claude-skills).

## The problem

Claude has no native video input. A video you hand it becomes a handful of still
frames, downscaled to roughly 1568px on the long edge. On a 4K screen recording
that shrinks 14px UI text to about 5 pixels — unreadable. Claude then fills the
gap with a plausible guess, stated confidently.

That last part is the real failure. Low quality you can work around. A confident
wrong answer you can't.

## What it does

| Step | Behavior |
|---|---|
| Probe | Reads duration, resolution and stream layout before touching anything |
| Extract | `mpdecimate` deduplication — **not** scene detection, which fails on screen capture |
| Budget | Counts frames first and thins to ~30, so long recordings don't flood context |
| Overview | Tiles frames into a single contact sheet: one read for the whole timeline |
| Auto-crop | Frame differencing computes the bounding box of what changed, and crops to it |
| Detail | Full-resolution frame, then crop + 2x lanczos upscale when text is too small |
| OCR | Runs `tesseract` as an independent cross-check; flags disagreement |
| Audio | Whisper transcription with timestamps, skipped when there's no audio track |
| Report | Timestamps throughout, and an explicit list of what it could not read |

## Why not scene detection

Nearly every ffmpeg frame-extraction guide recommends `select='gt(scene,0.15)'`.
On a screen recording it's broken — `scene` scores whole-frame change, so a
recording where only a text line or panel changes against a static background
scores near zero.

Measured on a 1080p recording with four completely different screens:
`lavfi.scene_score = 0.001`, about 150x below the standard threshold. Scene
detection returned **0 frames**. `mpdecimate` returned exactly **4**, one per
screen, with correct timestamps.

## Install

```
/plugin marketplace add HunterLabsDev/claude-skills
/plugin install video-analysis@hunterlabs-skills
```

## Requirements

Needs a shell — works in Claude Code, Cowork, or anywhere Claude can run
commands.

Required:

```bash
ffmpeg
ffprobe
```

Optional but recommended:

```bash
tesseract        # OCR cross-check
faster-whisper   # audio transcription (pip)
opencv-python    # auto-crop via frame differencing (pip)
```

On Debian/Ubuntu:

```bash
apt-get install -y ffmpeg tesseract-ocr
pip install faster-whisper opencv-python
```

The skill checks for each and degrades gracefully rather than failing.

## Usage

There's no command to run. Hand Claude a video and ask a question about it:

- "What error is showing in this recording?"
- "Walk me through what happens in this screencast"
- "Transcribe this and tell me where they mention pricing"
- "At around 1:20 something breaks — what is it?"

Naming a moment ("around 1:20") is the single biggest quality win. It lets the
skill extract densely from that window instead of spreading frames thin across
the whole file.

## Getting better results

| Do | Why |
|---|---|
| Record the window, not the whole desktop | Less to downscale |
| Zoom the app to 150% first | Text survives the resize |
| Use 1080p, not 4K | 4K downscales *harder* |
| Pause 2–3s on anything important | A flash frame can fall between samples |
| Trim before sending if the file is >200MB | Faster, and no quality cost |
| Paste text instead of screenshotting it | A stack trace beats a picture of one |

## License

MIT
