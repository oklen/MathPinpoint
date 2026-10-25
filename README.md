<h1 align="center">MathPinpoint</h1>

<h3 align="center">题目级（problem-level）数学检索的训练数据与测试集</h3>

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

MathPinpoint 是一个**题目级数学检索**数据集。给一道数学题，任务是找到解答*这道*题、并且解对了的网页。每个 (query, page) pair 都按同一份书面 rubric [`prompts/judge_prompt.md`](prompts/judge_prompt.md) 标了 0、1 或 2 的相关性 label。这份 rubric 在两方面很严格：

- **同一道题，而不是同一个主题。** 网页如果讲的是另一道题（`wrong_problem`）、改了条件（`condition_mismatch`）、求的是另一个目标（`target_mismatch`），或者只是主题或关键词相同（`topic_only`），都记 0 分。
- **考虑对错（correctness-aware）。** 网页的核心数学如果是错的，即使讲的是同一道题，也记 0 分。用错误的推理得到正确的最终答案，同样记 0 分。

|               |                                                                                  |
|---------------|----------------------------------------------------------------------------------|
| 任务          | retrieval：一道数学题 → 解答它的网页                                             |
| 领域          | 数学，网页                                                                       |
| 语言          | 英语                                                                             |
| Corpus        | 3,525,546 篇文档，两个 split 共用                                                |
| 训练数据      | 1,853,389 条 query，43,506,169 个判过的 (query, document) pair                   |
| 测试数据      | 6,173 条 query，83,833 个相关的 (query, document) pair                           |
| Label         | 0 / 1 / 2。训练集：GPT-5.6-Sol，或从它蒸馏出的一个 8B 模型。测试集：两个 LLM judge 加一个 arbiter |
| 指标          | nDCG@10、Recall@100                                                              |
| 来源          | [UltraData-Math](https://huggingface.co/datasets/openbmb/UltraData-Math) L2 层的网页 |

## Quick start

数据集是 gated 的，申请自动通过：在它的 [Hugging Face 页面](https://huggingface.co/datasets/oklenAI/MathPinpoint)上接受条款，然后在本地 `hf auth login`。

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

给 `load_dataset` 传 `revision="v2.0"` 可以固定到这个版本。

## 评测

对每条测试 query 在整个 corpus 里检索，再用 `qrels` 给排序打分。下面的函数计算本 README 报告的两个指标。所有相关文档的 label 都是 2，所以 graded nDCG 和 binary nDCG 是一样的。

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

没被判过的文档按不相关计算。所以，如果某个 retriever 的排序和构建 test pool 的那些系统差别很大，只有它找到的那些相关文档会被算成不相关，它会因此吃亏。见 [limitations](docs/limitations.md#test-split)。

## 数据集结构

| config | split | 行数 | 内容 |
|---|---|---:|---|
| `corpus` | train | 3,525,546 | 去重后的数学文档，两个 split 共用 |
| `queries` | train | 1,853,389 | query 文本、它是从哪个页面抽出来的、抽取的 metadata |
| `judgments` | train | 43,506,169 | 每个判过的 (query, document) pair 一行：它在每一路检索里的排名、label、由哪个 judge 给出，有的话还有 raw score |
| `queries` | test | 6,173 | held-out 的测试 query |
| `qrels` | test | 83,833 | 每条测试 query 的相关文档 |

Hugging Face 把一个 config 唯一的 split 叫作 `train`，所以 corpus 的 split 是 `train`。各列的 schema 见 [`docs/schema.md`](docs/schema.md)，test split 的说明见 [`docs/test_split.md`](docs/test_split.md)。

## Label 是怎么来的

1. **文档。** 数学网页经过 content extractor 抽取、normalize，再用 MinHash 去重。抽取结果退化（degenerate）的文档，以及捆绑了五道及以上题目的文档，都被去掉。见 [`docs/corpus.md`](docs/corpus.md)。
2. **Query。** 每个页面生成一条 query，用的是一个 minimal-edit prompt：如果页面里有一道真有人问过的题，这道题*就是* query，尽可能少改地照抄过来。prompt 是 [`prompts/query_gen_prompt.txt`](prompts/query_gen_prompt.txt)。之后对 query 去重和过滤，去掉和评测集的 leakage，并去掉两个 LLM judge 都判为不 self-contained 的 query。见 [`docs/queries.md`](docs/queries.md)。
3. **候选。**
   - 三路检索各贡献 top 100：dense（Qwen3-Embedding-4B）、BM25 和 word 5-gram，再用加权 RRF 融合（dense 权重 2，k = 60）。
   - 一条 query 的候选是它的 dense top 20 和融合后的 top 20。平均每条 query 有 4.55 个候选只出现在融合 top 20 里。
   - 见 [`docs/judging.md`](docs/judging.md)。
4. **Judging。**
   - LLM judge（GPT-5.6-Sol）按 `prompts/judge_prompt.md` 标注：dense 排名 1–5；dense 排名 6–20 里，zero-shot reranker 排出的 top 5，以及所有长于 8,144 token 的文档；还有只出现在融合 top 20 里的候选中，排在融合第 1–5 名、长于 8,144 token 的文档。
   - 一个用 LLM label 训练的 8B relevance 模型判其余所有的 pair。它也给 reranker 挑出的那些 pair 打分，所以这些 pair 同时带有两种 label。这个模型已发布为 [oklenAI/MathPinpoint-Judge-8B](https://huggingface.co/oklenAI/MathPinpoint-Judge-8B)。
   - 每一行都记录了 label 由哪个 judge 给出，凡是有 8B raw score 的地方都保留了。见 [`docs/judging.md`](docs/judging.md)。

## 在 MathPinpoint 上训练 retriever

用这次发布的数据构造训练样本，fine-tune 了 Qwen3-Embedding-0.6B，并在 test split（6,173 条 query）上和 fine-tune 之前的同一个模型比较。

| 指标 | fine-tune 前 | fine-tune 后 | 差值 [95% CI] |
|---|---:|---:|---:|
| nDCG@10 | 0.441 | 0.477 | +0.036 [+0.028, +0.045] |
| R@100 | 0.561 | 0.568 | +0.007 [−0.001, +0.016] |

两个指标都把没判过的文档算作不相关，而 test pool 对两个模型的覆盖并不均衡：fine-tune 前模型的 top 10 有 66% 被判过，fine-tune 后的只有 35%。如果只对判过的文档排序（condensed nDCG@10），fine-tune 前是 0.486，后是 0.617，差 +0.131 [+0.123, +0.139]。在一个两个模型 top 10 都被完整判过的 500 条 query 样本上，nDCG@10 的差是 +0.114，接近 condensed 的值。

<details>
  <summary>训练样本、训练和评测设置（点击展开）</summary>

- **训练样本。** 每条 query 生成一条：
  - positive 从这条 query 得 2 分的候选里随机抽一个；
  - dense 排名最靠前的三个 0 分候选作为 hard negative；不在 dense top 100 里的候选排在最后；
  - 0 分候选少于三个的 query 跳过。

  两个 judge 的 label 都会用到：
  - 两个 judge 都标过的 pair，用 LLM 的 label。
  - 只有 8B 模型标过的 pair，raw score ≥ 1.5 的记为 2 分，< 0.5 的记为 0 分。
  - 不用 source-page check 产生的行，标为 not evaluable 的 pair 也丢掉。检索也返回了 query 自己的 source page 时，它和其他候选一样对待；在 46.8% 的样本里它是 positive。

  文档截到 4,000 个字符，query 截到 2,000 个字符。这样得到 1,225,057 条样本。
- **训练。**
  - Loss：MultipleNegativesRankingLoss（in-batch negatives 加三个 hard negative），再加一个在 768、512、256、128 维上的 Matryoshka loss。
  - Batch：effective batch 512（8 张 GPU × 64，GradCache mini-batch 32）。
  - Schedule：learning rate 2e-5，10% warmup，sequence length 512，1,000 步。所以一次运行只看到 1,225,057 条里的 512,000 条。
  - 运行：三次，seed 分别是 42、43、44。逐 query 的分数在三次之间取平均。
- **评测。**
  - 在全部 3,525,546 篇 corpus 文档上检索，last-token pooling，长度 512 token。每条 query 前面加下面这段 instruction；文档不加前缀。

    ```
    Instruct: Given a web search query, retrieve relevant passages that answer the query
    Query:
    ```

  - 相关文档是 test split 的 `qrels`：判为 2 分的文档，也就是把同一道题解对的文档。指标就是[评测](#评测)一节里 `evaluate` 计算的那两个。
  - 测试 query 是 held-out 的：没有任何一条训练 query 和它们逐字相同、word 5-gram Jaccard ≥ 0.50（穷举检查），或被判为同一道题的语义重复。
  - CI 来自对 query 做 paired bootstrap，重采样 10,000 次。

</details>

## 文档

| 文档（英文） | |
|---|---|
| 📚 [Corpus](docs/corpus.md) | source page、抽取、从文档里删掉页面自带的题目、去重、degenerate 和多题文档、document id |
| ❓ [Queries](docs/queries.md) | query 抽取、去重、去 leakage、source-page check、self-containedness check |
| ⚖️ [Candidates and judging](docs/judging.md) | 候选、谁判了什么、rubric、LLM judge、8B 模型 |
| 🧪 [Test split](docs/test_split.md) | 测试 query，以及它们的相关文档是怎么判的 |
| 🗂️ [Schema](docs/schema.md) | 发布的目录结构和每一列 |
| ⚠️ [Limitations](docs/limitations.md) | 这些 label 能告诉你什么、不能告诉你什么 |

## Prompt 与 schema

`prompts/` 里是生成这些数据时用的原始文本，[`prompts/SHA256SUMS`](prompts/SHA256SUMS) 固定了它们的哈希。judge prompt 里的 `guideline_version` 字符串在输出 schema 里是写死的，所以它说明不了某个 label 是由哪一份 prompt 文本生成的；SHA-256 可以。

| 文件 | sha256 |
|---|---|
| `judge_prompt.md` | `031ac72b649ae979…` |
| `query_gen_prompt.txt` | `d75d30a5b8cb0b03…` |
| `query_check_prompt.md` | `5de21b985641bc6a…` |

## 来源与许可

- **数据。** Hugging Face 数据集里的所有内容（corpus、queries、judgments 和 test qrels）以 [CC0 1.0](https://creativecommons.org/publicdomain/zero/1.0/) 发布。
- **代码。** 本仓库里的 prompt、schema 和代码以 [Apache License 2.0](https://www.apache.org/licenses/LICENSE-2.0) 发布。

文档是 [UltraData-Math](https://huggingface.co/datasets/openbmb/UltraData-Math) L2 层网页清洗后的抽取结果，UltraData-Math 以 Apache-2.0 发布；我们用的是它的一个副本，这个副本给每个页面加了难度分（见 [corpus](docs/corpus.md)）。原始网页的权利归其所有者。

## 引用

如果你用了 MathPinpoint，请引用它。它的文档来自 UltraData-Math，所以也请一并引用那个数据集。

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

根据发布的数据表计算。长度单位是字符（Unicode code point）。

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
