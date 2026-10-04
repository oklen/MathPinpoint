# MathPinpoint

> **Status: v2, private preview.** Every count below comes from the assembled v2 table. For what a retriever fine-tuned on it gains, see [Training a retriever on MathPinpoint](#training-a-retriever-on-mathpinpoint).

MathPinpoint is training data for **problem-level math retrieval**. Given a math question, the task is to find the web page that solves *that* problem, and solves it correctly. Each (query, page) pair carries a relevance label of 0, 1 or 2 under one written rubric, [`prompts/judge_prompt.md`](prompts/judge_prompt.md). The rubric is strict in two ways:

- **Same problem, not same topic.** A page gets 0 when it addresses a different problem (`wrong_problem`), changes the conditions (`condition_mismatch`), answers a different target (`target_mismatch`), or only shares the topic or keywords (`topic_only`).
- **Correctness-aware.** A page whose central mathematics is wrong gets 0, even when it addresses the right problem. A correct final answer reached through invalid reasoning is also 0.

## What the release contains

| config | rows | content |
|---|---:|---|
| `queries` | 2,032,033 | query text, the page it was extracted from, extraction metadata |
| `corpus` | 3,676,820 | deduplicated mathematical documents |
| `judgments` | 51,158,695 | one row per judged (query, document) pair: its rank in each retrieval route, label, which judge produced it, and raw score where available |

Column-level schemas are in [`docs/schema.md`](docs/schema.md).

## How the labels were made

1. **Documents.** Mathematical web pages go through a content extractor, are normalized, and are deduplicated with MinHash. See [`docs/corpus.md`](docs/corpus.md).
2. **Queries.** One query per page, produced with a minimal-edit prompt: if the page contains a question somebody actually asked, that question *is* the query, copied with as few changes as possible. The prompt is [`prompts/query_gen_prompt.txt`](prompts/query_gen_prompt.txt). Queries are then deduplicated and filtered, and leakage against evaluation sets is removed. See [`docs/queries.md`](docs/queries.md).
3. **Candidates.**
   - Three retrieval routes each contribute their top 100: dense (Qwen3-Embedding-4B), BM25, and word 5-gram. They are combined with weighted reciprocal rank fusion (dense weight 2, k = 60).
   - A query's candidates are its fused top 20 together with its dense top 20.
   - v1 judged the dense top 20 only, which departed from this design. v2 adds the 9,949,472 fused candidates that the dense top 20 missed, and judges the 37,228 dense top-20 pairs that v1 had left without a judgment.
   - See [`docs/judging.md`](docs/judging.md).
4. **Judging.**
   - The LLM judge (GPT-5.6-Sol) labels, under `prompts/judge_prompt.md`: dense ranks 1–5; the top 5 within dense ranks 6–20 as ordered by a zero-shot reranker; documents in dense ranks 6–20 too long for that reranker; and, among the new candidates, documents too long for it at fused positions 1–5.
   - An 8B relevance model trained on the LLM labels scores every short document in dense ranks 6–20. That includes the reranker's picks, so those pairs carry both labels. In v2 it also labels every new candidate that the LLM judge does not.
   - Every row records which judge produced its label and in which round (`label_round`), and the 8B's raw score is kept wherever it exists. See [`docs/judging.md`](docs/judging.md).

## Training a retriever on MathPinpoint

Qwen3-Embedding-0.6B was fine-tuned on training rows built from v2 of this release and compared with the same model before fine-tuning, on a held-out test set of 2,303 queries.

| metric | before | after | difference [95% CI] |
|---|---:|---:|---:|
| nDCG@10 | 0.403 | 0.451 | +0.048 [+0.035, +0.061] |
| R@100 | 0.470 | 0.490 | +0.020 [+0.009, +0.031] |

- **Training rows.** Each query gives one row:
  - the positive is drawn at random from the query's score-2 candidates;
  - the three score-0 candidates with the best dense rank are the hard negatives; candidates outside the dense top 100 come last;
  - queries with fewer than three score-0 candidates are skipped.

  Labels from both judges are used:
  - Where both judges labeled a pair, the LLM label is used.
  - A pair labeled only by the 8B model counts as score 2 if its raw score is at least 1.5, and as score 0 if it is below 0.5.
  - A query's own source page is not used, and pairs marked not evaluable are dropped.

  Documents are cut to 4,000 characters and queries to 2,000. This gives 1,403,375 rows.
- **Training.**
  - Loss: multiple-negatives ranking loss, using in-batch negatives plus the three hard negatives, and a Matryoshka loss over 768, 512, 256 and 128 dimensions.
  - Batch: effective batch 512 (8 GPUs × 64, GradCache with mini-batch 32).
  - Schedule: learning rate 2e-5 with 10% warmup, sequence length 512, 1,000 steps. A run therefore sees 512,000 of the 1,403,375 rows.
  - Runs: three, with seeds 42, 43 and 44. Per-query scores are averaged over the three.
- **Evaluation.**
  - Search runs over all 3,676,820 corpus documents, with last-token pooling at 512 tokens. Each query is prefixed with the instruction below; documents get no prefix.

    ```
    Instruct: Given a web search query, retrieve relevant passages that answer the query
    Query:
    ```

  - The test set is held out: none of its queries matches a training query after NFKC normalization, whitespace folding and lowercasing.
  - Only documents judged score 2, which answer the same problem correctly, count as relevant.
  - Intervals come from a paired bootstrap over queries, with 10,000 resamples.

## Documentation

| page | covers |
|---|---|
| [`docs/corpus.md`](docs/corpus.md) | source pages, extraction, question removal, dedup, document ids |
| [`docs/queries.md`](docs/queries.md) | query extraction, dedup, leakage removal, source-page check |
| [`docs/judging.md`](docs/judging.md) | candidates, who judged what, the rubric, the LLM judge, the 8B model |
| [`docs/schema.md`](docs/schema.md) | release layout (proposal) |
| [`docs/limitations.md`](docs/limitations.md) | what the labels do and do not tell you |

## Prompts and schemas

`prompts/` holds the exact texts that produced the data. [`prompts/SHA256SUMS`](prompts/SHA256SUMS) pins them. The judge prompt's `guideline_version` string is fixed in the output schema, so it does not identify which prompt text produced a label; the SHA-256 does.

| file | sha256 |
|---|---|
| `judge_prompt.md` | `031ac72b649ae979…` |
| `query_gen_prompt.txt` | `d75d30a5b8cb0b03…` |

## Provenance and license

TBD before any public release. The source pages come from a gated upstream dataset that declares no license, and the documents here are derived from that content.
