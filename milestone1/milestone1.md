# Progress Report: Volumetric Self-Transfer for Real-Time Relighting

**Project:** Volumetric Precomputed Radiance Transfer (PRT) in Participating Media  
**Student:** [Your Name]  
**Date:** February 10, 2026

---

## 1. Project Overview
This project implements a **Volumetric Self-Transfer** system inspired by Sloan et al.’s *Precomputed Radiance Transfer (PRT)*. The goal is to achieve real-time relighting of participating media—specifically clouds or fog—under dynamic HDR environment maps. By decoupling light transport from the lighting environment, the expensive computation of volumetric self-shadowing is moved to an offline precomputation phase, enabling interactive lighting exploration at runtime.

## 2. Technical Implementation Progress

### 2.1 Ray Marching and Density Modeling (Completed)
I have implemented a GPU-based ray marcher within a fragment shader. 
* **Density Field:** The volume is defined by an analytic ellipsoid Signed Distance Function (SDF) with an exponential falloff to simulate a realistic "fuzzy" boundary.
* **Transmittance:** Light attenuation is calculated via the Beer-Lambert Law.
* **Optimization:** The system utilizes ray-ellipsoid intersection for early optimization and early ray termination once transmittance falls below a specific threshold.

### 2.2 Image-Based Lighting via Spherical Harmonics (Completed)
To represent the infinite lighting of an HDR environment map, I utilize **Spherical Harmonics (SH)**.
* **Projection:** HDR environment maps are projected into 3rd-order SH (9 coefficients) using an offline Python pipeline.
* **Runtime Reconstitution:** The shader reconstructs the lighting signal using SH basis functions implemented in GLSL, allowing for smooth, low-frequency global illumination without sampling the environment map directly per step.

> **[Space for Image: HDR Environment Maps (EXR) and their SH Projections]**

### 2.3 Volumetric Self-Transfer & PRT Core (Completed)
The core of the project involves precomputing a **Transfer Function** $T(p, \omega)$ for every voxel in a $64^3$ grid.
* **Precomputation:** For each voxel, I sample 512 directions (Fibonacci sphere) and march rays outward to calculate self-occlusion. These results are projected into SH coefficients $T_{\text{SH}}(p)$.
* **Runtime Integration:** The radiance at a point $p$ is calculated as a dot product:
    $$L_{\text{inscatter}}(p) \approx \frac{1}{4\pi} \sum_i \frac{L_{\text{stored}}^{(i)}}{A_\ell} \cdot T_{\text{SH}}^{(i)}(p)$$
* **Kernel Un-baking:** I manually remove the Lambertian cosine kernel ($A_\ell$) from the lighting coefficients in the shader to ensure the math correctly reflects volumetric scattering rather than surface irradiance.

> **[Space for Image: Output comparison showing "No Self-Transfer" vs "PRT Self-Shadowing"]**

---

## 3. Current Status & Debugging
The system is currently fully interactive using **Dear ImGui**.
* **Environment Switching:** Any HDR-derived JSON file in the assets folder can be loaded at runtime to instantly change the lighting environment.
* **Visualization Modes:** I have implemented debug modes to view the raw **Transfer DC** (self-shadow map) and **PRT Raw** (un-normalized) to verify the integrity of the 3D texture data.

---

## 4. Future Work: Experiments & IBR Exploration
I plan to dedicate the final stage to systematic evaluation and exploring the limits of this Image-Based Rendering (IBR) technique.

### 4.1 Quantitative & Qualitative Evaluation
* **Voxel Resolution:** Compare visual fidelity and memory overhead of $32^3$ vs $64^3$ vs $128^3$ transfer grids.
* **SH Order Analysis:** Evaluate if 9 coefficients are sufficient for "peaked" lighting (e.g., sunsets) or if higher orders are necessary to capture directional shadows.

### 4.2 IBR-Focused Extensions
* **Phase Function Variation:** Currently, the system uses an isotropic phase function. I plan to implement the **Henyey-Greenstein** phase function to see how forward-scattering interacts with the precomputed SH transfer.
* **Hybrid Lighting:** Explore combining the global SH lighting with a single high-frequency directional light (the Sun) to see if it resolves the "softness" limitation of low-order PRT.

---

## 5. Questions for the Professor

1.  **SH Frequency Limits:** Given that SH is inherently low-frequency, I lose the "sharp" edges of the sun. In IBR practice, is it better to move to higher SH orders (e.g., Order 5), or to treat the primary light source as a separate analytic light while using PRT for the rest of the environment?
2.  **Dynamic Geometry:** My transfer function is currently tied to a static ellipsoid density. If I were to animate the cloud's shape (deforming the SDF), the precomputation becomes invalid. Are there common "warping" techniques in PRT that allow for simple geometric deformations without re-baking the entire 3D texture?
3.  **Multiple Scattering Approximation:** My current model focuses on single scattering. Would adding a simplified "ambient" SH term to the transfer function be a valid way to approximate multiple scattering in this framework, or does that break the PRT mental model?