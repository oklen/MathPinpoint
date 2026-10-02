# Candidates and judging

The candidate design fuses three retrieval routes. **v1 departed from that design: its candidates came from the dense route alone**, the top 20 per query. v2 adds the documents that the BM25 and word 5-gram routes bring into the fused top 20, and judges the pairs v1 had left without a judgment; see [v2](#v2-adding-the-missing-routes). Every v1 label carries over, except for 31,626 reranker picks that had no usable LLM label in v1: v2 gives them one and keeps their 8B score.

Every candidate is judged 0 / 1 / 2 under one rubric, by one of two judges:

- an LLM judge, for the candidates that most need it;
- an 8B relevance model distilled from the LLM judge, for the rest.

Every label records which judge produced it.

## The candidate design

Three routes each contribute their top 100 documents:

- **Dense:** Qwen3-Embedding-4B, configured as described under [v1](#v1-the-dense-route-only).
- **BM25:** k1 = 1.2, b = 0.75, over lowercased `[a-z0-9]+` tokens.
- **Word 5-gram overlap.**

BM25 and 5-gram statistics are computed over the 4,340,031 documents from before near-duplicate removal. As with the dense route, documents removed as near-duplicates are excluded at search time.

The routes are combined with **weighted reciprocal rank fusion**: score(d) = Σ_r w_r / (k + rank_r(d)), with weight 2 for the dense route, weight 1 for BM25 and for 5-gram, and k = 60. Ties are broken by the document's best rank across the routes. The original design fused by best rank alone; weighted RRF replaced it after the probe below.

**Probe.** 1,000 queries; the union of the three routes' top 100 was fully judged by the LLM judge. The table counts positives (score 2) per query:

| order | top 5 | top 10 | top 20 |
|---|---:|---:|---:|
| dense only | 1.61 | 2.56 | 3.99 |
| best rank across routes (original design) | 1.78 | 2.70 | 4.03 |
| weighted RRF, dense ×2, k = 60 | 1.92 | 2.97 | 4.56 |

Under the best-rank order, documents that BM25 ranked high but that were not relevant pushed good dense results down. Weighting the dense route fixes this: 97.8% of the fused top 5 stays inside the dense top 5, while the positives that only the other routes find are still added.

## v1: the dense route only

In v1, a reranker orders dense ranks 6–20, and each group goes to a different judge:

```
query ── dense retrieval (Qwen3-Embedding-4B) ──▶ top 20
   ranks 1–5 ───────────────────────────────────────────────▶ LLM judge
   ranks 6–20 ── rerank (Qwen3-Reranker-4B, zero-shot) ─▶ top 5 ─▶ LLM judge
              ├─ documents longer than 8,144 tokens ────────▶ LLM judge
              └─ every document up to 8,144 tokens ─────────▶ 8B relevance model
```

- **Retriever: Qwen3-Embedding-4B.**
  - Documents: last-token pooling, up to 32,768 tokens, no instruction.
  - Queries: up to 512 tokens, with the prefix `Instruct: Given a math question, retrieve documents that answer it\nQuery: `.
  - Vectors are L2-normalized.
- **Search.** The index holds the 4,340,031 documents from before near-duplicate removal. Documents removed as near-duplicates are masked out of the index before search, so the stored top 100 is over the deduplicated corpus. Search is an exact inner product. The top 20 per query are judged.
- **Sanity check.** The LLM judge's score-2 rate falls steadily over dense ranks 1–5: 59.2 / 36.0 / 26.6 / 22.2 / 19.6%.

## v2: adding the missing routes

**New candidates** are the documents in a query's fused top 20 that v1 did not already judge. That excludes the dense top 20 and the query's own source page. There are 9,949,472 of them, 4.90 per query. 1,087,062 (10.9%) are longer than 8,144 tokens, and 1,002,885 (10.1%) are not in the dense top 100 at all: only BM25 or 5-gram found them.

- The 162,464 long documents at fused positions 1–5 go to the **LLM judge**, which reads up to 100,000 characters.
- Every other new candidate goes to the **8B relevance model**: 8,862,410 documents up to 8,144 tokens and 924,598 long documents at fused positions 6–20.
  - 240 of those long documents come from a 500-query trial in which the LLM judged every long new candidate. They carry both labels.
- The zero-shot reranker is not used for new candidates. In v1 its only job was to pick which short documents the LLM judge would see, and short new candidates go to the 8B model instead.
- Token counts use the same table as v1's long-document threshold.

**v1's gaps.** 37,228 pairs in v1's dense top 20 had no judgment, because their LLM call never produced a valid result: 35,477 in ranks 1–5 and 1,751 long documents in ranks 6–20. A further 31,626 reranker picks had no usable LLM label. v2 sends all of them to the LLM judge. v1's own count of unjudged dense top-20 pairs, 40,967, also included 3,739 query source pages that already carried a source-page label.

After v2, every pair in a query's dense top 20 and fused top 20 has a judgment row.

## Who judged which pairs

| Candidates | Judge | Pairs sent | Judged |
|---|---|---:|---:|
| Dense ranks 1–5 | LLM, reading up to 30,000 characters | 10,160,165 | 10,120,955 (99.61%) |
| Ranks 6–20: the top 5 of the 4B reranker's order | LLM, reading up to 30,000 characters | 10,154,925 (5,299,442 in three rounds + 4,855,483 in the final round) | 10,122,745 (99.68%) |
| Ranks 6–20: documents longer than 8,144 tokens | LLM, reading up to 100,000 characters | 2,095,018 (884,238 queries) | 2,093,238 (99.92%) |
| Ranks 6–20: all documents up to 8,144 tokens | 8B relevance model | 28,411,189 | 28,411,189 |
| Each query's own source page (before retrieval) | LLM | 2,901,088 | see [queries](queries.md) |

The table above is v1. In v1, 37,228 pairs in the dense top 20 had no judgment; v2 judged them, together with the following:

| v2 candidates | Judge | Pairs |
|---|---|---:|
| Dense ranks 1–5 with no v1 judgment | LLM, reading up to 30,000 characters | 35,477 |
| Long documents in dense ranks 6–20 with no v1 judgment | LLM, reading up to 100,000 characters | 1,751 |
| Reranker picks with no usable v1 LLM label | LLM, reading up to 30,000 characters | 31,626 |
| New candidates longer than 8,144 tokens at fused positions 1–5 | LLM, reading up to 100,000 characters | 162,464 |
| All other new candidates | 8B relevance model | 9,787,008 |

Every v2 request produced a valid judgment. In 6 requests (60 pairs) the LLM wrote LaTeX backslashes that are not valid JSON escapes. Those backslashes were doubled before parsing, which cannot change a label field; `json_escape_repaired` marks the 10 affected rows.

- **The reranker picks.** Qwen3-Reranker-4B scores ranks 6–20 zero-shot, with the instruction to judge whether the document answers "the same problem with the same particulars". Its top 5 go to the LLM judge. It was chosen on a 196-query probe whose three-route candidate pool was fully judged: there, its AUC for separating score 2 from score 0 was 0.921, against 0.599 for the fused retrieval order. Retrieval is good at getting relevant documents into the pool but poor at ordering them within it; the reranker fixes the order.
- **The long-document threshold.** The reranker reads at most 8,144 tokens of a document. Longer documents therefore skip the reranker and go straight to the LLM judge, which reads up to 100,000 characters.
- **Overlap.** 10,192,093 pairs carry both an LLM label and an 8B score: 10,154,365 reranker picks, 25,689 long-document pairs that the reranker had scored before long documents were routed to the LLM, 11,799 source pages, and the 240 trial pairs among v2's new candidates. In the table the LLM label wins and the 8B score is kept in its own column; see [schema](schema.md). The overlap is also the largest available sample for checking the 8B against the LLM on real candidates. Any pair that was in the 8B's training data must be excluded from that check.

## The rubric

[`prompts/judge_prompt.md`](../prompts/judge_prompt.md) (sha256 `031ac72b…`) scores usefulness toward a *correct* answer to *the same* problem.

- **2, highly relevant:** directly and substantially answers the same mathematical need, with correct central reasoning and conclusion.
- **1, partially relevant:** substantive, mathematically correct material that advances the same problem but leaves important work, for example a correct lemma, method or solved subcase. Topical overlap alone does not qualify.
- **0, not relevant or unusable:** any of the following.
  - A different problem or changed conditions.
  - Keyword overlap only.
  - No answer.
  - A major mathematical error.
  - A correct-looking conclusion with no reliable basis.
  - Mathematics that cannot be verified.
- **Unevaluable query:** when the query itself cannot be interpreted (missing figure, missing context, corrupted text), the judge sets `query_evaluable=false` and `score=null` instead of forcing a 0.

The output has exactly 8 fields: `task_id`, `guideline_version`, `query_evaluable`, `score`, `answer_validity`, `reason_codes`, `confidence`, `rationale`.

- `reason_codes` holds at least one of 18 codes, for example `full_solution`, `wrong_problem`, `condition_mismatch`, `topic_only` or `major_math_error`.
- The schema ties the codes and `answer_validity` to the score.

`guideline_version` is always the string `strict-math-retrieval-v1`, so it cannot tell prompt texts apart. The SHA-256 of the prompt identifies the text.

## The LLM judge

- **Model.** GPT-5.6-Sol. Reasoning effort was not held fixed.
  - v1's LLM labels came from several runs, and the effort behind each label was not recorded. Part of v1's final round went through an endpoint that applies its own default effort when none is requested, and none was requested. The endpoint documents that default as `low`; on three test requests its output was about half as long as with `xhigh`.
  - v2's LLM labels came through that same endpoint at its default, or through a second endpoint at an explicitly requested `low`. `llm_effort` records which.
- **Batching.** Each request holds 10 items.
  - In v1 they are laid out as 2 queries × 5 documents. The layout is fixed because it measurably changes labels. Two waves were accidentally packed as 6 queries × 1–2 documents, and their score-2 rate at rank 3 rose by 2.76 points. Both waves were re-judged.
  - v2's requests did not use that layout. New-candidate requests mostly hold 7–10 different queries, and gap requests mostly hold 3.
  - Given the effort and layout differences, v2's LLM labels are not strictly comparable with v1's.
- **Label mix.**

| Stage (judge) | Round | Rows | 0 | 1 | 2 | unevaluable |
|---|---|---:|---:|---:|---:|---:|
| Dense ranks 1–5 (LLM) | v1 | 10,120,955 | 35.8% | 30.6% | 32.7% | 0.84% |
| Reranker picks in ranks 6–20 (LLM) | v1 | 10,122,739 | 38.2% | 37.9% | 22.6% | 1.39% |
| Long documents in ranks 6–20 (LLM) | v1 | 2,093,238 | 58.8% | 25.9% | 12.6% | 2.76% |
| Other short documents in ranks 6–20 (8B) | v1 | 18,219,336 | 49.3% | 42.9% | 7.8% | — |
| Dense ranks 1–5 (LLM) | v2 gap | 35,477 | 37.8% | 33.4% | 27.3% | 1.43% |
| Reranker picks in ranks 6–20 (LLM) | v2 gap | 31,626 | 36.9% | 39.3% | 22.5% | 1.34% |
| Long documents in ranks 6–20 (LLM) | v2 gap | 1,751 | 56.5% | 27.0% | 13.1% | 3.37% |
| New candidates, long, fused positions 1–5 (LLM) | v2 | 162,464 | 38.0% | 34.8% | 25.7% | 1.52% |
| New candidates, short (8B) | v2 | 8,862,410 | 38.6% | 41.8% | 19.6% | — |
| New candidates, long, fused positions 6–20 (8B) | v2 | 924,358 | 46.5% | 38.8% | 14.7% | — |

The 240 trial pairs at fused positions 6–20 that carry an LLM label are not in the table. The v2 gap rows are pairs that failed in v1, so their mix need not match v1's for the same stage.

**Reliability against a second judge.** DeepSeek-V4-Pro independently re-judged 337,032 training candidates, and 306,993 of them have a combined label:

| | LLM judge alone | Second judge | Combined |
|---|---:|---:|---:|
| score 0 | 55.0% | 58.2% | 68.6% |
| score 1 | 35.4% | 28.4% | 22.8% |
| score 2 | 9.6% | 13.4% | 8.5% |

- The two judges agree exactly on 77.6% of pairs, and 7.2% needed arbitration.
- Only 1.0% are a 0 against a 2. The disagreement is about how strict to be, not about misreading the problem.
- The LLM judge is more lenient at the 0/1 boundary: it calls 45.0% of candidates at least partially relevant, against 31.4% for the combined label.
- Score 1 is the least stable label. Treat it as "partially relevant, uncertain".

## The 8B relevance model

- **Model.** Qwen3-Reranker-8B with a new single-output regression head, trained with MSE against the LLM judge's 0/1/2 labels.
- **Input.** The tokenizer's (query, document) pair, with the document cut to its first 32,000 characters and the pair truncated to 8,192 tokens. No chat template.
- **Training data.** 9,777,827 LLM-labeled pairs from the reranker-pick and long-document phases, which are the same rank band as the pairs the model later judges.
- **Held-out set.** 31,830 pairs over 6,000 queries, disjoint by query from training. Its LLM labels are 41.4 / 37.0 / 21.6% (0 / 1 / 2).

Most of the label variance, 61.2%, lies between queries. That makes pooled R² flattering, so the headline metric is within-query R².

| Training step | Within-query R² [95% CI] | Exact agreement with LLM (cuts 0.5 / 1.5) | LLM 2 kept as 2 | 0 ↔ 2 flips |
|---:|---:|---:|---:|---:|
| 5,000 | .3282 [.312, .344] | | | |
| **10,000 (used)** | **.3641 [.349, .380]** | **73.2%** | 63.3% | 1.11% |
| 12,500 | .3712 [.356, .387] | 73.6% | 64.6% | 1.10% |
| 15,000 (plateau) | .3728 [.358, .389] | 73.7% | 67.0% | 1.12% |

The 10,000-step checkpoint labels the data. Later steps add at most 0.5 points of agreement.

**How the 8B's cut labels spread within each LLM label, at 10,000 steps:**

| LLM label | 8B → 0 | 8B → 1 | 8B → 2 |
|---|---:|---:|---:|
| 0 | 75.4% | 23.1% | 1.4% |
| 1 | 17.1% | 76.5% | 6.4% |
| 2 | 2.4% | 34.3% | 63.3% |

The regression pulls toward the middle: a third of the LLM's 2s land at 1. The fixed cuts at 0.5 and 1.5 are a default, not a calibration. The raw score `s` ships with every 8B label so users can choose their own threshold.

For comparison, a 0.6B arm with the same data and configuration reached within-query R² .140. Capacity is the bottleneck.

**The labeling run.**

- **Packing.** Pairs are packed into rows of up to 65,536 tokens, with position ids restarting at each pair and no attention mask. Scoring uses FlashAttention-2 varlen in bf16.
- **Self-test gate.** Before producing any output, each worker scores 256 held-out pairs and compares them with the training-time predictions. It must reach Pearson ≥ 0.999; the measured value was 1.00000.
- **Label mix.** Over all 28,411,189 pairs it scored in v1, the 8B's labels split 44.3 / 44.3 / 11.4%. Over v2's 9,787,008 new candidates they split 39.4 / 41.5 / 19.1%. See [limitations](limitations.md) for why the v1 mix is not uniform across the run.
