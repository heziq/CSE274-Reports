# Milestone 1 Progress Update

## Project Overview

This project implements a **volumetric self-transfer** system for image-based relighting, extending Precomputed Radiance Transfer (PRT) to participating media such as fog and clouds. The core innovation is decoupling light transport from lighting by precomputing how a volume attenuates light from all directions, then reusing this precomputed response at runtime with dynamic HDR environment maps. Unlike traditional volumetric rendering, which computes shadowing every frame, this approach moves self-shadowing computation entirely to precomputation, enabling real-time relighting using low-frequency spherical harmonics (SH) representations.

## Completed Work (Stages 1–4)

### Stage 1: Volume Rendering Foundation
- Implemented ray marching in OpenGL fragment shaders with analytic density field
- Ellipsoid SDF model with exponential falloff for foggy appearance
- Beer-Lambert transmittance and alpha compositing
- Early ray termination optimization when transmittance falls below threshold
- Result: visible volumetric ellipsoid rendered in real-time

### Stage 2: Local Directional Lighting
- Single hand-written directional light with Lambertian shading
- Approximate surface normals (normalized position vector for ellipsoid geometry)
- Configurable light intensity, direction, and fog color
- Verified correct light direction mapping to surface brightness variation

### Stage 3: Spherical Harmonics Lighting
Replaced directional lighting with environment-style lighting using SH coefficients:

**Why Spherical Harmonics?**
- Signal representation for lighting functions $L(\omega)$
- Enable fast runtime evaluation via dot product instead of many ray traces
- Low-frequency nature suits volumetric scattering well

**Implementation Details:**
- **Order 3 (9 coefficients)** chosen for balance between quality and performance
- Coefficients represent order 0 (ambient), order 1 (directional), and order 2 (quadratic variations)
- GLSL implementation: `evalSHBasis(vec3 dir, out float basis[9])` computes basis functions
- SH reconstruction: $L(\omega) = \sum_{i=0}^{8} L_i \cdot Y_i(\omega)$
- 9 vec3 uniforms (27 floats total) for RGB SH coefficients
- Clamped to non-negative values (lighting cannot be negative)

**Visual Result:** Softer, more ambient illumination from all directions rather than a single light source

### Stage 3.5: HDR Environment Maps
- Python pipeline (Jupyter notebook) to project HDR images onto SH coefficients
- Offline preprocessing: HDR → SH JSON files in `assets/precompute/sh_lighting/`
- ImGui integration for interactive runtime environment switching
- Auto-discovery of SH JSON files at startup
- Color shifts visible (blue sky, warm sunlit ground) demonstrate image-derived lighting
- **Key insight:** Lighting clearly originates from environment images, not hand-tuned values

### Stage 4: Volumetric Self-Transfer (PRT Core)

This is the main contribution. Self-transfer encodes how each voxel in the volume blocks light from different directions.

**Precomputation Pipeline (Offline, Python):**
- Grid resolution: 64³ voxels spanning $[-1.2, 1.2]^3$
- Direction sampling: 512 quasi-uniform directions via Fibonacci sphere
- For each voxel $p$:
  - March rays outward along each direction $\omega_j$
  - Accumulate Beer-Lambert transmittance: $T(p, \omega_j) = \exp(-\sigma \int \text{density} \, dt)$
  - Project transmittance onto SH: $T_{\ell m}(p) = \frac{4\pi}{N} \sum_{j=1}^{N} T(p, \omega_j) \cdot Y_{\ell m}(\omega_j)$
- Output: Binary file `transfer_64x64x64x9.bin` (~9.4 MB, float32)
- Device support: CUDA, Apple Silicon MPS, or CPU fallback

**Transfer Data Characteristics:**
- **DC coefficient ($T_0^0$):** Range [1.09, 3.48]
  - Center voxel ≈ 1.09 (only ~31% transmittance, heavy self-shadowing)
  - Corner voxels ≈ 3.48 (near theoretical max)
- **Directional coefficients (SH 1–8):** Magnitude ~1.2, zero-mean
  - Encode directional asymmetry of blocking
  - Strongest at surface, weakest at center (uniform occlusion)

**Runtime Rendering (C++ / GLSL):**
- Load binary → pack into 3 × `GL_TEXTURE_3D` (RGB32F, 64³ each)
  - Texture 0: SH coefficients 0, 1, 2 → R, G, B channels
  - Texture 1: SH coefficients 3, 4, 5 → R, G, B channels
  - Texture 2: SH coefficients 6, 7, 8 → R, G, B channels
- Sample with hardware trilinear interpolation for smooth spatial variation
- **PRT rendering equation:**
  $$L_{\text{inscatter}}(p) = \frac{1}{4\pi} \sum_{i=0}^{8} \frac{L_{\text{stored}}^{(i)}}{A_\ell} \cdot T_i(p)$$
  
  Where:
  - $L_{\text{stored}}^{(i)}$ are SH lighting coefficients (includes Lambertian kernel)
  - $A_\ell$ are kernel factors: $A_0 = \pi$, $A_1 = 2\pi/3$, $A_2 = \pi/4$
  - Dividing by $A_\ell$ removes Lambertian kernel (incorrect for volumetric scattering)
  - $\frac{1}{4\pi}$ is isotropic phase function for volumetric scattering
  - $T_i(p)$ are sampled transfer SH coefficients from 3D textures

**Why Undo the Lambertian Kernel?**
- The precomputed SH lighting coefficients include a Lambertian cosine kernel
- This kernel is correct for diffuse surface irradiance rendering
- But for volumetric PRT, the angular integration is already in the transfer function
- Applying the kernel twice would inflate DC by 3.14× and distort band ratios, causing oversaturation
- Solution: Divide each SH band by its $A_\ell$ factor in the shader

**Auto-Exposure Normalization:**
- Different HDR environments vary enormously in total energy (e.g., meadow vs dim scenes: 20×+ difference)
- Auto-exposure: `effectiveExposure = 15.0 / length(L_DC)`
- Normalizes all environments to produce similar brightness (~0.7 before tone mapping)
- User's exposure slider multiplies on top of this auto-normalization

**Debug Visualization Modes:**
- **PRT Normal:** Full pipeline with Lambertian kernel removal, phase function, and auto-exposure. Final output showing colored, directional self-shadowing.
- **Transfer DC:** Raw self-shadow map showing precomputed DC coefficient, normalized to [0,1]. Dark center, bright edges verify correct precomputation.
- **PRT Raw:** Same as PRT Normal but without phase function (~12.6× brighter). Verifies phase function correctness.

## Technical Stack

**Runtime:**
- C++ with OpenGL 3.3 Core
- GLSL fragment shaders for ray marching and PRT
- GLFW for window and input management
- GLM for mathematics
- Dear ImGui for interactive parameter control

**Precomputation:**
- Python with NumPy and PyTorch (Jupyter notebooks)
- CUDA/MPS support for GPU acceleration

**Build:**
- CMake 3.10+ with Homebrew dependency detection
- External GLAD for OpenGL function loading

## Next Steps

### Immediate (Finishing Stage 5)
- **Comparative Analysis:** Quantitatively compare rendering quality across different configurations
  - SH lighting with vs without self-transfer
  - Different SH orders (4 vs 9 vs 16 coefficients) and their impact on quality
  - Different voxel resolutions (32³ vs 64³ vs 128³) and performance scaling
- **HDR Environment Variety:** Test with diverse lighting conditions (directional sunlight, diffuse cloudy sky, mixed indoor/outdoor)
- **Failure Case Documentation:** Identify and document scenarios where low-frequency SH representation breaks down (e.g., strong high-frequency lighting patterns)
- **Performance Analysis:** Measure frame rates, precomputation time, and memory usage across configurations

### IMR-Focused Exploration (Advanced)
- **Multiple Scattering:** Extend single-scattering PRT to include second-order bounces. Self-transfer currently accounts only for occlusion along outgoing rays; multiple scattering would capture light bouncing between volume elements.
- **Anisotropic Phase Functions:** Replace isotropic $\frac{1}{4\pi}$ with Henyey-Greenstein or other phase functions to model forward/backward scattering more realistically.
- **Temporal Coherence & Motion:** Explore precomputing directional transfer for multiple volume poses/deformations, enabling animated self-shadowing without recomputation.
- **Frequency Analysis:** Study which SH bands contribute most to self-shadowing. Can we predict perceptually important bands without computing all 9?
- **Hybrid Approaches:** Compare against lightweight alternatives:
  - Normal-based SH (no precomputation, surface-only approximation)
  - Per-voxel cone tracing for dynamic occlusion
  - Neural networks to compress or predict transfer functions

## Questions for Discussion

1. **Exploration Direction:** Should I prioritize completing Stage 5 evaluation first, or would you recommend diving into multiple-scattering extension? What would be more valuable for IMR community?

2. **SH Order Tradeoff:** Is there literature or guidance on predicting the minimum SH order needed for a given scene's lighting complexity without exhaustive testing?

3. **Volume Deformation:** If the ellipsoid shape changes at runtime (e.g., animated clouds), would it be feasible to precompute transfer for multiple keyframe poses and interpolate, or does this break the method fundamentally?

4. **Validation:** For comparing against ground truth, would ray-traced volumetric shadows per frame be the best reference, or is there a faster offline method?

5. **Practical Limitations:** Are there known failure cases in PRT-style methods when applied to volumetric media that I should be aware of?

## Visual Placeholders

![EXR render showing volumetric self-shadowing with directional environment lighting](path/to/exr_render.exr)

![Comparison of different visualization modes: PRT Normal vs Transfer DC vs PRT Raw](path/to/visualization_modes.png)

![Transfer DC coefficient visualization showing self-shadow intensity variation](path/to/transfer_dc.png)