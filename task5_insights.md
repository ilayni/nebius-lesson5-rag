# Task 5 – Debug: small vector store sanity check

## Motivation

The Task 5 discussion ended with a debugging tip from the assignment: build a FAISS index over **only 1–2 entire PDFs** and see what gets retrieved. The goal was to isolate the failure mode on `financebench_id_00206` (JPM gross-margin question), which retrieval missed in Tasks 3–5. Two competing hypotheses:

1. **Chunk-granularity issue** — the right doc is in the index but the right *page* never wins the top-k.
2. **Vocabulary mismatch** — the query words (`"gross margin"`) simply don't appear in the filing, so the embedder has nothing to anchor to.

## What we did

Completely independent cell, new variable names (prefixed `small_`), no mutation of `df`, `vectorstore`, `embeddings`, or `answer_with_rag`:

- Loaded a single PDF: `JPMORGAN_2022_10K.pdf` (382 pages).
- Same chunking as Task 3: `RecursiveCharacterTextSplitter(chunk_size=1000, chunk_overlap=150)` → 1965 chunks.
- Same embedder: `BAAI/bge-small-en-v1.5`.
- Built a fresh in-memory FAISS index over those 1965 chunks (1965 vectors total).
- Ran the full `00206` question as the query and retrieved top-8 with `similarity_search_with_score`.
- Separately scanned **every page of the raw PDF** (not just retrieved chunks) for the literal phrase `"gross margin"`.

Evidence page from FinanceBench for `00206`: **page 2**.

## Results

Top-8 retrieved pages: **208, 28, 27, 78, 196, 344, 151, 97**. Page 2 (the evidence page) does not appear. Distances are bunched between ~0.537–0.567 — a flat, low-confidence distribution with no strong winner.

Literal-phrase scan across all 382 pages: `"gross margin"` appears on **zero pages** of the filing.

## Conclusion

This is a **vocabulary mismatch**, not a chunk-granularity issue. Two layers of evidence:

1. With 40+ unrelated filings stripped out, the embedder still can't route the query to page 2 — so noise from other docs was never the blocker.
2. The phrase doesn't occur in JPM's 10-K *at all*, so even a hypothetical perfect reranker that handed page 2 back at rank 1 wouldn't rescue the answer: under our strict "answer only from context" prompt, the generator would still refuse, because the context lacks the phrase/concept being asked about.

In other words, the right content (page 2 describes JPM's business and revenue structure, implying gross margin is inapplicable for a bank) is *present*, but the bridge word isn't. BGE-small, a general-purpose embedder, can't cross that semantic gap; banks report **net interest margin**, **net interest income**, **non-interest income**, **efficiency ratio** — not "gross margin."

## Possible solutions

Ordered roughly from cheapest to most invasive:

1. **Query expansion / rewriting.** Rewrite the user's query with an LLM *before* embedding it for retrieval, adding domain synonyms: `"gross margin"` → `"gross margin, profitability metric, net interest margin, revenue structure, applicability to banks"`. Cheap, no re-indexing. The generator later still sees the original question (often the original + the rewrite), so the user-facing answer is unchanged; only the search key is enriched. Most effective for questions that say "if X isn't relevant, explain why" — the rewriter can pick that clause up and push the search toward the company's business description.

2. **HyDE (Hypothetical Document Embeddings).** Another LLM drafts a hypothetical answer to the query *first*; we embed that hypothetical answer and retrieve against it instead of against the raw query. A natural concern here is: *"won't we be searching with hallucinated numbers?"* — yes, and it still works, for three reasons:

   - The hypothetical is used **only as a retrieval key**, never returned to the user. The real answer is still generated from the *real* chunks the search brought back.
   - Embeddings match on **vocabulary and shape**, not on truth. A hallucinated JPM answer will say things like *"JPMorgan's net interest margin was X%, non-interest income grew Y%…"* — the numbers are wrong, but `"net interest margin"` and `"non-interest income"` are exactly the terms that **do** appear on page 2 of the 10-K. The short, vocabulary-poor query `"gross margin"` can't match those pages; a fluent fake answer can.
   - Flow: bad-vocab query → fake answer with right-vocab → cosine-matches the real evidence page → real chunks go to the generator → real, grounded answer. The fake numbers are discarded before generation.

   HyDE's own failure mode is different: it helps when query and evidence are phrased very differently, and it hurts when the hypothetical is so off-topic that it pulls retrieval the wrong way. For `00206`, where the gap is purely vocabulary, it's a natural fit. Cost: one extra LLM call per query, no re-indexing.

3. **Hybrid retrieval (dense + BM25).** Wouldn't help here (the lexical term is also absent), but helps adjacent cases where the phrase *is* in the document but the dense embedder scores it below a fluent-but-irrelevant passage.

4. **Prompt relaxation for "explain why N/A" questions.** Allow the generator to say "this metric is not applicable to this industry" from its own reasoning when context supports the inapplicability even without the exact phrase. Breaks the strict no-outside-knowledge guarantee, so needs care — best combined with (1) or (2) so refusals only happen when retrieval genuinely fails.

5. **Domain-specific embedder.** A finance-tuned encoder (e.g., FinBERT-style) may already map "gross margin" ≈ "net interest margin" for banks. Heaviest option: re-embed the whole corpus.

For this assignment, **(1) query rewriting** is the natural Task 7 experiment — single variable changed (prompt before retrieval), no re-indexing, and it directly targets the failure mode this debug run identified.
