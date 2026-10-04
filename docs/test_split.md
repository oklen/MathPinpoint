# Test split

The test split is 2,284 held-out queries and their relevant documents. The `queries` config holds the queries as split `test`, and the `qrels` config lists the relevant documents. The training result in the [README](../README.md#training-a-retriever-on-mathpinpoint) is measured on this split.

| | |
|---|---:|
| queries | 2,284 |
| relevant (query, document) pairs | 26,948 |
| distinct relevant documents | 26,412 |
| relevant documents per query | median 3, mean 11.8, 90th percentile 36, maximum 135 |
| pairs where the document is the query's own source page | 1,992 |

## Queries

- The test queries come from the same extraction as the training queries (see [queries](queries.md)): 8,000 queries were set aside for evaluation.
- No training query matches a test query verbatim, at word Jaccard ≥ 0.50, or as a semantic duplicate judged to be the same problem. The training set was filtered against all 8,000; see [evaluation leakage](queries.md#evaluation-leakage).
- A test query's id is `"q_" + sha256(lowercase(normalized_text)).hexdigest()[:24]`, with the same normalization as document ids: NFKC, runs of whitespace collapsed to one space, trimmed.

## Labels

Candidate pools were built for 5,000 of the 8,000 queries, over the 4,340,031 documents from before near-duplicate removal. Each pool holds:

- the query's own source page;
- the BM25 top 100;
- the first 50 documents of the bge-base-en-v1.5 top 100 that are not already in the pool.

Every pair was judged independently by GPT-5.6-Sol and DeepSeek-V4-Pro under [`prompts/judge_prompt.md`](../prompts/judge_prompt.md). The two scores are combined as follows:

| The two judges | Combined score |
|---|---|
| agree | that score |
| differ by one, and neither gave 2 | the lower score |
| differ by two, or only one gave 2 | Gemini-3.1-Pro-Preview arbitrates |
| either judged the query unevaluable | the pair is dropped |

- The judges agreed on 80.8% of the 380,416 combined pairs. 15,362 pairs were arbitrated.
- 5,415 pairs (1.4%) needed arbitration but have no arbitration result. They take the lower score.
- A query is kept only when both judges scored every pair in its pool. 2,617 queries qualify.

**Relevant documents.** A document is relevant when its combined score is 2; 2,303 of the 2,617 queries have at least one. Only documents in the corpus are kept: 6,220 relevant pairs point to documents that near-duplicate removal took out, and they are dropped. 2,284 queries keep at least one relevant document.

Documents outside a query's pool were never judged and count as not relevant; see [limitations](limitations.md#test-split).
