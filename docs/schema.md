# Schema (proposal)

> This is the layout of the assembled v1 build. It stays a proposal until the release is published.

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
| `dense_rank` | int8 | 1–20; null for a source page that was not retrieved |
| `stage` | string | `dense_top5`, `rerank_top5`, `long_doc`, `short_doc` or `source_page` (see [judging](judging.md)) |
| `label` | int8 | 0 / 1 / 2; null when the query was judged unevaluable |
| `query_evaluable` | bool | false when the LLM judge found the query itself uninterpretable |
| `is_source_page` | bool | the document is the query's own source page |
| `judge` | string | `llm` or `rm8b`, whichever produced `label` |
| `rm8b_score` | float32 | the 8B model's raw score. Present on every short-document pair, including those whose `label` comes from the LLM |
| `dense_score` | float32 | cosine similarity from the dense retriever |
| `reranker_score` | float32 | zero-shot score from Qwen3-Reranker-4B (logit of yes minus no), for ranks 6–20 |
| `answer_validity`, `reason_codes`, `confidence`, `rationale` | | the LLM judge's structured output; null on 8B-only rows |
| `rubric_sha256` | string | SHA-256 of the prompt text that produced an LLM label; null on 8B-only rows |

Rows: 41,171,995.

**Rules.**

- When both judges labeled a pair, `label` and `judge` come from the LLM, and `rm8b_score` is kept.
- An 8B `label` applies the default cuts 0.5 and 1.5 to `rm8b_score`. Use the raw score to pick your own threshold.
- No `qid` overlaps the evaluation queries: not verbatim, not at word Jaccard ≥ 0.50, and not as a semantic duplicate judged to be the same problem.
