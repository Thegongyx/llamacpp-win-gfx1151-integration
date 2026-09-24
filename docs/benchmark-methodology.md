# lemonade 官方引擎测速 —— 测试标准 / 方法 / 提示词

> 本文件记录对**新编译官方引擎**（`vulkan_official` / `roc_official`，源自 `ROCmFPX/ROCmFPX` main HEAD `2249677`）做长上下文测速的**统一标准、测量方法与所用提示词**。测试均通过 **lemonade**（HTTP 客户端，端口 `13305`）驱动，结果见 `../README.md` 与 `../../scripts/bench_official_results.json`。

---

## 一、测试标准（与历史一致的固定条件）

| 项 | 约定 |
|---|---|
| **电源/频率** | AMD 电源档位拉满，**整机功耗约 132W** |
| **提示词长度** | **~1 万 token** 长上下文（散文目标 **10123**，代码目标 **9950**；用固定填充文本拼接构造，见第三节） |
| **输出长度** | `max_tokens = 3000`，实际连续输出约 3000 token |
| **DECODE 统计段** | 仅在累计输出 **≥ 1000 token** 后取 **[1000, 末尾] 稳定段**（约 2000 token）统计，剔除首段过渡不稳定 |
| **场景** | **散文 / 代码** × **思考模式 off / low**（共 4 个场景） |
| **加载方式** | **每个场景独立冷启动模型**：`lemonade load → 测 → lemonade unload`，杜绝 KVCache 复用污染；每场景只发一次请求 |
| **驱动** | 全部通过 **lemonade** 加载（`--llamacpp vulkan/rocm` + `--no-merge-args --llamacpp-args <lean 参数>`），lemonade 按模型 label 自动注入 `-m` / `-md` / `--spec-type` / `--jinja` / 端口 / `--ctx-size` |

---

## 二、测量方法

### 2.1 引擎与参数（`--llamacpp-args`，仅含可通过 lemonade 保留参数检查的部分）

> lemonade **保留参数**（不可自定义）：`--ctx-size, --device, --jinja, --mmproj, --model, --model-draft, --port, -c, -dev, -m, -md, --mmproj…`。这些由 lemonade 自动注入；`--llamacpp-args` 只传剩余项。

| 引擎 | 模型 | 后端/设备 | `--llamacpp-args`（投机说明） |
|---|---|---|---|
| `vulkan_official` | Qwen3.8-27B-ROCmFP4-FAST | vulkan / Vulkan0 | `-ngl 99 -fa on --no-mmap -ctk q8_0 -ctv q8_0 -b 2048 -ub 512`（label=dflash+mtp，lemonade 自动注入 DFlash2 投机，`draft_n>0`） |
| `roc_official` | Qwen3.8-27B-ROCmI4-MTP | rocm / ROCm0 | `-ngl 999 -b 2048 -ub 512 -t 16 -tb 32 -fa on -ctk f16 -ctv f16 --spec-type draft-mtp --spec-draft-device ROCm0 --spec-draft-ngl all --spec-draft-type-k f16 --spec-draft-type-v f16 --spec-draft-n-max 6 --spec-draft-n-min 0 --spec-draft-p-min 0.6 --spec-draft-p-split 0.10 --no-spec-draft-backend-sampling --spec-draft-poll 1 --spec-draft-poll-batch 1`（内嵌 MTP） |
| `roc_official` | Ornith-1.5-35B-A3B-ROCmFP4 | rocm / ROCm0 | `-ngl 999 -fa on --no-mmap -b 2048 -ub 512 --spec-type draft-mtp --spec-draft-n-max 4 --spec-draft-p-min 0.6`（内嵌 MTP，MoE） |

> **关键约定**：官方引擎**不支持** `--spec-draft-adaptive`（报 `invalid argument`）与 DFlash2 外部草稿，因此统一用**固定 `--spec-draft-n-max`**；且若模型已注册外部 draft（mtp/dflash label + draft checkpoint），**不要重复传** `--spec-type`，由 lemonade 自动注入，避免冲突。
> **注意**：`Qwen3.8-Flash-Next-ROCmFP4-FAST-imatrix`（qwen4exp）在官方引擎上**无法加载**（per-head PLE 布局与官方单张量读取不匹配；v2 单张量需 `--ngram-on-disk` 而官方无此参数），故不在官方表内——详见 README「兼容性说明」。

### 2.2 思考模式控制

| 场景 | 实现方式 |
|---|---|
| **think-off** | 加载时 `--llamacpp-args` 追加 `--reasoning off`（模型不输出 reasoning_content；实测 0 个 reasoning chunk） |
| **think-low** | 加载时不加 `--reasoning off`（保持开启），请求体传 `reasoning_effort: "low"`（Qwen3.8 模板读取该键，`"none"/"off"` 无效） |

### 2.3 指标计算（流式，逐 token 时间戳）

发送 `POST /v1/chat/completions`，`stream: true`，`max_tokens=3000`，`temperature=0`；用 HTTP 流式读取每个 `data:` chunk，记录每个含 `content`/`reasoning_content` 的 chunk 的到达时间。

- **PREFILL** = 服务端 `timings.prompt_per_second`（= `prompt_n / prompt_ms`，最准，不含网络）→ 取最后一个 chunk 的 `timings` 字段。
- **DECODE** = 客户端流式时间戳在 **[1000, 末尾] 稳定段**上计算：
  `decode = (predicted_n - 1000) / (t_last_token - t_token_at_cum_1000)`
  其中 `predicted_n` 取服务端 `timings.predicted_n`；若输出不足 ~1000 token（早停），则回退用 `timings.predicted_per_second`（全段），并在结果里标注 `seg=None`。
- 同时记录 `draft_n` / `draft_n_accepted`（判定投机是否生效）。

### 2.4 结果字段

每次场景记录：`engine / model / kind / think / prompt_n / predicted_n / prefill(t/s) / decode([1000,end] tok/s) / n_seg / predicted_ps / draft_n / draft_accepted / args`。

---

## 三、所用提示词（固定填充文本构造）

> 提示词 = **续写指令（引导长输出，避免模型早停）** + **固定填充文本 × 重复次数**。重复次数按目标 token 数调校（散文 ~10123 / 代码 ~9950）；实际每类 prompt 的 `prompt_n` 随模型 tokenizer 略有不同（实测散文 ~10079–10153，代码 ~10125–10165），以每次运行读到的 `prompt_n` 为准。

### 3.1 散文（prose）

```
{PROSE_INSTRUCTION}{PROSE_PARA × 57}
```

- **续写指令** `PROSE_INSTRUCTION`：
  > 请根据以下材料，撰写一篇非常长且详尽的学术技术文章，逐段深入展开每个论点，内容充实，写满篇幅，不要提前结束。

- **填充段落** `PROSE_PARA`（重复 57 次）：
  > 机器学习和人工智能技术正在快速发展，大型语言模型在自然语言处理、代码生成和知识问答等领域展现出强大的能力。这些模型基于深度神经网络和海量训练数据，通过自注意力机制理解上下文语义。在实际应用中，推理速度、内存占用和量化精度之间的平衡是部署的关键考量因素。ROCmFPX 是针对 AMD GPU 优化的量化格式，利用原生 WMMA 指令加速矩阵运算。投机解码通过草稿模型预测多个候选 token，再由目标模型验证，从而在保持输出质量的同时显著提升生成速度。对于庞大的长上下文输入，预填充效率成为决定首字延迟的核心因素。大型语言模型在多轮对话中需要通过 KV 缓存复用前缀来降低重复计算的开销。AMD Strix Halo 平台借助统一内存架构实现了 GPU 与 CPU 之间的高效数据共享，使得高带宽显存成为可能。

> ⚠️ 针对 Ornith（简洁 MoE，续写指令容易被忽略）单独使用**更强**的续写指令：
> 请把以下材料作为文章开头，极其详尽地续写并扩展成一篇能写多久就写多久的超长学术技术文章。要求：持续扩展每一个观点，充分举例、分层论证、反复换角度重述，绝不提前总结或收尾，写到篇幅尽可能长为止。

### 3.2 代码（code）

```
{CODE_INSTRUCTION}{CODE_UNIT × 31}
```

- **续写指令** `CODE_INSTRUCTION`：
  > 请根据以下编码规范与示例，编写一个功能完整、结构清晰的大型程序，详细实现每一个模块，并补充大量必要的注释，代码量越大越好，不要提前结束。

- **填充代码单元** `CODE_UNIT`（重复 31 次；每次前后带 ``` 代码围栏）：

````
```python
def compute_metrics(predictions, references, metrics=["precision", "recall", "f1"]):
    """Compute standard classification metrics.

    Args:
        predictions (List[int]): Model output labels.
        references (List[int]): Ground-truth labels.
        metrics (List[str]): The metrics to compute, e.g. precision/recall/f1.
    Returns:
        Dict[str, float]: Metric name to score mapping.
    """
    results = {}
    for m in metrics:
        if m == "precision":
            tp = sum(1 for p, r in zip(predictions, references) if p == 1 and r == 1)
            pp = sum(1 for p in predictions if p == 1)
            results[m] = tp / pp if pp else 0.0
        elif m == "recall":
            tp = sum(1 for p, r in zip(predictions, references) if p == 1 and r == 1)
            rn = sum(1 for r in references if r == 1)
            results[m] = tp / rn if rn else 0.0
        elif m == "f1":
            precision = compute_metrics(predictions, references, ["precision"])["precision"]
            recall = compute_metrics(predictions, references, ["recall"])["recall"]
            results[m] = 2 * precision * recall / (precision + recall) if (precision + recall) else 0.0
    return results
```
````

---

## 四、执行流程（每次场景）

1. `lemonade load <model> --llamacpp <backend> --ctx-size 262144 --no-merge-args --llamacpp-args "<lean args>"`（think-off 时追加 `--reasoning off`）。
2. 轮询 `lemonade status` 直到该模型出现 `ready`。
3. 发送一次流式 `chat/completions`（`max_tokens=3000`，`temperature=0`；think-low 加 `reasoning_effort=low`），逐 chunk 记时间戳。
4. 从最后一个 chunk 读 `timings`，客户端计算 PREFILL / DECODE[1000,end]。
5. `lemonade unload <model>`，等待退出（下一次场景重新冷加载）。

---

## 五、备注 / 已知边界

- **投机解码开启**：本表为模型卡推荐的 MTP/DFlash2 投机下测速，decode 高于旧版"无投机 baseline"。
- **HIP prefill 显著快于 Vulkan**（~430 vs ~110 t/s）；MoE 的 Ornith 最快（prefill ~1200 t/s，decode 59–102 tok/s）。
- **早停**：Ornith think-off 两个场景输出不足 1000 token（281 / 448），用全段 `predicted_per_second`，标注 `seg=None`。
- **Flash-Next（qwen4exp）**：官方引擎无法加载（per-head PLE / `--ngram-on-disk` 均不支持），属 fork 专属；已在 README 标注。


---

## 六、strixllama 引擎（`roc_strixllama` / `roc_strixllama_env`）

同一套方法也可用于 strixllama 引擎，但有两点不同：

- **环境变量**：该引擎的性能特性由 `getenv()` 的 gate 控制、默认关（MMB / HC / QSA 等）。
  `roc_strixllama_env` 已把默认值**烘焙进二进制**，直接用 `--llamacpp-args` 跑即可；
  `roc_strixllama` 需要启动方提供这些 env。清单、原因与验证方法见
  [`BUILD-STRIXLLAMA-WINDOWS.md`](build/BUILD-STRIXLLAMA-WINDOWS.md)。
- **推荐参数**（作者实测配置）：
  `-ngl 999 -b 8192 -ub 8192 -t 16 --poll 0 --fit off -np 1 -fa on -ctk f16 -ctv f16 --jinja --cache-prompt --cache-ram 1024 --no-cache-idle-slots --ctx-checkpoints 8 --checkpoint-min-step 32768 --load-mode none --lazy-mode on-direct --spec-draft-n-max 3 --spec-draft-p-min 0.3`
  （`-md` / `--mmproj` 由 lemonade 按模型 checkpoint 自动注入，勿手写。）
- **注意**：`--cache-ram` / `--no-cache-idle-slots` / `STRIX_PROMPT_CACHE_*` 只影响 prompt cache 与多槽切换，
  **不改变 prefill/decode 的 t/s**；测速率时用 `cache_prompt:false` 冷启动，避免缓存复用污染。
  带 mmproj 的（视觉）模型**不要**开磁盘层（`STRIX_PROMPT_CACHE_DIR`），否则 `server_tokens::get_tokens()`
  的 `GGML_ASSERT(!has_mtmd)` 会中止服务器。
