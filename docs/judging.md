# Candidates and judging

For every query, the top 20 documents from a dense retriever are judged 0 / 1 / 2 under one rubric. There are two judges:

- an LLM judge, for the candidates that most need it;
- an 8B relevance model distilled from the LLM judge, for the rest.

Every label records which judge produced it.

## Candidates

- **Retriever: Qwen3-Embedding-4B.**
  - Documents: last-token pooling, up to 32,768 tokens, no instruction.
  - Queries: up to 512 tokens, with the prefix `Instruct: Given a math question, retrieve documents that answer it\nQuery: `.
  - Vectors are L2-normalized.
- **Search.** The index holds the 4,340,031 documents from before near-duplicate removal. Search is an exact inner product, and removed near-duplicates are filtered out of the results. The top 20 per query are judged.
- **Sanity check.** The LLM judge's score-2 rate falls steadily over dense ranks 1–5: 59.2 / 36.0 / 26.6 / 22.2 / 19.6%.

## Who judged which pairs

| Candidates | Judge | Pairs sent |
|---|---|---:|
| Dense ranks 1–5 | LLM, reading up to 30,000 characters | ≈ 10.15M |
| Ranks 6–20: the top 5 of the 4B reranker's order | LLM, reading up to 30,000 characters | 8,214,071 (= 5,299,442 in three rounds + 2,914,629 in the final round) |
| Ranks 6–20: documents longer than 8,144 tokens | LLM, reading up to 100,000 characters | 2,095,018 (884,238 queries) |
| Ranks 6–20: all documents up to 8,144 tokens | 8B relevance model | 28,411,189 |
| Each query's own source page (before retrieval) | LLM | 2,901,088 |

- **The reranker picks.** Qwen3-Reranker-4B scores ranks 6–20 zero-shot, with the instruction to judge whether the document answers "the same problem with the same particulars". Its top 5 go to the LLM judge. On a fully judged probe, its AUC for separating score 2 from score 0 is 0.921, against 0.599 for the retrievers' fused ranking.
- **The long-document threshold.** The reranker reads at most 8,144 tokens of a document. Longer documents therefore skip the reranker and go straight to the LLM judge, which reads up to 100,000 characters.
- **Overlap.** The 8B set covers every short document in ranks 6–20, so it includes the ≈ 8.2M reranker picks that the LLM also judged. These pairs carry both labels. The planned default is that the LLM label wins and the 8B score is kept in its own column; see [schema](schema.md). The overlap is also the largest available sample for checking the 8B against the LLM on real candidates. Any pair that was in the 8B's training data must be excluded from that check.

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

- **Model.** GPT-5.6-Sol at reasoning effort `xhigh`.
- **Batching.** Each request holds 10 items, laid out as 2 queries × 5 documents.
  - The layout is fixed because it measurably changes labels. Two waves were accidentally packed as 6 queries × 1–2 documents, and their score-2 rate at rank 3 rose by 2.76 points. Both waves were re-judged.
- **Label mix.**

| Candidates | 0 | 1 | 2 | unevaluable |
|---|---:|---:|---:|---:|
| Dense ranks 1–5 | 35.5% | 31.0% | 32.7% | 0.9% |
| Reranker picks, 4,000-row sample | 57.5% | 26.4% | 13.0% | 3.15% |

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
- **Label mix so far.** With 81.5% of pairs done, the 8B's labels split 43.9 / 45.2 / 11.0%. See [limitations](limitations.md) for why this mix is not uniform across the run.
