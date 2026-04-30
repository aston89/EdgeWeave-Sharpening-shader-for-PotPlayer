# EdgeWeave: sharpening shader for PotPlayer and other Media Players

EdgeWeave Sharpen is not a traditional sharpening filter, it's a **structure-aware reconstruction enhancer shader** that prioritize perceptual clarity and stability over raw sharpness amplification.
It is intended for live video playback enhancement in media players that expose a pixel shader hook in the rendering pipeline (e.g. PotPlayer, MPC-HC variants, etc.).
It sits between classical sharpening and edge-aware reconstruction filtering, focusing on preserving visual integrity rather than maximizing contrast.
Designed for legacy DirectX 9 / Pixel Shader 3.0 video pipelines.

---

## What it is
EdgeWeave Sharpen is a **per-frame, non-temporal, edge-gated sharpening filter**.
Unlike traditional sharpening filters that uniformly amplify high-frequency contrast, EdgeWeave selectively reconstructs perceived detail based on local edge structure and signal stability.

The goal is not to "increase sharpness" but to improve **perceptual clarity without introducing artifacts such as ringing, halos, or edge inflation**.

---

## What it is useful for
EdgeWeave Sharpen is designed for:
- Anime and animation content (clean line preservation)
- Film and cinematic content (grain-safe enhancement)
- Compressed video sources (artifact-aware refinement)
- Low-to-mid resolution upscaled playback (720p to 1080p / 4K displays)

It's especially effective in cases where traditional sharpening produces:
- halo artifacts
- edge overshoot
- noisy texture amplification
- overly harsh contrast transitions

---

## How it works
EdgeWeave Sharpen operates entirely in a **DX9 Pixel Shader 3.0 (PS_3_0) pipeline**.
Core characteristics:
- 9-tap local sampling (3x3 neighborhood)
- Sobel-style gradient estimation
- Edge gating (activation based on structural confidence)
- Local mean reconstruction baseline
- Detail extraction instead of global contrast boosting
- CAS-inspired local clamp limiting to prevent overshoot

The shader operates strictly **per frame**.

### Pipeline stages:
1. Luma conversion (perceptual weighting)
2. Gradient computation (edge strength + direction)
3. Local structure estimation
4. Detail extraction (high-pass relative to local blur estimate)
5. Edge-gated sharpening application
6. Clamp-based reconstruction stabilization

---

## Key difference vs traditional sharpening

Most sharpening filters (CAS-like, unsharp mask, Laplacian-based):
- apply a fixed high-pass amplification
- treat all pixels equally regardless of context
- risk haloing and ringing when pushed
- do not distinguish between noise and structure

EdgeWeave Sharpen instead:
- activates sharpening only where structural confidence is high
- suppresses amplification in flat or noisy regions
- adapts strength based on local edge reliability
- avoids artificial edge thickening
- prioritizes perceptual clarity over raw contrast increase

Result:
> less “crispy but broken”, more “clear but stable”

---

## Limitations
EdgeWeave Sharpen is intentionally constrained by design:
- Limited performance scaling on extremely low resolution content (≤480p)
- Cannot fully recover detail that does not exist in source signal

At very high strength values, the filter may still introduce:
- mild edge thickening
- structural exaggeration on high-contrast transitions

---

## Performance and structure
- Shader model: Pixel Shader 3.0 (ps_3_0)
- API target: DirectX 9 class pipeline
- Input: single texture sampler (s0)
- Constants: screen width/height, texel size
- No compute shaders, no multi-pass accumulation, no temporal storage.

---

## Usage in PotPlayer
EdgeWeave Sharpen is compatible with PotPlayer’s built-in pixel shader pipeline.

### Recommended placement:

**Post-resize (recommended default)**
Best used after scaling because:
- image is already upscaled and softened by interpolation
- sharpening operates on final pixel grid
- reduces risk of aliasing amplification

This is the most stable and visually consistent configuration.

**Pre-resize (advanced use)**
Can be used before scaling when:
- source is already high quality (Blu-ray, clean anime encodes)
- scaling algorithm is high quality (Lanczos, Bicubic sharp)

However:
- sharpening may be partially altered by subsequent scaling
- edge reconstruction may be slightly less stable

---

## Renderer considerations
Best results are obtained when:
- Direct3D 9 / legacy pipeline is used
- pixel shader stage is fully active (no hardware bypass overlay path)
- hardware video acceleration does not bypass final shader stage

If no visible effect is observed, ensure:

- pixel shader support is enabled in renderer settings
- only one shader chain is active (avoid override conflicts)

---

## Compatibility
EdgeWeave Sharpen can also be used in:
- MPC-HC / MPC-BE (pixel shader support enabled and .txt changed into .hlsl)
- any DirectX 9 compatible video renderer exposing PS_3_0 hooks
- shader injection pipelines that emulate DX9-style post-processing
