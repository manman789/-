# 基于大语言模型的古今异义词判别系统

> **Ancient-Modern Semantic Shift Classifier** — A prompt-engineered few-shot system using Qwen (通义千问) to automatically identify whether a Classical Chinese word sense has undergone significant semantic change in Modern Chinese.

---

## 目录 Table of Contents

- [项目简介](#项目简介)
- [核心思路](#核心思路)
- [项目结构](#项目结构)
- [数据集说明](#数据集说明)
- [环境依赖](#环境依赖)
- [快速开始](#快速开始)
- [模块说明](#模块说明)
- [实验结果](#实验结果)
- [作者](#作者)

---

## 项目简介

本项目是一个**无需传统模型训练**的古今异义词自动判别系统，核心任务为：

> 给定一个古汉语词义项及其对应的现代汉语义项列表，判断该词义是否属于**"古今异义"**现象。

**标签定义：**

| 标签 | 含义 | 典型场景 |
|------|------|---------|
| `1` | 古今异义 | 古义消亡、语义转移、仅存于书面语/成语 |
| `0` | 语义延续 | 今义是古义的自然延续，或仅新增义项 |

**技术亮点：**
- 🔍 **提示工程驱动**：通过精心设计的三步逻辑树 Prompt，引导模型按照语言学家的判别路径推理
- 🛠️ **特征工程增强**：将频率、语体标签、关键词匹配等语言学信号转化为自然语言特征嵌入 Prompt
- 🔁 **少样本学习**：从高置信训练样本中均衡采样正负例，构建 Few-shot 示例
- ⚙️ **工程鲁棒性**：内置重试机制、JSON 强制输出、结果校验等保障模块

---

## 核心思路

系统采用 **"数据清洗 → 特征工程 → Prompt 构造 → LLM 推理 → 结果验证"** 的端到端流程：

```
原始数据 (train.xlsx / test.xlsx)
    │
    ▼
[0] 异常值清洗  ──────────────────────────────── 删除负频次样本
    │
    ▼
[1] 特征工程（4 项语言学特征）
    ├── match_status    古义关键词在现代义项中的重合度
    ├── modern_flags    现代义项的限制性语体标签（<书>/<古>）
    ├── decay_pattern   历史词频衰减模式（先秦→民国）
    └── rarity          词汇生僻度（基于 all_freq）
    │
    ▼
[2] Few-shot 采样  ───────────────────────────── 高置信样本，正负均衡
    │
    ▼
[3] Prompt 构造（三步逻辑树）
    ├── Step 1: 语义匹配  → 是否有含义一致的现代义项？
    ├── Step 2: 标签检查  → 匹配项是否带 <书>/<古> 等标签？
    └── Step 3: 频率验证  → 历史使用是否已显著衰退？
    │
    ▼
[4] 通义千问 (qwen-plus) 推理
    │   JSON 强制输出 + 最多 3 次重试
    │
    ▼
[5] 结果解析 & 输出
    └── test_predict_yecongrong.csv  (id, label)
```

---

## 项目结构

```
.
├── LLM代码文件.ipynb          # 完整代码（6 个功能模块）
├── train.xlsx                 # 原始训练集（423 条样本）
├── train_cleaned.xlsx         # 清洗后训练集（417 条，自动生成）
├── test.xlsx                  # 待预测测试集
├── test_predict.csv  # 最终预测结果（自动生成）
├── validation_errors.csv      # 验证集错误样本分析（自动生成）
└── README.md
```

---

## 数据集说明

本项目数据集为课程提供，**不包含在本仓库中**，请自行准备符合以下格式的文件并放置于项目根目录。

### `train.xlsx` — 训练集

| 列名 | 类型 | 说明 |
|------|------|------|
| `id` | int | 样本唯一 ID |
| `sense` | str | 古汉语词义项描述 |
| `modern_sense_list` | str | 对应现代汉语义项列表 |
| `label` | int | 标签：`1`=古今异义，`0`=语义延续 |
| `all_freq` | int | 该义项历史语料总频次 |
| `先秦_freq` / `漢_freq` / ... `民國_freq` | int | 各朝代词频（共 9 列） |

> `_count` 后缀列与 `_freq` 格式相同，数据清洗时会一并检测负值异常。

### `test.xlsx` — 测试集

与训练集列结构相同，但**不含 `label` 列**，由系统预测输出。

### 输出文件（运行后自动生成）

| 文件 | 说明 |
|------|------|
| `train_cleaned.xlsx` | 清洗后的训练集 |
| `test_predict.csv` | 测试集预测结果（`id` + `label` 两列） |
| `validation_errors.csv` | 验证集错误样本分析 |

---

## 环境依赖

**Python 版本：** 3.8+

```bash
pip install openai pandas openpyxl scikit-learn
```

**API 配置：**

本项目使用通义千问（Qwen）API，通过 DashScope 兼容 OpenAI 协议的接口接入。运行前需设置环境变量：

```bash
export DASHSCOPE_API_KEY="your_api_key_here"
```

> 获取 API Key：[阿里云 DashScope 控制台](https://dashscope.console.aliyun.com/)

---

## 快速开始

**1. 克隆仓库并准备数据**

```bash
git clone https://github.com/your-username/your-repo-name.git
cd your-repo-name
```

将 `train.xlsx` 和 `test.xlsx` 放入项目根目录。

**2. 配置 API Key**

```bash
export DASHSCOPE_API_KEY="your_api_key_here"
```

**3. 按顺序运行 Notebook 各模块**

在 Jupyter Notebook 中依次执行各 Cell（模块 0 → 模块 6），或直接运行全部：

```bash
jupyter nbconvert --to notebook --execute LLM代码文件.ipynb
```

**4. 查看结果**

预测完成后，结果保存至 `test_predict_yecongrong.csv`，格式如下：

```
id,label
1,1
2,0
3,1
...
```

---

## 模块说明

### 模块 0：数据异常值处理
- 检查 `train.xlsx` 必需列（`id` / `sense` / `modern_sense_list` / `label`）
- 清除各朝代词频列（`_freq` / `_count`）中含负值的异常样本
- 输出：`train_cleaned.xlsx`（417 条有效样本）

### 模块 1：特征工程（增强版预处理）
核心函数：

| 函数 | 功能 |
|------|------|
| `format_modern_sense` | 将现代义项列表格式化为带序号的清晰文本 |
| `compute_keyword_match` | 计算古义关键词在现代义项中的字面重合度（高度/部分/几乎未出现） |
| `extract_modern_flags` | 提取限制性语体标签（`<书>` / `<古>` / 方言 / 成语） |
| `compute_decay_feature` | 基于先秦至民国九朝词频计算历史使用重心（古代/居中/近现代） |
| `numeric_to_text` | 整合以上特征，转化为自然语言描述注入 Prompt |

### 模块 2：Few-shot 示例采样
- 优先从 `all_freq ≥ 20` 的高置信样本中采样
- 正负类（`label=0/1`）均衡，各默认取 3 例
- 固定随机种子（`random_state=42`）确保可复现

### 模块 3：Prompt 构造
- **系统提示**：定义"严谨的语言学家"角色 + 三步不可跳过的逻辑树
- **示例部分**：每例展示词语、古义、现代义项、辅助特征及推理参考
- **用户提示**：传入待预测样本的完整特征信息
- **输出约束**：强制返回包含 `id`、`reasoning`、`label` 的 JSON

### 模块 4：LLM 客户端初始化 & 单样本预测
- `init_llm_client()`：从环境变量加载 API Key，初始化 `qwen-plus` 客户端
- `predict_single_sample_sync()`：单样本同步预测，最多重试 3 次，失败保守返回 `label=1`

### 模块 5：验证集评估
- 训练集按 8:2 拆分为演示池和验证集
- 输出：准确率、精确率/召回率/F1、混淆矩阵（TP/FP/TN/FN）
- 错误样本保存至 `validation_errors.csv`

### 模块 6：测试集批量预测
- 复用训练集特征工程逻辑，保证预处理一致性
- 逐样本调用预测函数，实时打印进度
- 结果以 UTF-8-BOM 编码导出（兼容 Excel 中文显示）

---

## 实验结果

在验证集（83 个样本）上的表现：

|  | 优化前 | 优化后 |
|--|--------|--------|
| **准确率** | 85.54% | **90.36%** |
| 非古今异义 F1 | 73.91% | **83.33%** |
| 古今异义 F1 | 90.00% | **93.22%** |
| 宏平均 F1 | 81.96% | **88.28%** |

**优化方向：**
1. 引入 `match_status` 显式关键词匹配特征，降低模型对模糊语义匹配的依赖
2. 重构 Prompt 为强制三步推理链，显著减少假阳性（`FP`）错误

