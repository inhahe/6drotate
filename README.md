# 6D Video Rotator

Load a video and rotate it in **six** dimensions. Every pixel of every frame
becomes a point in 6D space — two spatial coordinates (X, Y), one **time**
coordinate (T, the frame's position in the clip), and three color coordinates.
Choose between **RGB**, **HSV**, and **HSL** color spaces. Fifteen sliders let
you rotate any axis toward any other, blending position, *time*, and color in
ways that don't exist in a normal video editor.

There's also a **Mono** mode that collapses the three color channels into a
single **luminance** axis (L). The voxel is then 4D — (X, Y, T, L) — so there
are only **six** rotation planes instead of fifteen, and the sliders shrink to
match.

Because time is now an axis, a rotation that touches T only reveals itself when
you **play** the clip: the app sweeps a thin "time slab" through the rotated 6D
volume, so you literally watch time pass through the rotation.

Open `6d-video-rotator.html` locally in a browser. No build step, no
dependencies.

## How it works

A pixel at column 80, row 40 of frame 12 (of 48) with color rgb(200, 100, 50)
becomes the 6D point:

    RGB mode:  (80, 40, t12, 200, 100, 50)
    HSV mode:  (80, 40, t12, 14°, 0.75, 0.78)
    HSL mode:  (80, 40, t12, 14°, 0.60, 0.49)
    Mono mode: (80, 40, t12, 120)          ← single luminance value

All six coordinates are centered and normalized so they carry roughly equal
weight during rotation. A 6×6 rotation matrix is built from however many of the
15 sliders are non-zero, then applied to every point. The output X and Y become
the point's new screen position; the output T decides *when* during playback it
appears; the output color channels become its new color.

At **identity** (all sliders zero) each point's time stays put, so pressing
**Play** just plays the video back frame by frame. As soon as you rotate a
plane involving T, position/color/time start mixing:

- **X↔T** shears the picture through time — the frame scrolls sideways as it
  plays.
- **R↔T** ties color to time — reds arrive at a different moment than blues.
- Spatial-only rotations (X↔Y, X↔R, …) behave like the 5D rotator on every
  frame at once.

## The 15 rotation planes

In 6D there are 6×5/2 = 15 unique rotation planes — every way to pick two axes
out of six:

    RGB mode:                       HSV mode:                       HSL mode:
    X↔Y  X↔T  X↔R  X↔G  X↔B          X↔Y  X↔T  X↔H  X↔S  X↔V          X↔Y  X↔T  X↔H  X↔S  X↔L
         Y↔T  Y↔R  Y↔G  Y↔B               Y↔T  Y↔H  Y↔S  Y↔V               Y↔T  Y↔H  Y↔S  Y↔L
              T↔R  T↔G  T↔B                    T↔H  T↔S  T↔V                    T↔H  T↔S  T↔L
                   R↔G  R↔B                         H↔S  H↔V                         H↔S  H↔L
                        G↔B                              S↔V                              S↔L

In **Mono mode** the color axes collapse to one luminance axis, leaving just
4×3/2 = 6 planes:

    X↔Y  X↔T  X↔L
         Y↔T  Y↔L
              T↔L

Each slider goes from -180° to +180°. Sliders that touch the **T** axis have a
gold thumb as a reminder that you'll need to play the clip to see their effect.
Double-click a slider to zero it.

**Live preview while dragging.** Rotating the whole cloud is an N×N transform of
*every* point (the visible time slab is defined in *rotated* time, so a slider
change can't be confined to one frame). On a huge clip that's too slow to do
every mouse-move, so while you're actively dragging a rotation, **Color wt** or
**Time wt** slider — or while **Spin** is running — only a strided sample of the
points (~1.2 M) is transformed for an instant, sparser preview. The moment you
release the slider (or stop Spin) the full-quality transform runs once. On small
clips the sample is the whole cloud, so you never see the difference.

## Scale factors

Two ratio sliders control how "long" the non-spatial axes are relative to the
X/Y plane:

- **Color wt** — color depth vs. spatial extent (same as the 5D rotator).
- **Time wt** — the **xy ↔ time ratio**. At 1.0 the whole clip's duration spans
  the same range as the image is wide. Higher values stretch the time axis, so
  rotations that mix time and space move points further per frame; lower values
  keep everything anchored near a single instant.

**Time thick** sets how many frames' worth of the time slab is visible at once
(in units of frame spacing). Thin = crisp single-frame slices; thick = motion
smear / ghosting.

## Playback

- **Play time** — sweep the time slab forward and loop. The demo auto-plays.
- **Speed** — full sweeps per second.
- **Time** scrubber — drag to inspect any single instant while paused; it tracks
  playback while playing.
- **Spin** — separately, randomize the *rotation* speeds and tumble the 6D
  orientation continuously (independent of time playback).

## 3D view orbit

The 15 rotation sliders rotate the 6D *cloud*; the three **View orbit** sliders
instead tumble the already-rendered 3D result on screen, like spinning a model in
a viewer. **X–Y** rolls it in the screen plane, and **X–Z** / **Y–Z** tilt it so
the depth axis — the rotated **time** coordinate — leans into view, letting you
see the time-extruded shape of a clip from an angle rather than dead-on. It's a
pure viewing transform: it doesn't change which points fall inside the time slab,
only where they land on screen (and, on the GPU, their depth ordering). Each
slider runs -180° to +180°; double-click to zero.

## Out-of-gamut clipping

When a rotation pushes a point's color outside the valid range it has left the
visible gamut. The **OOB color** toggle renders those points as **Magenta**
(default, obvious), **Black** (vanish), or **Clamp** (channels clamped to
range). The info bar shows the current clip percentage (a dash while the GPU
renderer is active, since a vertex shader can't count clipped points); a faint
dashed rectangle marks the original frame bounds. The info bar also shows a
**GPU / CPU** indicator reporting which renderer is currently drawing the live
view — it turns cyan on **GPU** and updates the moment a clip is loaded and the
renderer is chosen. See [GPU rendering](#technical-notes) below.

## Controls

### Sidebar

Controls with a **dotted underline** (Dot scale, Color wt, Time wt, Time thick,
Resolution, Frames, OOB color) show an explanatory tooltip on hover.

| Control      | What it does |
|--------------|-------------|
| Drop zone    | Load an MP4/WebM/MOV **or an animated GIF** (works in every browser, including Safari). The clip's native size and frame count are read first, then a **load dialog** (below) lets you pick the resolution and frame count *before* it decodes. |
| Load dialog  | Pops up on every video/GIF load. It shows the clip's **native** width×height and frame count, and gives you a resolution slider (long side, up to the native size) and a frames slider (up to the native count), each with a matching number box for exact entry. A **live memory estimate** updates as you drag so you can see the cost before committing. **Load** decodes at your chosen settings; **Cancel** keeps the current clip. Enter = Load, Esc = Cancel. This is how you reach **full native resolution** without loading blind — pick exactly what fits in memory up front instead of decoding at a fixed cap and reloading. |
| Dot scale    | Multiplier on the auto-computed splat size. 1.0 = seamless tiling. |
| Zoom         | Scale the cloud on the canvas (also mouse wheel). At 1.0 the cloud is 1:1 with the working-resolution frame (one source pixel per screen pixel). |
| Resolution   | Working resolution: the long-side pixel count the clip is downsampled to before every pixel becomes a 6D point. Its ceiling is whatever resolution you loaded in the load dialog (the in-memory master), and it defaults to that full value. Dragging it **down** re-samples from the master live — no re-decode; to go **higher** than the loaded master, reload the clip and pick a bigger resolution in the dialog. Higher = sharper/more detail but slower (cost grows with the square). Takes effect on release. |
| Frames       | How many time slices the clip is sampled to along the **T** axis (default 48). More frames = smoother motion and finer time-mixing rotations, but memory and compute grow with the count. The slider's ceiling is the source's real frame count — **exact** for GIFs, **estimated** for video (duration × fps, where fps is measured on load via `requestVideoFrameCallback`), with a generous fallback when fps can't be measured. Your chosen count is remembered as a *preference* and re-applied (clamped to each clip's real length) on the next load, so a short clip that clamps 48→N doesn't leave you stuck at N when a longer clip follows. There's no fixed memory cap: if a request is simply too big to allocate, the app aborts cleanly and asks you to reduce Frames or Resolution rather than crashing the tab. Takes effect on release (for video this re-seeks the source). |
| All frames   | Checkbox below the Frames slider. Tick it to extract **every** frame the source has instead of sampling down to the Frames count (the readout shows `All (N)` in gold and the Frames slider is disabled while it's on). All-frames mode stays on across loads, so a freshly loaded clip comes in at full frame count without a reload. |
| Est. memory  | Live estimate of how much RAM the point cloud will take at the current Frames, Resolution and color space (worst case: every pixel visible). Updates as you drag the Frames/Resolution sliders — *before* you release. If a load fails with an out-of-memory error during this session, the readout turns **red** whenever the estimate reaches or exceeds that failed size (resets on page reload). Click the number to toggle between MB/GB and an exact comma-grouped byte count. |
| Color wt     | Ratio of color depth to spatial extent. |
| Time wt      | Ratio of time extent to spatial extent (xy ↔ time). |
| Time thick   | Slab thickness in frame-spacings — how much time is shown at once. |
| Color space  | RGB / HSV / HSL / Mono toggle. Mono collapses color to a single luminance axis (6 sliders). Slider labels and count update to match. |
| OOB color    | Magenta / Black / Clamp for out-of-gamut points. |
| Play time    | Toggle time playback. |
| Speed        | Playback speed (sweeps/sec). |
| Time         | Scrub / read the current instant. |
| View orbit   | Three sliders (X–Y, X–Z, Y–Z) that tumble the *rendered* 3D result on screen without re-rotating the cloud — see [3D view orbit](#3d-view-orbit). -180 to +180 each; double-click to zero. |
| Rotation sliders | 15 in RGB/HSV, 6 in Mono. -180 to +180 degrees each. Double-click to zero. |
| Reset All    | Zero every rotation slider. |
| Spin         | Randomize rotation speeds and tumble continuously. |
| Randomize    | Jump to random angles on all 15 planes. |
| Export PNG   | Save the current slab as a PNG. |
| Export video | Record one full time-sweep (playPhase 0→1) to a video file, played back at the current **Speed**. If **Spin** is on, the rotation tumble is captured too. Where the browser exposes **WebCodecs** H.264 (Chrome, Edge), frames are encoded and muxed into a self-contained **MP4** — this path is **frame-exact and faster than real time**. Otherwise it falls back to the browser's `MediaRecorder` (MP4/H.264 where supported, else WebM), which runs in real time. Either way there are no external libraries, and progress shows behind an "encoding video…" overlay. |

### Keyboard

| Key   | Action |
|-------|--------|
| Space | Toggle time playback |
| A     | Toggle rotation spin |
| R     | Reset all rotations |
| E     | Export PNG |
| V     | Export video |
| C     | Cycle color space (RGB → HSV → Mono) |

Scroll wheel over the canvas to zoom; pinch to zoom on touch devices.

## Technical notes

**Saved settings.** Every sidebar knob is persisted to `localStorage` and
restored on the next page load: Dot scale, Zoom, Color wt, Time wt, Time thick,
Play speed, the color space, the OOB color mode, the Resolution and Frames
preferences, the **All frames** toggle, the memory-readout format, and the
current rotation angle on every plane. Only transient state is *not* saved — play/pause, spin, the scrub
position, and the loaded clip itself (each session starts on the demo). Writes
are debounced so dragging a slider doesn't hammer storage, and a private-mode or
full-storage failure is swallowed silently. Delete the `6drotate.settings.v1`
key (or clear site data) to get the factory defaults back.

**Two-stage render.** The heavy 6×6 transform of every point is cached and only
recomputed when the rotation, weights, clip mode, or color space change. Each
displayed frame just re-gates the cached cloud by the time slab and splats it —
so scrubbing and playback stay smooth even with hundreds of thousands of points
(Frames × up to Resolution² px — e.g. 48 × 100×100 at the defaults).

**Time-bucket index.** Only a thin slab of the cloud is visible at any playback
time, so scanning every point each frame just to skip ~99.9% of them wastes work
that grows with the *whole* cloud (Frames × Resolution²), not the visible slice.
When the rotation cache is rebuilt, the points are counting-sorted into bins by
their rotated time coordinate (`timeOrder` lists point indices grouped by bin;
`binStart[b]..binStart[b+1]` is bin *b*'s slice). Rendering then walks only the
bins the current slab spans — so a frame's cost tracks the number of *visible*
points, not the total. This is what keeps playback fast at high frame counts
(e.g. Resolution 168 × 3888 frames ≈ 60M points, of which only a few thousand
are on screen per frame). If every point lands at one time (no time rotation),
the index is degenerate and rendering falls back to a full scan.

**GPU rendering (WebGPU).** When the browser exposes **WebGPU** and the clip is
small enough to fit the device's single-buffer limit, the whole cloud is
transformed *and* rasterized on the GPU every frame — there's no CPU
counting-sort, no cache, and no readback. The point buffer is uploaded once; a
vertex shader then does the full weighted N×N rotation, the color-space
conversion and out-of-gamut clip, the time-slab cull, the gap-free splat
footprint, and the 3D view orbit, emitting one instanced quad per point with a
depth-tested `z` from the rotated time axis. The result is drawn on a dedicated
WebGPU canvas, and in GPU mode the 2D overlay canvas is hidden entirely so that
canvas is the *only* layer the browser composites — the dashed frame-bounds
outline and time readout are plain DOM elements positioned over it instead of a
second, transparent canvas stacked on top. (An overlapping transparent 2D canvas
made Chrome's compositor intermittently drop the WebGPU layer, blanking the cloud
while the outline kept painting.) The 2D canvas keeps its bitmap while hidden, so
CPU render and export still read from it. "Small enough" means `points × N × 4`
bytes fits within
`min(maxStorageBufferBindingSize, maxBufferSize)` — the largest single buffer the
GPU allows. This is **not** a measure of your VRAM: WebGPU exposes no total-memory
figure, and this per-binding cap is a fixed structural limit (commonly ~2 GiB) that
reads the same on a 24 GB card as on a small laptop GPU, so the app doesn't try to
guess or flag against your card's real capacity. Anything that doesn't fit that one
buffer, or any WebGPU/pipeline failure at any point, silently falls back to the CPU
renderer for the rest of the session, so the app behaves identically with or
without a GPU. Which renderer is live is shown by the **GPU / CPU** indicator in
the info bar — it updates the instant a load allocates the cloud and the renderer
is chosen (or a GPU failure falls back). On the GPU the clip-percentage readout
shows a dash instead, because a vertex shader can't count clipped points without
atomics. Because a video **export** reads the 2D canvas, it always forces the CPU
transform first, so exports match the on-screen result — view orbit included — even
in GPU mode.

**Working resolution.** On load the clip is decoded and downsampled once to a
master whose long side is the resolution you picked in the load dialog (anything
up to the source's native size); that master is held in memory. The point cloud
is then derived from that master at the current **Resolution** setting, which
starts at the master's full size and can be dragged down. Moving the Resolution
slider re-samples from the master rather than re-decoding the source, so it's
cheap — the heavy re-extract happens only on release (a brief "resampling…"
overlay shows while it runs). Going above the master means loading again and
choosing a larger resolution in the dialog, which re-decodes the source. The
dialog's up-front memory estimate is the guard rail: full native resolution ×
every frame can be enormous, so you decide what fits before it allocates.

**Frame count & memory.** The **Frames** slider re-samples the *original*
decoded media (kept in memory) rather than re-downloading it — instant for GIFs
and the demo, a re-seek for video. The slider's ceiling is the source's true
frame count: GIFs report it exactly, while for video (which exposes neither fps
nor a frame count) the app measures the frame rate on load by playing muted for
a fraction of a second and reading successive `mediaTime` values via
`requestVideoFrameCallback`, then multiplies the median frame rate by the
duration. Where `requestVideoFrameCallback` is unavailable it falls back to a
generous fixed ceiling. Some clips (notably `yt-dlp` WebM/MP4 downloads) ship
without a duration in their metadata, so the browser reports it as `Infinity`
until you seek past the end; the loader detects this and forces the real duration
by seeking to a huge time and waiting for `durationchange` before sampling —
otherwise extraction would sample only the first second and you'd get a blank or
near-empty result. If a clip still decodes no visible pixels (an unsupported
codec), the app says so instead of leaving a silently empty canvas. The count you set is stored as a *preference* distinct
from the count actually extracted: on each load the preference is re-applied and
clamped to the new clip's real length, so a short clip clamping it down doesn't
overwrite your intent for the next (longer) clip. The **All frames** checkbox
switches to **All-frames** mode, which extracts the source's full frame count and
persists across loads. There is **no artificial memory cap** — the real limit is
just your free RAM. Most browsers *do* refuse to allocate a single `ArrayBuffer`
larger than ~2 GB (Chrome/Safari ~2 GB; Firefox 8 GiB), which used to cap the
point cloud even when plenty of RAM was free. To sidestep that, the cloud is
**chunked**: instead of one giant typed array per field, each field is split
across several ≤ 1.5 GiB blocks, and a point's global index is decoded into a
`(chunk, offset)` pair with a shift and a mask. Every field shares the same
points-per-chunk, so that split is identical across all four arrays. Total size
is then bounded only by available memory, not the per-buffer cap. The point cloud
is `frames × resolution² × per-point bytes` (at the widest color space), so cost
grows with the frame count and the *square* of the Resolution slider. Before each
re-extract the old buffers are released first so the transient peak stays near 1×
rather than 2×, and the allocation is wrapped so a genuine out-of-memory failure
aborts cleanly with a "reduce Frames or Resolution" message instead of crashing
the tab.

**Memory estimate.** The **Est. memory** readout sizes the four point-cloud
arrays before they're allocated: `frames × width × height × (N·4 + 19)` bytes,
where `N` is 6 in RGB/HSV/HSL and 4 in Mono (so 43 or 35 bytes per point —
`points` at `N·4`, plus `rPos` 12, `rCol` 3, `timeOrder` 4). Width/height come from
the source's aspect ratio scaled to the pending Resolution, and the frame count
is the pending slider value (or the full count in All-frames mode) — all read
live from the controls, so the number moves *while you drag* rather than only on
release. It assumes the worst case that every pixel is visible (fully opaque),
which matches video and slightly over-counts GIFs with transparency. It counts
only the point cloud (the thing that OOMs), not the smaller held frame buffers.

**Normalization.** Spatial coords are divided by `max(w, h)`; time is the frame
index mapped to [-0.5, 0.5]; colors are centered/normalized as in the 5D
rotator. In Mono mode a single luminance value `0.299·R + 0.587·G + 0.114·B`
(Rec. 601) is centered at 128 and divided by 256. The **Time wt** and **Color
wt** sliders scale their axes before rotation.

**Time slab.** After rotation, a point is drawn only if its new time coordinate
lies within ±(Time thick × frame-spacing / 2) of the current playback time, so
playback marches that window across the rotated volume. Playback maps 0→1 onto the
part of the rotated time axis where that moving window stays *well-populated*, not a
fixed ±½. A rotation that tilts time into another axis spreads each frame's pixels
along the time axis, so how many points the thin slab catches peaks in the middle
and thins to almost nothing at the extremes (only a few corner pixels reach the true
min/max). Sweeping out to those ends would fade the frame to a near-blank for part
of every cycle. To prevent that, a histogram of the rotated time is built and the
sweep is trimmed to the range over which the slab always captures at least ~20% of
its peak population — what the eye sees is the slab's *catch*, which depends on the
slab width, not raw point density. (The threshold is deliberately modest: trimming
the sweep also raises how long each point dwells inside the moving window, so a
tighter trim would keep runs of pixels — often out-of-gamut magenta — on screen for
too much of the cycle; ~20% is the knee that keeps the ends visible without that
lingering.) A uniform distribution — such as the identity,
where every frame is one dense spike of pixels — never drops much below half its
peak even at the ends, so its range is left at exactly ±½·(Time wt) and ordinary
scrubbing and playback are unchanged.

**Continuous (gap-free) splats.** A source sample isn't really a bare point —
it's a cell in the full grid of (x, y, t) *and* colour. After rotation each
sample's neighbours land somewhere else on screen, so the renderer stretches
every splat to meet them instead of drawing a fixed square. For dense (opaque)
clips the renderer knows each pixel's exact grid neighbours (+x, +y, +t) and
reads their *already-rotated* screen positions straight from the point-cloud
cache — and because those cached positions fold in the colour rotation too, the
bridge closes gaps opened by **any** slider, spatial *or* colour. Tip the time
axis into a screen direction (an `X↔T`/`Y↔T` plane) and each cell stretches to
cover the background stripes a square splat would leave; rotate a colour axis
into screen space (`X↔R`, `X↔H`, …) and cells spread apart by colour still tile
seamlessly rather than scattering into a sparse point cloud. At identity every
splat collapses back to the original one-pixel square, so ordinary playback stays
pixel-crisp. Extremely stretched cells (very few frames combined with a strong
rotation and high **Time wt** or **Color wt**) are clamped to a maximum footprint
to protect the frame rate. Sparse clips (transparent GIFs, where samples don't
fill a regular grid) fall back to an edge-vector bounding box built from the
rotation matrix, which still bridges **x, y, and time**; **Dot scale** remains
the master knob for splat size (values below 1 open pointillist gaps everywhere;
above 1 fatten every splat).

**GIF decoding.** Animated GIFs use the browser's native WebCodecs
`ImageDecoder` where available (Chrome, Edge, Firefox) and fall back to a
self-contained pure-JS GIF decoder (LZW + full frame-disposal/transparency
compositing) everywhere else, so **Safari** works too. No external libraries.

**Video export.** `Export video` has two paths, both dependency-free. The live
animation loop is suspended and the exporter drives `playPhase` from 0 to 1 in
`round(30 × sweepSeconds)` steps (clamped 30–1800, where `sweepSeconds` follows
the **Speed** setting), so the clip is one seamless time-loop at the same speed
as the preview; if Spin is active the rotation is advanced and re-transformed per
frame as well.

*Preferred — WebCodecs.* Where the browser exposes `VideoEncoder`/`VideoFrame`
(Chrome, Edge), each rendered frame is drawn to an even-dimensioned scratch
canvas, wrapped in a `VideoFrame`, and handed to a hardware/software H.264
encoder (`avc1.640028`→…→`avc1.42001f`, first supported wins) in `realtime`
latency mode so decode order equals presentation order (no B-frame reordering,
so no `ctts` box is needed). Encoder backpressure is respected via
`encodeQueueSize`, and the emitted chunks are muxed into a minimal ISO-BMFF
**MP4** by a hand-written muxer — `ftyp`/`mdat`/`moov` with a single-chunk sample
table (`co64` 64-bit offset, one-run `stsc`, constant `stts`, keyframe `stss`,
per-sample `stsz`) and an `avc1`/`avcC` sample entry whose parameter sets come
from the encoder's `decoderConfig.description`. This path is faster than real
time and frame-exact. No external muxing library.

*Fallback — MediaRecorder.* Where WebCodecs H.264 isn't available, the canvas is
recorded with `MediaRecorder` fed by `canvas.captureStream()`. Frames are pushed
deterministically via `track.requestFrame()` when supported (otherwise the stream
auto-samples at 30 fps), paced in real time to 30 fps. The container is chosen by
capability — MP4/H.264 first, then WebM (VP9/VP8) — and the bitrate scales with
canvas area. A missing `MediaRecorder`, a failed recorder start, or an empty
recording is reported in the info bar.

## License

Public domain. Do whatever you want with it.
