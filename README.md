# HunterLabsDev Skills

Practical [Claude](https://claude.com) skills, built by a solo dev and published
free. Each one exists because I hit a real limitation and got tired of working
around it by hand.

Built and maintained by **[HunterLabsDev](https://github.com/HunterLabsDev)** ·
[hunterlabsct.com](https://hunterlabsct.com)

---

## Install

Add this marketplace, then install the skills you want:

```
/plugin marketplace add HunterLabsDev/claude-skills
/plugin install video-analysis@hunterlabs-skills
```

Works in Claude Code and Cowork — anywhere Claude can run shell commands.

---

## Skills

### 📹 video-analysis

**Makes screen recordings as easy to work with as a photo.**

Claude has no native video input. When you hand it a screen recording, it sees a
handful of stills that have been downscaled to roughly 1568px on the long edge.
On a 4K capture, your 14px UI text becomes about 5 pixels tall. Claude then
guesses at it — confidently, and often wrong.

That last part is the real failure. Low quality you can work around. A confident
wrong answer you can't.

**The three-tier read** is the core idea: one image for the whole timeline, then
drill in only where it matters.

| Tier | What | Why |
|---|---|---|
| 1 | **Contact sheet** — all frames tiled into a single grid | One read shows the entire arc of the recording |
| 2 | **Full-resolution frame** at the moment that matters | At 1080p this usually OCRs cleanly on its own |
| 3 | **Crop + 2x lanczos upscale** of the exact region | Rescues text too small to survive downscaling |

**Plus the part I haven't seen elsewhere: it computes the crop instead of
guessing it.** Frame differencing finds the bounding box of whatever changed
between two frames — which is almost always the thing you care about — and crops
straight to it. No eyeballing coordinates off a blurry screenshot.

Also: deduplicated frame extraction with a hard frame budget, timestamps for
everything, OCR as an independent cross-check, Whisper transcription when
there's audio — and an explicit list of what it *couldn't* read, instead of a
plausible guess.

**Requires** `ffmpeg` + `ffprobe`. `tesseract`, `faster-whisper` and
`opencv-python` are optional but recommended; the skill checks for each and
degrades gracefully.

---

## The thing this gets right that most advice gets wrong

Nearly every guide to extracting frames recommends ffmpeg scene detection:

```bash
-vf "select='gt(scene,0.15)'"
```

**On a screen recording, that is broken.** `scene` scores whole-frame change, so
a recording where only a text line or a panel changes against a static
background scores near zero.

Measured on a 1080p recording containing four completely different screens:

```
lavfi.scene_score = 0.001
```

That's ~150x below the standard threshold. Scene detection returned **0 frames**.

This skill uses `mpdecimate` instead, which returned exactly **4** — one per
screen, with correct timestamps at 0s, 6s, 12s and 18s.

Every command in these skills is tested before it ships, including the negative
results. A few other things that testing changed:

- `-q:v` is a **no-op on PNG output** — verified identical byte size with and
  without it. It's in a lot of frame-extraction snippets anyway.
- `mpdecimate` **ignores your frame budget**. A 10-minute recording produced 100
  frames, so the skill counts first and thins to target in a second pass.
- Contact sheets are readable at `2x2` and not at `4x3`. The skill says to
  confirm any quoted text against the full-resolution frame rather than trusting
  the sheet.

---

## Recording tips

Most "Claude can't read my video" problems are fixable at capture time:

| Do | Why |
|---|---|
| Record the window, not the whole desktop | Less to downscale |
| Zoom the app to 150% first | Text survives the resize |
| Use 1080p, not 4K | Counterintuitive — 4K downscales *harder* |
| Pause 2–3s on anything important | A flash frame can fall between samples |
| Paste text instead of screenshotting it | A stack trace beats a picture of one |

---

## Contributing

Issues and PRs welcome. If a skill misfires on a real file of yours, open an
issue with the `ffprobe` output — that's usually enough to diagnose it.

## License

MIT — see [LICENSE](LICENSE). Use it, fork it, ship it.
