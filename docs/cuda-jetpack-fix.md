# CUDA + JetPack 7.x Guidelines for NVIDIA Thor

## The #1 Rule: Never Use pip nvidia-cublas on Jetson

When installing PyTorch via pip on any Jetson device (Thor, Orin, etc.), pip pulls
generic `nvidia-cublas`, `nvidia-cudnn`, etc. packages that are **NOT compiled for
Jetson aarch64 GPUs**. They silently replace the JetPack system libraries and cause:

```
RuntimeError: CUDA error: CUBLAS_STATUS_INVALID_VALUE when calling `cublasSgemm(...)`
RuntimeError: CUDA error: CUBLAS_STATUS_NOT_INITIALIZED when calling `cublasLtMatmulAlgoGetHeuristic(...)`
```

These errors happen on **any** matmul operation, even a simple 2x2 FP32 multiply.

## Fix: Symlink pip cuBLAS to JetPack System Libraries

After any `pip install torch` or `pip install` that pulls nvidia packages:

```bash
NVIDIA_LIB=$HOME/comfyui-env/venv/lib/python3.12/site-packages/nvidia/cu13/lib

# Backup pip versions — CRITICAL: backup names must NOT end in a digit!
# torch/__init__.py uses glob("libcublas.so.*[0-9]") to find libraries,
# so a backup like "libcublas.so.13.pip.bak2" matches (ends in "2") and
# gets loaded INSTEAD of the symlink. Always use .old suffix.
mv $NVIDIA_LIB/libcublas.so.13 $NVIDIA_LIB/libcublas.so.pip-backup.old
mv $NVIDIA_LIB/libcublasLt.so.13 $NVIDIA_LIB/libcublasLt.so.pip-backup.old
mv $NVIDIA_LIB/libnvblas.so.13 $NVIDIA_LIB/libnvblas.so.pip-backup.old

# Symlink to JetPack
ln -sf /usr/local/cuda/lib64/libcublas.so.13 $NVIDIA_LIB/libcublas.so.13
ln -sf /usr/local/cuda/lib64/libcublasLt.so.13 $NVIDIA_LIB/libcublasLt.so.13
ln -sf /usr/local/cuda/lib64/libnvblas.so.13 $NVIDIA_LIB/libnvblas.so.13
```

Verify with:

```bash
python3 -c "import torch; print(torch.randn(2,2,device='cuda') @ torch.randn(2,2,device='cuda'))"
```

## Why Backup Naming Matters

`torch/__init__.py` uses `glob("libcublas.so.*[0-9]")` to discover and load cuBLAS.
If a backup file like `libcublas.so.13.pip.bak2` exists in the same directory, it
matches the glob (ends with digit `2`) and PyTorch loads the **bad pip version**
instead of the JetPack symlink.

This was discovered by tracing library loading with `LD_DEBUG=libs` and inspecting
`/proc/<pid>/maps` — the backup file appeared in the process memory map despite
the symlink being correct.

**Always name backups with a non-digit suffix** like `.old`.

## Affected Packages

Any pip install that pulls these will break cuBLAS on Jetson:

| Package | What it does |
|---|---|
| `nvidia-cublas` | Replaces system cuBLAS (the main culprit) |
| `nvidia-cublas-cu13` | Same, different naming |
| `nvidia-cudnn-cu13` | May conflict with JetPack cuDNN |
| `nvidia-cusparselt-cu13` | Required by torch, usually safe |

## How to Detect the Problem

```bash
# Check if pip cuBLAS exists (files, not symlinks)
find ~/comfyui-env/venv -name 'libcublas*' -not -type l

# If files (not symlinks) exist -> problem
# Compare versions:
pip show nvidia-cublas  # pip version (e.g., 13.2.1.1 - generic)
ls /usr/local/cuda/lib64/libcublas.so.*  # JetPack version (e.g., 13.0.0.19 - Thor-optimized)
```

## ComfyUI Launch Flags for Thor

```bash
python3 main.py \
  --listen 0.0.0.0 \
  --port 8188 \
  --highvram \
  --fp8_e4m3fn-unet \
  --bf16-text-enc \
  --bf16-vae \
  --disable-cuda-malloc \
  --preview-method auto \
  --cuda-device 0
```

Key flags:
- `--fp8_e4m3fn-unet`: FP8 native on Blackwell UNET (fast)
- `--bf16-text-enc`: Forces T5-XXL/CLIP to bf16 (avoids FP8 cuBLAS issues in text encoding)
- `--bf16-vae`: VAE in bf16 (stable)
- `--disable-cuda-malloc`: Uses native allocator, not cudaMallocAsync (avoids NOT_INITIALIZED)

Environment variables:

```bash
export PYTORCH_CUDA_ALLOC_CONF=expandable_segments:True
export CUDA_MODULE_LOADING=LAZY
export CUBLAS_WORKSPACE_CONFIG=:4096:8
```

## Flux.1 dev on Thor: Use Separated Model Loading

Do NOT use `CheckpointLoaderSimple` with `flux1-dev-fp8.safetensors` — the embedded
T5-XXL in FP8 crashes cuBLAS even with JetPack libraries.

Use separate nodes:

| Node | File |
|---|---|
| Load Diffusion Model (UNETLoader) | `flux1-dev-fp8.safetensors` (weight_dtype: fp8_e4m3fn) |
| DualCLIPLoader | `clip_l.safetensors` + `t5xxl_fp16.safetensors` (type: flux) |
| Load VAE | `flux-ae-bf16.safetensors` (extracted from checkpoint) |

## Prevention: Pin pip nvidia packages

After fixing, prevent pip from overwriting the symlinks:

```bash
# Create a constraints file
cat > ~/comfyui-env/constraints.txt << 'EOF'
nvidia-cublas==13.1.0.3
nvidia-cudnn-cu13==9.15.1.9
EOF

# Use when installing packages
pip install --constraint ~/comfyui-env/constraints.txt <package>
```

Or always re-run the symlink commands after any pip install that touches nvidia packages.
