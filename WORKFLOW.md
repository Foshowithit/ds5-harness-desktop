# The pipeline

Five stages. Each one is a single script with a gate. Run them in order.

```
app/ + src/            the React + framer-motion film source
        |
        v
harness/build.mjs      -> dist/bundle.js      (published surface)
                          dist/__cap.js       (capture surface, NEVER published)
        |
harness/bands.mjs      -> harness/bands.bin   (measured spectral-flux onsets)
        |
harness/capture.mjs    -> frames/f0000..f0504.png + evidence/capture.json
        |
harness/encode.mjs     -> out/ds5-film-1080p.mp4, out/ds5-film-x.mp4 + evidence/encode.json
        |
harness/publish.mjs    -> pages/              (the tree this repo publishes)
```

## Stage 1 — build

`node harness/build.mjs`

Emits **two** bundles from one source tree:

* `dist/bundle.js` <- `harness/entry-publish.tsx` — the public surface. Contains no
  capture machinery.
* `dist/__cap.js` <- `harness/entry.tsx` — the capture surface. Exposes
  `window.__ds5`, including `pin(t)`, which freezes the film's virtual clock at an
  arbitrary time so a frame can be captured without playing.

`build.mjs` gates the boundary between them: it throws if `__ds5` appears in
`bundle.js` ("machinery leaked into the published surface") and if it is *absent*
from `__cap.js`. Both halves are needed — a one-sided check passes when the capture
build silently stops emitting the handle.

## Stage 2 — measure the score

`node harness/bands.mjs`

Reads `audio/ds5-film-bed.mp3` and writes `harness/bands.bin` (387,840 B): the
per-frame spectral-flux envelope the scenes are cut against. The film does not use a
hand-written cue sheet; the cut points are measured off the audio.

## Stage 3 — capture

`node harness/capture.mjs`

Renders 505 frames over the Chrome DevTools Protocol. Three things here are load-bearing:

1. **`--autoplay-policy=no-user-gesture-required`.** Without it `el.play()` is
   rejected, and the blocked-audio path force-lights the player HUD, making
   `--hud off` impossible.
2. **The pin recipe.** `beginRewind()` is the only public method that sets the
   virtual clock without playing, but it also sets `rewindMode`, which paints the VHS
   tracking overlay into the captured frame. So the driver writes the private field
   directly and keeps `playing` true. Order matters: set `window.__vc.now = t`
   FIRST, then `pin(t)`, so the pin's origin equals `t` exactly and `dt` is 0 for
   the whole pump.
3. **The clock is milliseconds.** `clock-init.js` overrides `performance.now()` to
   return `vc.now * 1000`, because every consumer in the page is written against ms
   (`motion-dom`'s `JSAnimation.updateTime`, the grain shader's 88 ms threshold, CSS
   `Animation.currentTime`). Returning seconds here once made no scene ever reveal —
   the film rendered as an empty grain field, and every fence still passed.

The gate: a frame "carries content" when at least 0.1% of its pixels have luma > 64.
The run fails if fewer than 25% of frames carry content. Measured — broken: 6/505
(1.2%); fixed: 441/505 (87.3%). The threshold sits between them with 20x margin below
and 3.5x above. A per-frame structure floor was tried and **rejected by measurement**
(the broken render's `f0216` had *higher* block standard deviation than the fixed
render's `f0003`), which is why the fence is aggregate.

## Stage 4 — encode

`node harness/encode.mjs`

1. **Inventories the frame sequence before encoding**, because ffmpeg's image2
   demuxer stops at the first gap and still reports success — a hole would otherwise
   ship as a silently truncated film.
2. Encodes a CRF 16 1080p master and a CRF 20 720p social cut with `+faststart`.
3. **Probes and asserts** codec, pixel format, dimensions, frame rate, frame count,
   audio presence and duration — then mutates the probe result four ways
   (wrong pixel format, wrong fps, audio removed, duration halved) and fails if the
   assertion accepts any of them.

## Stage 5 — publish

`node harness/publish.mjs`

Assembles the published tree from an **allowlist** and gates it with five independent
predicates: no capture machinery by filename, none by content, required assets
present, no origin-absolute asset paths, and every `index.html` reference resolving
to a real file. It then mutates the finished tree four ways and fails if any mutation
survives.

Predicate 4 exists because a GitHub Pages *project* site is served from
`https://<user>.github.io/<repo>/`. An origin-absolute `/bundle.js` resolves to the
domain root and 404s — and this is invisible locally, where a server rooted at the
repo serves `/bundle.js` fine.

## The three defects this pipeline caught

1. **The unit mismatch.** The clock returned seconds where the page expected
   milliseconds. Result: 505 frames of empty grain, and every existing fence passed.
   Fixed at the `clock-init.js` boundary; the per-frame `perfNow` field is now a
   permanent regression detector.
2. **The gate that could not see the failure.** The original pixel fence judged frames
   individually and could not distinguish an empty grain field from a dark scene. It
   was replaced by the aggregate content fence after the per-frame alternative was
   rejected by measurement.
3. **The publish boundary.** `build.mjs` stages both surfaces into one directory and
   its boundary gate only inspects bundle *contents*, never the published *file set* —
   so `__cap.js` (1.19 MB of capture machinery) would have gone public. Stage 5
   closes this.

## Running it as an Archon workflow

The same five stages are packaged as an Archon lane, so the whole film can be
reproduced with one dispatch:

```bash
archon workflow run ds5-film-render-v1 '<task>'
```

The lane is [`workflow/ds5-film-render.yaml`](workflow/ds5-film-render.yaml). It is
**bash-only** — it declares no provider and no model, so it spends no credits and cannot
be killed by an unhealthy model provider. Every one of its nodes is an `ssh` hop, because
the render runs on a Mac while the workflow engine runs on a Linux host.

Its environment is **five two-sided levers**, not constants. Each one has an
environment variable and a space-free `KEY=` token on the task string. The lane's own
header is canonical for this list; it is mapped here so the lane need not be opened to
learn what can be aimed:

```
MAC       FILM_MAC        the ssh alias the render runs on
ROOT      FILM_ROOT       the scratch tree on that machine
NODE_BIN  FILM_NODE       the Node binary the harness is run with
DEP_WS    FILM_DEP_WS     the shared node_modules the harness resolves from
SRC_WS    FILM_SRC_WS     the film source tree
```

Every default is the literal that was previously hard-wired, so a run that passes no
override behaves exactly as before. `SRC_WS`'s default contains a space, which the
`KEY=` form cannot express — it is reachable by environment variable only.

`LANE_DIR` is deliberately **not** a lever. `assert.mjs` hardcodes
`const LANE = '/tmp/ds5-lane'`, so overriding the lane's scratch directory would
silently split it from the directory its own assertion helper reads. A lever that
cannot act is worse than no lever.

That makes the lane **configurable, not yet relocatable**: the lane's own constants
are levers, but the Mac-side scripts it drives still resolve their environment
absolutely — `build.mjs` and `og-card.mjs` each hardcode the same Node binary path,
and the dependency workspace is resolved by `createRequire` in six separate files
rather than from one place. A second operator on another machine must still reproduce
that environment.

Every stage asserts the **recorded evidence** in `evidence/*.json`, never an exit code. A
stage that exits 0 having produced nothing still fails — which is exactly the failure
that once shipped 505 frames of empty grain while every other check passed.
