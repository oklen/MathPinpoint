# Limitations

## Coverage

- **v1 labels only the dense top 20, so unlabeled does not mean irrelevant.** In an audit of a fully judged evaluation pool, the top 100 of Qwen3-Embedding-4B alone reached at most 73.8% of the known full answers. Another 23.4% were found only by BM25.
  - v2 adds the BM25 and 5-gram candidates that reach the fused top 20.
  - Even so, a document outside a query's fused top 20 can still answer it.
- **Positives are heavy-tailed.** On a 1,000-query probe, the number of full answers per query had a median of 7, a 90th percentile of 230 and a maximum of 1,840. 10.5% of queries had none in the pool. Standard textbook problems are answered by hundreds of pages, so a handful of queries hold most of the positives.

## Label quality

- **The labels are not human gold.** Each label comes from a single LLM judge or from the 8B model distilled from it. There is no human review.
  - The LLM judge is lenient at the 0/1 boundary; see [judging](judging.md).
  - Score 1 is the least stable label.
- **The 8B scores regress toward the middle.**
  - A third of the LLM's 2s receive an 8B label of 1. The cuts at 0.5 and 1.5 are defaults, not calibrated thresholds.
  - The 8B's held-out set comes from the reranker-pick and long-document pairs, not from its target population, which is all short documents in ranks 6–20. Calibration on the target population is still open.
- **The 8B's label mix is not uniform across the run.** Score 2 is 8.7% among the first 12.2M pairs and 13.5% among the rest. The first part of the run processed the shortest pairs first, so the two parts probably hold different pairs rather than showing drift. This has not yet been checked by length bucket.
- **Labeling conditions are not stored per row.**
  - The LLM labels were produced over about two weeks through more than one serving channel.
  - The prompt, the batch layout and the reasoning effort were held fixed where the channel allowed it, but none of them is recorded on individual labels.
- **Truncation differs by stage:**

  | Stage | What it reads |
  |---|---|
  | LLM judge, dense ranks 1–5 and reranker picks | first 30,000 characters |
  | LLM judge, long documents | first 100,000 characters |
  | 4B reranker | first 8,144 tokens |
  | 8B model | first 32,000 characters, then 8,192 tokens |
- **The reranker's top-5 selection is fragile.** Scores near ranks 5 and 6 are very close. Computing the same pairs along two numerically different paths, with scores about 0.5% apart, changed 7 of 20 top-5 sets. Which candidates the LLM judged, as opposed to the 8B, is therefore partly arbitrary.
- **Silent gaps.**
  - 40,967 pairs in the dense top 20 (0.10%) have no label because their LLM call never produced a valid result.
  - 283,627 LLM rows judged the query itself unevaluable; they carry `label=null`, and 141,455 of them still have an 8B score.
- **Two judgments of the same source page.** 1,140,217 source pages were also retrieved and judged as candidates. The candidate-stage label is the one kept; it agrees with the source-page check on a 2 in 95.27% of cases.
- **Missing source pages.** 307,715 queries (15.1%) have no `source_did`: their source page was removed as a near-duplicate, and the dedup kept no record of which copy survived.
- **Prompt provenance.** Every LLM candidate label carries the prompt hash `031ac72b…`. For the dense-rank stage the prompt file was copied from a backup with that hash, but the hash was not recorded per batch.

## Corpus

- **Extraction errors.** 1.1% of extractions are degenerate, mostly repetition loops. They are flagged, not removed.
- **Residual near-duplicates.** Pairs near the dedup threshold can survive; the normalized re-run is pending. See [corpus](corpus.md).
- **Mirror pages.** A query's question is removed from its own page but can still appear on other pages that repost the same problem.
- **Multi-problem documents** (2.6%) remain in the corpus and in the candidate lists.
