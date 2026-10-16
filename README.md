# Colexchange — WebGPU Color-Transfer Console
 
Colexchange is a single-file, browser-based tool that **extracts the color look of one video and bakes it into a 3D LUT**, then applies that LUT to another video in real time. Everything runs locally on your GPU via WebGPU. No frames or files ever leave the page.
 
---
 
## Features
 
- **Look extraction:** drop a source clip and its color distribution is sampled on the GPU and baked into a 3D LUT.
- **Real-time preview:** the LUT is applied to your target footage in a WebGPU program monitor.
- **Adaptive transfer:** optionally re-maps the look onto the target clip's own statistics for a more natural match.
- **Split / Original / Graded views:** with a draggable wipe line for before/after comparison.
- **Intensity control:** blend the LUT from 0% to 100%.
- **LUT export:** save as a `.cube` file (works in most NLEs and grading tools) or as a PNG preview ribbon.
- **LUT import:** load any `.cube` file and preview it on your target as-is.
- **Analysis scopes:** log-LMS histograms plus stats chips (exposure, contrast, temperature, tint, resolution, duration).
- **Selectable LUT resolution:** 17³, 33³, 48³ or 64³.
---
 
## Requirements
 
- A browser with **WebGPU** support: Chrome, Edge or Opera 113+, or Chrome on Android.
- If WebGPU isn't enabled by default, try `chrome://flags` → `#enable-unsafe-webgpu`.
- A video format your browser can decode (mp4, webm, mov, m4v).
- Internet access is only needed to load the Google Fonts (Chakra Petch, IBM Plex Mono). The tool works with fallback fonts otherwise.
---
 
## Getting Started
 
1. Open `index.html` in a supported browser. No build step or dependencies.
   - If your browser blocks WebGPU on `file://`, serve it locally, e.g. `python -m http.server`, then open `http://localhost:8000`.
2. **Module 01 — Extract Look:** drop (or click to choose) the clip whose look you want to copy.
3. **Module 03 — Apply To:** drop the clip you want to grade.
4. **Module 02 — The Look:** adjust resolution, intensity and adapt mode, then export if you like.
5. Use the **Program Monitor** to compare the original and graded result.
---
 
## Interface Overview
 
| Module | Purpose |
|---|---|
| **01 Extract Look** | Loads source footage, samples 14 frames across its duration, shows a filmstrip, histogram scope and stats chips. Includes *Re-analyze* and *Eject*. |
| **02 The Look** | Shows a ribbon preview (hue ramp on top, grayscale ramp below) through the LUT. Controls: LUT resolution, intensity, adapt toggle. Export `.cube` / PNG, import `.cube`. |
| **03 Apply To** | Loads target footage, samples 8 frames, shows its scope and stats. Includes *Re-analyze* and *Eject*. |
| **Program Monitor** | Live WebGPU preview with Original / Split / Graded modes and transport controls (play, mute, scrub, loop). |
 
### Keyboard & Mouse
 
| Input | Action |
|---|---|
| `Space` | Play / pause |
| `1` / `2` / `3` | Original / Split / Graded view |
| `←` / `→` | Scrub ∓2 seconds |
| Click-drag on monitor | Move the wipe line (switches to Split) |
| Double-click monitor | Toggle Split ↔ Graded |
 
---
 
## How It Works
 
1. **Sampling.** Frames are evenly sampled from the clip, downscaled (max 1024 px) and uploaded to a GPU texture. A compute shader draws 120,000 pseudo-random pixels per frame and accumulates them into per-channel histograms with atomic adds.
2. **Color space.** Pixels are linearized from sRGB, converted to **LMS** (Hunt-Pointer-Estevez matrix) and log10-encoded, following the statistics-based color transfer of **Reinhard et al. (2001)**.
3. **Statistics.** Mean (μ) and standard deviation (σ) of each log-LMS channel are computed from the histograms.
4. **LUT baking.** For each grid point of the 3D LUT, the input color is moved into log-LMS, shifted and scaled from the target's (or a neutral) distribution to the source's distribution, then converted back to sRGB. Scale ratios are clamped to 0.2–5× for stability.
5. **Adapt mode.** With *Adapt transfer to target stats* on and a target loaded, the transfer starts from the target's own μ/σ. Otherwise it starts from a neutral baseline (mid-gray 0.18, σ 0.45).
6. **Playback.** Each video frame is copied into a texture and a fragment shader samples the 3D LUT with trilinear filtering, blending by intensity. A subtle vignette is applied to the monitor view.
---
 
## Export Formats
 
- **`.cube`**: standard `LUT_3D_SIZE` file with `DOMAIN_MIN/MAX` 0–1. The domain is **display-referred, sRGB-encoded**, so it is intended for Rec.709/sRGB-style workflows and not for log footage.
- **PNG**: a ribbon image (hue ramp over grayscale ramp) rendered through the LUT, useful as a quick visual reference.
When a `.cube` is imported, it's applied as-is and the resolution/adapt controls are locked until you load new source footage.
 
---
 
## Limitations & Notes
 
- Operates on the video's decoded, display-referred frames. It is not color-managed for log, HDR or wide-gamut sources.
- The LUT is uploaded to an 8-bit `rgba8unorm` 3D texture for preview, so the on-screen preview is quantized to 8 bits per channel. The exported `.cube` keeps full float precision.
- Transfer is global and statistical, so it matches overall tone and color balance, not local or spatial looks.
- Sampled frames are drawn from evenly spaced seek points, so very short or heavily varying clips may give a less representative look. Use *Re-analyze* if needed.
- Codec support depends on your browser.
- Audio is muted by default; use the speaker button to toggle.
---
 
## Project Structure
 
```
index.html   # Entire app: HTML, CSS, JS and WGSL shaders in one file
```
 
Key parts inside `index.html`:
 
- `WGSL_ANALYZE` — compute shader for histogram sampling
- `WGSL_APPLY` — fragment shader applying the 3D LUT to video
- `WGSL_RIBBON` — LUT preview ribbon
- `WGSL_PATTERN` — "no signal" color-bar idle screen
- `bakeLUT()` — builds the 3D LUT from source/target statistics
- `analyze()` — frame sampling and GPU analysis pipeline
- `exportCube()` / `importCube()` — `.cube` I/O
---
 
## Privacy
 
All processing happens locally in your browser. Videos are never uploaded.
 
---
 
## Credits
 
Color transfer method: E. Reinhard, M. Adhikhmin, B. Gooch, P. Shirley, *"Color Transfer between Images"*, IEEE Computer Graphics and Applications, 2001.
