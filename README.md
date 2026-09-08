# llamacpp-win-gfx1151-integration

本仓库的**主要目的**是沉淀：各类 **ROCmFPX / ROCmFP4 版 llama.cpp 引擎** 在
**Windows + AMD gfx1151 GPU（Radeon 8060S / Strix Halo）** 环境下的——

- **编译方法**（见 `docs\build\*.md`，含通用 HIP 版与 ciru-rocmfpx Kairic Edge 分支版）；
- 以及 **编译好的引擎**（`llamacpp-engines\` 内各引擎目录，含全部依赖 DLL，可直接本地推理）。

**附带功能**：把这些引擎打包成 **NovaMax 可用的适配版**。`llamacpp-engines\` 内每个引擎目录
即"完整编译产物 + `.installed` 适配标记"，NovaMax 通过 `.installed` 识别即可选用
（详见下文「（附带）NovaMax 适配」）。NovaMax 适配是打包后的额外便利，**不是本仓库的主要目的**。

## 目录结构

```
llamacpp-win-gfx1151-integration\
├── README.md                            ← 本文件（引擎总览 / 编译方法入口 / NovaMax 适配说明）
├── docs\                                ← 文档
│   ├── build\                           ← 【主】编译方法
│   │   ├── BUILD-ROCM-HIP-GENERIC.md    ← Windows HIP/ROCm 通用编译（gfx1151）
│   │   ├── BUILD-CIRU-ROCMFPX-WINDOWS.md← ciru-rocmfpx Kairic Edge（Windows + 3 处 Windows 移植）
│   │   └── BUILD-VULKAN-ROCMFPX-WINDOWS.md ← ROCmFPX 官方 Vulkan 版（vulkan_official + ROCmFPXVulkan0 插件）
│   └── novamax-llamacpp-skill\
│       └── SKILL.md                     ← 【附】NovaMax 接入详细技能文档
└── llamacpp-engines\                    ← 【主】编译好的引擎（含全部依赖 DLL），并附 .installed 适配标记
    ├── roc_official\                    ← 官方版：HIP/ROCm（官方 ROCmFPX 主线）
    ├── vulkan_official\                 ← 官方版：Vulkan（官方 ROCmFPX 主线）
    ├── vulkan_official_cm1\             ← 官方版：Vulkan + 实验性 CM1（cooperative-matrix，官方 ROCmFPX 主线）
    ├── roc_k2horizon\                   ← 官方版：HIP+Vulkan 双后端（K2-Horizon 专用，官方 ROCmFPX 主线）
    ├── rocm_w4a4\                       ← fork版：HIP/W4A4（charlie12345/ROCmFPX）
    ├── roc_rocmfp4\                     ← fork版：HIP/ROCmFP4（charlie12345/ROCmFPX）
    ├── rocm_ciru\                       ← fork版：HIP/Kairic Edge（ciru-rocmfpx，Qwen3.8-27B IU4 Kairic Edge）
    └── vulkan_qwen4exp\                 ← fork版：Vulkan/qwen4exp（LaurentZuijdwijk）
```

> 说明：`llamacpp-engines\` 内每个引擎目录 = **完整编译产物** + `.installed` 适配标记。
> 编译产物本身可直接用 `llama-server.exe --list-devices` / `--version` 验证；`.installed` 仅是为
> 让 NovaMax 能识别该引擎而附带的标记（NovaMax 仅通过 `.installed` 识别引擎，见下文「引擎发现原理」）。
> 原始编译产物与适配版仅差一个 `.installed` 文件，故不单独存放原件。

## 引擎一览

本包含 **8 个引擎**：4 个官方版（来自官方主线 `ROCmFPX/ROCmFPX`）+ 4 个 fork 版（来自 charlie12345/ROCmFPX、ciru-rocmfpx 与 LaurentZuijdwijk 的 fork）。

| 引擎目录 | 来源 | variant 归属 | llama-server.exe |
|---|---|---|---|
| `roc_official` | `ROCmFPX/ROCmFPX`（HIP，gfx1151） | rocm | ~10KB（启动壳，核心在 `llama-server-impl.dll`） |
| `vulkan_official` | `ROCmFPX/ROCmFPX`（Vulkan，gfx1151） | vulkan | ~10KB（启动壳，核心在 `llama-server-impl.dll`） |
| `vulkan_official_cm1` | `ROCmFPX/ROCmFPX`（Vulkan + 实验性 CM1，gfx1151） | vulkan | ~10KB（启动壳，核心在 `llama-server-impl.dll`） |
| `roc_k2horizon` | `ROCmFPX/ROCmFPX`（HIP+Vulkan 双后端，K2-Horizon 专用，gfx1151） | rocm | ~10KB（启动壳，核心在 `llama-server-impl.dll`） |
| `rocm_w4a4` | charlie12345/ROCmFPX @ main（HIP/W4A4） | rocm | ~4.2MB（单文件） |
| `roc_rocmfp4` | charlie12345/ROCmFPX @ main（HIP/ROCmFP4） | rocm | ~4.2MB（单文件） |
| `rocm_ciru` | ciru-rocmfpx @ `release/kairic-edge-qwen38-27b-v1.2`（HIP/Kairic Edge，IU4） | rocm | ~4.1MB（单文件） |
| `vulkan_qwen4exp` | LaurentZuijdwijk/llama.cpp @ `vulkan/qwen4exp-rocmfpx` | vulkan | ~10KB（启动壳，核心在 `llama-server-impl.dll`） |

> 引擎都已**实测可用**：从各自目录运行 `llama-server.exe --list-devices` 均能列出 GPU（`ROCm0` / `Vulkan0: AMD Radeon 8060S`），依赖齐全。

> **`vulkan_official_cm1`（新增，官方 Vulkan + CM1）**：与 `vulkan_official` **同源**（同一官方主线 `ROCmFPX/ROCmFPX`），仅额外开启了**实验性 Vulkan ROCmFP4 CM1（cooperative-matrix，F32 累加）** 构建选项（`-DGGML_VULKAN_ROCMFP4_COOPMAT=ON`，默认 OFF），让标准 `Vulkan0` 后端能把 `Q4_0_ROCMFP4` / `Q4_0_ROCMFP4_FAST` 的矩阵乘走 CM1 通道。它**仍然内置 Charlie ROCmFP4 插件**（`ggml-rocmfpx-vulkan.dll` + `rocmfpx-vulkan-plugin.dll`，暴露 `ROCmFPXVulkan0`）。
>
> 两个可选快路径**互斥、性能等价**：走 **`ROCmFPXVulkan0`**（Charlie 插件，只需设 `ROCMFPX_PLUGIN_PATH`），或走 **`Vulkan0` + 环境变量 `GGML_VK_ROCMFP4_COOPMAT=1`**（CM1，需确保插件未接管/`ROCMFPX_PLUGIN_PATH` 去掉）。二者不可叠加。构建与启用方法详见 [`docs\build\BUILD-VULKAN-ROCMFPX-WINDOWS.md`](docs/build/BUILD-VULKAN-ROCMFPX-WINDOWS.md) §5。

> **重要（2026-09 更新）**：官方 `roc_official` / `vulkan_official` 现已**吸收 fork 全部能力**——支持 per-head PLE（`ple_ngram`）+ **MTP 投机** + `--spec-draft-adaptive` + DFlash2，**速度与其他支线（`vulkan_qwen4exp` / `rocm_w4a4`）一致**。下方 fork 测速数据即对应官方引擎的表现，两者可互相替代；fork 引擎已被官方引擎取代（推荐改用 official）。

## 编译方法（主）

本仓库**核心**是把这些引擎在 **Windows + gfx1151** 下的编译方法落成文档，并附带已编译产物：

| 文档 | 目标引擎 | 说明 |
|---|---|---|
| [`docs\build\BUILD-ROCM-HIP-GENERIC.md`](docs/build/BUILD-ROCM-HIP-GENERIC.md) | HIP/ROCm 通用版 | Windows HIP/ROCm 通用编译（rocm-7.14 clang + MSVC 14.44，gfx1151） |
| [`docs\build\BUILD-CIRU-ROCMFPX-WINDOWS.md`](docs/build/BUILD-CIRU-ROCMFPX-WINDOWS.md) | ciru-rocmfpx Kairic Edge | ciru 分支（Qwen3.8-27B IU4 Kairic Edge / PromptForge），含 3 处 Windows 移植与踩坑 |
| [`docs\build\BUILD-VULKAN-ROCMFPX-WINDOWS.md`](docs/build/BUILD-VULKAN-ROCMFPX-WINDOWS.md) | `vulkan_official`（官方主线 Vulkan） | Windows Vulkan 编译（MSVC + VULKAN_SDK 1.4.357.0），含 `ROCmFPXVulkan0` 快速后端与使用 |

各引擎的**完整编译产物**见 `llamacpp-engines\<引擎>\`（含全部依赖 DLL，可直接运行；
`llama-server.exe --list-devices` 应列出 `ROCm0` / `Vulkan0`）。引擎性能实测见下文
「满电源频率速度测试」。

---

## （附带）NovaMax 适配

> 以下章节为**附带功能**：把上面编译好的引擎注册为 NovaMax 可用引擎。若你只是想本地跑引擎，
> 直接使用 `llamacpp-engines\<引擎>\llama-server.exe` 即可，无需看这部分。

### 1. 拷贝引擎到 NovaMax 引擎目录

variant 归属决定目标目录：

| 引擎 | 目标目录 |
|---|---|
| `roc_official` | `C:\Linglong\NovaStudio\NovaMax\external\llamacpp\rocm\` |
| `roc_k2horizon` | `C:\Linglong\NovaStudio\NovaMax\external\llamacpp\rocm\` |
| `vulkan_official` | `C:\Linglong\NovaStudio\NovaMax\external\llamacpp\vulkan\`（目录可能需新建） |
| `rocm_w4a4` | `C:\Linglong\NovaStudio\NovaMax\external\llamacpp\rocm\` |
| `roc_rocmfp4` | `C:\Linglong\NovaStudio\NovaMax\external\llamacpp\rocm\` |
| `rocm_ciru` | `C:\Linglong\NovaStudio\NovaMax\external\llamacpp\rocm\` |
| `vulkan_qwen4exp` | `C:\Linglong\NovaStudio\NovaMax\external\llamacpp\vulkan\`（目录可能需新建） |

```powershell
# 例：把 vulkan_official 接入 NovaMax
$dst = "C:\Linglong\NovaStudio\NovaMax\external\llamacpp\vulkan\vulkan_official"
New-Item -ItemType Directory -Path $dst -Force
Copy-Item -Path ".\vulkan_official\*" -Destination $dst -Recurse -Force
```

> 本包每个引擎已带 `.installed` 标记。NovaMax 靠 `.installed` + 目录名识别引擎（发现机制、`.installed` 字段说明、重载步骤、常见坑 → 见 `docs\novamax-llamacpp-skill\SKILL.md`）。


## 各引擎适配的模型（HF 镜像链接）

### 引擎适配矩阵（官方 2 引擎 + fork 1 引擎）

| 引擎目录 | 后端 | 适配模型 | HF 镜像仓库链接 | 满频实测 |
|---|---|---|---|---|
| `vulkan_official` | Vulkan | Qwen3.8-27B-ROCmFP4-FAST（DFlash2 投机，新版官方已支持） | https://hf-mirror.com/julianmb/Qwen-3.8-27B-ROCmFP4-FAST-GGUF | ✅ |
| `vulkan_official` | Vulkan | Qwen3.8-Flash-Next-ROCmFP4-FAST-imatrix（qwen4exp） | https://hf-mirror.com/agentionai/Qwen3.8-Flash-Next-ROCmFP4-FAST-imatrix-GGUF · **官方用 `v2/Qwen3.8-Flash-Next-ROCmFP4-FAST-imatrix-v2.gguf`（单张量）** | ✅（无投机，见下） |
| `roc_official` | HIP | Qwen3.8-27B-ROCmI4-MTP-GGUF | https://hf-mirror.com/cafonez/Qwen3.8-27B-ROCmI4-MTP-GGUF | ✅ |
| `roc_official` | HIP | Qwen3.8-Flash-Next-ROCmFP4-FAST-imatrix（qwen4exp） | https://hf-mirror.com/agentionai/Qwen3.8-Flash-Next-ROCmFP4-FAST-imatrix-GGUF · **官方用 `v2/Qwen3.8-Flash-Next-ROCmFP4-FAST-imatrix-v2.gguf`（单张量）** | ✅（无投机，decode 慢，见下） |
| `roc_official` | HIP | Ornith-1.5-35B-A3B-ROCmFP4-GGUF | https://hf-mirror.com/julianmb/Ornith-1.5-35B-A3B-ROCmFP4-GGUF | ✅ |
| `vulkan_qwen4exp` | Vulkan | Qwen3.8-27B-ROCmFP4-FAST（DFlash2） | https://hf-mirror.com/julianmb/Qwen-3.8-27B-ROCmFP4-FAST-GGUF | ✅ |
| `vulkan_qwen4exp` | Vulkan | Qwen3.8-Flash-Next-ROCmFP4-FAST-imatrix（MTP，per-head） | https://hf-mirror.com/agentionai/Qwen3.8-Flash-Next-ROCmFP4-FAST-imatrix-GGUF · **fork 用 `Qwen3.8-Flash-Next-ROCmFP4-FAST-v2-ple16.gguf`（per-head PLE）** | ✅（fork 支持 MTP） |
| `vulkan_qwen4exp` | Vulkan | sh0wie-Qwen3.8-Flash-Next-REAP-288-Q4_0_ROCMFP4_STRIX_LEAN（REAP-288 剪枝） | https://hf-mirror.com/rcmorano/sh0wie-Qwen3.8-Flash-Next-REAP-288-ROCMFPX · 文件 `sh0wie-Qwen3.8-Flash-Next-REAP-288-Q4_0_ROCMFP4_STRIX_LEAN.gguf` | ✅（无投机，见下测速） |
| `roc_k2horizon` | HIP+Vulkan | K2-Horizon-MoVA-36B-A4B（Q8_0_ROCMFPX_AGENT） | https://hf-mirror.com/kingjones777/K2-Horizon-MoVA-36B-A4B-ROCmFP4-GGUF | —（未测速） |

> **引擎更替说明**：官方项目 `ROCmFPX/ROCmFPX` 的 **`roc_official`（HIP 构建）已可替代此前两个个人/社区 fork 引擎** —— `rocm_w4a4`（charlie12345/ROCmFPX，HIP/W4A4）与 `roc_rocmfp4`（charlie12345/ROCmFPX，HIP/ROCmFP4），后两者已弃用并从 NovaMax 移除。

> **Flash-Next 文件选择（重要，与引擎绑定）**：`agentionai/Qwen3.8-Flash-Next-ROCmFP4-FAST-imatrix-GGUF` 仓库**同一权重、两种布局**，选哪份由引擎定：
> - **官方引擎（`vulkan_official`/`roc_official`）→ 用 `v2/Qwen3.8-Flash-Next-ROCmFP4-FAST-imatrix-v2.gguf`**（单张量 `per_layer_token_embd.weight`）——旧版（HEAD `2249677`）只能无投机；**新版（HEAD `c3b1c99`）已吸收 per-head PLE（`ple_ngram`）+ MTP/adaptive，可加载 `...-v2-ple16.gguf`（per-head）并跑 MTP**（见下方投机实测）。
> - **fork 引擎（`vulkan_qwen4exp`）→ 用 `Qwen3.8-Flash-Next-ROCmFP4-FAST-v2-ple16.gguf`**（per-head `ple_ngram_embd.0..15.weight`）——支持 **MTP 投机**。
> - MTP 外部草稿 `agentionai/Qwen3.8-Flash-Next-MTP-ROCmFP4-FAST-GGUF`（2.28GB）：**旧版官方**加载报 `blk.0.hc_attn_norm.weight not found`（仅 fork 可用）；**新版官方（HEAD `c3b1c99`）已按 nextn-only 草稿迭代，可正常 MTP 投机**（见下方实测）。

> **兼容性说明**：
> - 官方版 `ROCmFPX/ROCmFPX` **旧版（HEAD `2249677`）不支持 `--spec-draft-adaptive`**（报 `invalid argument`）、不支持 DFlash2 外部草稿；**新版（HEAD `c3b1c99`）已吸收 fork 的 `--spec-draft-adaptive` / DFlash2 / MTP / per-head 支持**。
> - `roc_official` 与 `vulkan_official` 是同一官方源码的 HIP/Vulkan 两套构建，量化支持范围一致（Q4_0_ROCMFP4/FAST、ROCMI4、Q2-Q8_ROCMFPX 全系列）。
> - **Flash-Next（qwen4exp）在官方引擎上的适配**：**新版**官方源码 `src/models/qwen4exp.cpp` 已**同时支持单张量** `per_layer_token_embd.weight` **与 per-head PLE**（`ple_ngram_embd.0..15.weight`，Laurent fork 布局，字段 `ple_ngram`），因此 `...-v2-ple16.gguf`（per-head）与 `v2/` 单张量均可加载；并**支持 MTP 外部草稿投机** `--spec-type draft-mtp`（新版 qwen4exp 已按 nextn-only 草稿迭代，草稿加载不再报 `blk.0.hc_attn_norm.weight not found`）。旧版（HEAD `2249677`）只读单张量、只能无投机（见上方旧版注）。
> - fork 版 `vulkan_qwen4exp` 支持 DFlash2 自适应投机；社区 fork 引擎 `rocm_w4a4`/`roc_rocmfp4` 已被官方 `roc_official` 替代并弃用。
>
> 镜像域名用 hf-mirror.com（国内直连）。lemonade 拉取时需 `HF_ENDPOINT=https://hf-mirror.com` + 显式 `--source huggingface`。

## 满电源频率速度测试（官方版 + fork 版）

**测试条件**（与之前一致）：
- **电源/频率**：AMD 电源档位拉满（**整机功耗约 132W**）
- **提示词长度**：~1 万 token 长上下文（散文 10123 / 代码 9950 token，固定填充文本构造）
- **输出长度**：`max_tokens=3000`，实际连续输出约 3000 token；decode 仅在累计 ≥1000 token 后取 [1000, 末尾] 稳定段（约 2000 token）统计
- **场景**：散文/代码 × 思考模式 off/low
- **加载方式**：每次场景独立冷启动模型（spawn → 测 → kill），杜绝 KVCache 复用污染

> **方法说明**：本版测试改为**通过 lemonade 加载**（不再直接 spawn），启用模型卡推荐的 **MTP 投机解码**（官方**新版**已支持 `--spec-draft-adaptive` 与 DFlash2；旧版只能无投机/固定 `--spec-draft-n-max`）。因此官方版 decode 高于旧版"无投机 baseline"。PREFILL 取服务端 `timings.prompt_per_second`；DECODE 用流式逐 token 时间戳取 **[1000, 末尾] 稳定段**。每场景 `lemonade load → 测 → unload` 冷启动。

### HIP 引擎（`roc_official`）—— Qwen3.8-27B-ROCmI4-MTP-GGUF-Q4_0（内嵌 MTP）

| 场景 | PREFILL | DECODE |
|---|---|---|
| 散文 think-off | 434.8 t/s | 20.28 tok/s |
| 代码 think-off | 435.5 t/s | 33.52 tok/s |
| 散文 think-low | 435.1 t/s | 19.43 tok/s |
| 代码 think-low | 428.5 t/s | 22.45 tok/s |

### HIP 引擎（`roc_official`）—— Ornith-1.5-35B-A3B-ROCmFP4-GGUF（MoE，内嵌 MTP）

| 场景 | PREFILL | DECODE |
|---|---|---|
| 散文 think-off | 1203.3 t/s | 59.59 tok/s * |
| 代码 think-off | 1218.4 t/s | 60.01 tok/s * |
| 散文 think-low | 1239.7 t/s | 62.43 tok/s |
| 代码 think-low | 1210.5 t/s | 101.82 tok/s |

### Vulkan 引擎（`vulkan_official`）—— Qwen3.8-27B-ROCmFP4-FAST（DFlash2 投机）

| 场景 | PREFILL | DECODE |
|---|---|---|
| 散文 think-off | 112.1 t/s | 21.67 tok/s |
| 代码 think-off | 111.9 t/s | 33.30 tok/s |
| 散文 think-low | 108.0 t/s | 23.36 tok/s |
| 代码 think-low | 108.2 t/s | 32.33 tok/s |

### Vulkan 引擎（`vulkan_official`）—— Qwen3.8-Flash-Next-ROCmFP4-FAST-imatrix-v2（qwen4exp，v2 单张量，**无投机**）

| 场景 | PREFILL | DECODE |
|---|---|---|
| 散文 think-off | 220.1 t/s | 21.53 tok/s |
| 代码 think-off | 199.1 t/s | 22.12 tok/s |
| 散文 think-low | 189.7 t/s | 21.06 tok/s |
| 代码 think-low | 197.8 t/s | 22.18 tok/s |

> 上表为官方引擎的 **v2 单张量、无投机 baseline**（旧版行为）；官方**新版已支持 Flash-Next 的 per-head + MTP 投机**（见下方 `vulkan_official` 投机实测）。参数：`-ngl 99 -fa on -ctk q8_0 -ctv q8_0 --no-mmap -b 2048 -ub 512`。

### HIP 引擎（`roc_official`）—— Qwen3.8-Flash-Next-ROCmFP4-FAST-imatrix-v2（qwen4exp，v2 单张量，**无投机**）

| 场景 | PREFILL | DECODE |
|---|---|---|
| 散文 think-off | 220.0 t/s | 5.09 tok/s |
| 代码 think-off | 208.1 t/s | 5.11 tok/s |
| 散文 think-low | 210.6 t/s | 5.10 tok/s |
| 代码 think-low | 206.2 t/s | 5.11 tok/s |

> **HIP decode 明显慢于 Vulkan**（~5.1 vs ~21-22 tok/s，约 4 倍）：v2 单张量的 51.2B 参数 PLE 表在 HIP 上落到 host 内存处理，吞吐瓶颈；Vulkan 自动溢出更优。故 Flash-Next 官方推荐用 **Vulkan 引擎**。（上表为 v2 无投机 baseline；新版 HIP 亦支持 MTP，见下方实测。）

> **备注**：
> - 带 **\*** 的 Ornith think-off 两场景因模型提前停止（仅 281/448 token，未达 1000 token）未取到 [1000,end] 段，用的是服务端全段 `predicted_per_second`；其余场景均为稳定的 [1000,end] 稳定段。
> - **Qwen3.8-Flash-Next-imatrix（qwen4exp）**：官方引擎**旧版只支持无投机**（MTP 外部草稿加载失败、ngram-cache 为负优化），上表为 v2 单张量 baseline；**新版已支持 per-head + MTP**，见下方 `vulkan_official` 投机实测。
> - **HIP prefill 显著优于 Vulkan**（~430 vs ~110 t/s，约 3.9 倍）；MoE 的 Ornith 最快（prefill ~1200 t/s，decode 59-102 tok/s）。
> - 本机电源档位拉满（整机功耗约 132W），条件与下方 fork 历史数据一致。

### Vulkan 引擎（`vulkan_official`）—— 含投机解码实测（DFlash2 / MTP，已对齐 fork）

> 说明：官方 `vulkan_official` **新版已支持 fork 的 DFlash2 / MTP 投机**（`--spec-draft-adaptive` / `--spec-type draft-mtp`），下方数据即官方引擎在**投机解码**下实测（lemonade 加载，方法与上方"无投机 baseline"不同，decode 更高）。经 Charlie ROCmFP4 快速后端（`ROCmFPXVulkan0`），速度与 fork `vulkan_qwen4exp` 一致。

**`vulkan_official`（官方，经 fork 对齐）—— Qwen3.8-27B-ROCmFP4-FAST（DFlash2）**

| 场景 | PREFILL | DECODE |
|---|---|---|
| 散文 think-off | 294.0 t/s | 17.4 tok/s |
| 代码 think-off | 295.2 t/s | 32.9 tok/s |
| 散文 think-low | 294.2 t/s | 24.2 tok/s |
| 代码 think-low | 292.5 t/s | 25.0 tok/s |

**`vulkan_official`（官方，经 fork 对齐）—— Qwen3.8-Flash-Next-ROCmFP4-FAST-imatrix（MTP）**

| 场景 | PREFILL | DECODE |
|---|---|---|
| 散文 think-off | 286.6 t/s | 17.4 tok/s |
| 代码 think-off | 291.0 t/s | 41.3 tok/s |
| 散文 think-low | 289.3 t/s | 18.3 tok/s |
| 代码 think-low | 289.9 t/s | 24.4 tok/s |

**`vulkan_official`（官方，经 fork 对齐）—— sh0wie-Qwen3.8-Flash-Next-REAP-288-Q4_0_ROCMFP4_STRIX_LEAN（REAP-288 剪枝）**

| 场景 | PREFILL | DECODE |
|---|---|---|
| 散文 think-off | 417.1 t/s | 26.68 tok/s * |
| 代码 think-off | 441.6 t/s | 30.02 tok/s |
| 散文 think-low | 454.7 t/s | 24.26 tok/s |
| 代码 think-low | 435.9 t/s | 28.96 tok/s |

> 参数：`-ngl 99 -fa on -ctk q8_0 -ctv q8_0 --no-mmap -b 2048 -ub 512`（含 `--no-mmap`）。REAP-288 为剪枝版，**prefill ~420-455 t/s（比官方全量 Flash-Next 的 ~200 t/s 快约 2 倍）**，decode ~24-30 tok/s。该数据经官方 `vulkan_official`（`ROCmFPXVulkan0` 快速后端）测得，效果与 Laurent fork 一致。\* = think-off 早停（未达 1000 token）用全段 pred_ps。

## 引用与致谢

本包中的两个 NovaMax 引擎（`roc_official` / `vulkan_official`）均基于以下开源 GitHub 仓库编译而来，特此声明并致谢：

### 编译依赖的 llama.cpp / ROCmFPx 仓库

| 仓库 | 分支 | 用途 |
|---|---|---|
| [ggml-org/llama.cpp](https://github.com/ggml-org/llama.cpp) | master | llama.cpp 上游官方仓库 |
| [ROCmFPX/ROCmFPX](https://github.com/ROCmFPX/ROCmFPX) | main | **官方 ROCmFPX 主线**（HEAD `c3b1c99`，compiled for `roc_official` + `vulkan_official`；已吸收 fork 的 per-head PLE/MTP/`--spec-draft-adaptive`/DFlash2）——Carlo Pasquale（charlie12345）创建，含 qwen4exp/qwen35 架构 + ROCmFPx 全量化（Q4_0_ROCMFP4/FAST、ROCMI4、Q2-Q8_ROCMFPX）+ HIP/Vulkan 双后端 |
| [charlie12345/ROCmFPX](https://github.com/charlie12345/ROCmFPX) | main | ROCmFPX 前身 fork（本次已弃用，被官方主线替代）；历史引擎来源 |
| [LaurentZuijdwijk/llama.cpp](https://github.com/LaurentZuijdwijk/llama.cpp) | `vulkan/qwen4exp-rocmfpx` | 历史 qwen4exp + Vulkan ROCmFPx fork（本次已弃用）；曾作为 vulkan_qwen4exp 引擎来源 |
| [ciru-ai/ROCmFPX](https://github.com/ciru-ai/ROCmFPX) | `ciru/upstream-rocmfpx-scale-eval` | ROCmFPx 量化内核（`rocmfp4_hip.cu`）来源调研与参考 |
| [daimonionnn/amd-rocmfpx-for-win](https://github.com/daimonionnn/amd-rocmfpx-for-win) | main | AMD Windows 平台 ROCmFPx 编译/基准参考（本机 Windows 编译参数借鉴于此） |

### 各引擎对应的构建来源

| 引擎目录 | 来源仓库/分支 | 本机编译参数 |
|---|---|---|
| `llamacpp-engines\roc_official` | ROCmFPX/ROCmFPX @ main（HIP，gfx1151） | `-DGGML_HIP=ON -DGGML_VULKAN=OFF -DGGML_HIP_FORCE_MMQ=ON -DCMAKE_HIP_ARCHITECTURES=gfx1151` + rocm-7.14 clang |
| `llamacpp-engines\vulkan_official` | ROCmFPX/ROCmFPX @ main（Vulkan，gfx1151） | `-DGGML_VULKAN=ON -DGGML_HIP=OFF -DGGML_CUDA=OFF` + MSVC 14.44 + VULKAN_SDK 1.4.357.0 |
| `llamacpp-engines\vulkan_official_cm1` | ROCmFPX/ROCmFPX @ main（Vulkan + 实验性 CM1，gfx1151） | `-DGGML_VULKAN=ON -DGGML_VULKAN_ROCMFP4_COOPMAT=ON -DROCMFPX_VULKAN_PLUGIN=ON -DBUILD_SHARED_LIBS=ON -DGGML_HIP=OFF -DGGML_CUDA=OFF` + MSVC + VULKAN_SDK 1.4.357.0（**Ninja** 生成器，VS 生成器在插件 ExternalProject 会报 FileTracker 错） |

### 致谢

- 感谢 **ggml-org** 维护 llama.cpp 及其社区，贡献了投机解码、prompt cache、Vulkan/ROCm 后端等核心能力。
- 感谢 **Carlo Pasquale (charlie12345)** 创建并维护 **ROCmFPX** 权重格式家族（ROCmFP3/4/6/8）及官方主线 `ROCmFPX/ROCmFPX`，使 AMD Radeon 8060S（Strix Halo）能跑通 Qwen3.8-27B-ROCmFP4-FAST 等专有量化模型。
- 感谢 **ciru-ai / LaurentZuijdwijk** 在 ROCmFPx 内核与 qwen4exp 架构上的早期贡献（已被官方主线吸收）。
- 感谢 **daimonionnn** 的 amd-rocmfpx-for-win，作为 AMD Windows 平台上的编译与性能评估参考。
- 适配目标平台 **NovaMax**（Linglong NovaStudio）与 **lemonade / LemonadeServer** 的引擎发现与模型加载机制为本次集成提供了基础。
- 本机工具链：Visual Studio 18 Build Tools (MSVC 14.44)、CMake 4.3.1、Ninja、Vulkan SDK `1.4.357.0`、ROCm 7.14 (HIP 工具链)、rocBLAS/hipBLAS，均用于上述引擎的本地编译。
