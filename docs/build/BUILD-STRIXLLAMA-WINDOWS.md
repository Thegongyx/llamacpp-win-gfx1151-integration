# BUILD-STRIXLLAMA-WINDOWS

在 **Windows + AMD gfx1151（Radeon 8060S / Strix Halo）** 上编译
[`rulith-dev/strixllama`](https://github.com/rulith-dev/strixllama) 补丁集的 llama.cpp，
产出本仓库的两个引擎：

- `roc_strixllama` —— 纯二进制，内核 gate 等环境变量由启动方提供；
- `roc_strixllama_env` —— 同上，但**默认环境变量在编译期烘焙进二进制**（外部 env 仍可覆盖）。

## 它是什么

strixllama 是 `pwilkin/llama.cpp`（pin **`f5daaa3cfa6358e5dd398911ec741813745a5440`**）之上的一套补丁集
（当前 39 个 `patches/apply_*.py`），面向 **Qwen3.8-Flash-Next（qwen4exp）**，核心包括：

- QSA 稀疏注意力：decode gather、block-key cache、small-batch causal mask 修复；
- IQ3_S / IQ4_XS 的 matrix-core（MMB/MMQ）内核与 expert gate/up 融合（`apply_moe_glu3`）；
- MTP 投机解码调优，按 shape 缓存的 HIP graph；
- server 侧：prompt cache 的磁盘层（`apply_prompt_cache_disk` / `apply_disk_tier_v2`）。

上游仓库也自带桌面 App / 管理器；本仓库只取它的**运行时**（`llama-server` + 依赖 DLL）。

## 前置

| 项 | 说明 |
|---|---|
| Python | 3.12+ |
| git / CMake / Ninja | `bootstrap.py` 会自行下载 ninja；CMake 需在 PATH |
| MSVC | 需存在 `vcvars64.bat`（ROCm 的 clang 仍链接 MSVC 运行时、用 Windows SDK 头） |
| ROCm | **不要**装系统 ROCm；`--toolchain` 会把 TheRock ROCm **10.1 nightly** 作为 Python wheel 装进 `toolchain/rocm-venv` |
| 磁盘 | 工具链 ~8GB、构建 ~10GB |

> 本机用的是 **VS 18 BuildTools**（`C:\Program Files (x86)\Microsoft Visual Studio\18\BuildTools`），
> 而 `bootstrap.py` 的 `VCVARS` 列表默认只找 VS 2022（且在 `C:\Program Files\...`）。
> 编译前需把 VS 18 的 `vcvars64.bat` 路径加进 `bootstrap/bootstrap.py` 的 `VCVARS`（或在 PATH 中让 CMake 可用）。

## 步骤

```powershell
# 0) 取源码（国内用镜像前缀）
git clone https://gh.xmly.dev/https://github.com/rulith-dev/strixllama.git
cd strixllama

# 1) 工具链：TheRock ROCm 10.1 nightly（wheel）+ ninja，装进 toolchain/rocm-venv
python bootstrap\bootstrap.py --toolchain
#   pin 为 10.1.0a20260910，来自滚动窗口约 27 天的 nightly 索引，过期后用：
#   python bootstrap\bootstrap.py --toolchain --rocm-version <index 现有版本>

# 2) 拉上游 + 打补丁（checkout pwilkin 的 pin，再按 PATCH_ORDER 套用补丁）
python bootstrap\bootstrap.py --fetch --patch
python bootstrap\bootstrap.py --verify      # 期望 “N of N patched files match the recipe”

# 3) 编译（HIP/ROCm，gfx1151）
#    需 CMake 在 PATH；bootstrap 会自行合并 vcvars64.bat 的环境
python bootstrap\bootstrap.py --build       # 产物在 bin\hip-rocm101\
```

### 打成自包含引擎目录

```powershell
# 收集 llama-server 及其传递依赖的 ROCm DLL、gfx1151 kernel 库、VC/OpenMP 运行时
python tools\make_runtime_bundle.py --out dist\runtime --python-zip <python-3.12-embed.zip>
# 结果在 dist\runtime\bin\hip-rocm101\（约 273 文件 / 311MB）
```

拷到一个以引擎命名的目录，并放 `.installed` 标记（**必须无 BOM**，否则 NovaMax 的 `JSON.parse` 会抛错）：

```json
{ "runtime": null, "installed_at": "<ISO8601>", "version": "roc_strixllama_env",
  "archiveSha256": "local-build-pwilkin-f5daaa3+strixllama-<rev>", "variant": "rocm", "engine": "llamacpp" }
```

> `version` 必须与目录名一致；`variant` 为 `rocm`（HIP 构建）。

## 编译期默认环境变量（仅 `roc_strixllama_env`）

补丁集把性能开关做成 `getenv()` 读取、默认**关**，由启动器显式打开。为让"裸跑 `llama-server.exe`"
也达实测配置，`roc_strixllama_env` 在 `src/llama.cpp` 里加了两处：

1. `src/strixllama-defaults.h`：默认值表 + `apply_default_env()`；
2. `llama_backend_init()` 起始调用 `strixllama::apply_default_env();`
   （所有 gate 都是函数内惰性 `getenv`，此处注入即可被读到）。

规则：**只在该变量未设置时写入**，所以 `lemonade`/NovaMax 的 per-model env 仍可覆盖。

烘焙的默认值：

```
LLAMA_MMB=1 LLAMA_MMB_MIN_T=512 LLAMA_MMB_BF16W=1 LLAMA_MMB_GLU=1 LLAMA_MMB_TALL=2
LLAMA_MMB_CACHE=4 LLAMA_MMB_F32SPLIT=2 LLAMA_MMB_HC16=0 LLAMA_MMB_SHADOW=2 LLAMA_MMB_DOWN16=1
LLAMA_HC_CN_SHAPE=1 LLAMA_HC_GATEMIX=1 LLAMA_HC_MIX_FUSE=1 LLAMA_HC_BLK16=1
LLAMA_HC_RES16=1 LLAMA_HC_PACK_DI=1
LLAMA_NORM_GATED=1 LLAMA_NORM_ROWS=1 LLAMA_IDX_RELU_SUM=1 LLAMA_PLE_CONV=1 LLAMA_GDN_CONV=1
STRIX_SPEC_DRAFT_UBATCH=2048 LLAMA_MTP_QSA=1
LLAMA_QSA_SPARSE=1 LLAMA_QSA_BLOCK_SELECTION=1 LLAMA_QSA_COMPACT_METADATA=1 LLAMA_QSA_DENSE_SHORTCUT=1
LLAMA_QSA_DIRECT_INDICES=1 LLAMA_QSA_FA_V3=1 LLAMA_QSA_FUSE_EXPAND=1 LLAMA_QSA_NO_DENSE_MASK=1
LLAMA_QSA_PACK_KEYS=1 LLAMA_QSA_PACK_VALUES=1 LLAMA_QSA_SCORE_BOUNDS=1 LLAMA_QSA_WHOLE_ATTN=1
LLAMA_QSA_DECODE_GATHER=1 LLAMA_QSA_BLOCK_KEY_CACHE=1 LLAMA_QSA_QUERY_STRIP=512
STRIX_PROMPT_CACHE_MIB=16384 STRIX_PROMPT_CACHE_BLOCK=4096
STRIX_PROMPT_CACHE_DIR=<引擎exe目录>\prompt-cache    (运行时派生, 可移植)
```

注意：`LLAMA_MMB_HC16` **必须为 0**（为 1 时在 Windows/TheRock/Clang 下输出会被 `/` 淹没）。

## 推荐的启动参数（作者实测配置）

```
-ngl 999 -c 262144 -b 8192 -ub 8192 -t 16 --poll 0 --fit off -np 1
-fa on -ctk f16 -ctv f16 --jinja
--cache-prompt --cache-ram 1024 --no-cache-idle-slots
--chat-template-kwargs {"enable_thinking":false}
--mmproj <mmproj-F16.gguf>
--load-mode none --lazy-mode on-direct
-m<draft> --spec-draft-n-max 3 --spec-draft-p-min 0.3 --spec-type draft-mtp
```

说明：
- `--cache-ram 1024` / `--no-cache-idle-slots` / `STRIX_PROMPT_CACHE_*` 只影响 **prompt cache 与多槽切换**
  （跨请求复用、省内存），**不改变 prefill/decode 的 t/s**。
- `-md`（草稿）与 `--mmproj` 若经 lemonade/NovaMax 加载，由模型的 checkpoint 自动注入，不要在自定义参数里手写。

## 验证

1. **注入是否生效**：写一个调用 `llama_backend_init()` 再 `getenv()` 的小程序，**用与引擎相同的 clang/CRT 编译**，
   放在引擎目录里运行。MSVC/ucrtbase 编的程序看不到引擎 DLL 的 CRT 环境块，会误判为"未注入"。
2. `llama-server.exe --list-devices` → 应列出 `ROCm0: AMD Radeon 8060S`。
3. 功能：`llama-server` 起服务后发一次 `chat/completions`，`timings` 里 `draft_n/draft_n_accepted` > 0 即 MTP 在工作。

## 发布物

GitHub Release 上按 `engine-<目录名>_gfx1151_win.zip` 命名（zip 顶层即引擎目录名，解压到引擎根目录即可）：

- `engine-roc_strixllama_gfx1151_win.zip`
- `engine-roc_strixllama_env_gfx1151_win.zip`

## 常见坑

- `.installed` 写成带 BOM 的 UTF-8（Windows PowerShell 5.1 的 `Set-Content -Encoding UTF8` 会加 BOM）
  → NovaMax 整个引擎列表请求失败。用 `UTF8Encoding($false)` 或 PowerShell 7 写。
- ROCm nightly 索引是滚动窗口（约 27 天），pin 会过期；换版本后需重新跑验证/数值检查。
- pwilkin 的 `master` 与 pin 点不同（更新但更慢、且在本机 Windows 上编译不过），**必须用 pin `f5daaa3`**。
- `--fetch --patch` 会重置 `src/llama.cpp`，自己加的改动（如默认 env 注入）需在 patch 之后重新应用。
