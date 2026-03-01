# ComfyUI + Flux.1 dev on NVIDIA Jetson Thor

Complete setup guide for image generation on the Jetson Thor.

## Quick Access

```
UI:  http://<THOR_IP>:8188
SSH: ssh <user>@<THOR_IP>
```

---

## 1. What Was Installed

### ComfyUI v0.15.1 (native install, NOT Docker)

Everything lives in `~/comfyui-env/` on Thor:

```
~/comfyui-env/
├── venv/                    # Python 3.12 virtual environment
├── start-comfyui.sh         # Optimized launch script
└── ComfyUI/
    ├── main.py
    ├── models/              # All models (~55 GB total)
    ├── custom_nodes/        # Installed plugins
    └── user/default/workflows/
```

### Installed Models

#### Checkpoints and Base Models

| Model | Size | Path | Usage |
|---|---|---|---|
| Flux.1 dev FP8 | 17 GB | checkpoints/ + diffusion_models/ (symlink) | Main generation |
| SD 1.5 Pruned | 4.0 GB | checkpoints/ | AnimateDiff base |
| CLIP-L | 235 MB | clip/ | Flux text encoder (lightweight) |
| T5-XXL FP16 | 9.2 GB | clip/ | Flux text encoder (USE THIS) |
| T5-XXL FP8 | 4.6 GB | clip/ | Backup (DO NOT use, crashes cuBLAS) |
| Flux VAE bf16 | 320 MB | vae/ | VAE extracted from checkpoint |

#### LoRAs (in models/loras/)

| LoRA | Size | Purpose | Recommended Strength |
|---|---|---|---|
| Flux_Perfect_Body_v48 | 877 MB | Realistic body anatomy and proportions | 0.7-0.9 |
| Flux_Uncensored_V2 | 656 MB | Full unrestricted detail | 0.6-0.8 |
| Flux_Super_Realism | 585 MB | General photorealism boost | 0.7-0.9 |
| Flux_Female_Anatomy_v2 | 188 MB | Precise female anatomy | 0.6-0.8 |
| Flux_Perfect_Skin | 147 MB | Realistic skin textures | 0.5-0.7 |
| Flux_Realism_Lora (XLabs) | 22 MB | Lightweight realism, stackable | 0.8-1.0 |

#### ControlNet (in models/controlnet/)

| Model | Size | Function |
|---|---|---|
| flux-canny-controlnet-v3 | 3.4 GB | Edge control (canny edges) |
| flux-depth-controlnet | 3.0 GB | Depth control (depth maps) |
| control_v11p_sd15_openpose | 1.4 GB | Human pose (for AnimateDiff with SD1.5) |

#### IPAdapter Flux (in models/ipadapter-flux/)

| Component | Size | Path |
|---|---|---|
| ip-adapter.bin | 5.0 GB | models/ipadapter-flux/ |
| SigLIP vision model | 3.3 GB | models/clip_vision/siglip-so400m-patch14-384/ |
| CLIP Vision ViT-H | 2.4 GB | models/clip_vision/ (for legacy IPAdapter SD1.5) |

#### AnimateDiff (in models/animatediff_models/)

| Motion Module | Size |
|---|---|
| v3_sd15_mm.ckpt | 1.6 GB |
| mm_sd_v15_v2.ckpt | 1.7 GB |

### Installed Custom Nodes

| Node | Repo | Purpose |
|---|---|---|
| ComfyUI-Manager | ltdrdata | Node management from UI |
| ComfyUI-IPAdapter-Flux | Shakker-Labs | IPAdapter for Flux (image transfer) |
| ComfyUI_IPAdapter_plus | cubiq | Legacy IPAdapter (SD1.5/SDXL) |
| comfyui_controlnet_aux | Fannovel16 | Preprocessors: openpose, depth, canny, etc. |
| ComfyUI-Advanced-ControlNet | Kosinkadink | Multi-ControlNet stacking |
| ComfyUI-AnimateDiff-Evolved | Kosinkadink | Short animations (4-16 frames) |
| ComfyUI-VideoHelperSuite | Kosinkadink | Video export (gif, mp4) |
| ComfyUI_InstantID | cubiq | Advanced facial preservation |

---

## 2. Start / Stop / Restart

ComfyUI runs as a systemd service. It starts automatically on boot.

```bash
# Check status
sudo systemctl status comfyui.service

# Start
sudo systemctl start comfyui.service

# Stop
sudo systemctl stop comfyui.service

# Restart (needed after installing nodes or after a crash)
sudo systemctl restart comfyui.service

# View logs in real time
sudo journalctl -u comfyui.service -f

# View last 50 log lines
sudo journalctl -u comfyui.service -n 50
```

For manual launch (debugging):

```bash
sudo systemctl stop comfyui.service
~/comfyui-env/start-comfyui.sh
# Ctrl+C to stop
```

### Disable auto-start

```bash
sudo systemctl disable comfyui.service
# To re-enable:
sudo systemctl enable comfyui.service
```

---

## 3. Workflows — How to Generate Images

### IMPORTANT: Do NOT use "Load Checkpoint" with the FP8 file

The flux1-dev-fp8.safetensors checkpoint is an all-in-one file but it embeds T5-XXL
in FP8, which crashes cuBLAS on Thor. ALWAYS use separate nodes:

### Basic workflow: txt2img with Flux

```
Required nodes:

1. Load Diffusion Model (UNETLoader)
   - unet_name: flux1-dev-fp8.safetensors
   - weight_dtype: fp8_e4m3fn

2. DualCLIPLoader
   - clip_name1: clip_l.safetensors
   - clip_name2: t5xxl_fp16.safetensors
   - type: flux

3. Load VAE
   - vae_name: flux-ae-bf16.safetensors

4. CLIPTextEncode (positive)
   - text: "your prompt here"

5. CLIPTextEncode (negative)
   - text: "" (empty for Flux)

6. EmptyLatentImage
   - width: 1024, height: 1024

7. KSampler
   - seed: random
   - steps: 20
   - cfg: 3.5 (Flux uses low CFG, 3.0-4.5)
   - sampler: euler
   - scheduler: normal
   - denoise: 1.0

8. VAEDecode -> SaveImage

Connections:
- UNETLoader MODEL -> KSampler model
- DualCLIPLoader CLIP -> both CLIPTextEncode
- CLIPTextEncode pos -> KSampler positive
- CLIPTextEncode neg -> KSampler negative
- EmptyLatentImage -> KSampler latent_image
- KSampler LATENT -> VAEDecode samples
- Load VAE -> VAEDecode vae
- VAEDecode IMAGE -> SaveImage
```

### img2img workflow with low denoising (preserve likeness)

Same as txt2img but change:

```
- Instead of EmptyLatentImage, use:
  LoadImage -> VAEEncode -> KSampler (latent_image)
  (also connect Load VAE to VAEEncode)

- KSampler denoise: 0.35-0.45
  (0.35 = subtle changes, face/body nearly intact)
  (0.45 = moderate changes, keeps overall structure)
  (0.55+ = creative changes, loses more original detail)
```

### img2img + LoRAs workflow

```
Chain Load LoRA nodes between UNETLoader and KSampler:

UNETLoader -> Load LoRA (Flux_Super_Realism, str: 0.8)
           -> Load LoRA (Flux_Perfect_Skin, str: 0.6)
           -> KSampler

Each Load LoRA has:
- model input/output (chain them)
- clip input/output (chain from DualCLIPLoader)
- lora_name: select the file
- strength_model: 0.6-0.9
- strength_clip: 0.6-0.9

You can stack 2-3 LoRAs without issues with 122GB RAM.
```

### img2img + ControlNet Depth workflow

```
Add these nodes:

1. LoadImage (reference image)
2. ControlNetLoader
   - control_net_name: flux-depth-controlnet.safetensors
3. DepthAnythingPreprocessor (from comfyui_controlnet_aux)
   - Connect LoadImage -> preprocessor
4. ControlNetApplyAdvanced
   - strength: 0.6-0.8
   - start_percent: 0.0
   - end_percent: 1.0
   - Connect: positive/negative conditioning + controlnet + processed image
   - Output: positive/negative -> KSampler
```

### IPAdapter Flux workflow (image transfer)

```
1. IPAdapterFluxLoader
   - ipadapter: ip-adapter.bin
   - clip_vision: google/siglip-so400m-patch14-384
   - provider: cuda

2. LoadImage (reference image for style/face)

3. ApplyIPAdapterFlux
   - model: connect from UNETLoader (or after LoRAs)
   - ipadapter_flux: from IPAdapterFluxLoader
   - image: from LoadImage
   - weight: 0.7 (0.5-0.9 depending on how much to transfer)
   - start_percent: 0.0
   - end_percent: 0.8
   - Output MODEL -> KSampler
```

### AnimateDiff workflow (short animations, uses SD1.5)

```
NOTE: AnimateDiff does NOT work with Flux. Use SD1.5.

1. CheckpointLoaderSimple
   - ckpt_name: v1-5-pruned-emaonly.safetensors

2. AnimateDiff Loader
   - model_name: v3_sd15_mm.ckpt (or mm_sd_v15_v2.ckpt)
   - beta_schedule: autoselect
   - motion_scale: 1.0
   - apply_v2_models_properly: true

3. CLIPTextEncode pos/neg (normal)

4. EmptyLatentImage
   - width: 512, height: 512
   - batch_size: 8-16 (= number of frames)

5. KSampler
   - steps: 20, cfg: 7.5, sampler: euler, scheduler: normal

6. AnimateDiff Combine
   - frame_rate: 8
   - format: gif or mp4

With ControlNet OpenPose for frame-by-frame pose guidance:
- Load pose images (one per frame)
- control_v11p_sd15_openpose.pth
- ControlNetApply strength: 0.7-0.9
```

---

## 4. Launch Flags (start-comfyui.sh)

Current script:

```bash
#!/bin/bash
export PATH=/usr/local/cuda/bin:$PATH
export LD_LIBRARY_PATH=/usr/local/cuda/lib64:$LD_LIBRARY_PATH
export PYTORCH_CUDA_ALLOC_CONF=expandable_segments:True
export CUDA_MODULE_LOADING=LAZY
export CUBLAS_WORKSPACE_CONFIG=:4096:8

source ~/comfyui-env/venv/bin/activate
cd ~/comfyui-env/ComfyUI

python3 main.py \
  --listen 0.0.0.0 \
  --port 8188 \
  --highvram \
  --fp8_e4m3fn-unet \
  --bf16-text-enc \
  --bf16-vae \
  --disable-cuda-malloc \
  --preview-method auto \
  --cuda-device 0 \
  "$@"
```

| Flag | What it does |
|---|---|
| --listen 0.0.0.0 | Accessible from entire network |
| --highvram | Keeps models in VRAM (Thor has 122GB, plenty) |
| --fp8_e4m3fn-unet | Native FP8 UNET (fast on Blackwell) |
| --bf16-text-enc | Text encoders in bf16 (avoids cuBLAS crash with FP8) |
| --bf16-vae | VAE in bf16 (stable) |
| --disable-cuda-malloc | Native allocator (avoids CUBLAS_NOT_INITIALIZED) |

---

## 5. Critical Bug: pip cuBLAS vs JetPack

### The Problem

pip install torch brings generic nvidia-cublas that does NOT work on Thor/Jetson.
Causes: CUBLAS_STATUS_INVALID_VALUE on ANY matmul, even 2x2.

### How to detect it

```bash
source ~/comfyui-env/venv/bin/activate
python3 -c "import torch; print(torch.randn(2,2,device='cuda') @ torch.randn(2,2,device='cuda'))"
```

If it crashes with CUBLAS error -> pip cuBLAS overwrote JetPack's version.

### How to fix it

```bash
NVIDIA_LIB=~/comfyui-env/venv/lib/python3.12/site-packages/nvidia/cu13/lib

# Backup pip versions — backup name must NOT end in a digit!
# torch uses glob("libcublas.so.*[0-9]") to find the library.
# If the backup is named "libcublas.so.13.pip.bak2", it ends in "2" and
# torch loads it INSTEAD of the symlink. Always use .old suffix.
mv $NVIDIA_LIB/libcublas.so.13 $NVIDIA_LIB/libcublas.so.pip-backup.old
mv $NVIDIA_LIB/libcublasLt.so.13 $NVIDIA_LIB/libcublasLt.so.pip-backup.old
mv $NVIDIA_LIB/libnvblas.so.13 $NVIDIA_LIB/libnvblas.so.pip-backup.old

# Symlink to JetPack
ln -sf /usr/local/cuda/lib64/libcublas.so.13 $NVIDIA_LIB/libcublas.so.13
ln -sf /usr/local/cuda/lib64/libcublasLt.so.13 $NVIDIA_LIB/libcublasLt.so.13
ln -sf /usr/local/cuda/lib64/libnvblas.so.13 $NVIDIA_LIB/libnvblas.so.13
```

IMPORTANT: This must be done EVERY TIME you pip install something that
pulls nvidia-cublas as a dependency. Also symlink libnvblas.

See full guide: [docs/cuda-jetpack-fix.md](docs/cuda-jetpack-fix.md)

---

## 6. Full Cleanup (free disk space)

See the "Free Resources / Cleanup" section in the [main README](../README.md).

---

## 7. Usage Tips

### Recommended resolution for Flux
- txt2img: 1024x1024 or 1024x768
- img2img: same resolution as input image (or multiple of 8)

### CFG for Flux
- Flux uses VERY low CFG: 3.0-4.5 (not 7-12 like SD1.5)
- CFG 1.0 = almost no prompt guidance
- CFG 3.5 = ideal balance
- CFG 7.0+ = oversaturated, artifacts

### Denoise for img2img (preserve likeness)
- 0.25-0.35: Minimal changes (lighting, color adjustments)
- 0.35-0.45: Moderate changes (keeps face, changes details)
- 0.45-0.60: Significant changes (overall structure maintained)
- 0.60+: Major changes (loses resemblance)

### Stacking LoRAs
- Maximum 2-3 LoRAs at once for optimal quality
- Recommended combo: Super_Realism (0.8) + Perfect_Skin (0.5)
- For anatomy: Perfect_Body (0.8) + Female_Anatomy (0.6)
- Uncensored_V2 can stack with any (0.6-0.7)

### IPAdapter weights
- 0.3-0.5: Subtle reference image influence
- 0.5-0.7: Balance between prompt and reference
- 0.7-0.9: Reference image dominates
- end_percent: 0.8 (let final iterations refine without IPAdapter)

### ControlNet strengths
- Depth: 0.5-0.8 (3D body structure)
- Canny: 0.4-0.7 (edges and contours)
- OpenPose: 0.7-0.9 (exact pose, for AnimateDiff)

### Memory (122 GB unified)
- Flux FP8 UNET: ~12 GB
- T5-XXL FP16: ~9 GB
- CLIP-L: ~0.2 GB
- VAE: ~0.3 GB
- LoRA (each): 0.02-0.9 GB
- IPAdapter + SigLIP: ~8 GB
- ControlNet: ~3 GB each
- Typical total with everything loaded: ~35-40 GB (~80 GB remaining)

---

## 8. Troubleshooting

### Error "CUBLAS_STATUS_INVALID_VALUE"
-> pip cuBLAS overwrote JetPack. See section 5.

### Error "CUBLAS_STATUS_NOT_INITIALIZED"
-> Corrupted CUDA context. Restart ComfyUI:
```bash
sudo systemctl restart comfyui.service
```

### Error "flipped_img_txt" in IPAdapter-Flux
-> Mismatch between ComfyUI and IPAdapter-Flux node. Patch:
```bash
# In ~/comfyui-env/ComfyUI/custom_nodes/ComfyUI-IPAdapter-Flux/flux/layers.py
# Line 31, change:
#   self.flipped_img_txt = original_block.flipped_img_txt
# To:
#   self.flipped_img_txt = getattr(original_block, 'flipped_img_txt', True)
```

### Error "header too large" in IPAdapter
-> Corrupt file or wrong extension. Verify with:
```bash
file models/ipadapter-flux/ip-adapter.bin
# Should say "Zip archive data" (PyTorch .bin format)
```

### Error "Allocation on device" (OOM)
-> Reduce resolution or batch size. Rare with 122GB but can happen
with many models loaded + high resolution + large batch.

### Black images
-> Change --bf16-vae to --fp32-vae in start-comfyui.sh

### LoRA not showing in list
-> Restart ComfyUI (detects new models on startup)

### ComfyUI won't start
```bash
# Check what failed
sudo journalctl -u comfyui.service -n 50

# Manual launch for debugging
sudo systemctl stop comfyui.service
~/comfyui-env/start-comfyui.sh
```
