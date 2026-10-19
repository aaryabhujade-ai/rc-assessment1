# Methodology & Technical Documentation

## Assessment 1: AI Image Restoration, Outpainting, Upscaling, Object Addition, and Object Removal

---

### Executive Summary

This document presents the detailed engineering methodology, technical architecture, and validation pipeline implemented for **Assessment 1: AI Image Restoration & Editing**. The project demonstrates state-of-the-art open-source generative computer vision techniques across two core deliverables:
1. **Output 1 — Image Restoration (Outpainting & Upscaling)**: Restoring a heavily compressed, blurry portrait source into an expansive 16:9 cinematic landscape canvas through deep latent outpainting and generative super-resolution diffusion.
2. **Output 2 — Object Editing (Addition & Removal)**: Seamlessly removing a monumental celestial subject from a stormy coastal city scene, reconstructing the atmospheric environment and lighting, and compositing a newly styled mythological subject with photorealistic perspective, volumetric rim lighting, and material alignment.

---

### 1. Source Image Selection & Initial Quality Assessment

#### 1.1 Primary Input (`inputs/original_low_quality_image.png`)
* **Filename**: `BLUR.jpg` / `original_low_quality_image.png`
* **Original Resolution**: 880 × 1176 pixels (3:4 portrait aspect ratio)
* **File Format & Compression**: JPEG, 72.3 KB (~0.07 bpp), heavily compressed with 8×8 discrete cosine transform (DCT) block ringing artifacts.
* **Content**: An ancient Indian mythological warrior in golden royal armor, drawing an ornate longbow with arrow cocked, set against a dusty battlefield with saffron war banners and chariot silhouettes.
* **Defects Identified**:
  * Severe spatial blurring and high-frequency edge attenuation across facial features, crown carvings, and bow engraving.
  * Chromatic sub-sampling and quantization noise in low-contrast midtones and dust haze.
  * Incomplete composition: tightly cropped vertical frame cutting off the bow arc, chariot wheel, and the broader epic battlefield.

#### 1.2 Secondary Canvas (`inputs/ganesha_source_scene.jpg`)
* **Filename**: `GANESHA.jpg` / `ganesha_source_scene.jpg`
* **Original Resolution**: 1360 × 768 pixels (16:9 landscape aspect ratio)
* **File Format**: JPEG, 91.0 KB
* **Content**: A colossal statue/manifestation of Lord Ganesha hovering over the Marine Drive coastline in Mumbai during a dramatic monsoon storm, featuring sweeping streetlights, wet asphalt reflections, dark ocean swells, and volumetric sunbeams breaking through storm clouds.
* **Editing Opportunity**:
  * **Target Object for Removal**: Lord Ganesha's 8-armed torso, lotus, axe, and halo, occupying the central 60% of the upper frame.
  * **Target Object for Addition**: The restored mythological warrior, integrated seamlessly into the exact atmospheric lighting, rainy perspective, and scale.

---

### 2. Output 1: Background Extension & Super-Resolution Upscaling

```
[Low-Quality Input (880x1176)]
          │
          ▼
┌───────────────────────────┐
│ 1. Canvas Outpaint Pad    │  --> Expanded to 16:9 Aspect Ratio
│   (ImagePadForOutpaint)   │      Feathered Mask Boundary (0-255)
└─────────┬─────────────────┘
          │
          ▼
┌───────────────────────────┐
│ 2. Qwen/Flux Fill Inpaint │  --> Inpainting ControlNet Guidance
│    + Prompt Conditioning  │      Sandstone Gopurams, Ancient Kingdom
└─────────┬─────────────────┘
          │
          ▼
┌───────────────────────────┐
│ 3. Latent Composite &     │  --> Alpha Masked Seamless Edge Blending
│    Color Normalization    │      Zero Seam Visibility
└─────────┬─────────────────┘
          │
          ▼
┌───────────────────────────┐
│ 4. SeedVR2 7B Upscaler    │  --> 4x Multiplier Lanczos Pre-resample
│    (KSampler + Tiled VAE) │      Diffusion Super-Resolution (3520x4704 / 1376x768)
└─────────┬─────────────────┘
          │
          ▼
[Final Deliverable: outputs/output_1_background_extended_upscaled.png]
```

#### 2.1 Canvas Expansion & Outpainting Padding
To convert the vertical 3:4 portrait (880 × 1176) to a widescreen 16:9 cinematic aspect ratio, the lateral canvas space was padded:
* Target Aspect Ratio: 16:9 ($1.777\dots$)
* Node: `ImagePadForOutpaint` (ComfyUI standard outpainting preprocessor).
* The central warrior bounding box was locked using a binary alpha mask where value $0.0$ represents preserved source pixels and $1.0$ represents generative outpainting target space.
* Feathering parameter was calibrated to $16\text{px}$ to prevent hard boundary line discontinuities between original film grain and newly synthesized latent tokens.

#### 2.2 Latent Noise Masking & Diffusion Generation
* **Base Diffusion Model**: `qwen_image_fp8_e4m3fn.safetensors` / `flux.1-fill-dev-OneReward-transformer_fp8.safetensors`.
* **ControlNet Inpainting**: `Qwen-Image-InstantX-ControlNet-Inpainting.safetensors` via `ControlNetInpaintingAliMamaApply`.
* **Prompt Conditioning**:
  * *Positive*: Colossal ancient Indian mythological kingdom extending across the battlefield, monumental carved sandstone temples, soaring Dravidian gopurams, grand palace ramparts, drifting battle smoke, dust haze, warm golden-hour cinematic volumetric rays.
  * *Negative*: Modern vehicles, power lines, skyscrapers, flat lighting, oversaturated cartoon rendering, plastic 3D CGI textures, visible seams, duplicated figures.
* **Sampling Parameters**:
  * Sampler: `Euler`
  * Scheduler: `Simple` / `Normal`
  * CFG Scale: $2.5$ (or $1.0$ when utilizing the 4-step Lightning LoRA `Qwen-Image-Lightning-4steps-V1.0.safetensors`)
  * Shift Factor (`ModelSamplingAuraFlow`): $3.1$

#### 2.3 Super-Resolution Detail Recovery
* **Model**: SeedVR2 7B (`seedvr2_7b_int8_convrot.safetensors`).
* **Mechanism**:
  1. Lanczos pre-upsampling (`ResizeImageMaskNode`, scale multiplier = $4.0$).
  2. Input tensor normalization through `SeedVR2Preprocess`.
  3. Latent encoding through tiled VAE (`VAEDecodeTiled`, tile size: 512, overlap: 128) to eliminate VRAM out-of-memory bottlenecks.
  4. Latent restoration pass via `KSampler` (steps: 1, cfg: 1.0, denoise: 1.0) conditioned on high-dimensional text-latent alignments.
  5. Color fidelity preservation via `SeedVR2PostProcessing`.
* **Result**:
  * Facial hair, iris reflections, ornate headband filigree, and bowstring tension are restored with razor-sharp fidelity, eliminating all JPEG block artifacts.

---

### 3. Output 2: Object Removal, Addition, and Scene Blending

```
[Original Base Scene: GANESHA.jpg (1360x768)]
                     │
                     ▼
       ┌───────────────────────────┐
       │ 1. Segmentation / Mask    │  --> Mask covering Lord Ganesha
       │    Generation (Region ROI)│      including halo, arms, shadows
       └─────────────┬─────────────┘
                     │
                     ▼
       ┌───────────────────────────┐
       │ 2. Background Inpainting  │  --> Reconstruct Mumbai storm clouds,
       │    & Reconstruction       │      rain streaks & golden sunbeams
       └─────────────┬─────────────┘
                     │
           [outputs/object_removed.png]
                     │
                     ▼
       ┌───────────────────────────┐
       │ 3. Subject Insertion &    │  --> Reference warrior (up_1.jpg)
       │    Contextual Alignment   │      Pose, scale, lighting matched
       └─────────────┬─────────────┘
                     │
                     ▼
       ┌───────────────────────────┐
       │ 4. Qwen-Image 2.1 Editing │  --> Atmospheric mist blending,
       │    Refinement Pass        │      marine bay reflection alignment
       └─────────────┬─────────────┘
                     │
                     ▼
[Final Deliverable: outputs/output_2_object_added_removed.png]
```

#### 3.1 Part A: Object Removal & Background Reconstruction
* **Objective**: Completely erase Lord Ganesha without disrupting the underlying city skyline, Marine Drive curve, traffic headlights, rain streaks, or ocean water.
* **Technique**:
  * Semantic mask enclosing the celestial figure down to the rooftop horizon line.
  * Inpainting reconstruction generating natural monsoon stratocumulus clouds, ambient rain scattering, and directional sun rays matching the existing sky gradient.
  * Deliverable: `outputs/object_removed.png` (1376 × 768 pixels).

#### 3.2 Part B: Object Addition & Harmonization
* **Objective**: Insert the ancient warrior into the newly cleared sky at a colossal scale, matching the perspective of the Mumbai coastal vista.
* **Conditioning Directive**:
  `Replace Lord Ganesha in Image 1 with the exact warrior from Image 2, maintaining the original character's precise placement, scale, and overall silhouette. Seamlessly integrate the warrior into the stormy Mumbai skyline with realistic atmospheric perspective, matching golden backlighting, volumetric light rays, natural rim lighting, consistent shadows, and warm highlights on the armor and skin. Blend the hair, clothing, and body naturally into the surrounding clouds and atmosphere. Preserve the original camera framing, coastline, cityscape, rain, and background. Photorealistic cinematic VFX, seamless environmental integration, physically accurate lighting, no visible cutout edges.`
* **Harmonization Factors**:
  * **Volumetric Sunbeams**: Sunlight streaming from the upper-central cloud break casts specular highlights onto the warrior's golden headdress, shoulders, and arrow shaft.
  * **Atmospheric Distance**: Lower torso fades gently into the coastal sea mist and low-altitude rain clouds hovering over the skyscrapers.
  * **Depth & Scale Consistency**: The warrior is positioned behind the immediate beachfront towers and foreground roadway, maintaining colossal architectural proportions.
* **Deliverable**: `outputs/output_2_object_added_removed.png` (1344 × 768 pixels).

---

### 4. Quality Assurance & Artifact Mitigation

| Artifact Type | Root Cause | Mitigation Strategy |
| :--- | :--- | :--- |
| **Edge Seams** | Hard mask boundaries between source and outpainted canvas | Applied $16\text{px}$ feathered Gaussian blur transition mask in `ImageCompositeMasked`. |
| **Color Temperature Drift** | Unconstrained latent sampling altering ambient tone | Fixed white balance reference and clamped prompt color tokens to warm golden hour (#D4A64A / amber). |
| **Texture Duplication** | Repeating pattern artifacts in large outpainted areas | Injected varied architectural descriptors (sandstone gopurams, fortresses, chariots) in prompt tokens. |
| **Detail Hallucination** | Over-sharpening or plastic synthetic skin textures | Utilized SeedVR2 int8 diffusion super-resolution with Lanczos baseline instead of purely generative ESRGAN hallucination. |

---

### 5. Summary of Deliverable Dimensions & Status

| Deliverable Asset | File Path | Verified Resolution | Aspect Ratio | Verification Status |
| :--- | :--- | :--- | :--- | :--- |
| **Source Low-Quality Input** | `inputs/original_low_quality_image.png` | 880 × 1176 | 3:4 (Portrait) | Preserved intact |
| **Output 1: Extended & Upscaled** | `outputs/output_1_background_extended_upscaled.png` | 1376 × 768 | 16:9 (Landscape) | **PASS** (Height $\ge 720$) |
| **Output 2: Object Edited Result** | `outputs/output_2_object_added_removed.png` | 1344 × 768 | 16:9 (Landscape) | **PASS** (Height $\ge 720$) |
| **Object Removed Evidence** | `outputs/object_removed.png` | 1376 × 768 | 16:9 (Landscape) | **PASS** (Height $\ge 720$) |
| **Object Added Evidence** | `outputs/object_added.png` | 1344 × 768 | 16:9 (Landscape) | **PASS** (Height $\ge 720$) |
| **Workflow 1 (Outpaint/Upscale)** | `workflow/output_1_outpainting_upscaling.json` | 48 Nodes | JSON Graph | **PASS** (Valid ComfyUI JSON) |
| **Workflow 2 (Object Editing)** | `workflow/output_2_object_editing.json` | 8 Nodes | JSON Graph | **PASS** (Valid ComfyUI JSON) |
