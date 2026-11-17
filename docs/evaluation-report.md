# Week 12 Search and RAG Evaluation Report

Online Bookstore: D1 + Vectorize + Workers AI. Tests were run on 2026-11-17 against the local Worker (`wrangler dev`), connected to the remote D1 database `w12-rag-db`, the Vectorize index `w12-rag-vectors` (384 dimensions, cosine) and Workers AI.

Raw outputs: [step7-fts-vs-semantic.txt](step7-fts-vs-semantic.txt) and [step8-rag-answers.txt](step8-rag-answers.txt).

## 1. Method

- **Catalog:** six books, each with one `book_documents` row (the book description).
- **Test questions:** the ten questions from the lab (Step 35). They replace the ten in `evaluation-template.md`.
- **Gold labels:** one relevant book per question, taken from the lab's reference table. They were fixed before the searches were run.
- **FTS5:** `GET /api/fts`. Keyword search over `books_fts`, with the question split into words joined by `OR`, ranked by `bm25`, limit 3.
- **Semantic:** `GET /api/semantic`. The question is embedded with `@cf/baai/bge-small-en-v1.5`, Vectorize returns the Top-3 by cosine similarity, and D1 supplies the book record with a JOIN.
- **Precision@3** = relevant books among the Top-3 / 3. With one relevant book per question, the best possible score is 1/3 = 0.333.

## 2. FTS5 versus semantic search

| # | Question | Gold | FTS5 Top-3 | Vector Top-3 | P@3 FTS5 | P@3 Vector | Semantic latency (ms) |
|---|---|---|---|---|---|---|---|
| 1 | I want to learn SQL from scratch. | 1 | 1, 3, 2 | 1, 4, 2 | 0.333 | 0.333 | 773 |
| 2 | How can I design normalized tables? | 2 | 2 | 2, 4, 1 | 0.333 | 0.333 | 680 |
| 3 | I want to learn machine learning. | 3 | 3, 1, 2 | 3, 4, 1 | 0.333 | 0.333 | 686 |
| 4 | How do LLMs use external documents? | 4 | (no results) | 4, 6, 5 | **0** | 0.333 | 705 |
| 5 | How can I build APIs using TypeScript? | 5 | 5, 4 | 5, 4, 3 | 0.333 | 0.333 | 677 |
| 6 | How do databases handle simultaneous transactions? | 6 | 6, 1, 2 | 6, 2, 1 | 0.333 | 0.333 | 641 |
| 7 | How do I organize relational data efficiently? | 2 | 6, 1 | 4, 2, 6 | **0** | 0.333 | 657 |
| 8 | How can I create a semantic search system? | 4 | 4, 1 | 4, 3, 2 | 0.333 | 0.333 | 622 |
| 9 | What is database consistency across servers? | 6 | 6, 2 | 6, 2, 1 | 0.333 | 0.333 | 682 |
| 10 | Recommend a beginner-friendly database book. | 1 | 1, 2, 6 | 1, 2, 4 | 0.333 | 0.333 | 631 |
| | **Average** | | | | **0.267** | **0.333** | 675 |

| Measure | FTS5 | Semantic |
|---|---|---|
| Gold book in the Top-3 | 8 of 10 | 10 of 10 |
| Gold book ranked first | 8 of 10 | 9 of 10 |
| Questions with fewer than 3 results | 6 of 10 | 0 of 10 |

### Observations

- **Question 4** ("How do LLMs use external documents?"): FTS5 returned nothing, because none of the words appear in the stored descriptions. *Practical RAG* says "vector databases and large language models". Semantic search matched on meaning and ranked it first.
- **Question 7** ("How do I organize relational data efficiently?"): FTS5 matched on the wrong words and returned books 6 and 1, missing *Database Design* (book 2). Semantic search returned it second, behind *Practical RAG*, so it did not rank first either.
- **Question 2:** FTS5 found only one book. Semantic search always fills all three slots, which is a strength for recall and a weakness for precision (see section 4).
- **Latency:** semantic search took 622–773 ms per query (embedding call, Vectorize query and three D1 lookups). The FTS5 query time was not measured.
- **Small sample:** ten questions over six books is not a benchmark. The results show the behaviour, not a reliable accuracy figure.

## 3. RAG answers

`GET /api/recommend` retrieves the Top-3 documents, builds a context string from D1 (title, author, price, stock, document text), and asks `@cf/meta/llama-3.3-70b-instruct-fp8-fast` to answer using only that context. Full answers are in `step8-rag-answers.txt`.

| # | Sources (book IDs) | Recommended | Grounded? | Latency (ms) |
|---|---|---|---|---|
| 1 | 1, 4, 2 | SQL for Beginners [D1], Database Design [D2] | Yes | 2924 |
| 2 | 2, 4, 1 | Database Design [D2], SQL for Beginners [D1] | Yes | 6817 |
| 3 | 3, 4, 1 | Introduction to AI [D3], Practical RAG [D4] | Partly: see note A | 2280 |
| 4 | 4, 6, 5 | Practical RAG [D4] | Yes | 5753 |
| 5 | 5, 4, 3 | TypeScript Projects [D5] | Yes | 1825 |
| 6 | 6, 2, 1 | Distributed Data Systems [D6], Database Design [D2] | Partly: see note B | 5435 |
| 7 | 4, 2, 6 | Database Design [D2], Distributed Data Systems [D6] | Yes | 3298 |
| 8 | 4, 3, 2 | Practical RAG [D4], Introduction to AI [D3] | Partly: see note C | 2209 |
| 9 | 6, 2, 1 | Distributed Data Systems [D6], Database Design [D2] | Yes | 2012 |
| 10 | 1, 2, 4 | SQL for Beginners [D1], Database Design [D2] | Yes | 2279 |

RAG latency ranged from 1.8 s to 6.8 s. These are single measurements.

### Grounding checks

| Check | Result |
|---|---|
| Did every cited document exist in `book_documents`? | Yes. Every cited `[D#]` was one of D1–D6. |
| Does each cited book exist in `books`? | Yes. |
| Do prices and stock match D1? | Yes. Prices and stock come from the `sources` array, which is read from D1 on every request. The one answer that quoted them (the price and stock test in [update-reindex-report.md](update-reindex-report.md)) matched exactly. |
| Were recommendations supported by the source text? | Mostly. Three answers added a small claim the text does not contain (notes below). |
| Did the LLM invent a title or store policy? | No invented titles, prices or policies in any of the ten answers. |

- **Note A (question 3):** the answer says *Practical RAG* covers large language models "which are related to machine learning". The document says only "semantic search systems with vector databases and large language models". The link to machine learning is the model's own inference.
- **Note B (question 6):** the answer says *Database Design* "includes a section on transactions". The text says it covers "transactions", with no mention of a section.
- **Note C (question 8):** the answer says *Introduction to AI* "may provide a foundational understanding" that is "useful in designing such a system". The document does not mention semantic search.

None of these are invented facts, but each goes slightly beyond the retrieved text, which the system prompt says not to do.

### An LLM failure

During the first loop of ten questions, and again during a second run, **question 1 returned corrupted output**: the string "Kauf partic" repeated until the 320-token limit was reached (21.9 s and 54.2 s, against about 2 s for normal answers). The retrieval for that question was correct (books 1, 4, 2), so the fault was in generation. After a throwaway warm-up request, question 1 returned a normal, correct answer, and it was clean in the final run.

Both failures were the first request in the loop after a period of inactivity. I did not confirm the cause, so this is an observation, not a diagnosis. It shows why the lab lists output verification as a future improvement: the application returned the corrupted text to the caller as if it were a valid answer.

## 4. Hallucination test

**Question:** "Do you sell Advanced Quantum Computing?" (the book does not exist in the catalog).

**Answer:** "I cannot confirm if we sell 'Advanced Quantum Computing' as it is not mentioned in the provided catalog context [D3], [D4], or [D5]."

| Retrieved | Score |
|---|---|
| Practical RAG (book 4) | 0.469 |
| Introduction to AI (book 3) | 0.457 |
| TypeScript Projects (book 5) | 0.444 |

- **Result:** the system did not claim to sell the book and did not invent a price. The system prompt ("If there is not enough evidence, clearly say you cannot confirm") worked in this test.
- **Retrieval still returns three books.** Vectorize always returns the Top-K nearest vectors, whether or not any are relevant. Here the scores (0.44–0.47) are lower than the top matches on questions that do have a relevant book (roughly 0.6–0.8 in this lab), but the application applies no score threshold, so unrelated books reach the LLM every time.
- **Remaining risk:** the protection is the prompt only. A model that ignored it, or a poorly matching but plausible-looking retrieved book, could produce a confident wrong recommendation.
- **Possible mitigations (not implemented):** a minimum similarity score before calling the LLM, checking that every cited `[D#]` is in the retrieved set, and using SQL to confirm exact conditions such as price or stock.

## 5. Limitations

| Issue | Observed in this lab |
|---|---|
| Top-K always returns results | The "Quantum Computing" test returned three unrelated books. |
| Exact constraints | "Under $25 and in stock" cannot be enforced by Vectorize. It needs a SQL filter in D1. |
| Output quality | One corrupted answer was returned as a normal response (section 3). |
| Run-to-run variation | Re-running the same question gave slightly different wording. |
| Index maintenance | Description changes need a manual reindex. See [update-reindex-report.md](update-reindex-report.md). |
| Sample size | Six books and ten questions. |

## 6. Conclusion

On this small catalog, semantic search put the correct book in the Top-3 for all ten questions and ranked it first for nine, while FTS5 managed eight and eight. FTS5 failed on questions where the wording differed from the stored text. Semantic search found books by meaning but cannot tell when nothing relevant exists, and the LLM's grounding depends on prompt instructions that held in ten answers and one hallucination test but were stretched slightly in three answers. D1 remains the source of truth for prices, stock and document text, and Vectorize only decides which documents to fetch.
