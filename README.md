<h1 align="center">MathPinpoint</h1>

<h3 align="center">Training data and a test set for problem-level math retrieval</h3>

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
        <a href="#evaluation">Evaluation</a> |
        <a href="#dataset-structure">Dataset structure</a> |
        <a href="#training-a-retriever-on-mathpinpoint">Baseline</a> |
        <a href="#documentation">Documentation</a> |
        <a href="#citing">Citing</a>
    </p>
</h4>

MathPinpoint is a dataset for **problem-level math retrieval**. Given a math question, the task is to find the web page that solves *that* problem, and solves it correctly. Each (query, page) pair carries a relevance label of 0, 1 or 2 under one written rubric, [`prompts/judge_prompt.md`](prompts/judge_prompt.md). The rubric is strict in two ways:

- **Same problem, not same topic.** A page gets 0 when it addresses a different problem (`wrong_problem`), changes the conditions (`condition_mismatch`), answers a different target (`target_mismatch`), or only shares the topic or keywords (`topic_only`).
- **Correctness-aware.** A page whose central mathematics is wrong gets 0, even when it addresses the right problem. A correct final answer reached through invalid reasoning is also 0.

|               |                                                                                  |
|---------------|----------------------------------------------------------------------------------|
| Task          | retrieval: a math question → the web pages that answer it                        |
| Domain        | mathematics, web pages                                                           |
| Language      | English                                                                          |
| Corpus        | 3,525,546 documents, shared by both splits                                       |
| Training data | 1,853,389 queries, 43,506,169 judged (query, document) pairs                     |
| Test data     | 6,173 queries, 83,833 relevant (query, document) pairs                           |
| Labels        | 0 / 1 / 2. Training: GPT-5.6-Sol, or an 8B model distilled from it. Test: two LLM judges and an arbiter |
| Metrics       | nDCG@10, Recall@100                                                              |
| Source        | web pages from the L2 tier of [UltraData-Math](https://huggingface.co/datasets/openbmb/UltraData-Math) |

## Quick start

The dataset is gated with automatic approval: accept the conditions on its [Hugging Face page](https://huggingface.co/datasets/oklenAI/MathPinpoint), then log in locally with `hf auth login`.

```python
from datasets import load_dataset

repo = "oklenAI/MathPinpoint"

# Corpus (shared by both splits) and test split
corpus = load_dataset(repo, "corpus", split="train")    # did, text, n_chars
queries = load_dataset(repo, "queries", split="test")   # qid, text
qrels = load_dataset(repo, "qrels", split="test")       # qid, did, label

# Training data
train_queries = load_dataset(repo, "queries", split="train")
judgments = load_dataset(repo, "judgments", split="train", streaming=True)  # 43.5M rows
```

Pass `revision="v2.0"` to `load_dataset` to pin this release.

## Evaluation

Search the whole corpus for every test query and score the ranking against `qrels`. The function below computes the two metrics reported in this README. Every relevant document has label 2, so graded and binary nDCG coincide.

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

Documents that were never judged count as not relevant, so a retriever that ranks differently from the systems that built the test pool is penalized for the relevant documents only it finds. See [limitations](docs/limitations.md#test-split).

## Dataset structure

| config | split | rows | content |
|---|---|---:|---|
| `corpus` | train | 3,525,546 | deduplicated mathematical documents, shared by both splits |
| `queries` | train | 1,853,389 | query text, the page it was extracted from, extraction metadata |
| `judgments` | train | 43,506,169 | one row per judged (query, document) pair: its rank in each retrieval route, label, which judge produced it, and raw score where available |
| `queries` | test | 6,173 | held-out test queries |
| `qrels` | test | 83,833 | the relevant documents of each test query |

Hugging Face names the only split of a config `train`, so the corpus is split `train`. Column-level schemas are in [`docs/schema.md`](docs/schema.md). The test split is described in [`docs/test_split.md`](docs/test_split.md).

## How the labels were made

1. **Documents.** Mathematical web pages go through a content extractor, are normalized, and are deduplicated with MinHash. Documents whose extraction degenerated, and documents that bundle five or more problems, are removed. See [`docs/corpus.md`](docs/corpus.md).
2. **Queries.** One query per page, produced with a minimal-edit prompt: if the page contains a question somebody actually asked, that question *is* the query, copied with as few changes as possible. The prompt is [`prompts/query_gen_prompt.txt`](prompts/query_gen_prompt.txt). Queries are then deduplicated and filtered, leakage against evaluation sets is removed, and queries that two LLM judges both find not self-contained are dropped. See [`docs/queries.md`](docs/queries.md).
3. **Candidates.**
   - Three retrieval routes each contribute their top 100: dense (Qwen3-Embedding-4B), BM25, and word 5-gram. They are combined with weighted reciprocal rank fusion (dense weight 2, k = 60).
   - A query's candidates are its dense top 20 and its fused top 20. On average, 4.55 candidates per query come from the fused top 20 alone.
   - See [`docs/judging.md`](docs/judging.md).
4. **Judging.**
   - The LLM judge (GPT-5.6-Sol) labels, under `prompts/judge_prompt.md`: dense ranks 1–5; within dense ranks 6–20, the top 5 as ordered by a zero-shot reranker and every document longer than 8,144 tokens; and, among the candidates from the fused top 20 alone, documents longer than 8,144 tokens at fused positions 1–5.
   - An 8B relevance model, trained on the LLM labels, judges everything else. It also scores the reranker's picks, so those pairs carry both labels.
   - Every row records which judge produced its label, and the 8B's raw score is kept wherever it exists. See [`docs/judging.md`](docs/judging.md).

## Training a retriever on MathPinpoint

Qwen3-Embedding-0.6B was fine-tuned on training rows built from this release and compared with the same model before fine-tuning, on the test split (6,173 queries).

| metric | before | after | difference [95% CI] |
|---|---:|---:|---:|
| nDCG@10 | 0.441 | 0.477 | +0.036 [+0.028, +0.045] |
| R@100 | 0.561 | 0.568 | +0.007 [−0.001, +0.016] |

Both metrics count unjudged documents as not relevant, and the test pool covers the two models unequally: 66% of the top 10 of the model before fine-tuning is judged, but only 35% of the fine-tuned model's. Ranking only the judged documents instead (condensed nDCG@10) gives 0.486 before and 0.617 after, a difference of +0.131 [+0.123, +0.139]. On a 500-query sample where both models' top 10 was judged in full, the nDCG@10 difference was +0.114, close to the condensed value.

<details>
  <summary>Training rows, training and evaluation setup (click to unfold)</summary>

- **Training rows.** Each query gives one row:
  - the positive is drawn at random from the query's score-2 candidates;
  - the three score-0 candidates with the best dense rank are the hard negatives; candidates outside the dense top 100 come last;
  - queries with fewer than three score-0 candidates are skipped.

  Labels from both judges are used:
  - Where both judges labeled a pair, the LLM label is used.
  - A pair labeled only by the 8B model counts as score 2 if its raw score is at least 1.5, and as score 0 if it is below 0.5.
  - Rows from the source-page check are not used, and pairs marked not evaluable are dropped. A query's own source page that retrieval also returned is a candidate like any other; it is the positive in 46.8% of rows.

  Documents are cut to 4,000 characters and queries to 2,000. This gives 1,225,057 rows.
- **Training.**
  - Loss: multiple-negatives ranking loss, using in-batch negatives plus the three hard negatives, and a Matryoshka loss over 768, 512, 256 and 128 dimensions.
  - Batch: effective batch 512 (8 GPUs × 64, GradCache with mini-batch 32).
  - Schedule: learning rate 2e-5 with 10% warmup, sequence length 512, 1,000 steps. A run therefore sees 512,000 of the 1,225,057 rows.
  - Runs: three, with seeds 42, 43 and 44. Per-query scores are averaged over the three.
- **Evaluation.**
  - Search runs over all 3,525,546 corpus documents, with last-token pooling at 512 tokens. Each query is prefixed with the instruction below; documents get no prefix.

    ```
    Instruct: Given a web search query, retrieve relevant passages that answer the query
    Query:
    ```

  - Relevant documents are the test split's `qrels`: documents judged score 2, which answer the same problem correctly. The metrics are the ones `evaluate` in [Evaluation](#evaluation) computes.
  - The test queries are held out: no training query matches one verbatim, at word 5-gram Jaccard ≥ 0.50 (checked exhaustively), or as a semantic duplicate judged to be the same problem.
  - Intervals come from a paired bootstrap over queries, with 10,000 resamples.

</details>

## Documentation

| Documentation | |
|---|---|
| 📚 [Corpus](docs/corpus.md) | source pages, extraction, question removal, dedup, degenerate and multi-problem documents, document ids |
| ❓ [Queries](docs/queries.md) | query extraction, dedup, leakage removal, source-page check, self-containedness check |
| ⚖️ [Candidates and judging](docs/judging.md) | candidates, who judged what, the rubric, the LLM judge, the 8B model |
| 🧪 [Test split](docs/test_split.md) | test queries and how their relevant documents were judged |
| 🗂️ [Schema](docs/schema.md) | release layout and every column |
| ⚠️ [Limitations](docs/limitations.md) | what the labels do and do not tell you |

## Prompts and schemas

`prompts/` holds the exact texts that produced the data. [`prompts/SHA256SUMS`](prompts/SHA256SUMS) pins them. The judge prompt's `guideline_version` string is fixed in the output schema, so it does not identify which prompt text produced a label; the SHA-256 does.

| file | sha256 |
|---|---|
| `judge_prompt.md` | `031ac72b649ae979…` |
| `query_gen_prompt.txt` | `d75d30a5b8cb0b03…` |
| `query_check_prompt.md` | `5de21b985641bc6a…` |

## Provenance and license

- **Data.** Everything in the Hugging Face dataset (corpus, queries, judgments and test qrels) is released under [CC0 1.0](https://creativecommons.org/publicdomain/zero/1.0/).
- **Code.** The prompts, schemas and any code in this repository are released under the [Apache License 2.0](https://www.apache.org/licenses/LICENSE-2.0).

The documents are cleaned extractions of web pages from the L2 tier of [UltraData-Math](https://huggingface.co/datasets/openbmb/UltraData-Math), which is released under Apache-2.0; we used a copy of it that adds a difficulty score to each page (see [corpus](docs/corpus.md)). Rights in the original web pages stay with their owners.

## Citing

If you use MathPinpoint, please cite it. Its documents come from UltraData-Math, so please cite that dataset as well.

<details>
  <summary>BibTeX citation (click to unfold)</summary>

```bibtex
@misc{mathpinpoint2026,
  title     = {{MathPinpoint}: Training Data and a Test Set for Problem-Level Math Retrieval},
  author    = {oklen},
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

## Dataset statistics

<details>
  <summary>Dataset statistics (click to unfold)</summary>

Computed from the released tables. Lengths are in characters (Unicode code points).

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
