# Schema (proposal)

> This is the layout of the assembled v2 build. It stays a proposal until the release is published. `queries` and `corpus` are byte-identical to v1.

The release has three configs. Ids are content hashes, so anyone holding the text can recompute them; see [corpus](corpus.md) and [queries](queries.md).

## `corpus`

| column | type | meaning |
|---|---|---|
| `did` | string | `"d_" + sha256(lowercase(normalized_text))[:24]` |
| `text` | string | normalized document text, with the source page's own question removed |
| `n_chars` | int32 | length of `text` |
| `multi_problem` | bool | the document contains five or more separate problems |

Rows: 3,676,820.

## `queries`

| column | type | meaning |
|---|---|---|
| `qid` | string | `"p_" + sha256(raw_page_text)[:24]`: the id of the source page the query was extracted from (raw upstream `content`, no normalization) |
| `text` | string | the query |
| `source_did` | string | the document the query was extracted from, with its own question removed. Null for 307,715 queries (15.1%) whose source page was later removed as a near-duplicate |
| `from_page_question` | bool | `true` if the query is the page's own question, minimally edited; `false` if the model composed it |

Rows: 2,032,033.

## `judgments`

One row per judged (query, document) pair.

| column | type | meaning |
|---|---|---|
| `qid`, `did` | string | the pair |
| `dense_rank` | int8 | rank in the dense route's top 100; null if the document is not in it. v1 recorded ranks 1–20 only. v2 fills ranks 21–100, which adds a rank to 190,540 source-page rows and covers the new candidates; no v1 value changes |
| `bm25_rank`, `ngram5_rank` | int8 | rank in the BM25 or word 5-gram route's top 100; null if the document is not in it |
| `fused_rank` | int8 | position in the fused top 20; null if the document is not in it |
| `rrf_score` | float64 | the fusion score 2 / (60 + `dense_rank`) + 1 / (60 + `bm25_rank`) + 1 / (60 + `ngram5_rank`), where a missing route adds 0; null if no route retrieved the document |
| `stage` | string | for the dense top 20: `dense_top5`, `rerank_top5`, `long_doc`, `short_doc` or `source_page`. For new candidates: `fused_long_top5` (longer than 8,144 tokens, fused positions 1–5), `fused_long` (longer than 8,144 tokens, positions 6–20) or `fused_short`. See [judging](judging.md) |
| `label` | int8 | 0 / 1 / 2; null when the query was judged unevaluable |
| `query_evaluable` | bool | false when the LLM judge found the query itself uninterpretable |
| `is_source_page` | bool | the document is the query's own source page |
| `judge` | string | `llm` or `rm8b`, whichever produced `label` |
| `rm8b_score` | float32 | the 8B model's raw score. Present on every short document in dense ranks 6–20 and on every new candidate except the 162,464 that the LLM judged at fused positions 1–5, including pairs whose `label` comes from the LLM |
| `dense_score` | float32 | cosine similarity from the dense retriever, for documents in its top 100 |
| `reranker_score` | float32 | zero-shot score from Qwen3-Reranker-4B (logit of yes minus no), for dense ranks 6–20; null for new candidates |
| `answer_validity`, `reason_codes`, `confidence`, `rationale` | | the LLM judge's structured output; null on 8B-only rows |
| `rubric_sha256` | string | SHA-256 of the prompt text that produced an LLM label; null on 8B-only rows |
| `label_round` | string | the round that produced `label`: `v1`; `v2_gap`, for pairs that v1 left without a usable LLM label; or `v2_fused`, for new candidates |

Rows: 51,158,695, of which 41,140,369 have `label_round` `v1`, 68,854 `v2_gap` and 9,949,472 `v2_fused`.

**Rules.**

- When both judges labeled a pair, `label` and `judge` come from the LLM, and `rm8b_score` is kept.
- An 8B `label` applies the default cuts 0.5 and 1.5 to `rm8b_score`. Use the raw score to pick your own threshold.
- No `qid` overlaps the evaluation queries: not verbatim, not at word Jaccard ≥ 0.50, and not as a semantic duplicate judged to be the same problem.
