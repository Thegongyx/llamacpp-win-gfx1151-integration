# 在 Windows 下编译 ROCmFPX 官方 Vulkan 版 llama.cpp 引擎（vulkan_official）

本文记录在本机 Windows + AMD **gfx1151**（Radeon 8060S / Strix Halo）环境下，编译
`ROCmFPX/ROCmFPX`（官方主线，HEAD `c3b1c99`；**实验性 CM1 加速需 ≥`aed0d5fd9`**，见 §5）的 **Vulkan 引擎**（即 `vulkan_official`）的步骤。
产物含 **Charlie ROCmFP4 Vulkan 快速后端插件**（`ROCmFPXVulkan0`）。

> 方法已在本机实测通过：`llama-server.exe --list-devices` 能列出 `Vulkan0` 与 `ROCmFPXVulkan0`，
> 依赖齐全。

文中路径均用**占位符**，请按实际环境替换（不写死绝对路径）：

| 占位符 | 含义 | 示例（仅示意） |
|---|---|---|
| `<repo_root>` | ROCmFPX/ROCmFPX 源码根目录 | `/path/to/ROCmFPX` |
| `<vs_root>` | Visual Studio / Build Tools 安装根目录 | `C:\Program Files (x86)\Microsoft Visual Studio\18\BuildTools` |
| `<vulkan_sdk>` | Vulkan SDK 安装根目录 | `C:\VulkanSDK\1.4.357.0` |
| `<build_dir>` | 构建目录（= `<repo_root>/build-win-vulkan`） | `<repo_root>\build-win-vulkan` |

---

## 1. 目标与范围

- **仓库**：`ROCmFPX/ROCmFPX`（git 官方主线，HEAD `c3b1c99`），分支 `main`。
- **目标**：编译 **Vulkan 后端** —— `ggml-vulkan`（含 Charlie `ROCMFPX_VULKAN_PLUGIN` 快速后端
  `ggml-rocmfpx-vulkan.dll`）+ `llama-server` 工具链。
- **只编 Vulkan**：`GGML_VULKAN=ON`、`GGML_HIP=OFF`、`GGML_CUDA=OFF`。
- 产物集中在 `<build_dir>\bin\Release\`，核心是 **`llama-server.exe`** 与两个 Vulkan 后端 DLL
  （`ggml-vulkan.dll` + `ggml-rocmfpx-vulkan.dll` + 插件包装器 `rocmfpx-vulkan-plugin.dll`）。

> **与 HIP 版的区别**：Vulkan 版用普通 **MSVC cl**（VS 生成器）即可，**不需要** ROCm/HIP clang、
> 不需要锁 MSVC 14.44（那是 HIP 编译的坑）。Vulkan 编译只要 **Vulkan SDK**（含 `glslc` 编 shader）。
> GPU 架构由 Vulkan 运行时决定，编译期**不需要** `gfx1151` 目标。

---

## 2. 前置条件（本机已装，实测路径）

| 组件 | 说明 | 关键点 |
|---|---|---|
| **MSVC (VS Build Tools)** | 提供 MSVC 编译器 | VS 生成器用默认工具集即可（Vulkan 无 HIP clang 冲突） |
| **Vulkan SDK** | 提供 `glslc`（编 shader）+ 头文件/lib | `glslc.exe` 在 `<vulkan_sdk>\Bin`；`Lib/cmake` 供 CMake find_package |
| **cmake** | 配置 | VS 内置或独立安装 |
| **git** | 拉取仓库 | — |

> **VULKAN_SDK 环境变量**：CMake 找 Vulkan 用 `<vulkan_sdk>/Lib/cmake`（见 §3 `CMAKE_PREFIX_PATH`）。
> `glslc` 需在 PATH 或由 CMake 找到；可用 `set "VULKAN_SDK=<vulkan_sdk>"` + 加 `Bin` 到 PATH。

---

## 3. 关键构建脚本（configure + build）

Vulkan 版用 **Visual Studio 生成器**（实测为 `Visual Studio 18 2026`，x64），命令：

```bat
@echo off
rem 1) 激活 MSVC x64 环境（默认工具集即可，Vulkan 无需锁 14.44）
call "<vs_root>\VC\Auxiliary\Build\vcvars64.bat" || exit /b 1

rem 2) 加 VS cmake + Vulkan SDK（glslc）到 PATH
set "PATH=<vs_root>\Common7\IDE\CommonExtensions\Microsoft\CMake\CMake\bin;<vulkan_sdk>\Bin;%PATH%"
set "VULKAN_SDK=<vulkan_sdk>"

set "SRC=<repo_root>"
set "BLD=%SRC%\build-win-vulkan"

rem 3) 配置：仅 Vulkan，含 Charlie ROCmFP4 插件
cmake -S "%SRC%" -B "%BLD%" -G "Visual Studio 18 2026" -A x64 ^
  -DCMAKE_BUILD_TYPE=Release ^
  -DBUILD_SHARED_LIBS=ON ^
  -DGGML_VULKAN=ON ^
  -DGGML_HIP=OFF ^
  -DGGML_CUDA=OFF ^
  -DGGML_CPU=ON ^
  -DGGML_OPENMP=ON ^
  -DGGML_NATIVE=ON ^
  -DROCMFPX_VULKAN_PLUGIN=ON ^
  -DCMAKE_PREFIX_PATH="<vulkan_sdk>/Lib/cmake" ^
  -DLLAMA_BUILD_SERVER=ON ^
  -DLLAMA_BUILD_WEBUI=ON ^
  -DGGML_BUILD_TESTS=OFF -DLLAMA_BUILD_TESTS=ON || exit /b 1

rem 4) 编译（VS 生成器在 Release 配置）
cmake --build "%BLD%" --config Release --target llama-server llama-cli llama-bench -j %NUMBER_OF_PROCESSORS% || exit /b 1

echo Build OK: %BLD%\bin\Release\llama-server.exe
exit /b 0
```

运行：`& cmd.exe /c "<repo_root>\build-win-vulkan.bat"`。

> **web UI（可选）**：`vulkan_official` 默认不带内嵌 web UI（`/` 返回 404）。若要把预编译的 LlamaUI
> 资产嵌入，先放 `<repo_root>\tools\ui\dist\`，再用 `cmake --build --config Release --target llama-server`
> 重编（日志出现 `UI: using pre-built assets ...`，`llama-server-impl.dll` 体积变大即嵌入成功）。

---

## 4. 关键 CMake 开关说明

| 开关 | 值 | 作用 |
|---|---|---|
| `GGML_VULKAN` | `ON` | 开启 Vulkan 后端（**必须**） |
| `GGML_VULKAN_ROCMFP4_COOPMAT` | `ON` | （实验性）让 ROCmFP4 / ROCmFP4-FAST 走 Vulkan **CM1（cooperative matrix，F32 累加）**；默认 `OFF`（标量）。须源码 ≥`aed0d5fd9` |
| `GGML_HIP` / `GGML_CUDA` | `OFF` | 只编 Vulkan，避免混淆（可选，显式关） |
| `ROCMFPX_VULKAN_PLUGIN` | `ON` | 启用 **Charlie ROCmFP4 Vulkan 快速后端**（`ROCmFPXVulkan0`，速度与 fork 一致）；`OFF` 则只有普通 `Vulkan0`（较慢） |
| `BUILD_SHARED_LIBS` | `ON` | 共享库（`ggml-vulkan`/`ggml-rocmfpx-vulkan` 为独立 DLL） |
| `GGML_CPU` / `GGML_OPENMP` | `ON` | CPU + OpenMP |
| `GGML_NATIVE` | `ON` | 宿主机 `-march=native` |
| `CMAKE_PREFIX_PATH` | `<vulkan_sdk>/Lib/cmake` | 让 CMake 找到 Vulkan 的 `Vulkan::Vulkan` 目标 |
| `LLAMA_BUILD_SERVER` / `LLAMA_BUILD_WEBUI` | `ON` / `ON` | 编 server；可选内嵌 web UI |

> **GPU 架构**：Vulkan 后端**不需要** `gfx1151` 之类的目标（Vulkan 运行期自动适配）；`glslc`
> 只把 shader 编译成 SPIR-V，与具体 GPU 无关。

---

## 5. 实验性 CM1（cooperative-matrix）加速构建（可选）

> 官方主线自 **`aed0d5fd9`**（PR #21/#22，qwen4exp CM1 公开发布）起，新增**实验性 Vulkan ROCmFP4 CM1**
> 加速通道：把 **ROCmFP4 / ROCmFP4-FAST** 的矩阵乘通过 **Vulkan cooperative matrix（CM1，F32 累加）** 执行。
> 默认**关闭**；源码须含 `GGML_VULKAN_ROCMFP4_COOPMAT` 选项（`c3b1c99` **不具备**，需 ≥`aed0d5fd9`）。

> **现成产物**：本仓库的 **`..\llamacpp-engines\vulkan_official_cm1\`** 已附带按本节省略编译好的 CM1 引擎
> （`ggml-vulkan.dll` 含 CM1 shader + Charlie 插件 `ggml-rocmfpx-vulkan.dll` / `rocmfpx-vulkan-plugin.dll`，
> 可直接 `llama-server.exe --list-devices` 验证，列出 `Vulkan0` 与 `ROCmFPXVulkan0`），并已带 `.installed` 标记，
> 作为 NovaMax 引擎（variant = vulkan）可直接选用。若只想要现成引擎、不重新编译，可跳过下方 §5.2-§5.4。

### 5.1 与普通打包的关系

- CM1 是**独立可选通道**：**不改** GGUF 格式、量化、权重、上下文窗口、chat template，与 MTP / n-gram 投机解码无关。
- 建议**独立构建目录**（`build-win-vulkan-cm1`），保留既有 `build-win-vulkan` 引擎不覆盖。
- 只有 **FP4 / FP4-FAST**（`Q4_0_ROCMFP4`、`Q4_0_ROCMFP4_FAST`）能选此路径；CM2、其他 ROCmFPX 格式、
  HIP、GGUF layout、模型权重均不变。

### 5.2 构建（在 §3 基础上只加一个开关）

配置时追加 `-DGGML_VULKAN_ROCMFP4_COOPMAT=ON`（其余同 §3），用独立目录：

```bat
set "SRC=<repo_root>"
set "BLD=%SRC%\build-win-vulkan-cm1"

cmake -S "%SRC%" -B "%BLD%" -G "Visual Studio 18 2026" -A x64 ^
  -DCMAKE_BUILD_TYPE=Release ^
  -DBUILD_SHARED_LIBS=ON ^
  -DGGML_VULKAN=ON ^
  -DGGML_VULKAN_ROCMFP4_COOPMAT=ON ^
  -DGGML_HIP=OFF ^
  -DGGML_CUDA=OFF ^
  -DGGML_CPU=ON ^
  -DGGML_OPENMP=ON ^
  -DGGML_NATIVE=ON ^
  -DROCMFPX_VULKAN_PLUGIN=ON ^
  -DCMAKE_PREFIX_PATH="<vulkan_sdk>/Lib/cmake" ^
  -DLLAMA_BUILD_SERVER=ON ^
  -DLLAMA_BUILD_WEBUI=ON ^
  -DGGML_BUILD_TESTS=OFF -DLLAMA_BUILD_TESTS=ON || exit /b 1

cmake --build "%BLD%" --config Release --target llama-server llama-cli llama-bench -j %NUMBER_OF_PROCESSORS% || exit /b 1
```

> shader 编译前置：`glslc`（<vulkan_sdk>\Bin）须支持 `GL_KHR_cooperative_matrix`——
> 配置日志应见 `GL_KHR_cooperative_matrix supported by glslc`（Vulkan SDK 2026.x 已含）。

### 5.3 运行时选择（环境变量，默认标量）

构建后**仍走标量路径**，需在启动模型**进程前**选择：

| 选择 | 启动前设置 |
|---|---|
| 启用实验路径 | `GGML_VK_ROCMFP4_COOPMAT=1` |
| 同构建回标量 | `GGML_VK_ROCMFP4_COOPMAT=0`，或取消该变量 |
| 构建排除 CM1 shader | `-DGGML_VULKAN_ROCMFP4_COOPMAT=OFF` |

启动日志出现 `ROCmFP4 CM1 request: enabled (experimental, F32 accumulation)` 即生效；
若见 `scalar fallback (build or device unsupported)` = 未激活（要求设备 **CM1 + F32 累加**，gfx1151 / Strix Halo 满足）。

### 5.4 验证

- 构建产物 `ggml-vulkan.dll` 体积 ~70+MB（CM1 shader `matmul_rocmfp4_f32` / `matmul_rocmfp4_fast_f32` 已编入）。
- `--list-devices` 仍可列出 `Vulkan0` 与 `ROCmFPXVulkan0`（前提同 §7 插件已加载）。

> ⚠️ 即使设了 `GGML_VK_ROCMFP4_COOPMAT=1`，**旧版构建**（如 c3b1c99 或未带该选项的打包）也不会获得该功能——
> 必须源码含 `GGML_VULKAN_ROCMFP4_COOPMAT` 并**重编**，启动日志才可能出现 `enabled`。

---

## 6. 验证

构建完成后（把 `<vulkan_sdk>\Bin` 加到 PATH），探测：

```powershell
$env:Path = "<vulkan_sdk>\Bin;$env:Path"
& "<build_dir>\bin\Release\llama-server.exe" --version
& "<build_dir>\bin\Release\llama-server.exe" --list-devices
```

预期（含插件时）：

```
ggml_vulkan: found 1 Vulkan devices:
  Vulkan0: AMD Radeon(TM) 8060S Graphics (...)
rocmfpx-plugin: info: registered 1 ROCmFPX Vulkan device(s)
rocmfpx-plugin: info: loaded rocmfpx-vulkan-charlie
Available devices:
  Vulkan0: ...
  ROCmFPXVulkan0: ...   <- 有这行说明 Charlie 插件已加载
```

---

## 7. 使用 `ROCmFPXVulkan0` 快速后端（Charlie ROCmFP4，可选）

> 官方 `vulkan_official` **默认用普通 `Vulkan0` 后端（较慢）**；快速通道是可选插件后端
> **`ROCmFPXVulkan0`**（`ROCMFPX_VULKAN_PLUGIN=ON` 编译的 "Charlie ROCmFP4 Vulkan" 路径，
> 速度与 `vulkan_qwen4exp` 相当）。插件**不替换** `Vulkan0`，须显式加载 + 选择。

**两个必要 DLL**（须同一编译，同置于引擎目录）：
- `rocmfpx-vulkan-plugin.dll`（ABI 包装器，`ROCMFPX_PLUGIN_PATH` 指向它）
- `ggml-rocmfpx-vulkan.dll`（~92MB 兄弟后端）

**启用步骤**：
1. 设用户级环境变量（持久化；**必须重启 NovaMax 后端**，llama-server 子进程才继承）：
   ```powershell
   [Environment]::SetEnvironmentVariable('ROCMFPX_PLUGIN_PATH',
     '<引擎目录>\rocmfpx-vulkan-plugin.dll','User')
   ```
2. 模型参数（webui 加额外/自定义参数；键映射成 `--<key> <value>`）：

   | 参数键 | 值 | 作用 |
   |---|---|---|
   | **`device`** | `ROCmFPXVulkan0` | **关键**——选快速插件后端（默认 `Vulkan0` 慢）|
   | `ctk` | `q8_0` | KV cache 键量化（提速 prefill/decode）|
   | `ctv` | `q8_0` | KV cache 值量化（提速）|

   ⚠️ **别用 `dev`**：`dev`→`--dev`，llama-server 报 `invalid argument: --dev`；须用 **`device`**→`--device ROCmFPXVulkan0`。
3. 启动模型，日志应出现：
   ```
   rocmfpx-plugin: info: registered 1 ROCmFPX Vulkan device(s)
   rocmfpx-plugin: info: loaded rocmfpx-vulkan-charlie
   ```
   且 `llama-server.exe --list-devices` 同时列出 `Vulkan0` 和 **`ROCmFPXVulkan0`**。

> **排错**：若报 `invalid device: ROCmFPXVulkan0` = 插件未加载，多半是**未重启 NovaMax**
> （环境变量没被 llama-server 子进程继承）。

---

## 8. 坑与注意事项

1. **`VULKAN_SDK` 环境变量**：CMake 找 Vulkan 用 `CMAKE_PREFIX_PATH=<vulkan_sdk>/Lib/cmake`；
   `glslc` 在 `<vulkan_sdk>\Bin`（编译 shader 必须）。若报 `Could not find Vulkan` 或 `glslc not found`，
   检查这两处。
2. **VS 生成器**：用 `-G "Visual Studio 18 2026" -A x64`，编译用 `--config Release`（不是 Ninja 的
   `Debug/Release` 一并）。若用 Ninja 生成器则去掉 `--config`。
3. **MSVC 工具集**：Vulkan 用默认工具集即可（本机 14.51，无 HIP clang 的 `isgreater` 冲突）。
   **不要**把 HIP 那条"锁 14.44"的坑套到 Vulkan 上。
4. **`ROCMFPX_VULKAN_PLUGIN` 必须与插件 DLL 同编译**：`rocmfpx-vulkan-plugin.dll` 与
   `ggml-rocmfpx-vulkan.dll` 须来自同一次编译，同置于引擎目录；`ROCMFPX_PLUGIN_PATH` 用**绝对路径**。
5. **普通 `Vulkan0` 较慢**：若没设 `device=ROCmFPXVulkan0`，会落到慢速普通 Vulkan 后端。
6. **web UI**：默认无；要内嵌需预编译 `tools\ui\dist` 后重编（见 §3 注）。

---

## 9. 一句话总结

Vulkan 编译成功的充要条件 = **Vulkan SDK（含 `glslc`）+ MSVC（VS 生成器）+ `-DGGML_VULKAN=ON` +
`-DROCMFPX_VULKAN_PLUGIN=ON`（要 Charlie 快速后端）+ `CMAKE_PREFIX_PATH=<vulkan_sdk>/Lib/cmake`**，
一次 `cmake --build --config Release --target llama-server`。要用 `ROCmFPXVulkan0` 再设
`ROCMFPX_PLUGIN_PATH` 指向插件 DLL 并重启后端。
