# Prism-Vega Baseline

Stand: 24.09.2026

## Projekt

Eigener Fork von PrismML llama.cpp für den Betrieb auf
2× AMD Radeon RX Vega 56 / gfx900.

GitHub:

* Eigener Fork: Sayrin/prism-vega
* Upstream: PrismML-Eng/llama.cpp
* Branch: prism

## Hardware

### GPUs

2× ASRock Phantom Gaming X Radeon RX Vega 56

Je GPU:

* Architektur: Vega 10
* Target: gfx900
* VRAM: 8176 MiB HBM2

PCIe:

* GPU0: 0000:03:00.0
* GPU1: 0000:06:00.0

### CPU

Intel Core i5-7600K

## Software

OS:

* Ubuntu 20.04.6 LTS
* Kernel: 5.4.0-216

AMD:

* ROCm: 6.1.0
* HIP: 6.1.40091
* AMD Clang: 17

CMake:

* 3.31.10
* `/opt/cmake/bin/cmake`

## Prism llama.cpp

Repository:

* https://github.com/Sayrin/prism-vega

Branch:

* prism

Prism Basis:

* prism-b10735-842b188

Baseline Commit:

* 86806d0d6
* ggml hip: fallback to device VRAM when hipMemGetInfo fails

### Build

```bash
/opt/cmake/bin/cmake -S ~/prism-llama.cpp -B ~/prism-llama.cpp/build \
  -DGGML_HIP=ON \
  -DAMDGPU_TARGETS=gfx900 \
  -DCMAKE_BUILD_TYPE=Release \
  -DLLAMA_OPENSSL=ON

cmake --build ~/prism-llama.cpp/build --config Release -j$(nproc)
```

## Vega VRAM Fix

Problem:

`hipMemGetInfo()` returns an error on the Vega/gfx900 setup.

This caused llama.cpp to report:

```text
ROCm0: Radeon RX Vega (0 MiB, 0 MiB free)
ROCm1: Radeon RX Vega (0 MiB, 0 MiB free)
```

The fallback in:

```text
ggml/src/ggml-cuda/ggml-cuda.cu
```

uses:

```text
cudaGetDeviceProperties()
```

to obtain `totalGlobalMem` when `hipMemGetInfo()` fails.

After the patch:

```text
ROCm0: Radeon RX Vega (8176 MiB, 8176 MiB free)
ROCm1: Radeon RX Vega (8176 MiB, 8176 MiB free)
```

Important:
The reported "free" value is the total device VRAM in the fallback path.
Actual VRAM usage is monitored through:

```text
/sys/class/drm/card*/device/mem_info_vram_used
```

Example:

```bash
watch -n 1 'for f in /sys/class/drm/card*/device/mem_info_vram_used; do printf "%s: %.0f MiB\n" "$f" "$(($(cat "$f") / 1024 / 1024))"; done'
```

## Model

Ternary Bonsai 2 27B

Repository:

```text
prism-ml/Ternary-Bonsai-2-27B-gguf
```

Main model:

```text
Ternary-Bonsai-2-27B-PQ2_0.gguf
```

Size:

```text
7,206,168,928 bytes
~6.8 GB
```

Base architecture:

```text
Qwen3.8-27B
```

Approximate parameters:

```text
27.36B
```

Native context:

```text
262,144
```

Model type:

```text
Hybrid attention
~75% linear attention
```

## Vision

Vision works with the separate mmproj model:

```text
Ternary-Bonsai-2-27B-mmproj-Q8_0.gguf
```

Size:

```text
629,246,976 bytes
~601 MB
```

Architecture:

```text
Image
  ↓
Vision encoder / mmproj
  ↓
Image embeddings
  ↓
Bonsai 2 27B
  ↓
Text response
```

Test image:

```text
/tmp/bonsai-test.png
```

The model correctly identified:

* red square
* blue circle
* green rectangle

This confirms that vision inference works on both Vega 56 GPUs.

## Dual GPU Configuration

Working configuration:

```text
-dev ROCm0,ROCm1
-sm layer
-ts 1,1
```

Architecture:

```text
Bonsai 2 27B
      │
      ├── ROCm0 → Vega 56 / 8 GB
      │
      └── ROCm1 → Vega 56 / 8 GB
```

## Stable Vision Configuration

Current selected baseline:

```text
80,000 context
2× Vega 56
PQ2_0
Vision enabled
```

Start command:

```bash
~/prism-llama.cpp/build/bin/llama-cli \
  -m ~/models/bonsai/Ternary-Bonsai-2-27B-PQ2_0.gguf \
  --mmproj ~/models/bonsai/Ternary-Bonsai-2-27B-mmproj-Q8_0.gguf \
  -dev ROCm0,ROCm1 \
  -sm layer \
  -ts 1,1 \
  -c 80000 \
  -n 128 \
  --reasoning off
```

At 80K context:

```text
Vision: WORKING
Dual GPU: WORKING
```

Observed VRAM during image inference:

```text
GPU0: ~8072 MiB
GPU1: ~7219 MiB
```

Observed performance:

```text
Prompt: ~26.0 t/s
Generation: ~12.7 t/s
```

## Context Tests

### 50K

WORKING

Vision:

```text
Prompt: ~27.0 t/s
Generation: ~12.6 t/s
```

### 80K

WORKING

Vision:

```text
Prompt: ~26.0 t/s
Generation: ~12.7 t/s
```

### 90K

NOT STABLE

Model loaded and accepted the image, but image inference caused:

```text
ROCm error: out of memory
current device: 0
Aborted (core dumped)
```

Therefore:

```text
80K = selected stable baseline
90K = OOM
```

## Large Context Without Vision

Previous tests:

```text
62.5K  → working
93.75K → working
105K   → working
109.375K → working
117.187K → OOM
125K → compute-buffer OOM
250K → KV-cache OOM
500K → KV-cache OOM
```

For the future Ollama integration we selected:

```text
105K without vision
80K with vision
```

## Earlier Model Tests

### Qwen2.5-Coder 7B Q4_K_M

1 GPU:

```text
Prompt: 84.63 t/s
Generation: 39.44 t/s
```

2 GPUs:

```text
Prompt: 47.91 t/s
Generation: 21.52 t/s
```

Conclusion:

Small models are faster on one Vega.

### Qwen2.5-Coder 14B

1 GPU:

```text
OOM
```

2 GPUs:

```text
-ts 1,1
Prompt: 37.56 t/s
Generation: 14.16 t/s
```

```text
-ts 3,2
Prompt: 37.14 t/s
Generation: 14.27 t/s
```

### Qwen2.5-Coder 27B

Model loads on both GPUs but inference fails because approximately:

```text
~287 MiB
```

additional VRAM is required.

## Important Lessons

1. Stock llama.cpp/Ollama did not correctly detect Vega VRAM through `hipMemGetInfo()`.

2. The device-property fallback fixes VRAM detection.

3. Both Vega 56 GPUs can run the Bonsai 2 27B PQ2_0 model.

4. Bonsai 2 PQ2_0 requires the PrismML llama.cpp implementation.

5. Vision requires the separate mmproj GGUF.

6. Vision inference works on the dual-Vega system.

7. 80K context with vision is currently the selected stable target.

8. 90K context with vision causes VRAM exhaustion during image inference.

9. Actual VRAM usage must be monitored through the DRM sysfs interface rather than trusting the fallback "free VRAM" value.

## Next Project Phase

Do NOT modify the working baseline until it is safely preserved.

Next goal:

```text
Prism-Vega
    ↓
Bonsai 2 27B PQ2_0
    ↓
Ollama integration
    ↓
Ollama API
    ↓
OpenCode
```

Long-term goal:

```text
OpenCode
    ↓
Ollama
    ↓
Bonsai 2 27B
    ├── Vega 56 #0
    └── Vega 56 #1
```

Vision should remain available through the corresponding mmproj model.

## Baseline Status

```text
[OK] Dual Vega 56 detected
[OK] gfx900 ROCm build
[OK] Prism llama.cpp
[OK] Vega VRAM fallback
[OK] Bonsai 2 27B PQ2_0
[OK] Dual GPU inference
[OK] Vision mmproj
[OK] 50K vision
[OK] 80K vision
[FAIL] 90K v
```

