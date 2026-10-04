# Limitations

## Coverage

- **Only each query's dense top 20 and fused top 20 are labeled, so unlabeled does not mean irrelevant.** A document outside both lists can still answer the query.
  - Some queries have many full answers. On a 1,000-query probe with 725,326 judged pairs, the number of full answers per query had a median of 7, a 90th percentile of 230 and a maximum of 1,840. Standard textbook problems are answered by hundreds of pages.
  - For such a query, most of its full answers are not among its labeled candidates.

## Label quality

- **The labels are not human gold.** Each label comes from a single LLM judge or from the 8B model distilled from it. There is no human review.
  - The LLM judge is lenient at the 0/1 boundary; see [judging](judging.md).
  - Score 1 is the least stable label.
- **The 8B scores regress toward the middle.** A third of the LLM's 2s receive an 8B label of 1. The cuts at 0.5 and 1.5 are defaults, not calibrated thresholds; the raw score ships with every 8B label.
- **The 8B's accuracy was measured on the reranker's picks and on long documents,** not on the pairs it labels: the other documents in dense ranks 6–20 and the fused-only candidates. Its accuracy on those pairs has not been measured.

## Corpus

- **Residual near-duplicates.** Pairs just above the dedup threshold can survive. See [corpus](corpus.md).
- **Mirror pages.** A query's question is removed from its own page but can still appear on other pages that repost the same problem.

## Test split

- **Only pooled documents were judged.** Every other document counts as not relevant. The pools come from BM25 and bge-base-en-v1.5, so a retriever that ranks differently from these two retrieves documents nobody judged, and its scores understate it.
