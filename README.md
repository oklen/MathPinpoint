# MathPinpoint

> **Status: v2.** Every count below comes from the released tables. For what a retriever fine-tuned on it gains, see [Training a retriever on MathPinpoint](#training-a-retriever-on-mathpinpoint).

MathPinpoint is training data for **problem-level math retrieval**. Given a math question, the task is to find the web page that solves *that* problem, and solves it correctly. Each (query, page) pair carries a relevance label of 0, 1 or 2 under one written rubric, [`prompts/judge_prompt.md`](prompts/judge_prompt.md). The rubric is strict in two ways:

- **Same problem, not same topic.** A page gets 0 when it addresses a different problem (`wrong_problem`), changes the conditions (`condition_mismatch`), answers a different target (`target_mismatch`), or only shares the topic or keywords (`topic_only`).
- **Correctness-aware.** A page whose central mathematics is wrong gets 0, even when it addresses the right problem. A correct final answer reached through invalid reasoning is also 0.

## What the release contains

| config | split | rows | content |
|---|---|---:|---|
| `corpus` | train | 3,525,546 | deduplicated mathematical documents, shared by both splits |
| `queries` | train | 1,853,389 | query text, the page it was extracted from, extraction metadata |
| `judgments` | train | 43,506,169 | one row per judged (query, document) pair: its rank in each retrieval route, label, which judge produced it, and raw score where available |
| `queries` | test | 6,173 | held-out test queries |
| `qrels` | test | 74,519 | the relevant documents of each test query |

Column-level schemas are in [`docs/schema.md`](docs/schema.md). The test split is described in [`docs/test_split.md`](docs/test_split.md).

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
| nDCG@10 | 0.436 | 0.478 | +0.042 [+0.034, +0.051] |
| R@100 | 0.560 | 0.577 | +0.016 [+0.008, +0.025] |

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

  - Relevant documents are the test split's `qrels`: documents judged score 2, which answer the same problem correctly.
  - The test queries are held out: no training query matches one verbatim, at word 5-gram Jaccard ≥ 0.50 (checked exhaustively), or as a semantic duplicate judged to be the same problem.
  - Intervals come from a paired bootstrap over queries, with 10,000 resamples.

## Documentation

| page | covers |
|---|---|
| [`docs/corpus.md`](docs/corpus.md) | source pages, extraction, question removal, dedup, degenerate and multi-problem documents, document ids |
| [`docs/queries.md`](docs/queries.md) | query extraction, dedup, leakage removal, source-page check, self-containedness check |
| [`docs/judging.md`](docs/judging.md) | candidates, who judged what, the rubric, the LLM judge, the 8B model |
| [`docs/test_split.md`](docs/test_split.md) | test queries and how their relevant documents were judged |
| [`docs/schema.md`](docs/schema.md) | release layout |
| [`docs/limitations.md`](docs/limitations.md) | what the labels do and do not tell you |

## Prompts and schemas

`prompts/` holds the exact texts that produced the data. [`prompts/SHA256SUMS`](prompts/SHA256SUMS) pins them. The judge prompt's `guideline_version` string is fixed in the output schema, so it does not identify which prompt text produced a label; the SHA-256 does.

| file | sha256 |
|---|---|
| `judge_prompt.md` | `031ac72b649ae979…` |
| `query_gen_prompt.txt` | `d75d30a5b8cb0b03…` |
| `query_check_prompt.md` | `5de21b985641bc6a…` |

## Provenance and license

TBD before any public release. The source pages come from a gated upstream dataset that declares no license, and the documents here are derived from that content.
