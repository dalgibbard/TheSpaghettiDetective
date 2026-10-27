---
title: Running Obico Server with AMD GPU acceleration (experimental)
---

:::info
Experimental. This guide applies to Linux hosts with an AMD GPU that ROCm
supports — Radeon RX 6000/7000 series cards and AMD APUs such as the Ryzen
5000G/6000G series iGPU (gfx1030). The AMD image currently targets amd64
hosts. AMD support was validated against the ROCm 7.2 compatibility matrix;
check your exact GPU model there before relying on it.
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

If the MIGraphX provider cannot initialize on your GPU, the loader retries
the same ONNX model with `CPUExecutionProvider`, so the server stays usable —
just slower. Check `/dev/kfd`, `iommu=pt`, and the ROCm compatibility matrix
for your GPU model in that case.

A host with no ROCm-capable GPU can still run this image: the wheel contains
a full CPU-capable ONNX Runtime, so leaving `ML_API_GPU_PROVIDER` unset (or
setting it to `CPUExecutionProvider`) works too.

## Selecting a device

For systems with more than one GPU, pin the device with
`ML_API_MIGRAPHX_DEVICE_ID` (default `0`) in a local Compose override.
