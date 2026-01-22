---
title: Qwen3-VL-Embedding 多模态嵌入模型深度解析
published: 2026-01-22
description: '阿里通义千问3多模态嵌入模型架构与工程实践完整指南'
image: ''
tags: ['学习','大模型应用', 'RAG', '多模态']
category: '大模型'
draft: false 
lang: ''
---

## 📖 项目概述

**Qwen3-VL-Embedding** 和 **Qwen3-VL-Reranker** 是阿里巴巴通义团队基于 [Qwen3-VL](https://huggingface.co/collections/Qwen/qwen3-vl) 视觉语言模型开发的多模态嵌入和重排序模型系列。

### 核心能力

| 特性 | 说明 | 
|------|------|
| 🎨 **多模态支持** | 文本、图像、截图、视频及混合模态输入 |
| 🔄 **统一表示空间** | 不同模态映射到统一的语义向量空间 |
| 🎯 **高精度重排** | 通过交叉注意力实现精细的相关性评分 |
| 🌍 **多语言支持** | 支持 30+ 种语言 |
| 📏 **灵活向量维度** | 支持 Matryoshka Representation Learning (MRL) |

---

## 🔄 项目整体流程

下图展示了 Qwen3-VL-Embedding 项目的整体流程框架，从环境准备到最终结果输出，分为四个核心阶段：

![千问3整体流程](./image/LLM/千问3整体流程.png)

### 流程说明

整个系统分为 **四个核心阶段**：

1. **阶段 1：环境准备** - 安装依赖、下载模型、配置环境
2. **阶段 2：离线索引** - 数据预处理、Embedding生成、向量入库
3. **阶段 3：在线查询** - 查询向量生成、向量召回、Reranker精排
4. **阶段 4：结果输出** - 排序文档、相关性分数、来源追溯

---

## ⚙️ 阶段 1：环境准备

在开始使用 Qwen3-VL-Embedding 之前，需要确保系统满足以下要求并正确配置环境。

![千问3环境配置](./image/LLM/千问3环境配置.png)

### 系统要求

| 项目 | 最低要求 | 推荐配置 |
|------|----------|----------|
| **操作系统** | Linux / Windows / macOS | Ubuntu 20.04+ / Windows 11 |
| **Python 版本** | Python ≥ 3.11 | Python 3.11.x |
| **GPU** | NVIDIA GPU (8GB+ 显存) | RTX 4090 (24GB) 或 A100 |
| **CUDA** | CUDA 11.8+ | CUDA 12.1+ |
| **磁盘空间** | 20GB+ | 50GB+ (含模型) |
| **内存** | 16GB RAM | 32GB+ RAM |

### 核心依赖说明

| 依赖包 | 版本 | 作用 | 必需? |
|--------|------|------|-------|
| `torch` | 2.8.* | 深度学习框架核心 | ✅ 必需 |
| `transformers` | ≥4.57.3 | 加载 Qwen3-VL 模型 | ✅ 必需 |
| `qwen-vl-utils` | ≥0.0.14 | 图像/视频预处理工具 | ✅ 必需 |
| `accelerate` | ≥1.12.0 | 分布式推理加速 | ✅ 必需 |
| `decord` | ≥0.6.0 | 高效视频解码 | ✅ 必需 |
| `flash-attn` | 最新 | 注意力加速 (2倍速) | ⭐ 推荐 |

### 安装步骤

```bash
# 1. 克隆项目
git clone https://github.com/QwenLM/Qwen3-VL-Embedding.git
cd Qwen3-VL-Embedding

# 2. 运行安装脚本 (Linux/Mac)
bash scripts/setup_environment.sh

# 3. 激活环境
source .venv/bin/activate

# 4. 下载模型 (国内推荐使用 ModelScope)
pip install modelscope
modelscope download --model qwen/Qwen3-VL-Embedding-2B --local_dir ./models/Qwen3-VL-Embedding-2B
```

---

## 📂 支持的文件类型

系统支持多种文件类型的处理，实现真正的多模态检索能力。

![千问3支持文件类型](./image/LLM/千问3支持文件类型.png)

### 文件类型详解

| 支持的文件类型 | 文件扩展名 | 处理方式 |
|----------------|-----------|----------|
| **文本文档** | `.txt`, `.md`, `.json` | 直接读取文本 |
| **PDF 文档** | `.pdf` | 提取文本 + 截图页面 |
| **图片文件** | `.jpg`, `.png`, `.webp` | 直接作为图像输入 |
| **视频文件** | `.mp4`, `.avi`, `.mov` | 提取关键帧 (最多64帧) |
| **网页文件** | `.html` | 解析提取正文内容 |

---

## 📊 阶段 2：离线索引

离线索引阶段是将原始文档、图片、视频转化为向量并存储到向量数据库的过程。

![离线索引流程](./image/LLM/离线索引流程.png)

### 离线索引五步流程

离线索引包含以下五个核心步骤：

| 步骤 | 名称 | 说明 | 输出 |
|------|------|------|------|
| 2.1 | **数据收集** | 遍历文件目录，收集待处理文件 | 文件列表 |
| 2.2 | **内容解析** | 根据文件类型解析内容 | 标准化内容 |
| 2.3 | **内容分块** | 将长内容切分为小块 | 文档块列表 |
| 2.4 | **Embedding生成** | AI生成向量表示 | [B, 2048] 向量 |
| 2.5 | **向量入库** | 存入向量数据库 | 持久化索引 |

---

### 步骤 2.2：内容解析

系统根据不同的文件类型采用不同的解析策略：

![千问3内容解析](./image/LLM/千问3内容解析.png)

#### 各类型解析详解

| 文件类型 | 解析工具 | 输出内容 | 注意事项 |
|----------|----------|----------|----------|
| PDF | PyMuPDF (fitz) | 每页文本 + 页面图片 | 表格/公式可能需要 OCR |
| 图片 | PIL.Image | RGB 图像对象 | 自动调整尺寸 ≤ 1280×1440 |
| 视频 | Decord | 帧序列 (fps=1, 最多64帧) | 长视频需要降采样 |
| 文本 | open() | UTF-8 字符串 | 注意编码问题 |

---

### 步骤 2.3：内容分块

长文档需要切分成小块，每块保持语义完整，方便后续检索。

![千问3内容分块](./image/LLM/千问3内容分块.png)

#### 分块策略对比

| 策略 | 优势 | 劣势 | 适用场景 |
|------|------|------|----------|
| **固定长度分块** | 实现简单 | 可能在句子中间切断 | 快速原型 |
| **语义分块** | 保持语义完整 | 实现复杂 | 技术文档、章节划分 |
| **滑动窗口分块** | 有重叠，不漏信息 | 数据量增加 | 对准确性要求高的场景 |

#### 分块参数建议

| 内容类型 | 推荐块大小 | 重叠大小 | 说明 |
|----------|-----------|----------|------|
| 技术文档 | 512 tokens | 50 tokens | 段落/小节为单位 |
| 新闻文章 | 256 tokens | 30 tokens | 较短，快速检索 |
| 法律合同 | 1024 tokens | 100 tokens | 保持条款完整 |
| 图片文档 | 1 页 = 1 块 | 无 | 每页作为一个文档 |

---

### 步骤 2.4：Embedding 生成

这是离线索引中最核心的步骤，使用 Qwen3-VL-Embedding 模型将文档块转换为 2048 维向量。

![千问3Embedding生成](./image/LLM/千问3Embedding生成.png)

#### Embedding 生成流程概览

Embedding 生成将多模态内容（文本+图片+视频）转换为统一的向量表示，使得不同类型的内容可以在同一个语义空间中进行比较。

---

### Embedding 模型处理流程详解

下图展示了 Embedding 模型内部的详细处理流程，从输入到输出的完整数据变换过程：

![千问3Embedding模型处理流程](./image/LLM/千问3Embedding模型处理流程.png)

#### 模型内部五步流程

基于源代码 `src/models/qwen3_vl_embedding.py`，模型内部经过以下 5 个核心步骤：

| 步骤 | 函数名称 | 作用 | 输出 |
|------|----------|------|------|
| 1 | `format_model_input()` | 构造对话格式 | List[Dict] 对话 |
| 2 | `_preprocess_inputs()` | Tokenize + 图像预处理 | Tensor Dict |
| 3 | `model.forward()` | Transformer 编码 | hidden_states [B,L,D] |
| 4 | `_pooling_last()` | EOS Pooling 提取 | [B, D] 向量 |
| 5 | `F.normalize()` | L2 归一化 | 单位向量 |

#### EOS Pooling 原理

```
Token序列:  [BOS] [产] [品] [介] [绍] [IMG] [IMG] ... [EOS] [PAD] [PAD]
attention:    1    1    1    1    1    1     1   ...   1     0     0
                                                        ↑
                                               提取这个位置的向量 [D]

为什么用 EOS?
├─ EOS token 通过因果注意力"看过"整个序列的所有信息
└─ 是对整个输入的最佳压缩表示
```

#### 关键代码实现

```python
def _pooling_last(hidden_state, attention_mask):
    # 翻转 mask，找到最后一个 1 的位置 (EOS 位置)
    flipped = attention_mask.flip(dims=[1])
    last_pos = attention_mask.shape[1] - flipped.argmax(dim=1) - 1
    row = torch.arange(hidden_state.shape[0])
    return hidden_state[row, last_pos]  # 取出 EOS 向量
```

---

### 步骤 2.5：向量入库

将生成的向量和元数据存储到向量数据库，建立索引以便快速搜索。

![千问3向量入库流程](./image/LLM/千问3向量入库流程.png)

#### 支持的向量数据库

| 数据库 | 优势 | 适用场景 | 部署难度 |
|--------|------|----------|----------|
| **Milvus** | 功能全面，大规模支持 | 生产环境、10亿+ 向量 | ⭐⭐⭐ |
| **Qdrant** | 易用，Rust 高性能 | 中小规模、快速上手 | ⭐⭐ |
| **Faiss** | 纯 Python，无服务依赖 | 本地开发、PoC 验证 | ⭐ |
| **Chroma** | 最简单，内嵌式 | 快速原型、学习测试 | ⭐ |

#### 索引类型说明

| 索引类型 | 说明 | 召回率 | 速度 | 适用场景 |
|----------|------|--------|------|----------|
| **FLAT** | 暴力搜索，无压缩 | 100% | 慢 | 小数据集 (<10万) |
| **IVF_FLAT** | 倒排索引 | ~95% | 中 | 中等数据集 |
| **HNSW** | 图索引 (推荐) | ~98% | 快 | 大数据集、实时搜索 |

---

## 🔍 阶段 3：在线查询

在线查询阶段处理用户实时输入（文字/图片/视频），快速找出最相关的结果返回给用户。

![千问3在线查询流程](./image/LLM/千问3在线查询流程.png)

### 在线查询五步流程

| 步骤 | 名称 | 说明 | 耗时 |
|------|------|------|------|
| 3.1 | **接收查询** | 解析用户输入 | 即时 |
| 3.2 | **查询向量生成** | Embedder 生成向量 | ~50-100ms |
| 3.3 | **向量召回** | ANN 搜索 Top-100 | ~10-50ms |
| 3.4 | **Reranker精排** | 深度交互打分 | ~30-50ms/对 |
| 3.5 | **结果组装** | 排序+元数据返回 | 即时 |

---

## 🎯 完整多模态知识库流程

下图展示了从知识库构建到查询的完整流程：

![千问3多模态知识库构建和查询流程](./image/LLM/千问3多模态知识库构建和查询流程.png)

### RAG 系统完整流程

```
┌───────────────────┐    ┌───────────────────┐    ┌───────────────────┐    ┌───────────────────┐
│   阶段1: 召回      │    │   阶段2: 精排      │    │  阶段3: 上下文    │    │   阶段4: 生成      │
│                   │    │                   │    │                   │    │                   │
│  Embedding        │───▶│  Reranker         │───▶│  Context          │───▶│  Qwen3-VL         │
│  向量检索         │    │  重排序            │    │  构建             │    │  生成回答          │
│                   │    │                   │    │                   │    │                   │
│   → Top-100      │    │   → Top-10        │    │   → Prompt组装    │    │   → 最终回答       │
└───────────────────┘    └───────────────────┘    └───────────────────┘    └───────────────────┘
```

---

## 📝 在线查询详细步骤

### 步骤 3.1：接收查询

系统支持多种查询类型，解析用户输入：

![在线查询接受查询](image/LLM/在线查询接受查询.png)

#### 支持的查询类型

| 查询类型 | 示例 | 说明 |
|----------|------|------|
| **纯文本查询** | "如何配置网络设置？" | 最常见，用文字搜内容 |
| **图片查询** | 用户上传商品图片 | 以图搜图，找相似商品 |
| **视频查询** | 上传视频片段 | 找相关视频内容 |
| **混合查询** | 图片 + "找蓝色款式" | 图文组合，精确检索 |

---

### 步骤 3.2：查询向量生成

用 Embedding 模型把用户查询转成向量，与索引阶段使用的是**同一个模型**：

![查询向量生成](image/LLM/查询向量生成.png)

#### 代码示例

```python
# 生成查询向量
query = {"text": "如何配置网络设置？", "instruction": "检索产品文档"}
query_embedding = embedder.process([query], normalize=True)
# 输出: [1, 2048] 的归一化向量
```

> ⚠️ **重要**: 查询和文档必须用**同一个 Embedding 模型**，否则向量空间不一致，无法比较！

---

### 步骤 3.3：向量召回

在向量数据库中进行 ANN (近似最近邻) 搜索，快速返回 Top-K 相似候选：

![向量召回](image/LLM/向量召回.png)

#### 召回参数配置

| 参数 | 推荐值 | 说明 |
|------|--------|------|
| `top_k` | 100 | 召回数量，过大影响精排速度 |
| `ef_search` | 128 | HNSW 搜索深度，越大越精确但更慢 |
| `distance` | COSINE | 距离度量，与归一化向量搭配 |

---

### 步骤 3.4：Reranker 精排

用 Reranker 模型对每个 (Query, Document) 对进行**深度交互打分**，比向量召回更精准：

![Reranker精排](image/LLM/Reranker精排.png)

#### Embedding vs Reranker 对比

| 特性 | Embedding 召回 (双塔) | Reranker 精排 (单塔) |
|------|----------------------|---------------------|
| **架构** | 独立编码 | 交叉注意力 |
| **优势** | 可预计算，毫秒级 | 深度交互，更精准 |
| **劣势** | 无交互，精度有限 | 需逐对计算，较慢 |
| **适用** | 百万级召回 | Top-20 精排 |

---

### Reranker 内部处理流程

下图详细展示了 Reranker 模型的内部工作原理：

![Ranker内部处理流程](image/LLM/Ranker内部处理流程.png)

#### Reranker 打分原理

Reranker 把问题转化为"判断文档是否满足查询需求"的二分类任务：

```python
# 构造的 prompt 格式
{
    "role": "system",
    "content": "Judge whether the Document meets the requirements based on the 
                Query and the Instruct. Note that the answer can only be 'yes' or 'no'."
},
{
    "role": "user",
    "content": [
        "<Instruct>: 检索相关产品文档",
        "<Query>: 如何配置网络设置？",
        "<Document>: 网络配置步骤：1. 打开设置面板..."
    ]
}
```

#### 关键代码实现

```python
def get_binary_linear(self, model, token_yes, token_no):
    lm_head_weights = model.lm_head.weight.data
    weight_yes = lm_head_weights[token_yes]  # "yes" token 的权重
    weight_no = lm_head_weights[token_no]    # "no" token 的权重
    
    linear_layer = torch.nn.Linear(D, 1, bias=False)
    linear_layer.weight[0] = weight_yes - weight_no  # 差值作为分类权重
    return linear_layer

def compute_scores(self, inputs):
    hidden_state = self.model(**inputs).last_hidden_state[:, -1]  # 取最后 token
    scores = self.score_linear(hidden_state)  # 线性变换
    scores = torch.sigmoid(scores)  # 归一化到 [0, 1]
    return scores
```

**为什么这样设计?**
- 利用模型预训练的 `yes`/`no` 语义知识
- 只需一次前向传播，比生成更高效
- 输出是连续分数，可排序

---

## 📊 性能对比

![性能对比](image/LLM/性能对比.png)

### Embedding vs Reranker 性能对比

| 指标 | Embedding 召回 | Reranker 精排 |
|------|----------------|---------------|
| **架构** | 双塔 (独立编码) | 单塔 (交叉注意力) |
| **延迟 (单条)** | 50-100ms | 30-50ms/对 |
| **吞吐量** | 可批处理，高 | 需逐对，低 |
| **精度** | 中等 (~80%) | 高 (~95%) |
| **适用场景** | 粗筛大规模候选 | 精排少量候选 |
| **典型调用** | Top-100 候选 | Top-20 精排 |

### 与其他方案的对比

| 方案 | 模态支持 | 视频支持 | 中文能力 | 开源 | 模型规模 |
|------|----------|----------|----------|------|----------|
| **Qwen3-VL-Embedding** | 文本+图像+视频 | ✅ 64帧 | ✅ 原生 | ✅ | 2B/8B |
| CLIP | 文本+图像 | ❌ |   | ✅ | 0.4B |
| OpenAI Embedding | 仅文本 | ❌ | ✅ | ❌ | 未知 |
| BGE-M3 | 仅文本 | ❌ | ✅ | ✅ | 0.6B |
| VLM2Vec | 文本+图像 | ⚠️ 有限 | ⚠️ 弱 | ✅ | 2B |

---

## 📤 阶段 4：结果输出

![结果输出](image/LLM/结果输出.png)

### 最终输出结构

```json
{
  "query": "用户的查询内容",
  "results": [
    {
      "rank": 1,
      "id": "doc_042",
      "content": "匹配的内容...",
      "embedding_score": 0.89,
      "rerank_score": 0.95,
      "metadata": {
        "source": "manual.pdf",
        "page": 42,
        "image_path": "/path/img.jpg"
      }
    }
  ],
  "total_time_ms": 156
}
```

---

## 📊 模型规格

| 模型 | 参数量 | 层数 | 序列长度 | 嵌入维度 | MRL支持 |
|------|--------|------|----------|----------|---------|
| Qwen3-VL-Embedding-2B | 2B | 28 | 32K | 2048 | ✅ |
| Qwen3-VL-Embedding-8B | 8B | 36 | 32K | 4096 | ✅ |
| Qwen3-VL-Reranker-2B | 2B | 28 | 32K | - | - |
| Qwen3-VL-Reranker-8B | 8B | 36 | 32K | - | - |

---

## 💡 部署建议

| 业务规模 | 推荐配置 | 月成本估算 |
|----------|----------|------------|
| **PoC/验证** | 1x RTX 4090，2B 模型 | ¥0 (自有) / ¥3k (云) |
| **小规模** | 2x A10，2B 模型 | ¥6-10k |
| **中规模** | 2x A100 40GB，8B 模型 | ¥30-50k |
| **大规模** | 4x A100 80GB + 向量库集群 | ¥100k+ |

### 适用场景评估

| 场景 | 推荐程度 | 说明 |
|------|----------|------|
| **多模态知识库** | ⭐⭐⭐⭐⭐ | 核心优势场景，图文视频混合检索 |
| **电商图搜图** | ⭐⭐⭐⭐⭐ | 图像理解能力强，支持文字搜商品图 |
| **视频内容检索** | ⭐⭐⭐⭐⭐ | 少有的视频嵌入支持，时刻检索 |
| **文档理解 OCR** | ⭐⭐⭐⭐ | ViDoRe 评测优秀，截图理解强 |
| **纯文本检索** | ⭐⭐⭐ | 可用但建议用 Qwen3-Embedding |
| **边缘设备** | ⭐ | 模型过大，不适合 |

---

## 📚 更多资源

- 📄 [技术报告 PDF](https://github.com/QwenLM/Qwen3-VL-Embedding)
- 📓 [Embedding 示例 Notebook](https://github.com/QwenLM/Qwen3-VL-Embedding/tree/main/examples)
- 🤗 [Hugging Face 模型](https://huggingface.co/collections/Qwen/qwen3-vl-embedding)
- 📦 [ModelScope 模型](https://modelscope.cn/organization/qwen/qwen3-vl-embedding)
