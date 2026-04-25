# Task 7 — Improvement Cycles: Implementation Plan

## Context

**Task 6 baseline** — the numbers we're trying to improve:

| metric        | value | notes                 |
|---------------|-------|-----------------------|
| correctness   | 0.190 | LLM-judge on 100 Q    |
| faithfulness  | 0.509 | Ragas on first 20 Q   |
| page-hit@1    | 0.200 | retrieval, n=100      |
| page-hit@3    | 0.330 | retrieval, n=100      |
| page-hit@5    | 0.400 | retrieval, n=100      |

**What Task 5 already told us** (cell 32–34, small-vector sanity check):
- On `financebench_id_00206` (JPM gross margin), even a mini FAISS over the correct 10-K didn't surface the evidence page.
- The phrase "gross margin" **doesn't appear on any page** of `JPMORGAN_2022_10K`.
- Conclusion written in the notebook: *"The Task 5 failure on 00206 is a vocabulary mismatch, not a chunk-granularity issue — no amount of reranking or larger k would have rescued it."*

**Assignment author's hint** (line 208):
> "A cross-encoder reranker rewards semantic similarity to the query — which makes the vocabulary mismatch worse. Chunk size and query reformulation should be higher leverage interventions than reranking for this hard dataset."

Both of these point away from reranker / k-tuning and toward **query reformulation** as the highest-leverage intervention on this dataset.

**User requirement:** at least 2 experiments; may combine. No kernel restart needed; each cell caches to disk so re-runs skip finished work.

## Reusable Task 6 artifacts (on disk)

- FAISS index: `faiss_chunk1000/` (BAAI/bge-small-en-v1.5, 24,218 vectors)
- Caches: `task6_rag.json` (100), `task6_correctness.json` (100), `task6_faithfulness.json` (20)
- Cell 36–39 kernel objects: `t6_df`, `t6_nebius`, `t6_embed`, `t6_vs`, `rag_answer_t6()`, `_t6_judge()`, `t6_faith_metric`, constants `T6_GEN_MODEL="meta-llama/Llama-3.3-70B-Instruct"`, `T6_JUDGE_MODEL="deepseek-ai/DeepSeek-V3.2"`, `T6_PAGE_HIT_K=[1,3,5]`
- Task 6 system prompt `T6_RAG_SYS` (answer-only-from-context, cite doc_name)

Task 7 reuses these directly — judge, faithfulness scorer, and page-hit logic are not re-implemented.

## Experiments (2 total, recommended)

### Experiment 1 — `exp1_chunk500` (indexing change, isolated)

**Hypothesis.** 1000-char chunks span multiple tables or narrative paragraphs; halving to 500 should give more precise page-level hits. Targets page-hit@k directly. Expected: page-hit@k rises, correctness moves slightly (may go either way — less context per chunk could hurt, but more-targeted retrieval could help).

**Change.** Rebuild FAISS with `chunk_size=500, chunk_overlap=75` → `faiss_chunk500/`. Everything else baseline (k=4, same prompt, same model, same query text).

**Cost.** One-time local embedding of ~2× the chunks (few minutes). Then 100 Llama RAG + 100 DeepSeek judge + 20 Ragas faithfulness runs.

### Experiment 2 — `exp2_hyde` (query reformulation, isolated)

**Hypothesis.** Task 5 proved the dominant failure mode is vocabulary mismatch — user questions use high-level financial terms ("gross margin", "working capital") that aren't literally in the filings. **HyDE**: ask the generator for a hypothetical answer to the question, then embed the hypothetical answer instead of the question. The hypothetical will contain filing-style vocabulary (line-item names, dollar amounts, 10-K phrasing), which FAISS can match to real pages. Expected: page-hit@k rises substantially on vocabulary-mismatch questions (like `00206`), correctness follows, faithfulness depends on whether the generator can ground in the now-better-retrieved chunks.

**Change.** Same index (`faiss_chunk1000`), same k=4, same generation prompt. Only the **retrieval query** changes. Implementation: call Llama once with a short "write a plausible 1–2 sentence answer; don't refuse even if unsure" prompt → concatenate `question + "\n" + hypothetical` → embed that concatenation for FAISS lookup → feed retrieved chunks + the **original question** to the generator.

**Cost.** +1 Llama call per question (100 extra). Then 100 RAG + 100 judge + 20 faithfulness runs.

### Optional Experiment 3 — `exp3_combined` (stack both, if budget allows)

**Hypothesis.** If exp1 and exp2 improve different pipeline stages (indexing vs retrieval-query), the combined change should stack. If they both help on the same questions, the combined effect will be sub-additive. Either way is informative.

**Change.** Use `faiss_chunk500/` from exp1 **and** HyDE from exp2.

**Default plan: skip exp3. Run it only if exp1 and exp2 are both positive and you want to see the stack effect.**

## Notebook cell layout (appended after current cell 40)

| #  | type     | content |
|----|----------|---------|
| 41 | markdown | `## Task 7 — improvement cycles`. Summary: 2 experiments (chunk=500, HyDE). Baseline context. References Task 5 vocabulary-mismatch finding. |
| 42 | code     | **Task 7 setup** — standalone. Re-loads `t7_df`, FAISS `chunk1000`, Nebius client, DeepSeek faithfulness metric. Defines generic `run_experiment(name, rag_fn)` that caches `task7_{name}_rag.json`, `task7_{name}_corr.json`, `task7_{name}_faith.json`. |
| 43 | markdown | `### Experiment 1 — chunk_size=500`. Hypothesis (1–2 sentences). |
| 44 | code     | Build `faiss_chunk500/` if absent; load it; define `rag_exp1_chunk500(query)`; call `run_experiment("exp1_chunk500", rag_exp1_chunk500)`. |
| 45 | markdown | Interpretation of exp1 (filled in after numbers are available). |
| 46 | markdown | `### Experiment 2 — HyDE query reformulation`. Hypothesis. |
| 47 | code     | Define `hyde_rewrite(query)` → Llama hypothetical; define `rag_exp2_hyde(query)` (retrieve on `question + hypothetical`, generate on `question`). `run_experiment("exp2_hyde", rag_exp2_hyde)`. |
| 48 | markdown | Interpretation of exp2. |
| 49 | code     | **Aggregation + xlsx**. Loads baseline from `task6_*.json`, then exp1 and exp2 caches. Builds 3-row dataframe with columns `experiment | change | correctness | faithfulness | page_hit_at_1 | page_hit_at_3 | page_hit_at_5`. Writes `assignment2_improvement_cycles_ilay_nisim_boris_kogan.xlsx`. Prints a summary table. |
| 50 | markdown | **Wrap-up paragraph** — retrieval vs generation: which fails more? What to try with another week? (Filled in after results.) |

## Shared helper (cell 42)

```python
def run_experiment(name, rag_fn, k_for_pagehit=5):
    rag_path   = f"task7_{name}_rag.json"
    corr_path  = f"task7_{name}_corr.json"
    faith_path = f"task7_{name}_faith.json"

    rag   = json.loads(Path(rag_path).read_text())   if Path(rag_path).exists()   else {}
    corr  = json.loads(Path(corr_path).read_text())  if Path(corr_path).exists()  else {}
    faith = json.loads(Path(faith_path).read_text()) if Path(faith_path).exists() else {}

    # 1) RAG answers + top-k retrieval snapshot (all 100)
    for _, row in t7_df.iterrows():
        fb = row["financebench_id"]
        if fb in rag: continue
        out = rag_fn(row["question"])
        rag[fb] = {
            "answer": out["answer"],
            "retrieved_chunks": out["retrieved_chunks"],
            "top5_pages": [{"doc_name": c["doc_name"], "page_number": c["page_number"]}
                           for c in out["retrieved_chunks"][:k_for_pagehit]],
        }
        Path(rag_path).write_text(json.dumps(rag, indent=1))

    # 2) Correctness (all 100)
    for _, row in t7_df.iterrows():
        fb = row["financebench_id"]
        if fb in corr: continue
        corr[fb] = _t6_judge(row["question"], row["ground_truth"], rag[fb]["answer"])
        Path(corr_path).write_text(json.dumps(corr, indent=1))

    # 3) Faithfulness (first 20 only) — same 20 as Task 6
    for _, row in t7_df.head(20).iterrows():
        fb = row["financebench_id"]
        if fb in faith: continue
        contexts = [c["content"] for c in rag[fb]["retrieved_chunks"]] or ["[NO CONTEXT]"]
        sample = SingleTurnSample(user_input=row["question"], response=rag[fb]["answer"],
                                  retrieved_contexts=contexts)
        try:
            s = t6_faith_metric.single_turn_score(sample)
            faith[fb] = float(s) if s is not None else None
        except Exception as e:
            faith[fb] = None
        Path(faith_path).write_text(json.dumps(faith, indent=1))

    return rag, corr, faith
```

**Every loop skips items already in cache** → a kill-and-resume picks up exactly where it stopped. Writes happen after every item, not at end.

## Per-experiment code sketches

### Exp 1 — chunk_size=500

```python
FAISS500_DIR = "faiss_chunk500"
if not Path(FAISS500_DIR).exists():
    # reuse Task 3 PDF-loading logic
    splitter = RecursiveCharacterTextSplitter(chunk_size=500, chunk_overlap=75)
    chunks500 = splitter.split_documents(pages)  # `pages` = per-page Documents from Task 3
    t7_vs500 = FAISS.from_documents(chunks500, t6_embed)
    t7_vs500.save_local(FAISS500_DIR)
else:
    t7_vs500 = FAISS.load_local(FAISS500_DIR, t6_embed, allow_dangerous_deserialization=True)

def rag_exp1_chunk500(query, k=4):
    chunks = t7_vs500.similarity_search(query, k=k)
    # identical body to rag_answer_t6, only the store changes
    ...

run_experiment("exp1_chunk500", rag_exp1_chunk500)
```

### Exp 2 — HyDE

```python
HYDE_SYS = (
    "You will receive a financial question. Write a SHORT, PLAUSIBLE hypothetical "
    "answer in the style of a 10-K filing — use specific line-item names and dollar "
    "figures even if you have to guess. 1–2 sentences. Do not refuse."
)

def hyde_rewrite(query):
    resp = t6_nebius.chat.completions.create(
        model=T6_GEN_MODEL,
        messages=[{"role":"system","content":HYDE_SYS},{"role":"user","content":query}],
        temperature=0.3, max_tokens=120,
    )
    return resp.choices[0].message.content.strip()

# Cache the hypothetical answers themselves — re-running is wasted tokens otherwise
HYDE_CACHE = "task7_exp2_hyde_hypotheticals.json"
hyde_cache = json.loads(Path(HYDE_CACHE).read_text()) if Path(HYDE_CACHE).exists() else {}

def rag_exp2_hyde(query, fb_id=None, k=4):
    # key the cache by question text (stable)
    if query not in hyde_cache:
        hyde_cache[query] = hyde_rewrite(query)
        Path(HYDE_CACHE).write_text(json.dumps(hyde_cache, indent=1))
    hyp = hyde_cache[query]
    retrieval_text = f"{query}\n{hyp}"
    chunks = t6_vs.similarity_search(retrieval_text, k=k)   # baseline index
    # same generation path as rag_answer_t6, but pass the ORIGINAL question to the LLM
    ...

# run_experiment passes only `query`; we look up fb via closure not needed if we key by query
run_experiment("exp2_hyde", rag_exp2_hyde)
```

**Note:** hypothetical is cached by question text, so re-running exp2 doesn't re-generate hypotheticals.

## Aggregation (cell 49)

Inputs (all from disk):
- baseline → `task6_rag.json` + `task6_correctness.json` + `task6_faithfulness.json`
- exp1 → `task7_exp1_chunk500_{rag,corr,faith}.json`
- exp2 → `task7_exp2_hyde_{rag,corr,faith}.json`

For each of the 3 rows, compute aggregates exactly as cell 40 does (average correctness, average faithfulness over non-None subset, page_hit@k = fraction of questions where top-k retrieval contains an evidence page in the right doc).

Output dataframe schema:

| experiment          | change                              | correctness | faithfulness | page_hit_at_1 | page_hit_at_3 | page_hit_at_5 |
|---------------------|-------------------------------------|-------------|--------------|---------------|---------------|---------------|
| baseline (Task 6)   | chunk=1000, k=4, question-only query| 0.190       | 0.509        | 0.200         | 0.330         | 0.400         |
| exp1_chunk500       | chunk_size 1000→500                 | ?           | ?            | ?             | ?             | ?             |
| exp2_hyde           | HyDE query rewrite for retrieval    | ?           | ?            | ?             | ?             | ?             |

Write to `assignment2_improvement_cycles_ilay_nisim_boris_kogan.xlsx`, single sheet, plus print the same table to stdout.

## Pitfalls & mitigations

1. **Same 20 faithfulness questions across experiments.** `t7_df` is sorted by `financebench_id` identically to `t6_df`; `.head(20)` returns the same set. The *contexts* passed to Ragas differ per experiment (that's the whole point — faithfulness measures how well *this experiment's* answer is grounded in *this experiment's* retrieved chunks).

2. **Page-hit@k uses each experiment's own retrieval.** `top5_pages` must be populated inside `run_experiment`. `k_for_pagehit=5` means we always store at least 5 pages even if the generator only saw 4.

3. **Chunk-500 rebuild is all-or-nothing.** If interrupted mid-embedding, `faiss_chunk500/` may be half-written — delete the directory and re-run. Acceptable risk (few minutes, local).

4. **HyDE hypotheticals must be cached** (`task7_exp2_hyde_hypotheticals.json`) — otherwise every RAG-cache regeneration double-bills the LLM.

5. **Wrap-up needs real numbers.** Write cell 50 *after* running all experiments; don't pre-write conclusions.

6. **Budget exposure per kill.** All loops cache-per-item; max loss from interrupt = one RAG round + one judge + one faithfulness sample (few thousand tokens).

## Files to modify

- `assignemnt2_rag_ilay_nisim_boris_kogan.ipynb` — append cells 41–50 as above.

## Files created at runtime

- `faiss_chunk500/index.faiss`, `index.pkl`
- `task7_exp1_chunk500_rag.json`, `..._corr.json`, `..._faith.json`
- `task7_exp2_hyde_rag.json`, `..._corr.json`, `..._faith.json`
- `task7_exp2_hyde_hypotheticals.json`
- `assignment2_improvement_cycles_ilay_nisim_boris_kogan.xlsx`

## Verification

1. Setup cell runs → `t7_df` has 100 rows; FAISS loads; Ragas metric initialises.
2. Each experiment cell: first run populates caches; **second run exits within seconds** (all items cached). If the second run still makes LLM calls, cache keying is wrong.
3. Aggregation cell: xlsx has 3 rows, 7 columns, baseline row matches Task 6 numbers exactly (0.190 / 0.509 / 0.200 / 0.330 / 0.400).
4. Spot-check `00206` (JPM gross margin) under HyDE: open `task7_exp2_hyde_rag.json`, read the retrieved_chunks — do they now surface the balance-sheet page? This is the single most-diagnostic case for whether HyDE is doing its job.
5. Interpretations (cells 45, 48) and wrap-up (cell 50) written after numbers are in.
