# MathPinpoint

> **Status: draft.** The data release is being assembled. Every number marked **TBD** gets filled in from the final assembly; nothing here is carried over from an estimate.

MathPinpoint is training data for **problem-level math retrieval**. Given a math question, the task is to find the web page that solves *that* problem, and solves it correctly. Each (query, page) pair carries a graded relevance label (0 / 1 / 2) under one written rubric, [`prompts/judge_prompt.md`](prompts/judge_prompt.md). The rubric is strict in two ways:

- **Same problem, not same topic.** A page gets 0 when it addresses a different problem (`wrong_problem`), changes the conditions (`condition_mismatch`), answers a different target (`target_mismatch`), or only shares the topic or keywords (`topic_only`).
- **Correctness-aware.** A page whose central mathematics is wrong gets 0, even when it addresses the right problem. A correct final answer reached through invalid reasoning is also 0.

## What the release contains

| config | rows | content |
|---|---:|---|
| `queries` | 2,032,033 | query text, the page it was extracted from, extraction metadata |
| `corpus` | 3,676,820 | deduplicated mathematical documents |
| `judgments` | 41,171,995 | one row per judged (query, document) pair: retrieval rank, label, which judge produced it, and raw score where available |

Column-level schemas are in [`docs/schema.md`](docs/schema.md).

## How the labels were made

1. **Documents.** Mathematical web pages go through a content extractor, are normalized, and are deduplicated with MinHash. See [`docs/corpus.md`](docs/corpus.md).
2. **Queries.** One query per page, produced with a minimal-edit prompt: if the page contains a question somebody actually asked, that question *is* the query, copied with as few changes as possible. The prompt is [`prompts/query_gen_prompt.txt`](prompts/query_gen_prompt.txt). Queries are then deduplicated and filtered, and leakage against evaluation sets is removed. See [`docs/queries.md`](docs/queries.md).
3. **Candidates.** The top 20 documents from a dense retriever, for every query.
4. **Judging.**
   - The LLM judge (GPT-5.6-Sol) labels three groups under `prompts/judge_prompt.md`: dense ranks 1–5; the top 5 within ranks 6–20 as ordered by a zero-shot reranker; and documents in ranks 6–20 too long for that reranker.
   - An 8B relevance model trained on the LLM labels scores every short document in ranks 6–20. That includes the reranker's picks, so those pairs carry both labels.
   - Every row records which judge produced its label, and the 8B's raw score is kept wherever it exists. See [`docs/judging.md`](docs/judging.md).

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
