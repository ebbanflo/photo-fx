# photo-fx

A single-file, GPU-accelerated photo effects playground with a hardware-inspired
interface. Everything — UI, shaders, and rendering — lives in `index.html` and
runs entirely in the browser via WebGL/GLSL. No build step, no external
libraries, no network calls.

**Live app:** https://ebbanflo.github.io/photo-fx

## Interface

The UI is styled after Teenage Engineering / OP-1 hardware: an aluminum chassis,
a dark inset "screen" bezel around the canvas, and every parameter is a colored
**rotary encoder** rather than a plain slider.

- **Drag a knob** vertically to change its value.
- **Scroll** over a knob to fine-tune.
- **Double-click** a knob to reset it to its default.

Under the hood each knob drives a hidden `<input type=range>`, so it stays fully
keyboard/programmatic-friendly while looking like a physical control.

## Usage

Open `index.html` in any WebGL-capable browser (or visit the live URL above). A
placeholder sunset image loads by default so you can see the effects immediately.

- **Load** — pick an image from disk, or drag & drop one onto the screen. Files
  are decoded with EXIF orientation applied, so phone photos land the right way up.
- **Random** — throws random values across every parameter at once for instant,
  unpredictable looks.
- **Reset** — restores every parameter to its default.
- **Rec** — records the live animation to a `.webm` clip via `MediaRecorder`
  (see below). The button turns red and the screen shows a running timer; press
  again to stop and download.
- **Export** — renders the current frame and downloads it as a PNG at the image's
  full resolution.

### Presets

A preset selector sits at the top of the control surface.

- Choose a **built-in** preset to jump to a curated look — Cinematic, Dreamwave,
  Kaleido Trip, Vortex, Sculpture 3D, Corrupted, Fishbowl, Ripple Pool, and the
  3D combo presets Crystal Ball, Hyperspace, Liquid Chrome, Wormhole, and
  Melting Glass.
- **Save** captures the current knob positions as a named preset, stored in
  `localStorage` so it persists across sessions.
- **Del** removes one of your saved presets (built-ins can't be deleted).

### Recording

**Rec** captures the canvas with `canvas.captureStream(30)` and encodes it with
`MediaRecorder` (VP9/VP8 WebM). Because the animation is live, effects like the
spinning kaleidoscope, zooming tunnel, orbiting relief light, grain, and glitch
are all captured in motion. The clip downloads as `photo-fx-clip.webm`.

## How it works

All effects are implemented as a single WebGL fragment shader that samples the
source image and applies each effect in sequence: warp, kaleidoscope, tunnel,
3D relief, glass sphere, 3D ripple, chromatic aberration, halation, color grade,
vignette, grain, and glitch/scanlines. A `requestAnimationFrame` loop redraws so
time-based effects animate continuously. Images are downscaled client-side to a
2048px max dimension before being uploaded as a texture, keeping the shader fast
on large photos. Textures are uploaded with `UNPACK_FLIP_Y_WEBGL` so images
render right-side up under the full-screen-quad shader.

## Effects

**Chromatic Aberration** — Splits the red and blue channels away from green,
offset radially from the image center and scaled by distance from center, mimicking
lens dispersion at the edges of a frame.
- *Amount* — strength of the per-channel offset.

**Halation** — Simulates the warm red/orange glow that bleeds from bright
highlights on film stock. Implemented as a ring-sampled bloom: bright pixels above
a threshold are gathered from neighboring texels, tinted warm, and added back in.
- *Threshold* — luminance level above which pixels start to bloom.
- *Intensity* — strength of the bloom added back into the image.
- *Radius* — how far the bloom sampling ring reaches, in texel multiples.

**Animated Film Grain** — Per-pixel-block noise that re-randomizes ~24 times a
second, similar to the flicker of real photographic grain.
- *Intensity* — strength of the noise added to the image.
- *Size* — size of each noise block, in pixels.

**Color Grading (Lift / Gamma / Gain)** — A standard ASC CDL-style grading
curve: `color = pow(color * gain + lift, 1/gamma)`.
- *Lift* — raises or lowers shadows/black point.
- *Gamma* — reshapes midtone contrast.
- *Gain* — scales highlights/overall exposure.

**Vignette** — Darkens the image toward the edges using a radial falloff from
center, independent of any warp distortion so it stays anchored to the frame.
- *Intensity* — how dark the vignette gets.
- *Radius* — distance from center where darkening begins.
- *Softness* — width of the falloff transition.

**Warp / Liquify** — A radial swirl distortion applied to the sampling
coordinates before the image is read, twisting pixels around the center.
- *Amount* — swirl strength and direction (positive/negative rotate opposite ways).
- *Radius* — how far from center the swirl effect reaches.

**Glitch / Scanline Corruption** — Digital-corruption look combining random
horizontal slice displacement, per-line RGB channel swapping, and CRT-style
scanline darkening, all driven by discrete time steps for a stuttering feel.
- *Glitch* — frequency/strength of slice displacement and channel corruption.
- *Scanlines* — strength of alternating horizontal line darkening.

**Kaleidoscope** — Folds the sampling coordinates into N mirrored angular wedges
around the center, with a time-based rotation, turning any image into a spinning,
symmetric mandala.
- *Segments* — number of mirrored wedges (0 = off).
- *Spin* — rotation speed of the symmetry.

**Warp Tunnel (Droste)** — A log-polar transform that maps the frame into an
infinite self-similar zoom tunnel, pulling the image recursively toward the
center with an optional spiral twist.
- *Amount* — blend between the normal image and the tunnel.
- *Speed* — how fast the tunnel zooms.
- *Twist* — spiral rotation applied with depth.

**3D Relief (Heightfield)** — Treats image luminance as a 3D heightfield: it
derives surface normals from the local brightness gradient, lights them with a
light source that orbits over time, and adds height-based parallax so the photo
looks sculpted and shifts in 3D.
- *Depth* — relief strength (normal steepness + parallax).
- *Orbit* — rotation speed of the light around the surface.

**Glass Sphere (Refraction)** — Maps the image onto a 3D glass orb: it builds a
hemisphere normal per pixel, refracts the view ray through it (`refract()` at an
IOR of ~1.33), and adds a specular highlight, so the image bulges and magnifies
like it's seen through a lens or crystal ball.
- *Bulge* — refraction strength.
- *Radius* — size of the orb on screen.

**3D Ripple (Pond)** — Animated concentric waves radiating from center, treated
as a 3D water surface: the wave slope displaces the sampling coordinates
(radial lensing) while crests and troughs are shaded for a shimmering surface
that moves over time.
- *Amount* — wave height / distortion strength.
- *Rings* — spatial frequency of the waves.
- *Speed* — how fast the ripples travel outward.

## Tech notes

- Pure HTML/CSS/JavaScript, no frameworks, no bundler, no CDN dependencies.
- Rendering uses raw WebGL1 (`canvas.getContext("webgl")`) with a single
  full-screen triangle-strip quad and one fragment shader.
- `preserveDrawingBuffer: true` is set on the context so the canvas can be read
  back for PNG export (`canvas.toBlob`) and streamed for recording
  (`canvas.captureStream`).
- File uploads are decoded via `createImageBitmap(..., { imageOrientation:
  "from-image" })` to normalize EXIF rotation.
- Saved presets live in `localStorage` under `photofx_presets_v1`.
- Tested with Chromium/WebGL (including software rendering via SwiftShader).
