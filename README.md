# Harness Desktop 5.0 — DeepSeek x RCOS

A 21-second deterministic motion-graphics promo film, plus the pipeline that renders it.

**Live:** https://foshowithit.github.io/ds5-harness-desktop/ (interactive film)
&middot; [watch the MP4](watch.html)

## What this is

Nine scenes, one score, cut to the music. The film is not screen-recorded and not
hand-edited: it is rendered frame-by-frame from a React + framer-motion source tree
driven by a pinned virtual clock, then encoded with ffmpeg.

| | |
|---|---|
| Runtime | 21.000 s (504 frames @ 24 fps) |
| Master | `film/ds5-film-1080p.mp4` — 1920x1080, H.264 High, CRF 16, AAC 48 kHz |
| For social | `film/ds5-film-x.mp4` — 1280x720, CRF 20, faststart |
| Score | `audio/ds5-film-bed.mp3` |
| Scenes | 9, cut to measured spectral-flux onsets |

## The page

`index.html` is the interactive film — the same nine scenes, running live in the
browser, with the score's beat detection driving the choreography in real time.
`watch.html` is the encoded MP4.

## Reproducing the film

Five stages, each a single script with a gate, documented step by step in
[WORKFLOW.md](WORKFLOW.md):

```bash
node harness/build.mjs     # source tree -> dist/ (published + capture surfaces)
node harness/bands.mjs     # measure the score's onsets -> bands.bin
node harness/capture.mjs   # headless Chrome, pinned clock -> frames/f0000..f0503.png
node harness/encode.mjs    # frames + audio -> out/*.mp4, with a probe gate
node harness/publish.mjs   # dist/ -> the published tree, with a boundary gate
```

This page is the **output** of that pipeline, not its source. The repository you are
reading ships the rendered film and the page that plays it; it does not ship the
`harness/` scripts or the React source tree they build from. To reproduce the film,
take the lane in [`workflow/ds5-film-render.yaml`](workflow/ds5-film-render.yaml) — it
drives all five stages in order and asserts each stage's recorded evidence rather than
its exit code, so a stage that exits 0 having produced nothing still fails.

## Why the gates exist

Every stage carries a gate that has been *seen to fail*. The render gate refuses a
frame sequence that carries no content (a regression that once shipped 505 frames of
empty grain while every other check passed); the encode gate refuses a file whose
probe cannot reject five known-bad mutations; the publish gate refuses to ship the
capture bundle. `WORKFLOW.md` names each one and the defect it caught.

## Licence

Source and film (c) the author. No model credits were spent producing this film —
every stage runs locally.
