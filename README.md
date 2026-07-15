# IP Detection — 知识产权侵权自动检测系统

面向电商场景的 IP 侵权自动检测**算法主仓库**，包含各模块的模型实现、服务封装示例，以及完整的在线检测流水线示例代码。

> **注意**：本仓库为算法研究与验证代码，真实线上检测链路由工程团队另行用 Java 实现和部署，本仓库的流水线代码供算法开发、离线验证和工程对接参考。

**核心输出**：给定一个 SKC 及其图片 URL，判断是否侵权（`is_infringement`）、侵犯了哪个权利人（`dominant_owner`）、风险等级（`risk_level`: high / mid / low）。

---

## 系统架构

### 检测流水线

```
图片输入
   │
   ▼
┌────────────────────────────────────────────────────────────────────────┐
│  Step 1: 召回（recall.py）                                              │
│  · Type 101: 案例图搜（向量检索）                                        │
│  · Type 102: 元素图搜（向量检索）                                        │
│  · Type 103: 文本匹配                                                   │
│  召回通道：OCR+文本 / Logo / 图搜元素库 / 商品盗图 / 人脸·人物 / 多模态案例库 │
└────────────────┬───────────────────────────────────────────────────────┘
                 │
                 ▼
┌────────────────────────────────────────────────────────────────────────┐
│  Step 2: RRF 融合（recall.py · fuse）                                   │
│  倒数排名融合，产出 rrf_score / hit_count / hit_keys                     │
└────────────────┬───────────────────────────────────────────────────────┘
                 │
                 ▼
┌────────────────────────────────────────────────────────────────────────┐
│  Step 3: 粗排（recall.py · coarse_rank）                                │
│  按命中数分 4 个置信层（Tier 1–4），硬预算上限 50                          │
└────────────────┬───────────────────────────────────────────────────────┘
                 │
                 ▼
┌────────────────────────────────────────────────────────────────────────┐
│  Step 4: 精排（rank.py · rerank）                                       │
│  Qwen3-VL 多模态精排，支持 aliyun / vllm API                            │
│  图片压缩转 base64 后送 LLM 判断，输出 is_infringing + 置信分             │
└────────────────┬───────────────────────────────────────────────────────┘
                 │
                 ▼
┌────────────────────────────────────────────────────────────────────────┐
│  Step 5: 跨图合并（rank.py · merge）                                    │
│  合并同一 SKC 多张图片的精排结果                                          │
└────────────────┬───────────────────────────────────────────────────────┘
                 │
                 ▼
┌────────────────────────────────────────────────────────────────────────┐
│  Step 6: 后处理（postprocess.py）7 阶段过滤                              │
│  A: rerank_score 阈值 + top_k                                           │
│  A2: 优先级元素组过滤                                                    │
│  B: Owner 投票（majority / score_sum / top1 / none）                    │
│  C: Element score 阈值（per-owner 自定义）                               │
│  C2: 元素 ID 阈值过滤                                                    │
│  D: 权利人名单过滤（可选）                                                │
│  E: 风险等级划分（high / mid / low）                                     │
└────────────────────────────────────────────────────────────────────────┘
```

### 检测链路

| 链路 | 目录 | 场景 |
|------|------|------|
| 公文检测 | `src/pipeline/detection/document_scan/` | 近实时，单SKC逐条检测 |
| 存量全量 | `src/pipeline/detection/stock_scan_full/` | 批量，全量元素扫描 |
| 存量快速回扫 | `src/pipeline/detection/stock_scan_fast/` | 批量，指定element_id过滤（T+1增量或手动指定） |

### 代码层划分

| 目录 | 职责 |
|------|------|
| `src/pipeline/detection/{scenario}/` | **主要工作代码**：各检测链路的层级编排与流水线串联 |
| `src/models/` | 纯计算逻辑（RRF、coarse_rank、postprocess 等，无外部调用，可单测） |
| `src/services/` | 具体子服务封装（图搜检索、OCR 识别等外部 API 调用） |
| `src/pipeline/data/` | 数据拉取脚本 |
| `src/pipeline/evaluation/` | 评估脚本 |

---

## 目录结构

```
ip-detection/
├── src/
│   ├── models/                       # 纯计算逻辑（无外部调用，可单测）
│   │   ├── retrieval/                # RRF 融合
│   │   ├── ranking/                  # 粗排逻辑
│   │   └── strategy/                 # 后处理逻辑
│   ├── services/                     # 具体子服务封装
│   │   ├── retrieval/                # 多通道召回服务
│   │   ├── ranking/                  # 精排服务
│   │   └── strategy/                 # 后处理服务
│   └── pipeline/
│       ├── detection/
│       │   ├── document_scan/              # 公文检测链路
│       │   │   ├── config.py         # PipelineConfig dataclass + YAML加载
│       │   │   ├── recall.py         # 召回 / RRF 融合 / 粗排
│       │   │   ├── rank.py           # Qwen3-VL 精排 / 跨图合并
│       │   │   ├── strategy.py       # 7 阶段后处理
│       │   │   ├── pipeline.py       # 流水线串联逻辑
│       │   │   ├── utils.py          # CSV 写入 / 终端展示
│       │   │   └── run.py            # CLI 批量推理入口
│       │   ├── stock_scan_full/       # 存量全量检测链路（开发中）
│       │   └── stock_scan_fast/       # 存量快速回扫链路
│       ├── data/                     # 数据拉取脚本
│       └── evaluation/               # 评估脚本
├── configs/
│   ├── detection/
│   │   ├── document_scan/                  # 公文链路配置
│   │   │   ├── pipeline_config.yaml
│   │   │   ├── element_config.json
│   │   │   └── element_id_2_decision_point.json
│   │   ├── stock_scan_full/           # 存量全量配置（开发中）
│   │   └── stock_scan_fast/           # 存量快速回扫配置
│   ├── data/                         # 数据拉取配置
│   └── service/                      # 子服务配置
├── scripts/
│   ├── detection/
│   │   ├── document_scan/                  # 公文链路脚本
│   │   ├── stock_scan_full/           # 存量全量脚本（开发中）
│   │   └── stock_scan_fast/           # 存量快速回扫脚本
│   ├── data/                         # 数据拉取脚本
│   └── eval/                         # 评测脚本
├── tests/                            # 单元测试（对应 src/models/）
├── docs/                             # 文档
└── results/                          # 运行输出（gitignore）
```

---

## Pipeline 使用指南

### 1. 公文检测链路（Document Scan）

**入口**：`src/pipeline/detection/document_scan/run.py`

**输入**：CSV 文件，必须包含 `skc` 和 `image_urls` 列（`image_urls` 为逗号分隔的图片 URL）

支持三种运行模式：

```bash
# 模式一：完整流水线（从召回开始）
python -m src.pipeline.detection.document_scan.run \
    --input  data/input.csv \
    --output results/output.csv \
    --config configs/detection/document_scan/pipeline_config.yaml

# 模式二：从粗排结果开始（step_coarse 列）
python -m src.pipeline.detection.document_scan.run \
    --input  data/input.csv \
    --output results/output.csv \
    --from-coarse

# 模式三：从工程格式召回结果开始（recall_result 列）
python -m src.pipeline.detection.document_scan.run \
    --input  data/input.csv \
    --output results/output.csv \
    --from-recall
```

**Shell 脚本**（封装了 Python 路径和默认配置）：

```bash
bash scripts/detection/document_scan/run_online_detection.sh --input data/input.csv
```

**Python 环境**：`infer-env`（或 `qwen-env`，当使用本地 vllm 时）

---

### 2. 数据拉取 Pipeline（Data）

所有数据拉取脚本均在 `src/pipeline/data/`，通过对应的 shell 脚本调用，配置文件在 `configs/data/`。

**Python 环境**：`data-env`

#### 主要脚本

| Shell 脚本 | 数据来源 | 输出 |
|-----------|---------|------|
| `scripts/data/pull_high_exposure_data.sh` | 高曝光审核任务 + SKC 图片 | `data/high_exposure/*.parquet` |
| `scripts/data/pull_algo_audit_result.sh` | 算法审核结果（侵权检测日志） | `data/algo_audit_result/*.parquet` |
| `scripts/data/pull_case_data.sh` | 案例库侵权案例 | `data/case_data/*.parquet` |
| `scripts/data/pull_skc_info.sh` | SKC 商品信息（图片/品类/品牌） | `data/skc_info/*.parquet` |
| `scripts/data/pull_skc_review_tp.sh` | 人工审核 TP 标注数据 | `data/skc_review_tp/*.parquet` |
| `scripts/data/pull_doc_audit_by_docid.sh` | 公文审核结果（按 docid 批量） | `data/doc_audit/*.parquet` |

#### 示例

```bash
# 拉取高曝光审核数据（指定日期范围）
bash scripts/data/pull_high_exposure_data.sh \
    --date-start 2026-05-01 --date-end 2026-05-07

# 拉取算法审核结果
bash scripts/data/pull_algo_audit_result.sh \
    --date-start 2026-05-01 --date-end 2026-05-07

# 拉取指定 SKC 列表的商品信息
bash scripts/data/pull_skc_info.sh \
    --skcs-file data/my_skcs.csv
```

> 更多脚本参数说明见 [`docs/data/script-guide.md`](docs/data/script-guide.md)

---

### 3. 评测 Pipeline（Evaluation）

支持三种评测场景，每个场景均为「拉数据 → 预处理 → 计算指标」三步流程。

**Python 环境**：`data-env`（拉数据）、`infer-env`（预处理 + 指标）

#### 场景一：在线高曝光评测

评测线上系统对高曝光商品的识别准召率。

```bash
# 拉取近 7 天数据并评测
bash scripts/eval/eval_online_high_exposure.sh

# 指定日期范围
bash scripts/eval/eval_online_high_exposure.sh \
    --date-start 2026-04-01 --date-end 2026-04-30

# 跳过拉取，直接用已有数据
bash scripts/eval/eval_online_high_exposure.sh \
    --input data/high_exposure/my_data.parquet
```

#### 场景二：离线 Batch 推理评测

评测离线批量推理结果（对比 GT 标注）。

```bash
bash scripts/eval/eval_offline_e2e_inference.sh \
    --batch-ids B-20260401-001,B-20260401-002
```

#### 场景三：Pipeline 各阶段评测

评测流水线中间结果，定位各模块（召回/排序/策略）的精召损失。

```bash
bash scripts/eval/eval_pipeline_stages.sh \
    --run-dir results/detection/<run_id>/ \
    --gt data/ground_truth.parquet
```

**输出**（`results/evaluation/`）：

- `metrics.json` — 整体 precision / recall / F1
- `per_owner_metrics.csv` — 按权利人粒度的准召
- `badcases/fp_cases.csv` — FP badcase（含图片 URL）
- `badcases/fn_cases.csv` — FN badcase

> 评测流程详见 [`docs/evaluation/online_detection_eval.md`](docs/evaluation/online_detection_eval.md)

---

## Claude Code Skills（AI 辅助工作流）

项目封装了两个 Claude Code Skill，可在对话中直接触发，AI 会交互式引导完成全流程。

### `/run-detection` — 运行识别链路

**触发词**：`跑识别`、`run detection`、`批量推理`、`run inference`

**功能**：
- 自动验证输入 CSV 格式（检查 `skc` 和 `image_urls` 列）
- 选择配置文件
- 后台执行推理，监控进度
- 推理完成后输出结果摘要

**示例对话**：
```
你：帮我跑一下 data/my_input.csv 的识别
Claude：（自动验证格式、选配置、后台启动推理，实时汇报进度）
```

---

### `/run-detection-eval` — 运行评测

**触发词**：`评测`、`跑一下指标`、`precision/recall`、`检出率`、`召回率`、`compute metrics`、`跑评测`

**功能**：
- 交互式确认评测场景（在线高曝光 / 离线 batch / pipeline stages）
- 自动收集所需参数（run-dir、GT 路径、日期范围等）
- 执行评测脚本，输出准召结果和 badcase 链接

**示例对话**：
```
你：评测一下上次跑的结果
Claude：请问是离线 batch 推理结果、线上高曝光数据，还是 pipeline 各阶段指标？
你：pipeline stages，run-dir 是 results/detection/run_20260510/
Claude：（自动确认 GT 路径，执行 eval_pipeline_stages.sh，输出结果）
```

---

## 配置说明

各链路配置独立存放：`configs/detection/{scenario}/pipeline_config.yaml`

以公文链路为例（`configs/detection/document_scan/pipeline_config.yaml`）：

| 参数 | 说明 |
|------|------|
| `key_configs` | 各召回通道（101/102/103/104）阈值及按元素类型的细粒度阈值 |
| `tier_quota` / `coarse_budget` | 粗排各置信层候选配额 |
| `qwen3_api_url` / `qwen3_max_workers` | Qwen3-VL 精排服务地址及并发数 |
| `owner_threshold` / `stage_b_mode` | Stage B owner投票策略（majority/score_sum/top1） |
| `owner_element_score_thresholds` | Stage C per-owner 自定义阈值（精度调优关键） |
| `risk_thresholds` / `owner_risk_thresholds` | Stage E 风险等级（high / mid / low）划分分数 |

---

## 环境说明

| 环境 | 路径 | 用途 |
|------|------|------|
| `data-env` | `/home/hadoop/data/share/ronan/venvs/data-env/bin/python` | 数据拉取（bigdata / Kyuubi） |
| `infer-env` | `/home/hadoop/data/share/ronan/venvs/infer-env/bin/python` | 推理、评测、预处理 |
| `qwen-env` | `/home/hadoop/data/share/ronan/venvs/qwen-env/bin/python` | Qwen3-VL 精排（vllm 本地部署） |
| `conda base` | `/opt/conda/bin/python` (Python 3.7) | 备用，部分旧脚本兼容 |

---

## 文档导航

| 文档 | 说明 |
|------|------|
| [`docs/evaluation/online_detection_eval.md`](docs/evaluation/online_detection_eval.md) | 在线评测完整流程 |
| [`docs/data/sql-recipes.md`](docs/data/sql-recipes.md) | 常用 SQL 查询示例 |
| [`docs/data/script-guide.md`](docs/data/script-guide.md) | 数据拉取脚本参数说明 |
| [`docs/scripts-reference.md`](docs/scripts-reference.md) | 所有脚本速查表 |
| [`docs/solutions/`](docs/solutions/) | 已记录的问题解决方案（bugs、最佳实践、工作流） |
| [`CLAUDE.md`](CLAUDE.md) | Claude Code 项目指引（架构、规范、常用命令） |
