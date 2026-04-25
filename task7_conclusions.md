# Task 7 — Conclusions Review

Reviewer-pass over the interpretations and wrap-up in cells 45, 48, and 50 of `assignemnt2_rag_ilay_nisim_boris_kogan.ipynb`. Numbers below are recomputed from the cached results on disk (`task6_*.json`, `task7_exp1_chunk1500_*.json`, `task7_exp2_hyde_*.json`).

## Verified aggregate numbers

| run | correctness | faithfulness (n=20) | page-hit@1 | page-hit@3 | page-hit@5 |
|---|---|---|---|---|---|
| baseline (chunk=1000) | 0.190 (19/100) | 0.509 | 0.200 | 0.330 | 0.400 |
| exp1 chunk=1500 | 0.210 (21/100) | 0.309 | 0.140 | 0.260 | 0.350 |
| exp2 HyDE | **0.230 (23/100)** | 0.437 | **0.240** | **0.410** | **0.500** |

## Review of Experiment 1 interpretation (cell 45)

**Two of the three direction calls are reversed.** Cell 45 needs this correction:

| your bullet | reality | fix |
|---|---|---|
| "Page-hit@k … improved a bit" | got *worse* on every k (0.200→0.140 / 0.330→0.260 / 0.400→0.350) | "got worse on every k" |
| "Correctness … got worse" | improved slightly (0.190→0.210, +2 questions) | "improved slightly (+2)" |
| "Faithfulness … got worse" | correct (0.509→0.309) | keep |

The conclusion sentence — *"Going larger didn't improve since the problem is not the size of retrieval, it is vocabulary mismatch"* — is correct in spirit, but the supporting bullets above were inverted, so the cell currently contradicts itself. Once the bullets are flipped, the story reads cleanly: bigger chunks averaged more topics into one embedding, so retrieval discrimination dropped (page-hit ↓); the lone gain on correctness comes from the few questions where retrieval still hit the right page and the larger chunk gave the generator more table context; faithfulness fell because longer chunks invite more unsupported claims per answer.

## Review of Experiment 2 interpretation (cell 48)

**All three direction calls are correct.** No fixes needed.

The JPM 00206 spot-check note is also accurate. Re-extracted from `task7_exp2_hyde_rag.json` for the record:

- **Question:** *Are JPM's gross margins historically consistent (not fluctuating more than roughly 2% each year)?* (evidence: JPMORGAN_2022_10K, page 2)
- **Baseline top-5:** zero JPMORGAN_2022_10K results (top-1 was a Best Buy chunk).
- **HyDE top-5:** all five results from JPMorgan filings, with `JPMORGAN_2022_10K` at rank #1 (page 48). Pages retrieved: 48, 4, 15, 4, 166 — none are the actual evidence page (page 2).
- **Verdict:** still `incorrect`. HyDE fixed the document-level miss but not the page-level miss, exactly as Task 5's small-vector sanity check predicted: "gross margin" doesn't appear verbatim in the 10-K, so even a vocabulary-shifted query can't pinpoint the right page in a 200+-page filing.

This is also why HyDE's faithfulness fell despite retrieval and correctness rising: the 100-question average pulls in many questions where HyDE retrieved a *plausible-looking* but irrelevant page, and the generator was more willing to write a confident-but-unsupported answer when the chunks looked topical.

## Review of wrap-up (cell 50)

**"Where does the pipeline fail most"** — the answer (retrieval is the dominant failure, but it propagates into generation; HyDE confirms generation has independent headroom because faithfulness didn't follow) is well-supported by the numbers. Page-hit@5 of 0.500 even on the best run means half the questions never see the evidence page, so retrieval is structurally bottlenecking.

**"What I'd try next" — review of the 6 directions:**

| # | direction | recommendation |
|---|---|---|
| (a) | combine HyDE with larger chunk size | **Keep — but flag the trade-off.** HyDE and larger chunks attack *different* bottlenecks (retrieval vocab vs. generator context per chunk), so the gains could compound on correctness even if page-hit slips. Worth running as a future experiment with the trade-off explicit. |
| (b) | improve generation model + prompt (incl. structured-output forcing line-item citations) | **Keep — strongest.** HyDE's faithfulness drop (0.509 → 0.437) despite retrieval gains is direct evidence the generator is the bottleneck for grounding. **Merged from old (b) + (f)** since structured-output is a specific instance of "improve the generation prompt", not a separate category. |
| (c) | hybrid retrieval (BM25 + dense) | **Keep.** Directly attacks vocabulary mismatch by matching surface forms — complements HyDE rather than overlapping it. |
| (d) | question decomposition for multi-hop numeric questions | **Keep, but lower priority.** We don't have a count of how many of the 100 questions are actually multi-hop, so the upside is uncertain. |
| (e) | table-aware PDF loader | **Keep — strongest single fix for FinanceBench.** PyPDFLoader collapses tables to whitespace-stripped text; on a dataset where most evidence lives inside balance sheets and income statements, this is likely the single largest available win. |

**Final list (5 items):** (a), (b)+(f) merged, (c), (d), (e).

## Main takeaway

> **You can't build a good RAG without knowing your data.** Every failure mode we hit traces back to a domain gap, not an algorithmic one:
> - *Financial language*: questions ask about "gross margin" or "working capital" — terms that don't appear verbatim in the filings. Knowing this would have driven us toward query reformulation (HyDE) or a finance-specific re-tokenization from the start.
> - *PDF structure*: PyPDFLoader flattens tables, but the evidence for most FinanceBench questions lives *inside* tables. A table-aware loader is not an optimization — it's a prerequisite.
> - *Specific fields*: to answer "working capital", a system needs to know it's `current assets − current liabilities`, retrieve both line items, and compute. A general-purpose retriever can't do this without either a finance-specific decomposer or pre-built field-extraction over the filings.
>
> Generic RAG tuning (chunk size, k, embeddings) gave us small, mostly-cancelling deltas. The lever that matters is encoding domain knowledge into the pipeline itself.

## Cell 45 — paste-ready replacement

```markdown
**Interpretation — Experiment 1 (chunk_size=1500):**

- Page-hit@k vs baseline (chunk=1000): got worse on every k (h@1 0.200→0.140, h@3 0.330→0.260, h@5 0.400→0.350).
- Correctness vs baseline: improved slightly (0.190 → 0.210, +2 questions).
- Faithfulness vs baseline: got worse (0.509 → 0.309).
- Context: the earlier chunk_size=500 attempt was also worse on every metric — fragmenting tables hurt. Going *larger* didn't help retrieval either: bigger chunks average more topics into a single embedding and dilute discrimination, so page-hit dropped. The small correctness bump comes from the subset where retrieval still hit the right page and the longer chunk gave the generator more table context. Faithfulness fell because longer chunks invite more unsupported claims per answer. Net read: chunk size is not the lever — vocabulary mismatch is.
```
