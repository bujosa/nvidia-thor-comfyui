# NVIDIA Thor — ComfyUI + Flux.1 dev Setup

Complete setup and operation guide for running ComfyUI with Flux.1 dev on NVIDIA Jetson Thor (JetPack 7.x, Blackwell, 122 GB unified memory).

## Quick Access

```
UI:  http://<THOR_IP>:8188
SSH: ssh <user>@<THOR_IP>
```

---

## Start / Stop / Restart

ComfyUI runs as a systemd service. It starts automatically on boot.

```bash
# Check status
sudo systemctl status comfyui.service

# Start
sudo systemctl start comfyui.service

# Stop (frees ~30 GB RAM)
sudo systemctl stop comfyui.service

# Restart (needed after installing nodes or after a crash)
sudo systemctl restart comfyui.service

# View logs in real time
sudo journalctl -u comfyui.service -f

# View last 50 log lines
sudo journalctl -u comfyui.service -n 50
```

### Disable auto-start (keep installed but don't run on boot)

```bash
sudo systemctl disable comfyui.service
# To re-enable:
sudo systemctl enable comfyui.service
```

### Manual launch (for debugging)

```bash
sudo systemctl stop comfyui.service
~/comfyui-env/start-comfyui.sh
# Ctrl+C to stop
```

---

## Free Resources / Cleanup

### Option A: Just stop ComfyUI (frees ~30 GB RAM, keeps everything installed)

```bash
sudo systemctl stop comfyui.service
sudo systemctl disable comfyui.service
```

### Option B: Delete models only (~55 GB disk)

```bash
sudo systemctl stop comfyui.service
rm -rf ~/comfyui-env/ComfyUI/models/checkpoints/*.safetensors
rm -rf ~/comfyui-env/ComfyUI/models/clip/*.safetensors
rm -rf ~/comfyui-env/ComfyUI/models/clip_vision/*
rm -rf ~/comfyui-env/ComfyUI/models/controlnet/*
rm -rf ~/comfyui-env/ComfyUI/models/loras/*.safetensors
rm -rf ~/comfyui-env/ComfyUI/models/ipadapter-flux/*
rm -rf ~/comfyui-env/ComfyUI/models/ipadapter/*
rm -rf ~/comfyui-env/ComfyUI/models/animatediff_models/*.ckpt
rm -rf ~/comfyui-env/ComfyUI/models/vae/*.safetensors
rm -rf ~/comfyui-env/ComfyUI/models/diffusion_models/*.safetensors
```

### Option C: Delete everything (ComfyUI + venv + models, ~60 GB)

```bash
sudo systemctl stop comfyui.service
sudo systemctl disable comfyui.service
sudo rm /etc/systemd/system/comfyui.service
sudo systemctl daemon-reload
rm -rf ~/comfyui-env
```

### Option D: Also remove swap (16 GB)

```bash
sudo swapoff /swapfile
sudo rm /swapfile
sudo sed -i '/swapfile/d' /etc/fstab
```

### Check disk after cleanup

```bash
df -h /
```

---

## Documentation

| Document | Description |
|---|---|
| [Setup Guide](docs/setup-guide.md) | Full guide: what was installed, workflows, flags, troubleshooting |
| [CUDA/JetPack Fix](docs/cuda-jetpack-fix.md) | Critical cuBLAS pip vs JetPack bug — detection and fix |
| [Device Specs](docs/device-specs.md) | Hardware, software stack, models, known issues |

---

## What's Installed

- **ComfyUI v0.15.1** (native install, not Docker)
- **Flux.1 dev FP8** (17 GB) — main generation model
- **SD 1.5** (4 GB) — for AnimateDiff
- **6 LoRAs** — realism, anatomy, skin textures
- **3 ControlNets** — canny, depth, openpose
- **IPAdapter Flux** — image-to-image style transfer
- **AnimateDiff** — short animations (4-16 frames, SD1.5 only)
- **8 custom nodes** — Manager, IPAdapter, ControlNet, AnimateDiff, etc.

Total disk usage: ~60 GB (models + code + venv)

## Resource Usage When Running

| Resource | Usage |
|---|---|
| RAM | ~30 GB (ComfyUI with models loaded) |
| Disk | ~60 GB |
| CPU | Near 0% when idle |
| GPU | Active only during generation |
