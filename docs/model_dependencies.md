# Model & Environment Dependencies

## Assessment 1: AI Image Restoration, Outpainting, Upscaling, Object Addition, and Object Removal

---

### 1. Required AI Model Weights

The workflows utilized in this project rely on official, open-source open-weights diffusion backbones, text encoders, VAEs, and ControlNets. Below is the complete catalog of required checkpoints, their storage directories inside standard ComfyUI installations, and official sources.

| Model File Name | Component Type | Size | Target Directory in ComfyUI | HuggingFace / Source Link | License |
| :--- | :--- | :--- | :--- | :--- | :--- |
| **`seedvr2_7b_int8_convrot.safetensors`** | Diffusion Upscaler / Super-Resolution | ~7.2 GB | `ComfyUI/models/diffusion_models/` | [Comfy-Org/SeedVR2](https://huggingface.co/Comfy-Org/SeedVR2) | Apache 2.0 / Open Access |
| **`qwen_image_fp8_e4m3fn.safetensors`** | DiT Inpainting / Generation Backbone | 19.03 GB | `ComfyUI/models/diffusion_models/` | [Comfy-Org/Qwen-Image_ComfyUI](https://huggingface.co/Comfy-Org/Qwen-Image_ComfyUI) | Tongyi Qianwen Community License |
| **`qwen_image_2.1_int8_convrot.safetensors`** | DiT Inpainting / Multimodal Editing | ~8.4 GB | `ComfyUI/models/diffusion_models/` | [Comfy-Org/Qwen-Image-2.1](https://huggingface.co/Comfy-Org/Qwen-Image_ComfyUI) | Tongyi Qianwen Community License |
| **`flux.1-fill-dev-OneReward-transformer_fp8.safetensors`** | Outpainting / Inpainting DiT | 11.9 GB | `ComfyUI/models/diffusion_models/` | [black-forest-labs/FLUX.1-Fill-dev](https://huggingface.co/black-forest-labs/FLUX.1-Fill-dev) | FLUX.1 Non-Commercial License |
| **`Qwen-Image-InstantX-ControlNet-Inpainting.safetensors`** | Inpainting ControlNet | 3.94 GB | `ComfyUI/models/controlnet/` | [Comfy-Org/Qwen-Image-InstantX-ControlNets](https://huggingface.co/Comfy-Org/Qwen-Image-InstantX-ControlNets) | Apache 2.0 |
| **`Qwen-Image-Lightning-4steps-V1.0.safetensors`** | Fast-Sampling LoRA (4-Step) | 1.58 GB | `ComfyUI/models/loras/` | [lightx2v/Qwen-Image-Lightning](https://huggingface.co/lightx2v/Qwen-Image-Lightning) | Apache 2.0 |
| **`qwen_2.5_vl_7b_fp8_scaled.safetensors`** | Multimodal Text & Vision Encoder | 8.74 GB | `ComfyUI/models/text_encoders/` | [Comfy-Org/Qwen-Image_ComfyUI](https://huggingface.co/Comfy-Org/Qwen-Image_ComfyUI) | Tongyi Qianwen Community License |
| **`qwen_image_vae.safetensors`** | Latent Variational Autoencoder | 242 MB | `ComfyUI/models/vae/` | [Comfy-Org/Qwen-Image_ComfyUI](https://huggingface.co/Comfy-Org/Qwen-Image_ComfyUI) | Apache 2.0 |

---

### 2. ComfyUI Custom Node Extensions

To open and execute the workflows without node resolution errors, install the following custom node packages via **ComfyUI Manager** or direct Git cloning into `ComfyUI/custom_nodes/`:

#### 2.1 Essential Node Packages
1. **ComfyUI-SeedVR2**
   * Repository: `https://github.com/SeedVR/ComfyUI-SeedVR2`
   * Provides: `SeedVR2Preprocess`, `SeedVR2Conditioning`, `SeedVR2PostProcessing`, `ResizeImageMaskNode`
2. **ComfyUI-Qwen-Image / InstantX**
   * Repository: `https://github.com/InstantX/ComfyUI-InstantX-Qwen`
   * Provides: `ControlNetInpaintingAliMamaApply`, `ModelSamplingAuraFlow`
3. **ComfyUI-Core Nodes**
   * Pre-installed with modern ComfyUI: `ImagePadForOutpaint`, `ImageCompositeMasked`, `VAEDecodeTiled`, `SaveImageAdvanced`, `ResolutionSelector`
4. **ComfyUI-Custom-Scripts**
   * Repository: `https://github.com/pythongosssss/ComfyUI-Custom-Scripts`
   * Provides: `ImageCompare`, `MarkdownNote`

#### 2.2 CLI Installation Command
```bash
cd ComfyUI/custom_nodes
git clone https://github.com/SeedVR/ComfyUI-SeedVR2.git
git clone https://github.com/InstantX/ComfyUI-InstantX-Qwen.git
git clone https://github.com/pythongosssss/ComfyUI-Custom-Scripts.git
pip install -r ComfyUI-SeedVR2/requirements.txt
```

---

### 3. Hardware Requirements & Performance Profiling

| Specification | Minimum Required | Recommended Configuration | Project Execution Environment |
| :--- | :--- | :--- | :--- |
| **GPU** | NVIDIA RTX 3060 (12 GB VRAM) | NVIDIA RTX 4080 / 4090 (16-24 GB VRAM) | NVIDIA RTX Series with CUDA 12.1+ |
| **System Memory (RAM)** | 16 GB DDR4 | 32 GB DDR5 | 32 GB |
| **Storage** | 45 GB Free SSD Space | High-Speed NVMe M.2 SSD | NVMe M.2 SSD |
| **Tiled VAE Execution** | Enabled (Tile Size: 512, Overlap: 128) | Enabled | Enabled |
| **Precision Quantization** | FP8 / INT8 ConvRot | FP8 / INT8 ConvRot | INT8 ConvRot / FP8 E4M3FN |

---

### 4. Licensing Verification & Attributions

* **Qwen-Image Model Suite**: Released by the Qwen Team, Alibaba Cloud under the [Tongyi Qianwen Community License](https://github.com/QwenLM/Qwen/blob/main/Tongyi%20Qianwen%20LICENSE%20AGREEMENT). Free for academic and commercial use under compliance guidelines.
* **SeedVR2 Super-Resolution**: Released by the SeedVR Research Team under the [Apache License 2.0](https://www.apache.org/licenses/LICENSE-2.0).
* **InstantX ControlNet Inpainting**: Released under [Apache License 2.0](https://huggingface.co/Comfy-Org/Qwen-Image-InstantX-ControlNets).
* **FLUX.1-Fill-dev**: Released by Black Forest Labs under the [FLUX.1-dev Non-Commercial License](https://github.com/black-forest-labs/flux/blob/main/model_licenses/LICENSE-FLUX1-dev).
* **ComfyUI**: Core engine released under [GPLv3](https://github.com/comfyanonymous/ComfyUI/blob/master/LICENSE).
