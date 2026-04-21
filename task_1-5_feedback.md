# Tasks 1–5 pre-submission feedback

## Summary
- **Blocker (5-min fix): Task 1 xlsx `verdict` column is wrong.** Cell 10 runs only the heuristic pre-fill; the manual-corrections block is entirely commented out. The saved xlsx has only `{wrong, refused}` (8 wrong, 2 refused), but the spec requires the full `{correct, partially correct, wrong, refused}` set, and the student's own discussion text says "2 correct / 5 partially correct / 2 wrong / 1 refused". A grader will see `financebench_id_00206` labeled `wrong` in the xlsx while the prose calls it correct.
- Otherwise the deliverables are all present, signatures and columns match the spec, the FAISS index is built and persisted, `answer_with_rag` has the required shape, and the 0-indexing of `page_number` is correctly aligned with `evidence_page_num` (verified against Corning p59).
- Discussion quality is a genuine strength: Task 5 in particular is specific, cites individual `financebench_id`s, and distinguishes "correct" from "partially correct but plausibly lucky" in a thoughtful way.
- Two reproducibility risks before submission: (a) the Task 1 and Task 5 xlsx files were saved in different kernel sessions — 5 of 10 naive-answer strings differ between them; (b) cell 25 (the `answer_with_rag` definition) has `execution_count=None` in the saved notebook, even though downstream cells that depend on it have counts.
- Nothing in Tasks 2, 3, or 4 is obviously broken. The Task 5 "small vector store" debug section is a strong bonus that actually answers the assignment's debugging tip.

---

## Task 1 – naive generation

### Spec checklist
- 10 questions, 5 per type, first 5 by sorted `financebench_id`: **PASS** — cell 7 sorts and concats; IDs in xlsx are `00005, 00070, 00080, 00206, 00215` (domain-relevant) and `00283, 00288, 00299, 00302, 00382` (novel-generated).
- Naive generation, no retrieval: **PASS** — cell 8 calls `llm_generate` with only `NAIVE_SYSTEM_PROMPT` + question.
- xlsx columns `financebench_id | question_type | question | naive_answer | ground_truth | verdict`: **PASS** — exact column set, exact order, 10 rows.
- xlsx filename `assignment2_naive_generation`: **PASS** (suffixed with student names; still starts with the required stem).
- Verdicts from `{correct, partially correct, wrong, refused}`: **FAIL** — only `wrong` (8) and `refused` (2) appear in the saved xlsx. The manual-corrections line in cell 10 is left as a commented-out template.
- Discussion addresses the 3 spec questions: **PASS** — all three are addressed in the Task 1 markdown cell, with concrete IDs.

### What's strong
- The discussion is sharp and specific: it distinguishes correct-from-reasoning (`00206`, `00382`) from correct-from-direction-with-fabricated-numbers (`00005`, `00299`), and calls out the "confident hallucination with plausible narrative" pattern on `00080` and `00283`. That's exactly the intuition the task is meant to build.
- Good choice to keep `task1_df` as a named slice so Task 5 can reuse it cleanly.

### What to fix or improve
- **Uncomment and fill in the `task1_df.loc[...] = "..."` lines in cell 10** for the 7 rows the discussion already judges (2 correct, 5 partially correct) and re-save. Without this, the xlsx and the prose disagree. The corrections implied by the current Task 1 discussion: `00206 → correct`, `00382 → correct`, `00005/00070/00215/00299/00302 → partially correct`, `00080/00283 → wrong`, `00288 → refused`. (Note: `task1_insights.md` actually says `00302` is partially correct, while the heuristic labels it `refused` because "I don't have" triggers — the Task 1 md file even flags this as a known heuristic bug, but the correction never lands.)
- The heuristic in cell 10 has a correctly-identified false-positive bug (`00302`); documenting it in a comment is fine, but the intended manual override must actually run.
- Minor: both refusals in the xlsx are labeled `refused`, but `financebench_id_00302` is arguably a substantive (if wrong) answer, not a refusal — resolve this one way or the other.

---

## Task 2 – RAG reminder

### Spec checklist
- Write-up per component (indexing / retrieval / generation): **PASS** — three clearly labeled subsections in the Task 2 markdown cell.
- Each component covers all 3 sub-questions (contribution / where-it-fails / once-vs-per-query): **PASS** — all three are answered for all three components.

### What's strong
- Concrete failure examples are given, not generic: "a financial table split across two chunks loses its row context"; "may attribute a number to the wrong company when two filings are in context". Exactly the kind of reasoning the task is asking for.

### What to fix or improve
- Prose is slightly breathless (long sentences, some repetition between the "how it contributes" and "where it fails" bullets in the indexing section). A tightening pass would help, but the content is complete.
- Small wording nit: "it can decide not to embed rare words" isn't really how embedding models fail (they embed everything, just poorly for OOV tokens). Reframe as "rare domain terms may get weak embeddings that don't cluster with their true semantic neighbors" if you touch it.

---

## Task 3 – embed documents

### Spec checklist
- PyPDFLoader, one Document per page: **PASS** — cell 19.
- Only PDFs for `doc_name`s in the filtered dataset: **PASS** — `needed_docs = sorted(df["doc_name"].unique())` → 42 PDFs.
- Metadata attached pre-split with `doc_name`, `company`, `doc_period`, `page_number`: **PASS** — cell 19 overwrites the metadata dict explicitly.
- `page_number` 0-indexed to match `evidence_page_num`: **PASS** — verified by loading `CORNING_2022_10K.pdf`: PyPDF index 59 is the Consolidated Balance Sheet, matching `evidence_page_num=59` for `financebench_id_00005`. Easy thing to get wrong; the student got it right.
- `RecursiveCharacterTextSplitter(chunk_size=1000, chunk_overlap=150)`: **PASS** — cell 20.
- FAISS + `BAAI/bge-small-en-v1.5`: **PASS** — cell 21.
- Index saved to disk: **PASS** — `faiss_chunk1000/` exists on disk; cell 21 has a load-or-build pattern.
- Sanity check on 2–3 questions covering right-doc / right-page / evidence text: **PASS** — 3 questions (`00005`, `00080`, `00206`), all three dimensions reported in console + `task3_observe_3docs.md`.
- Markdown observations cell: **PASS** — in-notebook Task 3 observations paragraph.

### What's strong
- The 8-gram-shingle measure of "evidence text visible" is a nicer, more conservative check than substring matching. It being 0/0/0 on all three samples tells you something real (pages that talk about WC are not the pages that contain the balance-sheet table).
- The observations paragraph correctly summarizes the pattern (`3/5`, `5/5`, `0/5` docs; zero pages; zero text overlap — total `8/15` docs), which matches the console output. No inflation.

### What to fix or improve
- The in-notebook Task 3 observations calls `id_00005` "Corning", `id_00080` "PayPal", `id_00206` "JPM" but doesn't mention that the PayPal hit of `5/5 right docs` is still a page miss. The trade-off between doc-recall and page-recall is the interesting finding — worth one sentence in the notebook (it's in `task3_observe_3docs.md` but not in the notebook discussion itself).
- The retrieval cell runs on a hardcoded `SAMPLE_IDS` list; consider noting somewhere in the markdown that these 3 IDs were chosen because they're the Task 1 domain-relevant set / overlap with Task 5, so the same failures carry through.

---

## Task 4 – RAG pipeline

### Spec checklist
- Function signature `answer_with_rag(query: str, k: int = 4) -> dict`: **PASS** — cell 25.
- Return dict keys `answer` (str) + `retrieved_chunks` (list): **PASS** — each chunk is a dict with `doc_name`, `page_number`, `content`.
- Chunks expose `doc_name` and `page_number` metadata: **PASS**.
- Empty-retrieval handled (model told there's no context, not handed a blank block): **PASS** — `NO_CONTEXT_BLOCK` sentinel; explicitly demonstrated in the markdown trace. Minor caveat: FAISS `similarity_search` will essentially always return `k` chunks when the index is non-empty, so the empty branch is mostly defensive — that's fine, and the empty-branch sanity-check call to `_format_context([])` in cell 26 is a nice touch.
- System prompt enforces: context-only / explicit no-answer / concise / cite `doc_name`: **PASS** — all four rules are in `RAG_SYSTEM_PROMPT`.
- Prompts included in the notebook: **PASS**.
- Clear separators between chunks + metadata shown to the model: **PASS** — `[Chunk i | doc_name=... | page_number=...]` header plus `\n\n---\n\n` separators.

### What's strong
- The strict "reply *exactly*: The provided context does not contain the answer" rule makes refusals easy to detect programmatically downstream — useful for Task 5 / 6 analysis.
- The Task 4 "Insights" markdown is candid about the sample result being three refusals, and honestly labels each one a save / wash / loss vs naive. That's the right framing.

### What to fix or improve
- `execution_count` on cell 25 is `None` in the saved notebook even though cells 26, 29, 30 that depend on it have counts (23, 24, 25). A "Restart & Run All" would still succeed because cell 25 is upstream in document order, but consider re-running and re-saving so the execution counts are contiguous — a grader who notices will wonder.
- The user-message tail (`"Answer using ONLY the context above. Cite doc_name for each fact."`) partially duplicates rules already in the system prompt. Not wrong, possibly slightly hurts instruction following — fine as-is but flag it.

---

## Task 5 – run and compare

### Spec checklist
- Same 10 questions from Task 1 run through RAG: **PASS** — cell 29 iterates over `task1_df`.
- xlsx columns `financebench_id | question_type | question | naive_answer | RAG_answer | ground_truth`: **PASS** — exact column set, exact order, 10 rows.
- xlsx filename `assignment2_run_and_compare`: **PASS**.
- Discussion covers "did RAG help / did RAG hurt / patterns by question_type": **PASS** — all three are addressed, in that order, with per-ID callouts.

### What's strong
- The discussion explicitly distinguishes "naive was fully correct" (`00206`) from "naive was partially correct but plausibly lucky" (`00070`, `00302`). This directly addresses the `task5_corrections.md` feedback about being more careful when claiming "correct → refusal". Good.
- `00288` (BestBuy cash drop) is correctly identified as the cleanest RAG win (training-cutoff case).
- The Task 5 debug cell at the end is substantive: it builds a proper mini-FAISS over the single filing, inspects top-8 scores, scans the raw PDF for the literal phrase, and uses that to distinguish chunk-granularity from vocabulary-mismatch failure modes. That's exactly what the assignment's debugging tip was asking for.
- The `task1_df` rehydration safety net in cell 29 (`if "task1_df" not in globals(): pd.read_excel(...)`) is a nice reproducibility touch.

### What to fix or improve
- **Naive-answer drift between the two xlsx files.** 5 of 10 rows (`00005`, `00283`, `00288`, `00299`, `00302`) have different `naive_answer` strings in `assignment2_naive_generation_*.xlsx` vs `assignment2_run_and_compare_*.xlsx`. At temperature 0.1 this happens when the naive cell is re-run in a later session; the Task 1 xlsx was saved early and not refreshed. Fix: re-save the Task 1 xlsx at the end of the run that produced Task 5 so both files come from the same LLM outputs. The differences are minor rewordings, not structural — but a grader comparing the two files will notice, and the Task 5 discussion references specific naive-answer phrasings (e.g. `00005` `$3.02B` and `00070` `-$120M`) that exist in the Task 1 xlsx but not all match the Task 5 xlsx row.
- The `00299` analysis ("retrieval did surface `JPMORGAN_2021Q1_10Q` p4, but the model still refused") is a concrete, actionable observation — consider saving the `retrieved_logs` dict to disk so a grader can confirm it independently; right now `retrieved_logs` is computed in cell 29 but only used in-memory.
- The Task 5 discussion has a very long bullet for `00283`. It's thoughtful but could be 2 sentences instead of 4 — the core point (both wrong with confidence, RAG adds a misleading citation) lands faster.

---

## Cross-task issues

- **Kernel-restart reproducibility.** Execution counts in the saved notebook jump 2, 3, 4, 5, 6, 7, 8, then 15, 16, 17, 20, then 23, 24, 25, 27. Cell 2 (pip install) and cell 18 (PDF download) both have `execution_count=None`. Cell 25 (`answer_with_rag` definition) has `execution_count=None` while cells depending on it have counts. In document order the notebook *should* run end-to-end (dependencies flow top-down), but the grader's sanity check "Restart & Run All" will produce a different execution-count sequence, and the current state suggests the notebook was edited after the last full run. Cheapest fix before submission: Restart, Run All, Save — one time.
- **Same-session xlsx generation.** Tied to the above: regenerate both xlsx files in the same final run so `naive_answer` columns agree.
- **Outputs are saved in the notebook.** Most code cells have captured outputs (markdown trace + console prints + dataframes). Cells 2, 18, 25 have no outputs; none of them produce load-bearing output, but cell 25 is suspicious given the downstream dependency.
- **File naming is consistent** across the two xlsx files — both follow the `<stem>_ilay_nisim_boris_kogan.xlsx` pattern, and both stems match the spec (`assignment2_naive_generation`, `assignment2_run_and_compare`).
- **Auxiliary markdown files** (`task1_insights.md`, `task3_observe_3docs.md`, `task4_generation.md`, `task5_corrections.md`, `task5_insights.md`) are substantive but clearly not the deliverable. `task1_insights.md` contains an older set of naive-answer numbers (`$7.85B / $4.93B / $2.92B` for Corning) that don't match either the Task 1 or Task 5 xlsx — not a blocker since it's auxiliary, but worth either deleting before zipping or noting "these reflect an earlier run and are not the deliverable" so the grader doesn't chase a discrepancy.
- **`.env` with `NEBIUS_API_KEY`** is loaded via `python-dotenv`. The spec asks for "placeholders for API key in case I do need to run it" — the current code reads from env and will fail with `KeyError` if `NEBIUS_API_KEY` isn't set. That's fine (env var is the standard placeholder), but make sure `.env` is *not* zipped into the submission (`.gitignore` currently excludes it from git; double-check the zip).
