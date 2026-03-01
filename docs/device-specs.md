# NVIDIA Jetson Thor — Device Specs

## Hardware

| Component | Spec |
|---|---|
| SoC | NVIDIA Thor |
| Architecture | Blackwell |
| GPU Cores | ~17,000 CUDA cores |
| Compute Capability | **11.0** |
| Memory | **122.8 GB** unified (CPU+GPU shared) |
| Storage | 937 GB NVMe |
| Swap | 16 GB (configured, persistent via /swapfile) |

## Software Stack

| Component | Version |
|---|---|
| OS | Ubuntu 24.04 LTS aarch64 |
| Kernel | 6.8.12-tegra (R38.2.0) |
| JetPack | **7.0-b128** |
| CUDA | **13.0** (V13.0.48) |
| cuDNN | 9.12.0 (system) / 9.15.1 (pip) |
| Driver | 580.00 |
| Python | 3.12.3 |
| PyTorch | 2.10.0+cu130 |
| ComfyUI | v0.15.1 |

## Services

| Service | Port | Status |
|---|---|---|
| ComfyUI | 8188 | systemd `comfyui.service` (enabled, auto-start) |

## Installed Models (~55 GB)

| Model | Size | Location | Purpose |
|---|---|---|---|
| Flux.1 dev FP8 | 17 GB | checkpoints/ + diffusion_models/ (symlink) | Main generation |
| SD 1.5 Pruned | 4.0 GB | checkpoints/ | AnimateDiff base |
| CLIP-L | 235 MB | clip/ | Flux text encoder |
| T5-XXL FP8 | 4.6 GB | clip/ | Flux text encoder (backup, causes cuBLAS issues) |
| T5-XXL FP16 | 9.2 GB | clip/ | Flux text encoder (USE THIS ONE) |
| CLIP Vision ViT-H | 2.4 GB | clip_vision/ | IPAdapter image encoding |
| SigLIP vision | 3.3 GB | clip_vision/siglip-so400m-patch14-384/ | IPAdapter Flux vision |
| Flux Canny ControlNet | 3.4 GB | controlnet/ | Edge-guided generation |
| Flux Depth ControlNet | 3.0 GB | controlnet/ | Depth-guided generation |
| SD1.5 OpenPose ControlNet | 1.4 GB | controlnet/ | Pose for AnimateDiff |
| IPAdapter Flux | 5.0 GB | ipadapter-flux/ | Style/face/pose transfer |
| AnimateDiff v3 | 1.6 GB | animatediff_models/ | Animation |
| AnimateDiff v2 | 1.7 GB | animatediff_models/ | Animation (alt) |
| 6 LoRAs | ~2.5 GB | loras/ | Realism, anatomy, skin |
| Flux VAE (bf16) | 320 MB | vae/ | Standalone VAE |

## Custom Nodes

- ComfyUI-Manager (ltdrdata)
- ComfyUI-IPAdapter-Flux (Shakker-Labs)
- ComfyUI_IPAdapter_plus (cubiq)
- comfyui_controlnet_aux (Fannovel16)
- ComfyUI-Advanced-ControlNet (Kosinkadink)
- ComfyUI-AnimateDiff-Evolved (Kosinkadink)
- ComfyUI-VideoHelperSuite (Kosinkadink)
- ComfyUI_InstantID (cubiq)

## Key Files

```
~/comfyui-env/
├── venv/                          # Python virtual environment
├── start-comfyui.sh               # Launch script (optimized for Thor)
├── constraints.txt                # pip constraints to prevent nvidia overwrites
└── ComfyUI/
    ├── main.py
    ├── models/
    │   ├── checkpoints/
    │   ├── clip/
    │   ├── clip_vision/
    │   ├── controlnet/
    │   ├── diffusion_models/      # symlink to checkpoints/flux1-dev-fp8
    │   ├── ipadapter-flux/
    │   ├── loras/
    │   ├── animatediff_models/
    │   └── vae/
    ├── custom_nodes/
    └── user/default/workflows/
```

## Known Issues

1. **pip nvidia-cublas breaks cuBLAS on Thor** — See [cuda-jetpack-fix.md](cuda-jetpack-fix.md)
2. **Backup file naming matters** — torch globs `libcublas.so.*[0-9]`, backups ending in digits get loaded
3. **Flux FP8 single-file checkpoint crashes CLIPTextEncode** — Use separated model loading (UNETLoader + DualCLIPLoader + Load VAE)
4. **T5-XXL FP8 incompatible with cuBLAS on Blackwell** — Use `t5xxl_fp16.safetensors` instead
5. **cudaMallocAsync can cause NOT_INITIALIZED** — Use `--disable-cuda-malloc`
6. **IPAdapter-Flux flipped_img_txt** — ComfyUI v0.15.1 lacks this attribute; patched with `getattr(..., True)`

## Quick Reference Commands

```bash
# Service management
sudo systemctl start|stop|restart|status comfyui.service

# View logs
sudo journalctl -u comfyui.service -f

# Check GPU
nvidia-smi

# Manual launch (after stopping service)
~/comfyui-env/start-comfyui.sh

# Fix cuBLAS after pip install
NVIDIA_LIB=~/comfyui-env/venv/lib/python3.12/site-packages/nvidia/cu13/lib
mv $NVIDIA_LIB/libcublas.so.13 $NVIDIA_LIB/libcublas.so.pip-backup.old
mv $NVIDIA_LIB/libcublasLt.so.13 $NVIDIA_LIB/libcublasLt.so.pip-backup.old
mv $NVIDIA_LIB/libnvblas.so.13 $NVIDIA_LIB/libnvblas.so.pip-backup.old
ln -sf /usr/local/cuda/lib64/libcublas.so.13 $NVIDIA_LIB/libcublas.so.13
ln -sf /usr/local/cuda/lib64/libcublasLt.so.13 $NVIDIA_LIB/libcublasLt.so.13
ln -sf /usr/local/cuda/lib64/libnvblas.so.13 $NVIDIA_LIB/libnvblas.so.13

# Test cuBLAS
source ~/comfyui-env/venv/bin/activate
python3 -c "import torch; print(torch.randn(2,2,device='cuda') @ torch.randn(2,2,device='cuda'))"
```
