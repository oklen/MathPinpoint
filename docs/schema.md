# Schema

The release has four configs. Ids are content hashes, so anyone holding the text can recompute them; see [corpus](corpus.md), [queries](queries.md) and [test split](test_split.md).

| config | split | rows |
|---|---|---:|
| `corpus` | train | 3,676,820 |
| `queries` | train | 2,032,033 |
| `queries` | test | 2,284 |
| `judgments` | train | 51,158,695 |
| `qrels` | test | 26,948 |

The corpus serves both splits. Hugging Face names the only split of a config `train`.

## `corpus`

| column | type | meaning |
|---|---|---|
| `did` | string | `"d_" + sha256(lowercase(normalized_text))[:24]` |
| `text` | string | normalized document text, with the source page's own question removed |
| `n_chars` | int32 | length of `text` |
| `multi_problem` | bool | the document contains five or more separate problems |

## `queries`

| column | type | meaning |
|---|---|---|
| `qid` | string | training queries: `"p_" + sha256(raw_page_text)[:24]`, the id of the source page the query was extracted from (raw upstream `content`, no normalization). Test queries: `"q_" + sha256(lowercase(normalized_text))[:24]`, a hash of the query text normalized as for document ids |
| `text` | string | the query |
| `source_did` | string | the document the query was extracted from, with its own question removed. Null for the 307,715 training queries (15.1%) whose source page was removed as a near-duplicate, and for all test queries |
| `from_page_question` | bool | `true` if the query is the page's own question, minimally edited; `false` if the model composed it. Null for test queries |

## `judgments`

One row per judged (training query, document) pair.

| column | type | meaning |
|---|---|---|
| `qid`, `did` | string | the pair |
| `dense_rank` | int8 | rank in the dense route's top 100; null if the document is not in it |
| `bm25_rank`, `ngram5_rank` | int8 | rank in the BM25 or word 5-gram route's top 100; null if the document is not in it |
| `fused_rank` | int8 | position in the fused top 20; null if the document is not in it |
| `rrf_score` | float64 | the fusion score 2 / (60 + `dense_rank`) + 1 / (60 + `bm25_rank`) + 1 / (60 + `ngram5_rank`), where a missing route adds 0; null if no route retrieved the document |
| `stage` | string | the group of candidates the pair belongs to; see below |
| `label` | int8 | 0 / 1 / 2; null when the query was judged unevaluable |
| `query_evaluable` | bool | false when the LLM judge found the query itself uninterpretable |
| `is_source_page` | bool | the document is the query's own source page |
| `judge` | string | `llm` or `rm8b`, whichever produced `label` |
| `rm8b_score` | float32 | the 8B model's raw score, on every pair it scored, including pairs whose `label` comes from the LLM |
| `dense_score` | float32 | cosine similarity from the dense retriever, for documents in its top 100 |
| `reranker_score` | float32 | zero-shot score from Qwen3-Reranker-4B (logit of yes minus no), for dense ranks 6–20; null elsewhere |
| `answer_validity`, `reason_codes`, `confidence`, `rationale` | | the LLM judge's structured output; null on 8B rows |
| `rubric_sha256` | string | SHA-256 of the judge prompt, on LLM-labeled candidates; null on 8B rows and on `source_page` rows |

| `stage` | pairs | judge |
|---|---|---|
| `dense_top5` | dense ranks 1–5 | LLM |
| `rerank_top5` | dense ranks 6–20, the reranker's top 5 | LLM, also scored by the 8B |
| `long_doc` | dense ranks 6–20, longer than 8,144 tokens | LLM |
| `short_doc` | dense ranks 6–20, all other documents | 8B |
| `fused_long_top5` | fused-only, longer than 8,144 tokens, fused positions 1–5 | LLM |
| `fused_long` | fused-only, longer than 8,144 tokens, fused positions 6–20 | 8B |
| `fused_short` | fused-only, up to 8,144 tokens | 8B |
| `source_page` | the query's own source page, labeled before retrieval, when no stage above covers it | LLM |

*Fused-only* means in the fused top 20 but not in the dense top 20; see [judging](judging.md).

**Rules.**

- When both judges labeled a pair, `label` and `judge` come from the LLM, and `rm8b_score` is kept.
- An 8B `label` applies the default cuts 0.5 and 1.5 to `rm8b_score`. Use the raw score to pick your own threshold.
- No training `qid` overlaps the test queries: not verbatim, not at word Jaccard ≥ 0.50, and not as a semantic duplicate judged to be the same problem.

## `qrels`

The relevant documents of each test query.

| column | type | meaning |
|---|---|---|
| `qid` | string | the test query |
| `did` | string | a relevant document |
| `label` | int8 | always 2: both judges, or the arbiter, found that the document answers the same problem correctly |
