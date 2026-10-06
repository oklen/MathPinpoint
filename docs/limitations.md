# Limitations

## Coverage

- **Only each query's dense top 20 and fused top 20 are labeled, so unlabeled does not mean irrelevant.** A document outside both lists can still answer the query.
  - Some queries have many full answers. On a 1,000-query probe with 725,326 judged pairs, the number of full answers per query had a median of 7, a 90th percentile of 230 and a maximum of 1,840. Standard textbook problems are answered by hundreds of pages.
  - For such a query, most of its full answers are not among its labeled candidates.

## Label quality

- **The labels are not human gold.** Each training label comes from a single LLM judge or from the 8B model distilled from it; test labels combine two LLM judges and an arbiter. There is no human review.
  - The LLM judge is lenient at the 0/1 boundary; see [judging](judging.md).
  - Score 1 is the least stable label.
- **The 8B scores regress toward the middle.** A third of the LLM's 2s receive an 8B label of 1. The cuts at 0.5 and 1.5 are defaults, not calibrated thresholds; the raw score ships with every 8B label.
- **About one in six 8B positives is a partial answer for the LLM judge.** On a 3,600-pair sample of the candidates the 8B labels on its own, judged by the LLM judge, 83.5% of the 8B's 2s are 2s for the LLM and most of the rest are 1s; 0.5% of its 0s are LLM 2s. See [judging](judging.md#the-8b-relevance-model).

## Corpus

- **Residual near-duplicates.** Pairs just above the dedup threshold can survive. See [corpus](corpus.md).
- **Mirror pages.** A query's question is removed from its own page but can still appear on other pages that repost the same problem.

## Test split

- **Only pooled documents were judged.** The pools come from BM25, bge-base-en-v1.5 and Qwen3-Embedding-4B. A retriever that ranks differently retrieves documents nobody judged: nDCG@10 counts them as not relevant, and R@100 is recall over the judged relevant documents only. Pool bias can therefore move either metric, and the difference between two retrievers, in either direction. In the README comparison, 66% of the top 10 of the model before fine-tuning is judged, but only 35% of the fine-tuned model's.
