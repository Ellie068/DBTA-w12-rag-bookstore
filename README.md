# Week 12: Online Bookstore — Vectorize + D1 + RAG

## Lesson relationship
Week 10: FTS5 → Week 11: embeddings and manual cosine search → Week 12: Vectorize and RAG → Week 15: production capstone.
This is a classroom demonstration, not a production-ready bookstore.

## 0. Install

```bash
npm install
node -v
npx wrangler login
```

## 1. Cloud services (create once, reuse existing resources)

```bash
npx wrangler d1 list
npx wrangler d1 create w12-rag-db
npx wrangler vectorize create w12-rag-vectors --dimensions=384 --metric=cosine
```

Put the `database_id` returned by D1 create into the `d1_databases` entry in `wrangler.jsonc` (this project's file already contains the ID for `w12-rag-db`; use your own if you create a new database). If the index exists, do not recreate it. Ensure its dimensions are 384 and metric cosine.

The commands in this README use bash syntax. On Windows PowerShell, use `curl.exe` instead of `curl`, and set the tokens with `$DEMO = "..."` and `$ADMIN = "..."` instead of `export`.

**WARNING: This project uses `remote:true` for D1. The next commands MODIFY your actual remote D1 database. Use a dedicated student database, never a database with real orders.**

## 2. SQL (one-time or idempotent initialization)

```bash
npx wrangler d1 execute w12-rag-db --remote --file=./sql/schema.sql
npx wrangler d1 execute w12-rag-db --remote --file=./sql/seed.sql
npx wrangler d1 execute w12-rag-db --remote --file=./sql/rag.sql
npx wrangler d1 execute w12-rag-db --remote --command="SELECT id,title,stock FROM books;"
npx wrangler d1 execute w12-rag-db --remote --command="SELECT document_id,book_id,title FROM book_documents;"
```

If upgrading an existing Week 11/15 project with other tables, inspect the schema and run ONLY the missing SQL statements; do not overwrite a production schema.

## 3. Set secrets

```bash
cp .dev.vars.example .dev.vars
```

Edit both tokens to unrelated long values. Never add `.dev.vars` to Git.

## 4. Run and index

```bash
npm run types
npm run typecheck
npm run dev
```

Open another terminal:

```bash
export DEMO_TOKEN='the-token-from-.dev.vars'
export ADMIN_TOKEN='the-other-token-from-.dev.vars'
curl http://localhost:8787/api/health
curl http://localhost:8787/api/books
curl http://localhost:8787/api/documents
curl -X POST http://localhost:8787/api/admin/index -H "Authorization: Bearer $ADMIN_TOKEN"
```

Vectorize indexing is asynchronous; search may return zero results immediately after upsert. Try again after indexing has propagated.

```bash
curl -G http://localhost:8787/api/fts -H "Authorization: Bearer $DEMO_TOKEN" --data-urlencode "q=database"
curl -G http://localhost:8787/api/semantic -H "Authorization: Bearer $DEMO_TOKEN" --data-urlencode "q=I want to learn SQL from scratch"
curl -G http://localhost:8787/api/recommend -H "Authorization: Bearer $DEMO_TOKEN" --data-urlencode "q=Recommend an easy database book and tell me the price and stock"
```

Open http://localhost:8787 to try the browser interface. Only enter `DEMO_TOKEN`, never the admin token.

## 5. Data freshness lab

```bash
npx wrangler d1 execute w12-rag-db --remote --command="UPDATE books SET price_cents=2599,stock=1 WHERE id=1;"
```

Search again: price and stock now come from D1 without recomputing embeddings. If editing book text, also update its `book_documents` content, then POST `/api/admin/index` to rebuild vectors. For deleted documents remove their corresponding vector IDs using `env.VECTORIZE.deleteByIds(["doc:1"])` in a secured admin operation or use Wrangler's vectorize delete-vectors command.

## 6. Optional deployment

```bash
npx wrangler secret put DEMO_TOKEN
npx wrangler secret put ADMIN_TOKEN
npm run deploy
```

This public classroom demo uses bearer tokens but lacks user accounts, per-user authorization and rate limiting. Turn it off after class.


## Compatibility with the earlier Vectorize-only Week 12 lab

The earlier Vectorize-only file uses IDs such as `"1"` for book vectors. This RAG document-based lab uses `"doc:1"`, etc. If the same `w12-rag-vectors` index has legacy IDs, they may occupy Top-K results and get skipped by the RAG lookup. For a dedicated classroom index that contains only your earlier sample book IDs, delete those six older IDs before indexing:

```bash
npx wrangler vectorize delete-vectors w12-rag-vectors --ids 1 2 3 4 5 6
```

**Only delete vectors that you own; never run this on a shared or production index.** Alternatively create a new, dedicated 384D cosine index and change `index_name` accordingly. Vectorize changes can take a few seconds to become queryable.

If FTS5 is added to a database where book rows already exist, an explicit one-time rebuild may be required:

```bash
npx wrangler d1 execute w12-rag-db --remote --command="INSERT INTO books_fts(books_fts) VALUES('rebuild');"
```

## 7. Model choices

Embeddings: `@cf/baai/bge-small-en-v1.5` (384D). Generation: `@cf/meta/llama-3.3-70b-instruct-fp8-fast` (currently documented and higher-cost than the lesson's fp8 model). The uploaded RAG material uses `@cf/meta/llama-3.1-8b-instruct-fp8`, which has a documented planned deprecation on November 20, 2026. Inspect Cloudflare's model catalog before instruction, and adjust `GENERATION_MODEL` if needed. All Workers AI usage is subject to limits and pricing.

## 8. Assignment

Run 10 queries. Log expected relevant Book IDs, Top-3, Precision@3, latency, and any unsupported LLM claim. Submit README, SQL, TypeScript, screenshots, evaluation and system diagram.

## 9. Results, reports and screenshots

Tests were run on 2026-11-17. Reports are in `docs/`:

| Document | Contents |
|---|---|
| [docs/architecture.md](docs/architecture.md) | System architecture diagrams (components, data model, indexing flow, RAG query flow) |
| [docs/evaluation-report.md](docs/evaluation-report.md) | FTS5 vs semantic search on ten questions (Precision@3), RAG grounding checks, hallucination test, limitations |
| [docs/update-reindex-report.md](docs/update-reindex-report.md) | Price/stock change (no reindex) and description change (reindex needed), synchronization problems |
| [docs/step7-fts-vs-semantic.txt](docs/step7-fts-vs-semantic.txt) | Raw output: FTS5 and vector Top-3 for the ten questions |
| [docs/step8-rag-answers.txt](docs/step8-rag-answers.txt) | Raw output: RAG answers for the ten questions |

Screenshots (in `docs/screenshots/`):

| # | File | Shows |
|---|---|---|
| 1 | [01-web-fts5.png](docs/screenshots/01-web-fts5.png) | Web app, FTS5 keyword search for "I want a book that explains how data remains consistent across multiple servers." |
| 2 | [02-web-semantic.png](docs/screenshots/02-web-semantic.png) | Same question, semantic search (Vectorize + D1 JOIN) with scores, prices and stock |
| 3 | [03-web-rag.png](docs/screenshots/03-web-rag.png) | Same question, RAG recommendation with the `sources` array |
| 4 | [04-fts-vs-semantic-table.png](docs/screenshots/04-fts-vs-semantic-table.png) | FTS5 vs semantic Top-3 for all ten test questions, with latency |
| 5 | [05-rag-ten-answers.png](docs/screenshots/05-rag-ten-answers.png) | RAG answers, sources and latency for the ten questions |
| 6 | [06-hallucination-test.png](docs/screenshots/06-hallucination-test.png) | "Do you sell Advanced Quantum Computing?": the answer says it cannot confirm |
| 7 | [07-index-call.png](docs/screenshots/07-index-call.png) | `POST /api/admin/index` returning `"indexed": 6` |
| 8 | [08-price-stock-change.png](docs/screenshots/08-price-stock-change.png) | Updated price ($25.99) and stock (1) appear in the answer with no reindexing |
| 9 | [09-reindex-before.png](docs/screenshots/09-reindex-before.png) | After the description change, **before** reindexing: score for doc 1 is 0.6091 |
| 10 | [10-reindex-after.png](docs/screenshots/10-reindex-after.png) | **After** reindexing: score for doc 1 is 0.6597 |

Summary of results: semantic search placed the expected book in the Top-3 for 10 of 10 questions (FTS5: 8 of 10). The RAG system refused to confirm a book that is not in the catalog. Prices and stock change instantly, while description changes need `POST /api/admin/index`.
