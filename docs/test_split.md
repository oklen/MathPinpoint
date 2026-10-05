# Test split

The test split is 6,173 held-out queries and their relevant documents. The `queries` config holds the queries as split `test`, and the `qrels` config lists the relevant documents. The training result in the [README](../README.md#training-a-retriever-on-mathpinpoint) is measured on this split.

| | |
|---|---:|
| queries | 6,173 |
| relevant (query, document) pairs | 83,833 |
| distinct relevant documents | 78,996 |
| relevant documents per query | median 4, mean 13.6, 90th percentile 43, maximum 140 |

## Queries

- The test queries come from the same extraction as the training queries (see [queries](queries.md)): 8,000 queries were set aside for evaluation.
- No training query matches a test query verbatim, at word 5-gram Jaccard ≥ 0.50, or as a semantic duplicate judged to be the same problem. The training set was filtered against all 8,000, and the lexical check is exhaustive; see [evaluation leakage](queries.md#evaluation-leakage).
- Test queries pass the same self-containedness check as training queries ([queries](queries.md#self-containedness-check)): 594 of the 8,000 were found not self-contained by both judges and are left out.
- Test queries then pass a stricter version of that check. Each one is judged by DeepSeek-V4-Pro, Gemini-3.1-Pro-Preview and GPT-5.6-Sol under the same prompt, one verdict per judge, and is left out when at least two of the three find it not self-contained. This removes 241 more.
- A test query's id is `"q_" + sha256(lowercase(normalized_text)).hexdigest()[:24]`, with the same normalization as document ids: NFKC, runs of whitespace collapsed to one space, trimmed.

## Labels

Every one of the 8,000 queries has a candidate pool, built over the documents from before near-duplicate removal. Each pool holds:

- the query's own source page;
- the BM25 top 100;
- the first 50 documents of the bge-base-en-v1.5 top 100 that are not already in the pool.

Only pool documents that are in the corpus are judged and kept: 947,537 pairs.

Every pair was judged independently by GPT-5.6-Sol and DeepSeek-V4-Pro under [`prompts/judge_prompt.md`](../prompts/judge_prompt.md). The two scores are combined as follows:

| The two judges | Combined score | Pairs |
|---|---|---:|
| agree | that score | 731,906 (77.2%) |
| differ by one, and neither gave 2 | the lower score | 129,952 (13.7%) |
| differ by two, or only one gave 2 | Gemini-3.1-Pro-Preview arbitrates | 52,968 (5.6%) |
| either judged the query unevaluable | the pair is dropped | 32,140 (3.4%) |

- 571 pairs (0.06%) needed arbitration but have no arbitration result, because the arbiter returned no valid output. They take the lower score.
- Where a judge scored the same pair twice, its later score is used.
- A query is kept only when both judges scored every pair in its pool. All 8,000 queries qualify.

**Second pooling round.** For the 6,173 queries of the split, the Qwen3-Embedding-4B top 20 over the corpus (document embeddings over up to 32,768 tokens) was added to the pool. Of its 123,460 (query, document) pairs, 54,062 had already been judged. The other 69,398 were judged by the same two judges under the same rules: 49,477 agree, 11,746 take the lower score, 7,507 are arbitrated, 28 take the lower score because the arbiter returned no valid output, and 640 are dropped as unevaluable.

**Relevant documents.** A document is relevant when its combined score is 2. After the 594 queries that fail the self-containedness check are removed, 6,414 queries have at least one relevant document; 992 have none. The stricter check then removes 241, leaving 6,173. The second pooling round adds 9,314 relevant pairs, for 83,833 in all.

Documents outside a query's pool were never judged and count as not relevant; see [limitations](limitations.md#test-split).
