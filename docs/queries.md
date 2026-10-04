# Queries

There is one query per document. Where the page contains a question somebody actually asked, that question *is* the query, copied with as few edits as possible. The **2,032,033** queries in MathPinpoint are the ones that survive deduplication and leakage removal, and whose source page was judged a full answer.

## Extraction

The extractor is GPT-5.6-Sol with the minimal-edit prompt [`prompts/query_gen_prompt.txt`](../prompts/query_gen_prompt.txt) (sha256 `d75d30a5b8cb0b03…`). It is allowed three kinds of edit:

- **remove** anything that is not part of the question: greetings, pleas for help, signatures, the asker's own failed attempt;
- **insert** a value, condition or definition that the question refers to but does not state, copied verbatim from elsewhere on the same page;
- **replace** a pointer such as "the equation above" with the thing it points to, as the page writes it.

It must not reword, normalize notation, reorder, or translate. When the page has no question, for example an expository article, the model composes a query and sets `from_page_question=false`.

A page is declined when it does not state a single, self-contained, mathematical problem that the page itself answers.

| | n | share |
|---|---:|---:|
| documents | 5,929,019 | |
| with a query | 5,044,362 | 85.1% |
| ↳ taken from the page's own question | 3,267,081 | 64.8% of queries |
| ↳ composed by the model | 1,777,281 | 35.2% of queries |
| declined | 884,657 | 14.9% |

The declined pages break down as:

| Reason | Share of declines |
|---|---:|
| Reference page only | 39.5% |
| Corrupted text | 24.8% |
| Needs a missing figure or context | 19.6% |
| Not mathematics | 8.5% |
| No problem stated | 7.7% |

**The edit is real.** On a sample of 210,439 queries, the median share of a query's characters that fall inside a run of 15 or more characters copied from its page is 1.000.

## From extracted queries to MathPinpoint queries

| Step | Queries | Change |
|---|---:|---|
| Non-empty queries | 5,044,326 | |
| Exact dedup of normalized text; one source page kept at random per query | 3,385,028 | −32.9% |
| Remove lexical overlap with evaluation queries (word Jaccard ≥ 0.50) | 3,373,109 | −0.35% |
| Remove semantic overlap with evaluation queries (cosine ≥ 0.70, and judged the same problem) | 3,357,494 | −0.46% |
| Remove queries from multi-problem pages (≥ 5 problems) | 2,911,860 | −13.3% |
| Source page judged | 2,901,088 | |
| **Source page judged a full answer (score 2)** | **2,032,033** | |

A query is identified by its source page: `qid = "p_" + sha256(raw_page_text).hexdigest()[:24]`, where `raw_page_text` is the upstream page's `content` field exactly as distributed (UTF-8, no normalization). There is one query per page. Note that this is a different recipe from the document id, which hashes the normalized extracted text; see [corpus](corpus.md).

### Why exact dedup only

In a probe of 1,000 queries, 44.7% had an exact duplicate after normalization. The duplication comes from the web, not from the pipeline: 45.9% of near-identical query pairs come from pages whose text is entirely different, typically classic problems reposted across many sites.

Merging near-duplicates was rejected. At word Jaccard 0.8–0.9, an LLM judge found that 13–16% of the pairs were different problems, for example the same method applied to a different object.

The page kept for a duplicated query is chosen at random. Taking the first one would tie the choice to shard order, which tracks crawl time and source site.

### Evaluation leakage

Before this step, all 8,000 held-out evaluation queries appeared verbatim in the raw query pool. After the two overlap filters, none of the 8,000 appears verbatim among the 2,032,033 queries. The test split is drawn from these 8,000; see [test split](test_split.md).

### Multi-problem pages

A page that bundles five or more problems produces a query that matches only a fraction of the page. 732,919 source pages (12.4%) were flagged by the detector, and their queries were removed.

## Checking each query against its own source page

Each query is judged against its own source page by the same LLM judge and rubric used for retrieved candidates (see [judging](judging.md)). The page has its question removed and is cut to 30,000 characters.

| Score | Share of 2,901,088 |
|---|---:|
| 2: the page fully answers the query | 70.04% (2,032,033) |
| 0: the page does not answer it | 11.70% |

Only queries whose source page scored 2 enter retrieval, which guarantees every query has at least one known full answer in the corpus.

**A second judge checked the positives.** DeepSeek-V4-Pro independently re-judged 1,453,403 of the 2,032,033 score-2 pages:

| Second judge's score | Share |
|---|---:|
| 2 | 95.00% |
| 1 | 4.14% |
| 0 | 0.85% |

The second judge gives 2 more readily than the first on retrieval candidates (13.4% vs 9.6%). So 95% agreement is an upper bound, and 0.85% strong disagreement is a lower bound.

**Why not stop at the source page.** Training only on (query, source page) pairs teaches a retriever to find the page a query was copied from, not every page that answers it. In our runs this hurt recall: Recall@100 fell by 0.088 with bge-base, and with Qwen3-Embedding-0.6B the recall gains disappeared as training continued. MathPinpoint therefore judges the retrieved candidates as well. See [judging](judging.md).
