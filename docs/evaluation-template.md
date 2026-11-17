# Week 12 RAG Evaluation Report

| Test | Customer Question | Relevant Book IDs (gold labels) | FTS5 Top-3 IDs | Vector Top-3 IDs | Precision@3 | RAG grounded? | Latency(ms) |
|---|---|---|---|---|---|---|---|
| 1 | Beginner SQL book | 1,2 | | | | | |
| 2 | Simultaneous database updates | 6 | | | | | |
| 3 | Creating APIs with TypeScript | 5 | | | | | |
| 4 | Retrieval augmented generation | 4 | | | | | |
| 5 | Introductory machine learning | 3 | | | | | |
| 6 | Database normalization | 2 | | | | | |
| 7 | Consistency across servers | 6 | | | | | |
| 8 | Data stored in relational tables | 1,2 | | | | | |
| 9 | Large language models and search | 4 | | | | | |
| 10 | Available AI title under $25 | 3 | | | | | |

Precision@3 = relevant retrieved / 3. This is a classroom metric; define gold labels before searching. Check actual returned data. For price/stock filtering, semantic retrieval alone cannot enforce catalog numeric constraints; use D1 validation/filtering in a production system.

## Grounding checks
- Did the cited document exist in book_documents?
- Does each cited book exist in books?
- Do prices and stock match D1?
- Are recommendations factually supported by source text?
- Did LLM invent a nonexistent title or store policy?
- What changed when a document was updated and reindexed?
