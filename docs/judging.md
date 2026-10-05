# Candidates and judging

Every candidate is judged 0 / 1 / 2 under one rubric, by one of two judges:

- the **LLM judge**, GPT-5.6-Sol, for the candidates that most need it;
- an **8B relevance model** distilled from the LLM judge, for the rest.

Every label records which judge produced it.

## Candidates

Three routes each contribute their top 100 documents:

- **Dense:** Qwen3-Embedding-4B.
  - Documents: last-token pooling, up to 32,768 tokens, no instruction.
  - Queries: up to 512 tokens, with the prefix `Instruct: Given a math question, retrieve documents that answer it\nQuery: `.
  - Vectors are L2-normalized, and search is an exact inner product.
- **BM25:** k1 = 1.2, b = 0.75, over lowercased `[a-z0-9]+` tokens.
- **Word 5-gram overlap.**

Each route searches an index of the 4,340,031 documents from before near-duplicate removal, with the removed documents masked out, so every top 100 is over the deduplicated corpus. Candidates that point at documents removed as degenerate or multi-problem (see [corpus](corpus.md)) are dropped.

The routes are combined with **weighted reciprocal rank fusion**: score(d) = Σ_r w_r / (k + rank_r(d)), with weight 2 for the dense route, weight 1 for BM25 and for 5-gram, and k = 60. Ties are broken by the document's best rank across the routes.

**A query's candidates are its dense top 20 and its fused top 20.**

- 8,426,786 candidates, 4.55 per query, are in the fused top 20 but not in the dense top 20. These docs call them **fused-only** candidates.
- 879,575 fused-only candidates are outside the dense top 100 altogether: only BM25 or 5-gram found them.
- The query's own source page is judged on its own before retrieval (see [queries](queries.md)) and is never a fused-only candidate. A surviving copy that stands in for a removed source page can be one.

**Why weighted RRF.** On a 1,000-query probe, the union of the three routes' top 100 was fully judged by the LLM judge. Positives (score 2) per query:

| order | top 5 | top 10 | top 20 |
|---|---:|---:|---:|
| dense only | 1.61 | 2.56 | 3.99 |
| best rank across routes | 1.78 | 2.70 | 4.03 |
| weighted RRF, dense ×2, k = 60 | 1.92 | 2.97 | 4.56 |

Fusing by best rank lets documents that BM25 ranks high but that are not relevant push good dense results down. Weighting the dense route avoids this: 97.8% of the fused top 5 stays inside the dense top 5, while the positives that only the other routes find are still added.

## Who judges which candidates

```
dense top 20
   ranks 1–5 ──────────────────────────────────────────────▶ LLM judge
   ranks 6–20 ─┬─ longer than 8,144 tokens ────────────────▶ LLM judge
               └─ up to 8,144 tokens ─┬─ all of them ──────▶ 8B model
                                      └─ reranker top 5 ───▶ LLM judge as well
fused-only candidates
   longer than 8,144 tokens, fused positions 1–5 ─────────▶ LLM judge
   everything else ───────────────────────────────────────▶ 8B model
```

| Candidates | Judge | Reads | Rows |
|---|---|---|---:|
| Dense ranks 1–5 | LLM | first 30,000 characters | 8,740,887 |
| Dense ranks 6–20: the reranker's top 5 | LLM; the 8B also scores them | first 30,000 characters | 8,682,170 |
| Dense ranks 6–20: longer than 8,144 tokens | LLM | first 100,000 characters | 1,275,579 |
| Dense ranks 6–20: all other documents | 8B | first 32,000 characters, then 8,192 tokens | 15,818,262 |
| Fused-only, longer than 8,144 tokens, fused positions 1–5 | LLM | first 100,000 characters | 113,876 |
| Fused-only, longer than 8,144 tokens, fused positions 6–20 | 8B | first 32,000 characters, then 8,192 tokens | 678,631 |
| Fused-only, up to 8,144 tokens | 8B | first 32,000 characters, then 8,192 tokens | 7,633,824 |
| The query's own source page, when no row above covers it | LLM | first 30,000 characters | 562,940 |

That is 43,506,169 rows, one per judged pair.

- **The reranker.** Qwen3-Reranker-4B scores dense ranks 6–20 zero-shot, with the instruction to judge whether the document answers "the same problem with the same particulars". Its top 5 go to the LLM judge.
  - It was chosen on a 196-query probe whose three-route candidate pool was fully judged. There, its AUC for separating score 2 from score 0 was 0.921, against 0.599 for the fused retrieval order.
  - Retrieval is good at getting relevant documents into the pool but poor at ordering them within it; the reranker fixes the order.
- **Long documents.** Token counts use the Qwen3 tokenizer. The reranker reads at most 8,144 tokens, so longer documents in dense ranks 6–20 go to the LLM judge, which reads up to 100,000 characters.
- **Pairs with both labels.** 8,726,880 pairs carry an LLM label and an 8B score:
  - the 8,682,170 reranker picks;
  - 8,717 long documents in dense ranks 6–20;
  - 11,122 source pages;
  - 24,705 surviving copies that stand in for a removed source page and that the 8B had scored as candidates (see [queries](queries.md));
  - 166 long fused-only documents at fused positions 6–20 that were also sent to the LLM judge.

  The LLM label is the one used, and the 8B score is kept in its own column; see [schema](schema.md). Once the 8B's own training pairs are excluded, these pairs also allow checking the 8B against the LLM on real candidates.

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

## Label mix

| Candidates | Judge | Rows | 0 | 1 | 2 | unevaluable |
|---|---|---:|---:|---:|---:|---:|
| Dense ranks 1–5 | LLM | 8,740,887 | 33.8% | 31.6% | 34.3% | 0.30% |
| Dense ranks 6–20: reranker's top 5 | LLM | 8,682,170 | 36.1% | 39.1% | 24.3% | 0.51% |
| Dense ranks 6–20: longer than 8,144 tokens | LLM | 1,275,579 | 56.4% | 27.7% | 15.1% | 0.74% |
| Dense ranks 6–20: all other documents | 8B | 15,818,148 | 47.5% | 44.2% | 8.2% | — |
| Fused-only, long, fused positions 1–5 | LLM | 113,876 | 38.3% | 33.3% | 27.7% | 0.63% |
| Fused-only, long, fused positions 6–20 | 8B | 677,817 | 45.6% | 38.2% | 16.2% | — |
| Fused-only, up to 8,144 tokens | 8B | 7,609,881 | 37.2% | 42.3% | 20.5% | — |

Rows in 8B stages that carry an LLM label (the 24,705 copies and the 166 long fused-only pairs) are left out of the table. Source-page rows are all 2: a source page, or a copy standing in for one, is marked only when it was judged a full answer.

## The LLM judge

- **Model.** GPT-5.6-Sol.
- **Reliability against a second judge.** DeepSeek-V4-Pro independently re-judged 337,032 training candidates, and 306,993 of them have a combined label:

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
- **Training data.** 9,777,827 LLM-labeled reranker picks and long documents from dense ranks 6–20, the same rank band as most of the pairs the model labels.
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

**Scoring.**

- **Packing.** Pairs are packed into rows of up to 65,536 tokens, with position ids restarting at each pair and no attention mask. Scoring uses FlashAttention-2 varlen in bf16.
- **Self-test gate.** Before producing any output, each worker scores 256 held-out pairs and compares them with the training-time predictions. It must reach Pearson ≥ 0.999; the measured value was 1.00000.
