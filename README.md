# photo-fx

A single-file, GPU-accelerated photo effects playground. Everything — UI, shaders,
and rendering — lives in `index.html` and runs entirely in the browser via WebGL/GLSL.
No build step, no external libraries, no network calls.

**Live app:** https://ebbanflo.github.io/photo-fx

## Usage

Open `index.html` in any WebGL-capable browser (or visit the live URL above). A
placeholder sunset image loads by default so you can see the effects immediately.

- **Load Photo** — pick an image from disk, or drag & drop one onto the canvas.
- **Sliders** — each effect has its own group of sliders in the right-hand panel,
  applied live every frame.
- **Randomize** — throws random values across every slider at once for instant,
  unpredictable looks.
- **Reset** — restores all sliders to their default values.
- **Export PNG** — renders the current frame and downloads it as a PNG at the
  image's full resolution.

## How it works

All effects are implemented as a single WebGL fragment shader that samples the
source image once per pixel, applying each effect in sequence: warp, chromatic
aberration, halation, color grade, vignette, grain, and glitch/scanlines. A
`requestAnimationFrame` loop redraws every frame so time-based effects (grain,
glitch bursts) animate continuously. Images are downscaled client-side to a
2048px max dimension before being uploaded as a texture, keeping the shader fast
on large photos.

## Effects

**Chromatic Aberration** — Splits the red and blue channels away from green,
offset radially from the image center and scaled by distance from center, mimicking
lens dispersion at the edges of a frame.
- *Amount* — strength of the per-channel offset.

**Halation** — Simulates the warm red/orange glow that bleeds from bright
highlights on film stock (caused by light bouncing back through the emulsion).
Implemented as a ring-sampled bloom: bright pixels above a threshold are
gathered from neighboring texels, tinted warm, and added back into the image.
- *Threshold* — luminance level above which pixels start to bloom.
- *Intensity* — strength of the bloom added back into the image.
- *Radius* — how far the bloom sampling ring reaches, in texel multiples.

**Animated Film Grain** — Per-pixel-block noise that re-randomizes ~24 times a
second, similar to the flicker of real photographic grain.
- *Intensity* — strength of the noise added to the image.
- *Grain Size* — size of each noise block, in pixels.

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
coordinates before the image is read, twisting pixels around the center like a
liquify brush.
- *Amount* — swirl strength and direction (positive/negative rotate opposite ways).
- *Radius* — how far from center the swirl effect reaches.

**Glitch / Scanline Corruption** — Digital-corruption look combining random
horizontal slice displacement, per-line RGB channel swapping, and CRT-style
scanline darkening, all driven by discrete time steps for a stuttering, glitchy
feel rather than smooth animation.
- *Glitch Intensity* — frequency/strength of slice displacement and channel corruption.
- *Scanlines* — strength of alternating horizontal line darkening.

## Tech notes

- Pure HTML/CSS/JavaScript, no frameworks, no bundler, no CDN dependencies.
- Rendering uses raw WebGL1 (`canvas.getContext("webgl")`) with a single
  full-screen triangle-strip quad and one fragment shader.
- `preserveDrawingBuffer: true` is set on the context so the canvas can be
  read back directly for PNG export via `canvas.toBlob`.
- Tested with Chromium/WebGL (including software rendering via SwiftShader).
