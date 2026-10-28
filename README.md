<h1 align="center">MathPinpoint</h1>

<h3 align="center">problem-level 数学检索的训练数据与测试集</h3>

<p align="center"><b>中文</b> | <a href="README.en.md">English</a></p>

<p align="center">
    <a href="https://huggingface.co/datasets/oklenAI/MathPinpoint">
        <img alt="Hugging Face dataset" src="https://img.shields.io/badge/%F0%9F%A4%97%20Hugging%20Face-oklenAI%2FMathPinpoint-ffcc4d">
    </a>
    <a href="https://huggingface.co/datasets/oklenAI/MathPinpoint/tree/v2.0">
        <img alt="Version" src="https://img.shields.io/badge/version-v2.0-blue">
    </a>
    <a href="https://creativecommons.org/publicdomain/zero/1.0/">
        <img alt="Data license: CC0 1.0" src="https://img.shields.io/badge/data-CC0%201.0-lightgrey">
    </a>
    <a href="https://www.apache.org/licenses/LICENSE-2.0">
        <img alt="Code license: Apache 2.0" src="https://img.shields.io/badge/code-Apache%202.0-blue">
    </a>
</p>

<h4 align="center">
    <p>
        <a href="#quick-start">Quick start</a> |
        <a href="#评测">评测</a> |
        <a href="#数据集结构">数据集结构</a> |
        <a href="#在-mathpinpoint-上训练-retriever">Baseline</a> |
        <a href="#文档">文档</a> |
        <a href="#引用">引用</a>
    </p>
</h4>

MathPinpoint 是一个题目级数学检索数据集。输入一道数学题，检索目标是给出该题正确解答的网页。每个查询与网页的配对（query, page）都按照 [`prompts/judge_prompt.md`](prompts/judge_prompt.md) 中的规则标注，相关性评分为 0、1 或 2。评分时检查题目是否一致，以及解答是否正确：

- 网页解答的是另一道题（`wrong_problem`）、题目条件不同（`condition_mismatch`）、求解目标不同（`target_mismatch`），或仅有主题、关键词相同（`topic_only`），均记 0 分。
- 即使题目一致，网页的关键数学内容有误，也记 0 分。推理错误但最终答案正确的解答同样记 0 分。

| 项目 | 说明 |
|---|---|
| 任务 | 输入数学题，检索解答该题的网页 |
| 领域 | 数学网页 |
| 语言 | 英语 |
| 文档库（corpus） | 3,525,546 篇文档，训练集与测试集共用 |
| 训练数据 | 1,853,389 条查询，43,506,169 条查询与文档的配对标注 |
| 测试数据 | 6,173 条查询，83,833 个相关的查询与文档配对 |
| 相关性标签 | 0 / 1 / 2。训练集由 GPT-5.6-Sol 或从它蒸馏的 8B 模型标注；测试集由两个 LLM 评审和一个仲裁器标注 |
| 指标 | nDCG@10、Recall@100 |
| 来源 | [UltraData-Math](https://huggingface.co/datasets/openbmb/UltraData-Math) L2 层的网页 |

## Quick start

下载前需要申请访问权限，申请会自动通过。在数据集的 [Hugging Face 页面](https://huggingface.co/datasets/oklenAI/MathPinpoint)接受条款后，在本地运行 `hf auth login` 登录。

```python
from datasets import load_dataset

repo = "oklenAI/MathPinpoint"

# Corpus（两个 split 共用）和 test split
corpus = load_dataset(repo, "corpus", split="train")    # did, text, n_chars
queries = load_dataset(repo, "queries", split="test")   # qid, text
qrels = load_dataset(repo, "qrels", split="test")       # qid, did, label

# 训练数据
train_queries = load_dataset(repo, "queries", split="train")
judgments = load_dataset(repo, "judgments", split="train", streaming=True)  # 43.5M 行
```

在 `load_dataset` 中指定 `revision="v2.0"`，可固定使用本次发布的版本。

## 评测

评测时，对每条测试查询在整个文档库中检索，再根据 `qrels` 计算排名指标。下面的函数用于计算本 README 中的两个指标。所有相关文档的标签均为 2，因此按分级相关性计算的 nDCG 与二值 nDCG 相同。

```python
import math
from collections import defaultdict

relevant = defaultdict(set)
for row in qrels:
    relevant[row["qid"]].add(row["did"])

def evaluate(run, relevant, k_ndcg=10, k_recall=100):
    """run: {qid: [did, ...]}, each list ranked best first."""
    ndcg, recall = [], []
    for qid, rel in relevant.items():
        ranked = run.get(qid, [])
        dcg = sum(1 / math.log2(i + 2) for i, did in enumerate(ranked[:k_ndcg]) if did in rel)
        idcg = sum(1 / math.log2(i + 2) for i in range(min(len(rel), k_ndcg)))
        ndcg.append(dcg / idcg)
        recall.append(len(rel & set(ranked[:k_recall])) / len(rel))
    return sum(ndcg) / len(ndcg), sum(recall) / len(recall)

# run = {qid: my_retriever(text, k=100) for qid, text in zip(queries["qid"], queries["text"])}
# ndcg_at_10, recall_at_100 = evaluate(run, relevant)
```

未标注的文档按不相关处理。如果检索器的排序与构建测试候选池的系统差异较大，它独自找到的相关文档可能尚未标注，也会被计为不相关，导致评测分数偏低。详见 [limitations](docs/limitations.md#test-split)。

## 数据集结构

| config | split | 行数 | 内容 |
|---|---|---:|---|
| `corpus` | train | 3,525,546 | 去重后的数学文档，训练集与测试集共用 |
| `queries` | train | 1,853,389 | 查询文本、来源页面及抽取元数据 |
| `judgments` | train | 43,506,169 | 每个已标注的查询与文档配对占一行，记录各路检索排名、相关性标签、标注模型，以及可用的原始分数（raw score） |
| `queries` | test | 6,173 | 独立留出的测试查询 |
| `qrels` | test | 83,833 | 每条测试查询对应的相关文档 |

Hugging Face 将配置中唯一的 split 命名为 `train`，因此 corpus 的 split 名称为 `train`。各列的字段定义见 [`docs/schema.md`](docs/schema.md)，测试集说明见 [`docs/test_split.md`](docs/test_split.md)。

## 标注流程

1. 文档处理。使用内容抽取器提取数学网页正文，规范化后用 MinHash 去重。剔除抽取结果退化（degenerate）的文档，以及包含五道及以上题目的文档。详见 [`docs/corpus.md`](docs/corpus.md)。
2. 查询生成。每个页面生成一条查询。如果页面中有实际提出的问题，就直接采用原题，尽量少改。所用提示词见 [`prompts/query_gen_prompt.txt`](prompts/query_gen_prompt.txt)。生成后进行去重和过滤，排除与评测集的数据泄漏，并剔除两个 LLM 评审均判定为题目信息不完整（not self-contained）的查询。详见 [`docs/queries.md`](docs/queries.md)。
3. 候选检索。分别取稠密检索（dense，使用 Qwen3-Embedding-4B）、BM25 和 word 5-gram 的前 100 个结果，再用加权 RRF 融合，dense 权重为 2，k = 60。每条查询的候选集由 dense 前 20 和融合结果前 20 合并而成。平均每条查询有 4.55 个候选仅出现在融合结果前 20 中。详见 [`docs/judging.md`](docs/judging.md)。
4. 相关性标注。
   - GPT-5.6-Sol 按 `prompts/judge_prompt.md` 标注以下候选：dense 排名第 1 至 5 的文档；dense 排名第 6 至 20 中，由零样本重排器（zero-shot reranker）选出的前 5 个结果，以及所有长度超过 8,144 token 的文档；仅出现在融合结果前 20 中，融合排名第 1 至 5、长度超过 8,144 token 的文档。
   - 其余配对由一个使用 LLM 标签训练的 8B 相关性模型标注。该模型也会对重排器选出的配对评分，因此这部分数据同时保留两种标签。模型已发布：[oklenAI/MathPinpoint-Judge-8B](https://huggingface.co/oklenAI/MathPinpoint-Judge-8B)。
   - 每行数据都记录标签的来源模型，并保留可用的 8B 模型原始分数。详见 [`docs/judging.md`](docs/judging.md)。

## 在 MathPinpoint 上训练 retriever

我们用本次发布的数据构造训练样本，微调 Qwen3-Embedding-0.6B，并在包含 6,173 条查询的测试集上比较微调前后的表现。

| 指标 | 微调前 | 微调后 | 差值 [95% CI] |
|---|---:|---:|---:|
| nDCG@10 | 0.441 | 0.477 | +0.036 [+0.028, +0.045] |
| R@100 | 0.561 | 0.568 | +0.007 [−0.001, +0.016] |

这两个指标都将未标注文档视为不相关。测试候选池对两个模型的覆盖率不同：微调前模型的前 10 个结果中，66% 已有标注；微调后这一比例为 35%。仅对已标注文档排序时（condensed nDCG@10），微调前后的分数分别为 0.486 和 0.617，差值为 +0.131 [+0.123, +0.139]。在一个包含 500 条查询、两个模型前 10 个结果均已完整标注的样本中，nDCG@10 差值为 +0.114，与 condensed 指标的差值接近。

<details>
  <summary>训练样本、训练和评测设置（点击展开）</summary>

- 训练样本：每条查询构造一条样本。
  - 从得 2 分的候选中随机抽取一个正例（positive）。
  - 选取 dense 排名最靠前的三个 0 分候选作为困难负例（hard negative）；不在 dense 前 100 中的候选排在最后。
  - 跳过 0 分候选不足三个的查询。

  两个标注模型的标签按以下方式使用：
  - 两个模型都标注过的配对，采用 LLM 标签。
  - 仅由 8B 模型标注的配对，原始分数 ≥ 1.5 时记为 2 分，< 0.5 时记为 0 分。
  - 排除来源页检查（source-page check）产生的记录，以及标为无法评估（not evaluable）的配对。如果检索结果包含查询的来源页，就按普通候选处理。在 46.8% 的样本中，来源页被选作正例。

  文档截断至 4,000 个字符，查询截断至 2,000 个字符，最终得到 1,225,057 条样本。

- 训练设置：
  - 损失函数采用 MultipleNegativesRankingLoss，使用批内负例（in-batch negatives）和三个困难负例，并在 768、512、256、128 维上加入 Matryoshka loss。
  - 有效批大小为 512（8 张 GPU × 64），GradCache mini-batch 为 32。
  - 学习率为 2e-5，warmup 比例为 10%，序列长度为 512，共训练 1,000 步。因此每次运行使用 1,225,057 条样本中的 512,000 条。
  - 共运行三次，随机种子分别为 42、43、44。每条查询的评测分数取三次运行的平均值。

- 评测设置：
  - 在全部 3,525,546 篇文档上检索，采用 last-token pooling，序列长度为 512 token。在每条查询前添加以下指令，文档不加前缀。

    ```
    Instruct: Given a web search query, retrieve relevant passages that answer the query
    Query:
    ```

  - 相关文档取自测试集的 `qrels`，均为正确解答同一道题、评分为 2 的文档。指标按[评测](#评测)一节中的 `evaluate` 函数计算。
  - 测试查询独立留出。训练查询中不存在与测试查询逐字相同、word 5-gram Jaccard ≥ 0.50（经穷举检查），或被判定为同一道题的语义重复项。
  - 置信区间（CI）通过对查询进行配对 bootstrap 估计，重采样 10,000 次。

</details>

## 文档

| 文档（英文） | |
|---|---|
| [Corpus](docs/corpus.md) | 来源页面、正文抽取、移除页面原题、去重、抽取退化与多题文档处理、文档 ID |
| [Queries](docs/queries.md) | 查询抽取、去重、排除数据泄漏、来源页检查、题目信息完整性检查 |
| [Candidates and judging](docs/judging.md) | 候选构建、标注分工、评分规则、LLM 评审与 8B 模型 |
| [Test split](docs/test_split.md) | 测试查询及相关文档的标注方法 |
| [Schema](docs/schema.md) | 发布目录结构与字段定义 |
| [Limitations](docs/limitations.md) | 标签的适用范围与局限 |

## Prompt 与 schema

`prompts/` 保存生成数据时使用的原始提示词，[`prompts/SHA256SUMS`](prompts/SHA256SUMS) 记录各文件的哈希。评审提示词中的 `guideline_version` 字符串由输出 schema 固定，无法用来区分生成标签时使用的提示词文本；可通过 SHA-256 确认对应的提示词文本。

| 文件 | sha256 |
|---|---|
| `judge_prompt.md` | `031ac72b649ae979…` |
| `query_gen_prompt.txt` | `d75d30a5b8cb0b03…` |
| `query_check_prompt.md` | `5de21b985641bc6a…` |

## 来源与许可

Hugging Face 数据集中的全部内容（corpus、queries、judgments 和 test qrels）以 [CC0 1.0](https://creativecommons.org/publicdomain/zero/1.0/) 发布。本仓库的提示词、schema 和代码以 [Apache License 2.0](https://www.apache.org/licenses/LICENSE-2.0) 发布。

文档来自 [UltraData-Math](https://huggingface.co/datasets/openbmb/UltraData-Math) L2 层网页，经正文抽取和清洗后收录。UltraData-Math 以 Apache-2.0 发布；我们使用的副本为每个页面添加了难度分，详见 [corpus](docs/corpus.md)。原始网页的权利归其所有者。

## 引用

使用 MathPinpoint 时，请同时引用 MathPinpoint 和文档来源数据集 UltraData-Math。

<details>
  <summary>BibTeX（点击展开）</summary>

```bibtex
@misc{mathpinpoint2026,
  title     = {{MathPinpoint}: Training Data and a Test Set for Problem-Level Math Retrieval},
  author    = {Cai, Zefeng and Shippole, Enrico},
  year      = {2026},
  note      = {Version 2.0},
  publisher = {Hugging Face},
  url       = {https://huggingface.co/datasets/oklenAI/MathPinpoint}
}

@misc{ultradata-math,
  title={UltraData-Math},
  author={Chuyue Zhou and Hongya Lyu and Xinle Lin and Hengyu Zhao and Junshao Guo and Xueren Zhang and Shuaikang Xue and Qiang Ma and Jie Zhou and Yudong Wang and Zhiyuan Liu},
  year={2026},
  url={https://huggingface.co/datasets/openbmb/UltraData-Math},
  publisher={Hugging Face}
}
```

</details>

## 数据集统计

<details>
  <summary>数据集统计（点击展开）</summary>

以下统计根据已发布的数据表计算，文本长度以字符（Unicode code point）为单位。

```json
{
    "train": {
        "num_samples": 5378935,
        "number_of_characters": 26524462290,
        "num_documents": 3525546,
        "min_document_length": 2,
        "average_document_length": 7418.14,
        "max_document_length": 168093,
        "unique_documents": 3525546,
        "num_queries": 1853389,
        "min_query_length": 3,
        "average_query_length": 200.42,
        "max_query_length": 8894,
        "unique_queries": 1853389,
        "num_judgments": 43506169,
        "min_judgments_per_query": 1,
        "average_judgments_per_query": 23.47,
        "max_judgments_per_query": 41,
        "judgments_per_label": {
            "0": 17512014,
            "1": 17025106,
            "2": 8888149,
            "unevaluable": 80900
        },
        "judgments_per_judge": {
            "llm": 19400323,
            "rm8b": 24105846
        }
    },
    "test": {
        "num_samples": 3531719,
        "number_of_characters": 26154284818,
        "num_documents": 3525546,
        "min_document_length": 2,
        "average_document_length": 7418.14,
        "max_document_length": 168093,
        "unique_documents": 3525546,
        "num_queries": 6173,
        "min_query_length": 13,
        "average_query_length": 206.93,
        "max_query_length": 2624,
        "unique_queries": 6173,
        "num_relevant_docs": 83833,
        "min_relevant_docs_per_query": 1,
        "average_relevant_docs_per_query": 13.58,
        "max_relevant_docs_per_query": 140,
        "unique_relevant_docs": 78996
    }
}
```

</details>
