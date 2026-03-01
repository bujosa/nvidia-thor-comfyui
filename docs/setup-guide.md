# ComfyUI + Flux.1 dev en NVIDIA Jetson Thor

Guia completa de la configuracion de generacion de imagenes en la Jetson Thor.

## Quick Access

```
UI:  http://<THOR_IP>:8188
SSH: ssh <user>@<THOR_IP>
```

---

## 1. Que se instalo

### ComfyUI v0.15.1 (instalacion nativa, NO Docker)

Todo esta en `~/comfyui-env/` en la Thor:

```
~/comfyui-env/
├── venv/                    # Python 3.12 virtual environment
├── start-comfyui.sh         # Script de arranque optimizado
└── ComfyUI/
    ├── main.py
    ├── models/              # Todos los modelos (~55 GB total)
    ├── custom_nodes/        # Plugins instalados
    └── user/default/workflows/
```

### Modelos instalados

#### Checkpoints y modelos base

| Modelo | Tamanio | Ruta | Uso |
|---|---|---|---|
| Flux.1 dev FP8 | 17 GB | checkpoints/ + diffusion_models/ (symlink) | Generacion principal |
| SD 1.5 Pruned | 4.0 GB | checkpoints/ | Base para AnimateDiff |
| CLIP-L | 235 MB | clip/ | Text encoder Flux (ligero) |
| T5-XXL FP16 | 9.2 GB | clip/ | Text encoder Flux (USAR ESTE) |
| T5-XXL FP8 | 4.6 GB | clip/ | Backup (NO usar, causa crash cuBLAS) |
| Flux VAE bf16 | 320 MB | vae/ | VAE extraido del checkpoint |

#### LoRAs (en models/loras/)

| LoRA | Tamanio | Para que sirve | Strength recomendado |
|---|---|---|---|
| Flux_Perfect_Body_v48 | 877 MB | Anatomia corporal y proporciones realistas | 0.7-0.9 |
| Flux_Uncensored_V2 | 656 MB | Detalle completo sin restricciones | 0.6-0.8 |
| Flux_Super_Realism | 585 MB | Boost de fotorrealismo general | 0.7-0.9 |
| Flux_Female_Anatomy_v2 | 188 MB | Anatomia femenina precisa | 0.6-0.8 |
| Flux_Perfect_Skin | 147 MB | Texturas de piel realistas | 0.5-0.7 |
| Flux_Realism_Lora (XLabs) | 22 MB | Realismo ligero, stackeable | 0.8-1.0 |

#### ControlNet (en models/controlnet/)

| Modelo | Tamanio | Funcion |
|---|---|---|
| flux-canny-controlnet-v3 | 3.4 GB | Control por bordes (canny edges) |
| flux-depth-controlnet | 3.0 GB | Control por profundidad (depth maps) |
| control_v11p_sd15_openpose | 1.4 GB | Pose humana (para AnimateDiff con SD1.5) |

#### IPAdapter Flux (en models/ipadapter-flux/)

| Componente | Tamanio | Ruta |
|---|---|---|
| ip-adapter.bin | 5.0 GB | models/ipadapter-flux/ |
| SigLIP vision model | 3.3 GB | models/clip_vision/siglip-so400m-patch14-384/ |
| CLIP Vision ViT-H | 2.4 GB | models/clip_vision/ (para IPAdapter legacy SD1.5) |

#### AnimateDiff (en models/animatediff_models/)

| Motion Module | Tamanio |
|---|---|
| v3_sd15_mm.ckpt | 1.6 GB |
| mm_sd_v15_v2.ckpt | 1.7 GB |

### Custom Nodes instalados

| Nodo | Repo | Para que |
|---|---|---|
| ComfyUI-Manager | ltdrdata | Gestion de nodos desde la UI |
| ComfyUI-IPAdapter-Flux | Shakker-Labs | IPAdapter para Flux (transferencia de imagen) |
| ComfyUI_IPAdapter_plus | cubiq | IPAdapter legacy (SD1.5/SDXL) |
| comfyui_controlnet_aux | Fannovel16 | Preprocessors: openpose, depth, canny, etc. |
| ComfyUI-Advanced-ControlNet | Kosinkadink | Multi-ControlNet stacking |
| ComfyUI-AnimateDiff-Evolved | Kosinkadink | Animaciones cortas (4-16 frames) |
| ComfyUI-VideoHelperSuite | Kosinkadink | Export de video (gif, mp4) |
| ComfyUI_InstantID | cubiq | Preservacion facial avanzada |

---

## 2. Como encender / apagar / reiniciar

ComfyUI corre como servicio systemd. Se prende automaticamente al encender la Thor.

```bash
# Ver estado
sudo systemctl status comfyui.service

# Encender
sudo systemctl start comfyui.service

# Apagar
sudo systemctl stop comfyui.service

# Reiniciar (necesario despues de instalar nodos o si hay crash)
sudo systemctl restart comfyui.service

# Ver logs en tiempo real
sudo journalctl -u comfyui.service -f

# Ver ultimos 50 logs
sudo journalctl -u comfyui.service -n 50
```

Para arranque manual (debug):

```bash
sudo systemctl stop comfyui.service
~/comfyui-env/start-comfyui.sh
# Ctrl+C para parar
```

### Desactivar auto-start

```bash
sudo systemctl disable comfyui.service
# Para reactivar:
sudo systemctl enable comfyui.service
```

---

## 3. Workflows - Como generar imagenes

### IMPORTANTE: No usar "Load Checkpoint" con el archivo FP8

El checkpoint flux1-dev-fp8.safetensors es un archivo todo-en-uno pero tiene el T5-XXL
en FP8 que crashea cuBLAS en Thor. SIEMPRE usa nodos separados:

### Workflow basico: txt2img con Flux

```
Nodos necesarios:

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
   - text: "tu prompt aqui"

5. CLIPTextEncode (negative)
   - text: "" (vacio para Flux)

6. EmptyLatentImage
   - width: 1024, height: 1024

7. KSampler
   - seed: random
   - steps: 20
   - cfg: 3.5 (Flux usa CFG bajo, 3.0-4.5)
   - sampler: euler
   - scheduler: normal
   - denoise: 1.0

8. VAEDecode -> SaveImage

Conexiones:
- UNETLoader MODEL -> KSampler model
- DualCLIPLoader CLIP -> ambos CLIPTextEncode
- CLIPTextEncode pos -> KSampler positive
- CLIPTextEncode neg -> KSampler negative
- EmptyLatentImage -> KSampler latent_image
- KSampler LATENT -> VAEDecode samples
- Load VAE -> VAEDecode vae
- VAEDecode IMAGE -> SaveImage
```

### Workflow img2img con bajo denoising (mantener likeness)

Igual que txt2img pero cambia:

```
- En vez de EmptyLatentImage, usa:
  LoadImage -> VAEEncode -> KSampler (latent_image)
  (conecta Load VAE tambien al VAEEncode)

- KSampler denoise: 0.35-0.45
  (0.35 = cambios sutiles, cara/cuerpo casi intactos)
  (0.45 = cambios moderados, mantiene estructura general)
  (0.55+ = cambios creativos, pierde mas detalle original)
```

### Workflow img2img + LoRAs

```
Encadena nodos Load LoRA entre el UNETLoader y el KSampler:

UNETLoader -> Load LoRA (Flux_Super_Realism, str: 0.8)
           -> Load LoRA (Flux_Perfect_Skin, str: 0.6)
           -> KSampler

Cada Load LoRA tiene:
- model input/output (encadenar)
- clip input/output (encadenar desde DualCLIPLoader)
- lora_name: seleccionar el archivo
- strength_model: 0.6-0.9
- strength_clip: 0.6-0.9

Puedes stackear 2-3 LoRAs sin problemas con 122GB de RAM.
```

### Workflow img2img + ControlNet Depth

```
Agrega estos nodos:

1. LoadImage (imagen de referencia)
2. ControlNetLoader
   - control_net_name: flux-depth-controlnet.safetensors
3. DepthAnythingPreprocessor (de comfyui_controlnet_aux)
   - Conectar LoadImage -> preprocessor
4. ControlNetApplyAdvanced
   - strength: 0.6-0.8
   - start_percent: 0.0
   - end_percent: 1.0
   - Conectar: positive/negative conditioning + controlnet + imagen procesada
   - Output: positive/negative -> KSampler
```

### Workflow con IPAdapter Flux (transferencia de imagen)

```
1. IPAdapterFluxLoader
   - ipadapter: ip-adapter.bin
   - clip_vision: google/siglip-so400m-patch14-384
   - provider: cuda

2. LoadImage (imagen de referencia para estilo/cara)

3. ApplyIPAdapterFlux
   - model: conectar desde UNETLoader (o despues de LoRAs)
   - ipadapter_flux: desde IPAdapterFluxLoader
   - image: desde LoadImage
   - weight: 0.7 (0.5-0.9 segun cuanto quieres transferir)
   - start_percent: 0.0
   - end_percent: 0.8
   - Output MODEL -> KSampler
```

### Workflow AnimateDiff (animaciones cortas, usa SD1.5)

```
NOTA: AnimateDiff NO funciona con Flux. Usa SD1.5.

1. CheckpointLoaderSimple
   - ckpt_name: v1-5-pruned-emaonly.safetensors

2. AnimateDiff Loader
   - model_name: v3_sd15_mm.ckpt (o mm_sd_v15_v2.ckpt)
   - beta_schedule: autoselect
   - motion_scale: 1.0
   - apply_v2_models_properly: true

3. CLIPTextEncode pos/neg (normales)

4. EmptyLatentImage
   - width: 512, height: 512
   - batch_size: 8-16 (= numero de frames)

5. KSampler
   - steps: 20, cfg: 7.5, sampler: euler, scheduler: normal

6. AnimateDiff Combine
   - frame_rate: 8
   - format: gif o mp4

Con ControlNet OpenPose para guiar poses frame a frame:
- Cargar imagenes de pose (una por frame)
- control_v11p_sd15_openpose.pth
- ControlNetApply strength: 0.7-0.9
```

---

## 4. Flags de arranque (start-comfyui.sh)

El script actual:

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

| Flag | Que hace |
|---|---|
| --listen 0.0.0.0 | Accesible desde toda la red |
| --highvram | Mantiene modelos en VRAM (Thor tiene 122GB, sobra) |
| --fp8_e4m3fn-unet | UNET en FP8 nativo (rapido en Blackwell) |
| --bf16-text-enc | Text encoders en bf16 (evita crash cuBLAS con FP8) |
| --bf16-vae | VAE en bf16 (estable) |
| --disable-cuda-malloc | Allocator nativo (evita CUBLAS_NOT_INITIALIZED) |

---

## 5. Bug critico: cuBLAS de pip vs JetPack

### El problema

pip install torch trae nvidia-cublas generico que NO funciona en Thor/Jetson.
Causa: CUBLAS_STATUS_INVALID_VALUE en CUALQUIER matmul, incluso 2x2.

### Como detectarlo

```bash
source ~/comfyui-env/venv/bin/activate
python3 -c "import torch; print(torch.randn(2,2,device='cuda') @ torch.randn(2,2,device='cuda'))"
```

Si crashea con CUBLAS error -> pip cuBLAS sobreescribio el de JetPack.

### Como arreglarlo

```bash
NVIDIA_LIB=~/comfyui-env/venv/lib/python3.12/site-packages/nvidia/cu13/lib

# Backup pip versions — el nombre del backup NO puede terminar en digito!
# torch usa glob("libcublas.so.*[0-9]") para buscar la libreria.
# Si el backup se llama "libcublas.so.13.pip.bak2", termina en "2" y
# torch lo carga EN VEZ del symlink. Siempre usar .old como sufijo.
mv $NVIDIA_LIB/libcublas.so.13 $NVIDIA_LIB/libcublas.so.pip-backup.old
mv $NVIDIA_LIB/libcublasLt.so.13 $NVIDIA_LIB/libcublasLt.so.pip-backup.old
mv $NVIDIA_LIB/libnvblas.so.13 $NVIDIA_LIB/libnvblas.so.pip-backup.old

# Symlink a JetPack
ln -sf /usr/local/cuda/lib64/libcublas.so.13 $NVIDIA_LIB/libcublas.so.13
ln -sf /usr/local/cuda/lib64/libcublasLt.so.13 $NVIDIA_LIB/libcublasLt.so.13
ln -sf /usr/local/cuda/lib64/libnvblas.so.13 $NVIDIA_LIB/libnvblas.so.13
```

IMPORTANTE: Hay que hacer esto CADA VEZ que se instala algo con pip que
trae nvidia-cublas como dependencia. Tambien symlink libnvblas.

Ver guia completa en: [docs/cuda-jetpack-fix.md](docs/cuda-jetpack-fix.md)

---

## 6. Como borrar todo (limpiar disco)

Ver la seccion "Free Resources / Cleanup" en el [README principal](../README.md).

---

## 7. Tips de uso

### Resolucion recomendada para Flux
- txt2img: 1024x1024 o 1024x768
- img2img: misma resolucion que la imagen input (o multiplo de 8)

### CFG para Flux
- Flux usa CFG MUY bajo: 3.0-4.5 (no 7-12 como SD1.5)
- CFG 1.0 = casi sin guia de prompt
- CFG 3.5 = balance ideal
- CFG 7.0+ = sobresaturado, artefactos

### Denoise para img2img (preservar likeness)
- 0.25-0.35: Cambios minimos (ajuste de iluminacion, color)
- 0.35-0.45: Cambios moderados (mantiene cara, cambia detalles)
- 0.45-0.60: Cambios significativos (estructura general se mantiene)
- 0.60+: Cambios grandes (pierde parecido)

### Stackear LoRAs
- Maximo 2-3 LoRAs a la vez para calidad optima
- Combo recomendado: Super_Realism (0.8) + Perfect_Skin (0.5)
- Para anatomia: Perfect_Body (0.8) + Female_Anatomy (0.6)
- Uncensored_V2 se puede stackear con cualquiera (0.6-0.7)

### IPAdapter weights
- 0.3-0.5: Influencia sutil de la imagen de referencia
- 0.5-0.7: Balance entre prompt y referencia
- 0.7-0.9: Imagen de referencia domina
- end_percent: 0.8 (dejar que las ultimas iteraciones refinen sin IPAdapter)

### ControlNet strengths
- Depth: 0.5-0.8 (estructura 3D del cuerpo)
- Canny: 0.4-0.7 (bordes y contornos)
- OpenPose: 0.7-0.9 (pose exacta, para AnimateDiff)

### Memoria (122 GB unified)
- Flux FP8 UNET: ~12 GB
- T5-XXL FP16: ~9 GB
- CLIP-L: ~0.2 GB
- VAE: ~0.3 GB
- LoRA (cada una): 0.02-0.9 GB
- IPAdapter + SigLIP: ~8 GB
- ControlNet: ~3 GB cada uno
- Total tipico con todo cargado: ~35-40 GB (sobran ~80 GB)

---

## 8. Troubleshooting

### Error "CUBLAS_STATUS_INVALID_VALUE"
-> pip cuBLAS sobreescribio JetPack. Ver seccion 5.

### Error "CUBLAS_STATUS_NOT_INITIALIZED"
-> Contexto CUDA corrupto. Reiniciar ComfyUI:
```bash
sudo systemctl restart comfyui.service
```

### Error "flipped_img_txt" en IPAdapter-Flux
-> Mismatch entre ComfyUI y el nodo IPAdapter-Flux. Parchar:
```bash
# En ~/comfyui-env/ComfyUI/custom_nodes/ComfyUI-IPAdapter-Flux/flux/layers.py
# Linea 31, cambiar:
#   self.flipped_img_txt = original_block.flipped_img_txt
# Por:
#   self.flipped_img_txt = getattr(original_block, 'flipped_img_txt', True)
```

### Error "header too large" en IPAdapter
-> Archivo corrupto o extension incorrecta. Verificar con:
```bash
file models/ipadapter-flux/ip-adapter.bin
# Debe decir "Zip archive data" (formato PyTorch .bin)
```

### Error "Allocation on device" (OOM)
-> Reducir resolucion o batch size. Con 122GB es raro pero puede pasar
con muchos modelos cargados + resolucion alta + batch grande.

### Imagenes negras
-> Cambiar --bf16-vae por --fp32-vae en start-comfyui.sh

### LoRA no aparece en la lista
-> Reiniciar ComfyUI (detecta modelos nuevos al arrancar)

### ComfyUI no arranca
```bash
# Ver que fallo
sudo journalctl -u comfyui.service -n 50

# Arranque manual para debug
sudo systemctl stop comfyui.service
~/comfyui-env/start-comfyui.sh
```
