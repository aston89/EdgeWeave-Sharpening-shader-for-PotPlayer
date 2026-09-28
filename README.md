
## Table of Contents

1. [EdgeWeave: sharpening shader for PotPlayer and other Media Players](https://github.com/aston89/EdgeWeave-Sharpening-shader-for-PotPlayer/tree/main#edgeweave-sharpening-shader-for-potplayer-and-other-media-players)
2. [EdgeWeave Sharpen LX](https://github.com/aston89/EdgeWeave-Sharpening-shader-for-PotPlayer/tree/main#edgeweave-sharpen-lx)
3. [Standard vs LX](https://github.com/aston89/EdgeWeave-Sharpening-shader-for-PotPlayer/blob/main/README.md#standard-vs-lx)



# EdgeWeave: sharpening shader for PotPlayer and other Media Players

EdgeWeave Sharpen is not a traditional sharpening filter, it's a **structure-aware reconstruction enhancer shader** that prioritize perceptual clarity and stability over raw sharpness amplification.
It is intended for live video playback enhancement in media players that expose a pixel shader hook in the rendering pipeline (e.g. PotPlayer, MPC-HC variants, etc.).
It sits between classical sharpening and edge-aware reconstruction filtering, focusing on preserving visual integrity rather than maximizing contrast.
Designed for legacy DirectX 9 / Pixel Shader 3.0 video pipelines.

## What it is
EdgeWeave Sharpen is a **per-frame, non-temporal, edge-gated sharpening filter**.
Unlike traditional sharpening filters that uniformly amplify high-frequency contrast, EdgeWeave selectively reconstructs perceived detail based on local edge structure and signal stability.

The goal is not to "increase sharpness" but to improve **perceptual clarity without introducing artifacts such as ringing, halos, or edge inflation**.

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

## Limitations
EdgeWeave Sharpen is intentionally constrained by design:
- Limited performance scaling on extremely low resolution content (≤480p)
- Cannot fully recover detail that does not exist in source signal

At very high strength values, the filter may still introduce:
- mild edge thickening
- structural exaggeration on high-contrast transitions

## Performance and structure
- Shader model: Pixel Shader 3.0 (ps_3_0)
- API target: DirectX 9 class pipeline
- Input: single texture sampler (s0)
- Constants: screen width/height, texel size
- No compute shaders, no multi-pass accumulation, no temporal storage.

## Update V2 (2026/05/11)

- Added edge coherence analysis to detect unreliable / diagonal edges and reduce over-sharpening in those regions
- Introduced staircase (aliasing) detection to suppress sharpening on stair-stepped diagonal patterns (hair, fine lines, textures)
- Reworked edge response using a sigmoid-based curve for smoother and more natural edge weighting
- Split detail processing into micro and macro components to better preserve structure while reducing noise amplification
- Improved stability of sharpening gain through combined coherence + stair-aware modulation
- Added edge-dependent clamp scaling to reduce artifacts in flat regions while preserving detail in strong edges
- Refined diagonal edge handling with a lightweight SMAA-inspired micro blend to reduce perceived aliasing without full blur/AA pass

---

# EdgeWeave Sharpen LX

**EdgeWeave Sharpen LX** is a more aggressive and lightweight variant of the original EdgeWeave Sharpen shader.
While the standard version focuses on conservative edge-aware sharpening, LX changes the way sharpening detail is extracted and applied. Instead of relying mainly on a general local detail estimate, LX uses **directional second-derivative information** to sharpen along the locally detected structure of an edge.

### How LX differs from the standard version

The standard EdgeWeave Sharpen pipeline is roughly:

```text
Sobel edge detection
        ↓
edge confidence
        ↓
coherence / stair-step analysis
        ↓
local detail extraction
        ↓
adaptive sharpening
        ↓
anti-ringing protection
```

The LX variant keeps the same edge-aware philosophy, but changes the sharpening stage:

```text
Sobel edge detection
        ↓
edge confidence
        ↓
local edge orientation
        ↓
directional detail extraction
        ↓
adaptive directional sharpening
        ↓
anti-ringing protection
```

Instead of simply increasing the contrast of a local high-pass signal, LX estimates whether the structure is primarily horizontal, vertical, or diagonal and applies the sharpening response along the corresponding direction.

This allows LX to enhance:

* fine edges and line structure more decisively
* thin high-contrast details
* diagonal structures without automatically treating them as unstable
* local detail while reducing unnecessary sharpening in flat or noisy regions

### Why LX can look sharper while using less processing

The LX implementation removes some of the more expensive operations used by the original shader, including the square-root gradient magnitude and exponential edge curve.

The edge response is built from simpler operations such as:

```text
abs
add
multiply
lerp
saturate
min / max
step
```

The shader still uses the same 3×3 neighborhood, so it does not achieve its lighter processing by throwing away spatial information. Instead, it uses the existing samples more directly.

### Sharpening philosophy

LX is **not a conventional unsharp-mask sharpener**.

It does not simply perform:

```text
output = image + (image - blur) × strength
```

The sharpening signal is derived from the local structure and its orientation. Edge confidence, detail thresholding, stair-step suppression, and local anti-ringing limits are still part of the process.

The result is intended to be more incisive than the standard EdgeWeave Sharpen while remaining selective about *where* sharpening is allowed to occur.

---

# Standard vs LX

|                              | EdgeWeave Sharpen        | EdgeWeave Sharpen LX          |
| ---------------------------- | ------------------------ | ----------------------------- |
| Sharpening style             | Conservative             | More aggressive               |
| Edge awareness               | Yes                      | Yes                           |
| Orientation awareness        | Limited                  | Directional                   |
| Diagonal handling            | Conservative suppression | Directional reconstruction    |
| Detail extraction            | Local blur/detail        | Directional second derivative |
| Expensive `sqrt()` / `exp()` | Yes                      | No                            |
| Processing cost              | Higher                   | Lower                         |
| Intended character           | Soft / natural           | Crisp / incisive              |

Both variants are intentionally kept in the project because they target different visual preferences. **Standard** prioritizes a softer and more conservative result, while **LX** prioritizes stronger structural definition with a lighter shader implementation.

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

## Renderer considerations
Best results are obtained when:
- Direct3D 9 / legacy pipeline is used
- pixel shader stage is fully active (no hardware bypass overlay path)
- hardware video acceleration does not bypass final shader stage

If no visible effect is observed, ensure:

- pixel shader support is enabled in renderer settings
- only one shader chain is active (avoid override conflicts)

## Compatibility
EdgeWeave Sharpen can also be used in:
- MPC-HC / MPC-BE (pixel shader support enabled and .txt changed into .hlsl)
- any DirectX 9 compatible video renderer exposing PS_3_0 hooks
- shader injection pipelines that emulate DX9-style post-processing
