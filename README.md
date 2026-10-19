# RAG: Baseline Retrieval vs. Re-Ranking

## 1. Setup and Approach

For this experiment, I used the Ball State University Faculty and Professional Personnel Handbook for 2026–2027. The PDF text was extracted page by page and split into chunks of 1,000 characters with an overlap of 200 characters. This produced 1,257 chunks.

I used `all-MiniLM-L6-v2` to create embeddings and FAISS to retrieve the most similar chunks. The embeddings were normalized, and inner product was used for similarity search.

In the baseline pipeline, the top three chunks from the ten retrieved candidates were sent directly to Gemini. In the second pipeline, the same ten candidates were scored using `cross-encoder/ms-marco-MiniLM-L-6-v2`, sorted again, and the top three were sent to Gemini. The same answer-generation prompt was used in both pipelines.

## 2. Bi-Encoder and Cross-Encoder

The Bi-Encoder converts the query and document chunks into embeddings separately. This makes it useful for searching a large collection because the document embeddings can be computed beforehand.

The Cross-Encoder considers the query and a candidate chunk together and produces a relevance score. It can compare the relationship between the query and the chunk more directly, but scoring every chunk this way would be more expensive. In this experiment, I used it only on the ten candidates retrieved by the Bi-Encoder.

## 3. Results and Failure Analysis

For the question about parental-leave duration, the baseline retrieved the eligibility passage at rank 1 and the six-week paid-leave passage at rank 2. After re-ranking, the paid-leave passage moved to rank 1, the eligibility passage moved to rank 2, and the general 26-week leave passage moved from rank 4 to rank 3.

Both generated answers gave the main leave limits. However, the re-ranked answer also included conditions about continuous service, eligibility for paid leave, and when the leave must be used. This made the answer more complete.

Two clearer improvements appeared in the stress tests. For the question about adoption by a new spouse, the baseline said that the supplied context did not contain the answer. The re-ranked pipeline retrieved the exception and correctly stated that this adoption situation was excluded from the policy.

For the question about when leave should be requested, the baseline again said that the context did not provide the information. The re-ranked answer identified the general requirement to request leave at least 30 days in advance, where possible, citing page 196.

The eligibility-category question showed that re-ranking does not always improve the rank of the most relevant chunk. The eligibility passage moved from rank 1 to rank 2. It still remained among the three chunks sent to Gemini, so this rank change did not necessarily harm the final answer.

Overall, these examples suggest that re-ranking can help when the initial retrieval places a useful passage too low in the ranking. However, the number of tested questions is limited, and these results do not establish that re-ranking improves every query.

## 4. Latency

I recorded the time taken by Gemini to generate answers from the baseline and re-ranked contexts. For the parental-leave duration question, generation took 4.05 seconds for the baseline and 4.70 seconds for the re-ranked version. For the adoption exception, the times were 4.95 and 2.74 seconds, respectively. For the request-timing question, they were 4.13 and 3.29 seconds.

The timings varied across questions. These measurements cover answer generation rather than the complete retrieval pipeline, so they cannot be used to conclude that re-ranking is faster overall. The extra Cross-Encoder stage also has a computational cost that should be included in a full latency comparison.

## 5. Why Not Use the Cross-Encoder on the Entire Collection?

The handbook was divided into 1,257 chunks. A Cross-Encoder would need to process the query together with each chunk, which means many query–chunk comparisons for a single question. A Bi-Encoder is more efficient for the first stage because the document embeddings can be computed in advance and searched using an index.

Therefore, the two-stage approach uses the Bi-Encoder to narrow the search to ten candidates and the Cross-Encoder to refine their order. This balances retrieval efficiency with more detailed relevance scoring.

## 6. Limitations

The experiment used a small set of questions, mostly about parental leave. More questions from different parts of the handbook would be needed to evaluate the approach more broadly. The recorded latency values are generation times, not end-to-end pipeline measurements. The results should therefore be interpreted as observations from this experiment rather than a general guarantee that re-ranking is always better.