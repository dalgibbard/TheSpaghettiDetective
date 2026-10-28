---
title: Running Obico Server with AMD GPU acceleration (experimental)
---

:::info
Experimental. This guide applies to Linux hosts with an AMD GPU that the
**ROCm 7.2** stack supports — Radeon RX 6000 (gfx1030), RX 7000 (gfx1100 /
gfx1101 / gfx1102) and RX 9000 (gfx1200 / gfx1201) dGPUs, Radeon PRO cards,
AMD Instinct accelerators, and the Ryzen AI Max 300 / selected Ryzen AI
300 & 400 series APUs. The AMD image currently targets amd64 hosts.

**Not supported: Ryzen 4000G/5000G iGPUs** (Renoir/Lucienne/Cezanne, which
report as `gfx90c`). ROCm 6.x and 7.x no longer ship compute kernels for that
architecture: its rocBLAS kernel sets cover `gfx908`, `gfx90a`, `gfx942` and
RDNA2+ only. Field-tested on a Ryzen 5700G, the ML container aborts with
`rocBLAS error: Cannot read TensileLibrary.dat: Illegal seek for GPU arch :
gfx90c` and then crashes. The older, deprecated ROCm execution provider was
also tested (ONNX Runtime 1.21 on ROCm 6.4.4): it loads and even assigns the
provider, but inference hits the same missing-kernels wall — forcing the
`gfx908` kernel set via `HSA_OVERRIDE_GFX_VERSION=9.0.8` ends in
`HSA_STATUS_ERROR_ILLEGAL_INSTRUCTION` because those CDNA kernels cannot run
on the iGPU's Vega hardware. Use the standard CPU image on those systems —
and it is not only a compatibility call: measured on the same Ryzen 5700G,
the model runs in ~38 ms on 8 CPU cores (~66 ms limited to 4 cores) versus
~201 ms through the iGPU via WebGPU, so GPU acceleration for AMD stays
scoped to RX 6000+ / Ryzen AI Max-class hardware.
Check your exact GPU model in the [ROCm compatibility matrix](https://rocm.docs.amd.com/en/latest/compatibility/compatibility-matrix.html)
before enabling AMD acceleration.
:::

## Why a separate image

ONNX Runtime ships a different Python distribution per GPU vendor. The CUDA
and OpenVINO builds replace the same `onnxruntime` package and cannot be
installed side by side, so AMD support is a third image, built from
`ml_api/Dockerfile.amd`. AMD deprecated ONNX Runtime's older ROCm execution
provider; the supported path today is the
[MIGraphX execution provider](https://onnxruntime.ai/docs/execution-providers/MIGraphX-ExecutionProvider.html),
which AMD publishes as a wheel per ROCm release. The image starts from AMD's
official `rocm/dev-ubuntu-24.04` image (Ubuntu 24.04 + AMD's signed apt
sources + the HIP runtime) and installs the `migraphx` packages and the
matching `onnxruntime_migraphx` wheel from the pinned ROCm 7.2.1 repository.

AMD's older, deprecated ROCm execution provider (last shipped as the
`onnxruntime_rocm` wheels, up to ONNX Runtime 1.21 / ROCm 6.4.4) is not kept
as a separate "legacy" image: it predates MIGraphX support for current
Radeon cards and still depends on rocBLAS, so it shares the same architecture
kernel coverage. A legacy-stack image would add build matrix and maintenance
without unlocking any GPU that the current image cannot already run on.

## Host requirements

The container carries the HIP and MIGraphX runtime libraries; the host only
needs the amdgpu kernel driver with KFD (compute) enabled:

- A recent Linux kernel and the `amdgpu` driver (any current distro kernel
  ships it).
- `rocm-smi` is not required on the host, but running it once is the quickest
  way to prove your GPU is usable:

```bash
ls -l /dev/kfd /dev/dri/render*
```

If `/dev/kfd` is missing, ROCm compute is not enabled. On systems with an
IOMMU, APUs and many dGPUs need `iommu=pt` on the kernel command line for
KFD to initialize. For Radeon/Ryzen specifics, see AMD's
[Use ROCm on Radeon and Ryzen](https://rocmdocs.amd.com/projects/radeon/en/latest/docs/compatibility.html)
compatibility guide and the [ROCm compatibility matrix](https://rocm.docs.amd.com/en/docs-7.2.0/compatibility/compatibility-matrix.html).

## Start the AMD image

From the repository root, run:

```bash
docker compose -f docker-compose.yml -f docker-compose.amd.yml up -d --build
```

The override builds `ml_api/Dockerfile.amd`, selects the MIGraphX execution
provider, and passes `/dev/kfd` and `/dev/dri` into the `ml_api` container.
Like the Intel override, it is explicit: a normal `docker compose up` keeps
the existing CUDA deployment behavior.

If you do not want to build locally, the fork CI publishes prebuilt images:

```bash
docker pull dalgibbard/obico-ml-api:amd-latest
```

and the Compose override can use `image: dalgibbard/obico-ml-api:amd-latest`
instead of the `build:` stanza.

## Confirm GPU inference

Inspect the ML API logs:

```bash
docker compose logs ml_api
```

A successful AMD load includes:

```
ONNX Runtime execution providers: ['MIGraphXExecutionProvider', 'CPUExecutionProvider']
----- Trying to load weights: /model_cache/ml_api/onnx/model-weights.onnx - use_gpu = True -----
Succeeded!
```

If the MIGraphX provider initializes but your GPU is not in the ROCm 7.2
kernel list, the provider can abort the process instead of falling back
(that is exactly what happens on `gfx90c` Renoir/Cezanne iGPUs), so
**verify your GPU first** with a throwaway probe before switching your ML
service over:

```bash
docker run --rm --device /dev/kfd --device /dev/dri \
  dalgibbard/obico-ml-api:amd-latest python -u -c "
import onnxruntime as o
s = o.InferenceSession('/model_cache/ml_api/onnx/model-weights.onnx',
    providers=[('MIGraphXExecutionProvider', {'device_id': '0'}), 'CPUExecutionProvider'])
print('assigned providers:', s.get_providers())
"
```

A supported GPU prints `assigned providers: ['MIGraphXExecutionProvider', ...]`
and exits 0. A crash means your GPU is not supported by the current ROCm
release — stay on the CPU image.

A host with a ROCm-supported GPU architecture but no usable device (e.g. no
`/dev/kfd` passed through) can still run this image: the wheel contains
a full CPU-capable ONNX Runtime, so leaving `ML_API_GPU_PROVIDER` unset (or
setting it to `CPUExecutionProvider`) works too.

## Selecting a device

For systems with more than one GPU, pin the device with
`ML_API_MIGRAPHX_DEVICE_ID` (default `0`) in a local Compose override.
