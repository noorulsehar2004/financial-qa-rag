# Week 4 — Financial Document QA Bot (RAG Foundations)

A bot that answers real questions about real SEC 10-K filings, with citations to the exact
filing and section — and correctly declines instead of guessing when it doesn't know.

**Dataset:** Apple, Microsoft, and Tesla 10-K filings, pulled directly from [SEC EDGAR](https://www.sec.gov/cgi-bin/browse-edgar).

---

## What's in this notebook

1. **Document ingestion** — real 10-K filings downloaded from SEC EDGAR, cleaned from Inline
   XBRL/HTML into readable text.
2. **Two chunking strategies, compared** — naive fixed-size character chunking vs.
   paragraph/sentence-boundary-aware recursive chunking, tested directly on the same financial
   tables to show a real, evidence-backed difference.
3. **Embeddings + distance metrics** — `all-MiniLM-L6-v2` sentence embeddings, FAISS index,
   L2 vs. cosine similarity compared side by side on identical queries.
4. **Citations at the filing + section level** — every retrieved answer is tagged with which
   company's filing and which 10-K "Item" section (e.g., Item 1A Risk Factors, Item 7 MD&A) it
   came from.
5. **"Not found" fallback** — declines instead of guessing when a question names a company not
   in the document set, asks about a year outside the filing's coverage, or is simply
   unrelated to the documents.
6. **Generalized ingestion pipeline** — company names, chunking, and section-tagging are all
   detected automatically from whatever `.htm` files are in a folder — no hardcoded company
   names. Pointing this at a new folder of filings works without touching the code.
7. **Saved, reloadable index** — the FAISS index and all chunk/metadata are saved to disk and
   reloaded fresh in a separate cell, to confirm the system works as a standalone artifact
   without needing to re-run ingestion from scratch.

## Key findings

**Chunking comparison:** on the same financial table (Apple's segment revenue breakdown),
fixed-size chunking started mid-word and only preserved the table header by chance. Recursive
chunking started at a clean sentence boundary and preserved the header by design. Confirmed with
a second example where fixed-size chunking lost a table header entirely.

**A real, recurring retrieval limitation:** found and documented twice independently — Tesla's
total revenue and Apple's R&D spending both live in terse, table-heavy chunks that ranked
*below* several topically-related-but-wrong chunks (headcount, cash flow, competitor discussion)
at a typical k=3-5. Both were only recoverable with a wider k (rank 7 of 8, and rank 8 of 30,
respectively). This shows **similarity score is not the same as "contains the answer."**

**Metadata filtering caught two real bugs during testing:**
- A question about Amazon (not in the document set) initially returned Microsoft's revenue
  figures with high confidence — fixed by checking retrieved results' company against the
  company actually named in the query.
- A question about Apple's 2010 revenue returned 2025 data with no warning — fixed with a
  year-range check against each filing's actual coverage.
- Generalizing the pipeline to auto-detect company names (instead of hardcoding "Apple",
  "Microsoft", "Tesla") silently broke the company-matching filter, since "Apple Inc." no
  longer matched a query saying just "Apple" — caught and fixed by normalizing legal suffixes
  before matching.

## Full write-up

The complete chunking comparison and a one-page model risk note are included as markdown cells
at the end of the notebook.

## Files

- `RAGFoundation_FinancialDocuments.ipynb` — full notebook, code + results + write-up
- `financial_qa_index.faiss` — saved FAISS vector index (1,345 chunks across 3 filings)
- Chunk/metadata pickle file not included here due to size — regenerate via the notebook's
  ingestion cells, or request separately.
