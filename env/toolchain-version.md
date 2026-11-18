# Windows GPU toolchain audit

Verified on 2026-11-18 on Windows 11 25H2 (build `26200.9711`).

## GPU and driver

```powershell
nvidia-smi --query-gpu=name,driver_version,compute_cap --format=csv,noheader
```

```text
NVIDIA GeForce RTX 4070 Laptop GPU, 610.47, 8.9
```

`vulkaninfo --summary` also listed `NVIDIA GeForce RTX 4070 Laptop GPU`.

### Original `nvidia-smi` output

```text
Wed Nov 18 15:47:23 2026
+-----------------------------------------------------------------------------------------+
| NVIDIA-SMI 610.47                 KMD Version: 610.47        CUDA UMD Version: 13.3     |
+-----------------------------------------+------------------------+----------------------+
| GPU  Name                  Driver-Model | Bus-Id          Disp.A | Volatile Uncorr. ECC |
| Fan  Temp   Perf          Pwr:Usage/Cap |           Memory-Usage | GPU-Util  Compute M. |
|                                         |                        |               MIG M. |
|=========================================+========================+======================|
|   0  NVIDIA GeForce RTX 4070 ...  WDDM  |   00000000:01:00.0 Off |                  N/A |
| N/A   43C    P3             20W /   50W |       0MiB /   8188MiB |      0%      Default |
|                                         |                        |                  N/A |
+-----------------------------------------+------------------------+----------------------+

+-----------------------------------------------------------------------------------------+
| Processes:                                                                              |
|  GPU   GI   CI              PID   Type   Process name                        GPU Memory |
|        ID   ID                                                               Usage      |
|=========================================================================================|
|  No running processes found                                                             |
+-----------------------------------------------------------------------------------------+
```

## Toolchain

| Tool | Observed version | Location on the audited PC |
| --- | --- | --- |
| MSVC (`cl.exe`) | VS 2026 Community, toolset `14.51.36231` | `C:/Program Files/Microsoft Visual Studio/18/Community/VC/Tools/MSVC/14.51.36231/bin/Hostx64/x64/cl.exe` |
| Windows SDK | `10.0.26100.0` | `C:/Program Files (x86)/Windows Kits/10/Include/10.0.26100.0` |
| CMake | `4.4.0` | `C:/Program Files/CMake/bin/cmake.exe` |
| Ninja | `1.13.2` | `C:/Program Files/Microsoft Visual Studio/18/Community/Common7/IDE/CommonExtensions/Microsoft/CMake/Ninja/ninja.exe` |
| clang-format | `22.1.3` | `C:/Program Files/Microsoft Visual Studio/18/Community/VC/Tools/Llvm/x64/bin/clang-format.exe` |
| CUDA Toolkit | `13.3` (`nvcc V13.3.33`) | `C:/Program Files/NVIDIA GPU Computing Toolkit/CUDA/v13.3/bin` (`nvcc.exe`, `ptxas.exe`, `cuobjdump.exe`, `nvdisasm.exe`) |
| Vulkan SDK | `1.4.350.0` | `C:/VulkanSDK/1.4.350.0` |
| glslangValidator | `16.2.0` | `C:/VulkanSDK/1.4.350.0/Bin/glslangValidator.exe` |
| spirv-val | `v2026.2` | `C:/VulkanSDK/1.4.350.0/Bin/spirv-val.exe` |
| Nsight Compute CLI | `2026.2.0.0` | `C:/Program Files/NVIDIA Corporation/Nsight Compute 2026.2.0/ncu.bat` |

Ninja and clang-format were available through the Visual Studio installation paths above and were absent from the audit shell's `PATH`. Nsight Graphics was absent from the Windows installed-app entries checked during this audit.

## Compile smoke tests

Run from the repository root in an x64 Visual Studio Developer PowerShell with `nvcc`, `cuobjdump`, `glslangValidator`, and `spirv-val` available on `PATH`. The commands create the test sources and outputs under the Git-ignored `build/` directory.

```powershell
$smoke_dir = 'build/toolchain-smoke'
New-Item -ItemType Directory -Path $smoke_dir -Force | Out-Null

Set-Content -LiteralPath "$smoke_dir/probe.cu" -Encoding ASCII -Value @'
extern "C" __global__ void probe(int* output) { output[0] = 1; }
'@

nvcc -arch=sm_89 --cubin "$smoke_dir/probe.cu" -o "$smoke_dir/probe.cubin"
if ($LASTEXITCODE -ne 0) { throw 'CUDA compilation failed' }
cuobjdump --list-elf "$smoke_dir/probe.cubin"
if ($LASTEXITCODE -ne 0) { throw 'CUDA cubin inspection failed' }

Set-Content -LiteralPath "$smoke_dir/probe.comp" -Encoding ASCII -Value @'
#version 450
layout(local_size_x = 1) in;
void main() {}
'@

glslangValidator -V --target-env vulkan1.0 -o "$smoke_dir/probe.spv" "$smoke_dir/probe.comp"
if ($LASTEXITCODE -ne 0) { throw 'GLSL compilation failed' }
spirv-val --target-env vulkan1.0 "$smoke_dir/probe.spv"
if ($LASTEXITCODE -ne 0) { throw 'SPIR-V validation failed' }
```

- **CUDA passed.** `nvcc` produced `probe.cubin`, and `cuobjdump --list-elf` listed `probe.sm_89.cubin`.
- **Vulkan passed.** `glslangValidator` produced `probe.spv`, and `spirv-val` exited with code `0` for the Vulkan 1.0 target environment.

This audit verifies GPU enumeration, CUDA device compilation, and Vulkan shader compilation and validation. GPU dispatch and output correctness are Phase 1 checks. Profiler counter permissions are pending the profiling smoke tests.
