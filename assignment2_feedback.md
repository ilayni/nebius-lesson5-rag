# Assignment 2 — RAG · Grading Feedback

**Students:** Ilay Nisim & Boris Kogan
**Reviewed against:** `Assignment_2_RAG.md`
**Submission files inspected:**
- `assignemnt2_rag_ilay_nisim_boris_kogan.ipynb` (51 cells, code + markdown discussions)
- `assignment2_naive_generation_ilay_nisim_boris_kogan.xlsx`
- `assignment2_run_and_compare_ilay_nisim_boris_kogan.xlsx`
- `assignment2_evaluation_ilay_nisim_boris_kogan.xlsx`
- `assignment2_improvement_cycles_ilay_nisim_boris_kogan.xlsx`

## Final grade: **100 / 100** (Bonus not attempted; optional)

| Task | Awarded | Max |
|------|---------|-----|
| 1 — Naive generation | 10 | 10 |
| 2 — RAG reminder | 5 | 5 |
| 3 — Embed documents | 15 | 15 |
| 4 — RAG pipeline | 25 | 25 |
| 5 — Run and compare | 10 | 10 |
| 6 — Evaluation | 20 | 20 |
| 7 — Improvement cycles | 15 | 15 |
| **Subtotal** | **100** | **100** |
| Bonus — Multi-scale chunking | 0 | 10 |

The assignment text says you "will not be graded on the quality of the scores" — only on pipeline implementation and what you did to improve results. Both are clearly demonstrated.

---

## Task 1 — Naive generation (10 / 10)

**Rubric checks**
- ✅ Used `Llama-3.3-70B-Instruct` via Nebius (no retrieval).
- ✅ First 5 of each `question_type` (5 domain-relevant + 5 novel-generated), sorted by `financebench_id`.
- ✅ xlsx has all 6 required columns: `financebench_id | question_type | question | naive_answer | ground_truth | verdict`.
- ✅ `verdict` ∈ {correct, partially correct, wrong, refused} — distribution 2 / 5 / 2 / 1.
- ✅ Discussion (markdown cell) addresses all 3 prompts:
  - Refusal: `00288` (training-cutoff explanation).
  - Confident answers: 2 correct (`00206`, `00382`), 5 partial, 2 hallucinated (`00080`, `00283` — convincingly wrong, well diagnosed).
  - Patterns by `question_type`: split reported and explained as concept- vs. figure-driven, with prediction for what RAG should fix.

**What stood out**
- Concrete diagnosis of *why* each partial/hallucinated case went wrong (e.g. `00005`: model used all current items rather than only operating items; `00299`: dropped negative sign).

---

## Task 2 — RAG reminder (5 / 5)

**Rubric checks**
- ✅ All 3 components covered (indexing, retrieval, generation).
- ✅ Each component answers the 3 sub-questions (contribution / where it can fail / once vs per query).
- ✅ Concrete failure examples (rare-term embeddings like "EBITDAR"/"MREL"; table split across chunks; wrong-document retrieval; multi-filing attribution errors).
- ✅ ~2–4 sentences per component, as requested.

**Minor (non-deductive)**: the generation section has some sentence-level repetition. Easy to tighten if you want to polish before submission.

---

## Task 3 — Embed documents (15 / 15)

**Rubric checks**
- ✅ `PyPDFLoader` (one Document per page).
- ✅ Only PDFs whose `doc_name` is in the filtered dataset are loaded.
- ✅ Metadata attached *before* splitting: `doc_name`, `company`, `doc_period`, `page_number` (0-indexed, matching `evidence_page_num`).
- ✅ `RecursiveCharacterTextSplitter(chunk_size=1000, chunk_overlap=150)`.
- ✅ FAISS + `BAAI/bge-small-en-v1.5` from HuggingFace.
- ✅ `vectorstore.save_local(...)` to `faiss_chunk1000/` for reuse in Tasks 4–7.
- ✅ Retrieval sanity check on 3 questions (Corning, PayPal, JPM) covering: right document, evidence-text match (8-gram), right page.
- ✅ Observations cell discusses each axis honestly and frames the doc-recall vs. page-recall tradeoff.

**What stood out**
- The 8-gram evidence-text check is a strong, quantitative way to sanity-check retrieval — more rigorous than what the assignment asked for.

---

## Task 4 — RAG pipeline (25 / 25)

**Rubric checks**
- ✅ Function signature exact: `answer_with_rag(query: str, k: int = 4) -> dict`.
- ✅ Returns `{"answer": str, "retrieved_chunks": list}`.
- ✅ Each chunk in the returned list carries `doc_name` and `page_number`.
- ✅ Embeds the query with the same `BAAI/bge-small-en-v1.5` (via the FAISS wrapper).
- ✅ Generation uses `Llama-3.3-70B-Instruct`.
- ✅ Prompt formatting: clear separators between chunks (`\n\n---\n\n`), each chunk header includes `doc_name` and `page_number`.
- ✅ Empty-retrieval branch: `NO_CONTEXT_BLOCK` injected — model is told there is no relevant context (not handed an empty block).
- ✅ System prompt enforces all 4 required behaviors:
  1. Answer ONLY from the provided context.
  2. Say so explicitly when context is missing (exact refusal phrase mandated).
  3. Concise (one short paragraph max).
  4. Cite `doc_name` for each fact.

---

## Task 5 — Run and compare (10 / 10)

**Rubric checks**
- ✅ Same 10 Task 1 questions run through `answer_with_rag`.
- ✅ xlsx has all 6 required columns: `financebench_id | question_type | question | naive_answer | RAG_answer | ground_truth`.
- ✅ Discussion addresses all 3 prompts:
  - **RAG helped**: `00288` (BestBuy Q2 FY24 — past training cutoff, RAG gave a grounded, cited answer); `00080` (refusal beats fabricated negative WC).
  - **RAG hurt**: `00206` (JPM gross-margin — naive memorized the right answer; RAG missed the filing → refusal); `00283` (RAG cited the *wrong* number with a misleading citation).
  - **Patterns by `question_type`**: argues the real signal is *what kind of knowledge* the question relies on (recent filing-specific facts vs. industry-general knowledge), not the type label.
- ✅ Bonus: ran the assignment's "small vector store" debug suggestion as an *independent* mini-FAISS over a single PDF. The conclusion (vocabulary mismatch, not chunk-granularity) directly motivated the Task 7 design.

**What stood out**
- The "naive correct" caveat (direction-correct on fabricated numbers ≠ knowledge) is exactly the kind of nuance the assignment was looking for.

---

## Task 6 — Evaluation (20 / 20)

**Rubric checks**
- ✅ All 100 filtered questions run through the RAG pipeline.
- ✅ **Correctness** — DeepSeek-V3.2 LLM-as-judge, JSON output `{verdict, justification}`, binary verdict, one-sentence justification, run on full dataset. Evaluated 100 / 100.
- ✅ **Faithfulness** — Ragas `Faithfulness`, `single_turn_score()` (synchronous as required), DeepSeek-V3.2 wrapped via `ragas.llms.llm_factory` with an OpenAI client pointing at Nebius. Evaluated on the first 20 sorted by `financebench_id`.
- ✅ **Page-hit@k** — for each question, retrieve top-k, mark hit if any chunk's `page_number` matches *any* `evidence_page_num`. Reported for k ∈ {1, 3, 5}.
- ✅ xlsx columns: `financebench_id | question | correctness | faithfulness | page_hit_at_1 | page_hit_at_3 | page_hit_at_5`.
- ✅ Aggregates reported in notebook:
  - Correctness: **0.19** (n=100)
  - Faithfulness: **0.509** (n=20)
  - Page-hit@1 / @3 / @5: **0.20 / 0.33 / 0.40**

**What stood out**
- Per-question caching to `task6_*.json` makes the evaluation cheap to resume after a rate limit — good engineering hygiene.

---

## Task 7 — Improvement cycles (15 / 15)

The assignment requires "at least 2 experiments" and "pick 3 or more from the list" of *things worth varying*. Two full experiments are submitted; both vary one thing at a time and follow the hypothesis → change → measure → interpret loop. A third row (chunk_size=500) is included as cached context with full transparency.

**Rubric checks**
- ✅ ≥ 2 experiments with explicit hypotheses (chunk_size=1500; HyDE).
- ✅ One thing varied per experiment (vs. Task 6 baseline).
- ✅ All 3 metrics re-run per experiment; faithfulness reuses the same first-20 subsample as required.
- ✅ Per-experiment interpretation comparing predicted vs. actual movement.
- ✅ xlsx with **baseline as the first row** + experiment rows; columns `experiment | change | correctness | faithfulness | page_hit_at_1 | page_hit_at_3 | page_hit_at_5`.
- ✅ Wrap-up answers both required questions: where the pipeline fails most (retrieval, propagating into generation) and what to try next (5 concrete candidates, prioritized).

**What stood out**
- Hypothesis discipline: HyDE was *predicted* to fix the vocabulary-mismatch failure surfaced in the Task 5 small-vector debug, and the Task 7 results confirmed it (page-hit@5 0.40 → 0.50; correctness 0.19 → 0.23). That's exactly the hypothesis → change → measure → interpret loop the rubric asks for.
- Honesty: the chunk_size=1500 interpretation acknowledges that page-hit *fell* despite the hypothesis — and the chunk_size=500 cached row is openly labeled "code not included in notebook," with the rationale spelled out in the markdown.

**Minor (non-deductive)**: if you want full reproducibility, either drop the chunk_size=500 row from the xlsx or paste the rebuild cell. As submitted, it is transparently caveated, so no points are at risk.

---

## Bonus — Multi-scale chunking (0 / 10)

Not attempted. The bonus is optional; this does not affect the 100 / 100 base grade. The infrastructure is already in place (`faiss_chunk500/`, `faiss_chunk1000/`, `faiss_chunk1500/` all exist on disk), so adding a single page-hit@5-per-index sweep + 1-paragraph discussion would be a low-effort path to bonus points if you have time.

---

## Pre-submission checklist (before zipping)

- [ ] **Zip filename includes both first and last names** (e.g. `assignment2_ilay_nisim_boris_kogan.zip`). General Instruction #3.
- [ ] **`.env` is NOT in the zip.** It is in the working directory and `.gitignore` excludes it from git, but a zip of the folder may still grab it. Double-check (`unzip -l <your.zip> | grep .env` should be empty).
- [ ] **Notebook outputs are present** for every code cell — General Instruction #5. (You ran cell-by-cell; just confirm before zipping that you didn't accidentally Restart-and-Clear afterwards.)
- [ ] **Placeholder for `NEBIUS_API_KEY`** is left in place (not your real key). General Instruction #4.
- [ ] **All 4 xlsx files included** with the names listed in the rubric (you have them).
- [ ] **Single notebook** (the assignment requires one notebook — the older `rag_demo.ipynb` and `rag_eval.ipynb` should not be in the zip).
- [ ] Optionally exclude the loose planning markdowns (`task*_plan.md`, `task*_insights.md`, etc.) and the `pdfs/` and `faiss_*/` folders if you want to keep the zip small — they are not deliverables.

---

## Optional polish (no points at stake)

1. **Empty markdown cell** at notebook cell index 13 — safe to delete.
2. **Task 2 generation paragraph** — slight repetition; could be tightened to ~3 sentences.
3. **Task 7 chunk_size=500 row** — either include the build cell or remove the row. Currently transparent, so this is purely cosmetic.

Good luck with the submission.
