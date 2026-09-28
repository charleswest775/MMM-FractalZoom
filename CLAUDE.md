# MMM-FractalZoom — context for Claude sessions

Charles's MagicMirror² module: an infinite zoom into the Mandelbrot set and Julia sets, for his
hallway mirror, as one page in a rotation of pages. Split out of MMM-ChaosTheory on 2026-09-28
with its history (it was the `zoom` simulation of that module's fractal page).

## Files

- `MMM-FractalZoom.js` — module shell: one canvas plus an HTML caption (equations + live readout,
  updated 2×/s). New sim every `cycleSeconds` and on each `resume()`. Loop: `setTimeout` until a
  frame is due, then one `requestAnimationFrame`. `suspend()` stops it; a sim with
  `resting = true` is polled only every 500 ms; while MagicMirror fades the module out (`hidden`
  is set at the start, `suspend()` comes after), frames draw nothing. `turns: { of, at }`: only
  every nth showing; otherwise the wrapper gets `display: none` and nothing starts. Calls
  `sim.dispose()` when it replaces a sim, which terminates the workers.
- `simulations/zoom.js` — the `zoom` sim on `window.FractalSimulations`: keyframes per doubling,
  two drawn scaled each frame; `WorkerPool` shares rows out over `zoomWorkers` web workers.
  Rests while waiting for a keyframe and at the end of the dive (~2³⁴, where 64 bits run out).
- `simulations/zoom-math.js` (`window.FractalZoomMath`) — escape times, Misiurewicz targets by
  Newton, colours, `maxDepth`; shared with `simulations/zoom-worker.js` (loaded by `importScripts`,
  so the page must be served over http). `simulations/common.js` — `sci()` formatting.
- UMD-style, so the maths runs in Node: `tests/zoom.test.js` (`node --test`, no dependencies).
- `node_helper.js` — the stats panel (`statsPanel: true`): CPU of Electron and cage, per core,
  temperature, from `/proc`, only while shown.
- `dev/preview.html` — runs the module in a desktop browser (`python3 -m http.server` in the repo,
  then `/dev/preview.html?target=seahorse`).

The shell (`MMM-FractalZoom.js`, `node_helper.js`'s stats panel, `dev/preview.html`) is shared in
spirit with the sibling modules (MMM-ChaosTheory, MMM-Chladni, MMM-Atom, MMM-SacredGeometry,
MMM-Tilings, MMM-PlanetsDance, MMM-SnowCrystal, MMM-NightSky, MMM-PhotoDeck): a fix there
probably belongs in the siblings too.

On the mirror it shares the fractal page with MMM-Chladni, taking turns
(`turns: { of: 2, at: 0 }` here, `{ of: 2, at: 1 }` there).

## Measured cost on the Pi

700², 12 fps, 2 workers: 155–168% of a core for the first 30 s of Elephant Valley, the star and
the north bulb; 234% Seahorse Valley, 270% the Julia spiral. ~130% of it is drawing (the whole
canvas changes every frame); workers ~25% each on cheap dives, up to ~100% each on the costly
ones. 70–73 °C. Hidden: 0.3% (baseline 0.2%), workers 0%. The costliest page of the family.

## Performance findings on the Pi (measured)

- A frame that changes the canvas costs ~2%/fps fixed; beyond that, cost scales with the
  **bounding box of everything changed in the frame**. Full redraws of a 900² canvas at 20 fps
  saturate the pipeline (~150%). JS on the main thread is never the bottleneck (<3 ms/frame).
- The zoom can't draw incrementally, so keep it small (700²) and slow (12 fps), and rest when the
  picture is static. Line width, opacity, `rAF` vs timer made no difference.
- MagicMirror applies `electronSwitches` after app ready, so `remote-debugging-port` can't be set
  that way; use `debugStats: true` and a `grim` screenshot to see fps on the Pi.

## Hard constraints: the target device

- **Raspberry Pi 3 B+, 905 MB RAM, 64-bit Debian 13.** Mirror runs Electron 42 in a cage
  Wayland kiosk.
- **No GPU acceleration, and it can't be enabled**: the Pi 3's VideoCore IV only does GLES 2.0,
  Chromium needs ES 3.0 (tested). All canvas drawing is CPU. **No WebGL / three.js.**
- Screen will be **portrait 1200×1920** once mounted (Dell U2413, rotated). Design for portrait.
- Electron baseline is ~0.5% of one core. **Measure, don't guess**: on the Pi,
  `~/.cache/mm-sample.sh 60` prints Electron CPU% and RSS over 60 s. Record before/after numbers
  in the README.
- The mirror rotates pages every 15-30 s (MMM-pages, which hides/shows modules). `suspend()` and
  `resume()` must fire on page changes, or the loop (and the workers) burn CPU 24/7.

## Deploying and testing

- This repo is public so the Pi can `git clone`/`git pull` without credentials.
- Pi access: `ssh fatherson@raspberrypi.local` (key auth). Module path:
  `~/MagicMirror/modules/MMM-FractalZoom`. Restart: `pm2 restart MagicMirror`
  (pm2 is in `~/.npm-global/bin`). Logs: `pm2 logs MagicMirror`.
- The mirror's **config.js lives in a separate private repo**, `charleswest775/magicmirror-setup`
  (cloned at `~/dev/magicmirror-setup`). Add the module's config block there, then
  `./deploy.sh diff` and `./deploy.sh push` (push validates config before restarting).
  Don't hand-edit config.js on the Pi without `./deploy.sh pull` afterwards.
- Faster iteration: run it in a desktop browser (`dev/preview.html`), then confirm performance
  on the Pi.
- Commit as Charles's GitHub noreply address (set in this repo's git config).
