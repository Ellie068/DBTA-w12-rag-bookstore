# Document Update and Reindexing Test Report

Online Bookstore: D1 + Vectorize + Workers AI. Tests were run on 2026-11-17 against the local Worker (`wrangler dev`) connected to the remote D1 database `w12-rag-db` and the Vectorize index `w12-rag-vectors`.

## 1. Purpose

The system stores each book's text in two places:

| Data | Where it lives | How it is used |
|---|---|---|
| Price, stock, description, document text | D1 (`books`, `book_documents`) | Read from D1 on **every** request, through a SQL JOIN |
| Embedding of the document text | Vectorize, ID `doc:<document_id>` | Used only to decide **which** documents match a question |

This report tests what happens when D1 changes: which changes show up immediately, and which need the vectors to be regenerated.

## 2. Starting state (Book 1, document 1)

| Field | Value |
|---|---|
| `books.price_cents` / `books.stock` | 1999 ($19.99) / 3 |
| `books.description` and `book_documents.content` | "Learn relational databases and SQL fundamentals with practical exercises." |
| Vector | `doc:1`, built from the text above |
| Vectors in the index | 6 (one per document) |

## 3. Test A: price and stock change (no reindexing)

**Change:**

```sql
UPDATE books SET price_cents=2599, stock=1 WHERE id=1;
```

**Query:** `GET /api/recommend?q=How much is SQL for Beginners and is it in stock?`

**Result:**

- Answer: "The book 'SQL for Beginners' is priced at $25.99 and we have 1 copy in stock [D1]."
- `sources[0]` for book 1: `price_cents: 2599`, `stock: 1`.

**Conclusion:** the new price and stock appeared on the very next request, with no embedding call and no reindexing. Prices and stock are not part of the document text, and the application reads them from D1 at query time. This is the reason prices and inventory are kept out of `book_documents`.

## 4. Test B: description change (reindexing needed)

**Change** (both tables, because the RAG code reads `book_documents.content`):

```sql
UPDATE books
SET description='A beginner-friendly guide to SQL, relational tables, joins, and database queries.'
WHERE id=1;

UPDATE book_documents
SET content=(SELECT description FROM books WHERE id=1), updated_at=CURRENT_TIMESTAMP
WHERE document_id=1;
```

**Query used for all three runs below:** `GET /api/semantic?q=SQL joins and database queries`

| Run | Condition | #1 | #2 | #3 | Score of doc 1 |
|---|---|---|---|---|---|
| 1 | Before the update | doc 1 | doc 2 | doc 4 | 0.6091 |
| 2 | D1 updated, **vector not yet regenerated** | doc 1 | doc 2 | doc 4 | **0.6091** (unchanged) |
| 3 | After `POST /api/admin/index` and a 30-second wait | doc 1 | doc 2 | doc 4 | **0.6597** |

Run 1 was reconstructed from the restore check in section 5; the original-state scores were not captured before the change, and the 0.6091 figure matches run 2 because the vector was identical.

**What changed and what did not:**

- In run 2 the `content` field returned by the API already showed the **new** description, because it is read from D1 live. The **score** was still 0.6091, because it is computed from the old vector. D1 and the vector index disagreed.
- After the reindex, the score for doc 1 rose from 0.6091 to 0.6597 (+0.051), because the new text mentions "joins" and "queries", which the query also does.
- Docs 2 and 4 kept the same scores (0.5814 and 0.5738), as expected: their text and vectors did not change.
- The ranking did not change, because doc 1 was already first on this query. The effect is visible only in the score. A bigger edit, such as rewriting a description to cover a different topic, could change the ranking, but this was not tested.

## 5. Restoring the original data

Book 1 was set back to price 1999, stock 3 and the original description, `book_documents` was updated to match, and `POST /api/admin/index` was run again.

Verified afterwards:

- D1: `price_cents = 1999`, `stock = 3`, and `books.description` equals `book_documents.content` (the original text).
- Search: `SQL joins and database queries` scored doc 1 at **0.6091** again, the same value as before the change, so the vector was regenerated from the original text.

## 6. The indexing delay

Shortly after the first indexing run (`"indexed": 6`), semantic search returned an **empty result list**, and `/api/recommend` answered "No relevant catalog documents found." When the index was checked a few minutes later, Vectorize reported 6 vectors, and the same searches then returned results. After a reindex, new vectors are therefore not guaranteed to be searchable immediately, which matches the note in the lab. The tests above waited 30 seconds after reindexing.

## 7. Synchronization problems

| Problem | Evidence |
|---|---|
| **Stale vectors.** The vector does not change when D1 text changes. Until someone calls the indexing API, searches rank by the old meaning while displaying the new text. | Test B, run 2 |
| **Eventual consistency.** New vectors take time to become searchable. | Section 6 |
| **Manual process.** Nothing triggers reindexing. Someone has to remember to call `POST /api/admin/index`, with the admin token. | Test B |
| **Whole-table reindex.** The indexing function embeds every document (up to 50) on each call, even when only one changed. | Code (`indexDocuments` in `src/index.ts`), not measured |
| **Deleted documents.** The code only upserts. A document deleted from D1 would keep its vector. Search drops matches whose D1 row is missing, so the result would be one fewer useful result in the Top-3. | Code reading only; **not tested** |
| **Two places to update.** The description lives in `books` and `book_documents`, and both had to be changed by hand. | Test B |

## 8. Possible improvements (not implemented)

- Put a Cloudflare Queue between document edits and indexing, so a change to `book_documents` triggers re-embedding of just that document, as suggested in the lab.
- Compare `book_documents.updated_at` with the last indexed time and re-embed only changed rows.
- Delete the matching vector when a document is deleted.
- Fill `book_documents.content` from `books.description` with a database trigger, so only one table needs editing.
