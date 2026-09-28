# Corpus

The corpus holds **3,676,820** documents. Each document is the cleaned mathematical content of one web page, with the page's own question removed. The corpus is deduplicated twice: once exactly, once for near-duplicates.

| Stage | Output | Change |
|---|---:|---|
| Upstream pages | 13,707,851 | |
| Drop pages with upstream difficulty score 0–1 | 13,514,716 | −1.4% |
| Exact dedup of raw page text | 6,029,052 | −55.4% |
| Content extraction (pages whose extraction came out empty are skipped) | 5,929,019 | −100,033 |
| Remove each page's own question from its text | 5,929,019 | 816 pages become empty |
| Normalize; drop empty documents; exact dedup | 4,340,031 | −26.8% of non-empty |
| Near-duplicate removal (MinHash, then LLM review) | **3,676,820** | −15.3% |

## Upstream pages

The pages come from the L2 tier of UltraData-Math: web pages that passed a quality classifier applied to its L1 tier. We used the copy `TeraflopAI/udml2-labeled` (117 parquet shards), which adds a 0–7 difficulty score to each page. Its row count, 13,707,851, equals that of the L2 tier.

The other tiers are not used:

- **L1** has had only heuristic cleaning.
- **L3** is synthetic. Each problem and its solution come from a single LLM call, so the question text leaks into the document, and retrieval becomes a string-matching exercise.

Pages carry no URL.

## Page filtering and exact dedup

- Pages with difficulty score 0 or 1 are dropped.
- Pages are deduplicated on the SHA-256 of the raw page text, with no normalization. 55.4% of rows are exact duplicates, leaving 6,029,052 unique pages.
- Pages have a median length of 5,084 characters. The 13.9% of pages longer than 20,000 characters are kept.

## Content extraction

An extraction model turns each page into its mathematical content. The model is Qwen3.5-2B distilled from GPT-5.6, run with greedy decoding.

- 100,033 pages produced an empty extraction and were skipped, which leaves 5,929,019 documents.
- 66,198 outputs (1.1%) fail a repetition test. 38,764 of these are repetition loops. They were flagged, not removed; see [limitations](limitations.md).

## Removing each page's own question

Every query is extracted from a page (see [queries](queries.md)). If that page still contained the question word for word, retrieving it would be trivial. So the verbatim question span, `question_verbatim`, is deleted from the document before indexing.

| How the span was located | Share of documents |
|---|---:|
| Exact match | 52.59% |
| Whitespace-tolerant match | 1.51% |
| Not found | 0.11% |
| Skipped: the query was composed and the page had no verbatim question | 45.79% |

- A second pass removes near-verbatim residue. It touched 3.06% of documents.
- **Guards:**
  - Never cut inside a formula block.
  - Never delete more than 15% of a document or more than 500 characters.
  - Only `question_verbatim` is ever a deletion target. When `query` was used as the target instead, manual review found that answer content, such as theorems and derived results, got deleted along with it.
- **Effect** (183,443 documents measured): documents that still contain their full question fell from 98.2% to 0.1%.
- 816 documents became empty. They held only the question and no answer, and are dropped in the next step.

## Normalization, exact dedup and document ids

- **Normalization:** Unicode NFKC, runs of whitespace collapsed to a single space, then trimmed. The stored text is this normalized form, with case preserved.
- **Document id:** `did = "d_" + sha256(lowercase(normalized_text)).hexdigest()[:24]`.
- **Exact dedup** on `did` removes 26.8% of the 5,928,203 non-empty documents, leaving 4,340,031.

## Near-duplicate removal

**Candidate pairs.**

1. Strip `\displaystyle` and apply NFKC.
2. Compute MinHash signatures with 128 permutations over word 5-grams.
3. Run LSH with two banding schemes: 16 bands × 8 rows, and 32 bands × 4 rows.
4. Keep a candidate pair only if its exact word 5-gram Jaccard is ≥ 0.80.

This proposes 762,610 deletions (17.6%). On a sample, 10.8% of the proposed deletions were wrong, so every deletion was reviewed.

**Review.**

- 583,077 of the proposals (76.5%) have no content that the copy being kept lacks. These are deleted.
- The other 179,533 (23.5%) go to an LLM judge (GPT-5.6-Sol). It was calibrated on 96 hand-checked items and got all 96 right. A failed call counts as "keep".
- Of the 177,431 proposals the judge reviewed, it kept 54.8%. 49.9% had substantive content the other copy lacks, and 4.9% were different documents.

**Result:** 663,211 deletions, with 99,399 documents rescued, leaving **3,676,820** documents.

**Known gap.** Pairs close to the 0.80 threshold can escape, because LSH candidate generation is probabilistic. For example, two documents that differ by a single `/` inside a formula have a Jaccard of 0.818 and survived. Normalizing before hashing fixes such pairs: lowercase, delete punctuation (deleting it outright, not replacing it with spaces), NFD, and fold whitespace. On a 120,000-document sample this merged 31 more groups (0.026%), and all of them were true duplicates. The full-corpus re-run has not been done.

## Multi-problem documents

114,070 documents (2.63%) contain five or more separate problems; this is a lower bound. They remain in the corpus. Queries extracted from multi-problem pages were removed from the query set (see [queries](queries.md)).
