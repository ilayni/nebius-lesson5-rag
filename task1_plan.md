# Task 1 Plan: Dataset Preparation + Naive Generation

## Context

Foundation for the entire Assignment 2 notebook. We install all dependencies, load/filter the FinanceBench dataset, and implement Task 1 (naive generation with Llama-3.3-70B-Instruct, no retrieval). Everything here is designed to be reused by later tasks.

## Forward-Looking Design

| Later Task | What it needs from us |
|---|---|
| Task 3 | Document metadata lookup (`doc_name → {company, doc_period, doc_link}`), parsed `evidence_page_nums` |
| Task 4 | Reusable `llm_generate` helper, same Nebius client |
| Task 5 | Exact same 10-question `task1_df` DataFrame with `naive_answer` column |
| Task 6 | All filtered questions, `evidence_page_nums` per row, `JUDGE_MODEL` constant |
| Task 7 | Same dataset, same evaluation infrastructure |

## Steps

### Step 1: Install dependencies

All packages for the entire assignment:

```bash
pip install openai pandas openpyxl datasets langchain langchain-community faiss-cpu sentence-transformers pypdf ragas
```

### Step 2: Setup & configuration

- Imports: `openai`, `pandas`, `datasets`, `json`
- Constants with placeholders:
  - `NEBIUS_API_KEY = "YOUR_NEBIUS_API_KEY"`
  - `NEBIUS_BASE_URL = "https://api.studio.nebius.ai/v1/"`
  - `LLM_MODEL = "meta-llama/Llama-3.3-70B-Instruct"`
  - `JUDGE_MODEL = "deepseek-ai/DeepSeek-V3-0324"` (for Task 6)
  - `EMBED_MODEL_NAME = "BAAI/bge-small-en-v1.5"` (for Task 3)
- Create `OpenAI` client pointed at Nebius
- Reusable helper:
  ```python
  def llm_generate(question: str, system_prompt: str = None, model: str = LLM_MODEL) -> str
  ```
  Single place for all LLM API calls. Used by Task 1 (naive), Task 4 (RAG), Task 6 (judge).

### Step 3: Load & filter dataset

1. `datasets.load_dataset("PatronusAI/financebench", split="train")` → pandas DataFrame
2. **Drop** `question_type == "metrics-generated"`
3. **Rename** `answer` → `ground_truth`
4. **Fix dead `doc_link`** → `https://raw.githubusercontent.com/patronus-ai/financebench/main/pdfs/{doc_name}.pdf`
5. **Parse `evidence` column**:
   - `evidence_page_nums`: list of ints per row (needed by Tasks 3, 6, 7)
   - `evidence_texts`: list of strings per row
6. **Build doc metadata lookup**: `doc_name → {company, doc_period, doc_link}` dict (needed by Task 3)
7. Sort by `financebench_id`, reset index
8. Print shape, `question_type` value counts, sample rows, unique doc count

### Step 4: Select the 10 Task 1 questions

```python
task1_domain = df[df.question_type == "domain-relevant"].head(5)
task1_novel = df[df.question_type == "novel-generated"].head(5)
task1_df = pd.concat([task1_domain, task1_novel]).sort_values("financebench_id")
```

Keep `task1_df` as a named variable — Task 5 reuses it.

### Step 5: Naive generation

```python
def naive_generate(question: str) -> str:
    return llm_generate(
        question,
        system_prompt="You are a helpful financial analyst. Answer the question to the best of your knowledge.",
    )
```

- No retrieval, no grounding instructions
- `temperature=0.1`, `max_tokens=500`
- Loop over `task1_df`, store in `naive_answer` column
- Print each Q/A as we go (outputs visible in notebook)

### Step 6: Display results & assign verdicts

- Show formatted table: `financebench_id | question | naive_answer | ground_truth`
- Add `verdict` column: `correct`, `partially correct`, `wrong`, `refused`
- Pre-fill with heuristic (e.g. "I don't have" → `refused`), then manual review

### Step 7: Export xlsx

```python
task1_df[["financebench_id", "question_type", "question", "naive_answer", "ground_truth", "verdict"]]\
    .to_excel("assignment2_naive_generation_ilay_nisim_boris_kogan.xlsx", index=False)
```

### Step 8: Discussion markdown cell

Template addressing the 3 required questions:
1. **Refusals** — which questions, why? (likely specific financial figures not in training data)
2. **Confident answers** — correct / partially correct / hallucinated?
3. **Patterns by question_type** — do domain-relevant vs novel-generated fail differently?

Placeholder text to update after reviewing actual outputs.

## Writing Style

- **Beginner-friendly**: Add clear comments and markdown explanations for each step. Explain *why* we do things, not just *what* we do. Written as if a student (Ilay Nisim & Boris Kogan) wrote it.
- **Natural tone**: Avoid overly polished or AI-sounding language. Use first person ("we", "I"), keep it conversational.
- **Comments in code**: Explain non-obvious lines. Don't over-comment trivial things like `import pandas`.

## Naming Convention

All generated files include student names:
- Notebook: `assignemnt2_rag_ilay_nisim_boris_kogan.ipynb` (keeping existing typo in filename)
- xlsx files: `assignment2_naive_generation_ilay_nisim_boris_kogan.xlsx`

## Output Files

- `assignemnt2_rag_ilay_nisim_boris_kogan.ipynb` — the notebook (created from scratch)
- `assignment2_naive_generation_ilay_nisim_boris_kogan.xlsx` — Task 1 deliverable

## Verification Checklist

- [ ] Notebook runs end-to-end
- [ ] No `metrics-generated` rows in filtered dataset
- [ ] `evidence_page_nums` column populated (list of ints per row)
- [ ] `task1_df` has exactly 10 rows (5 domain-relevant + 5 novel-generated)
- [ ] `naive_answer` column populated with model responses
- [ ] `assignment2_naive_generation_ilay_nisim_boris_kogan.xlsx` exists with correct columns
- [ ] `llm_generate` helper is reusable for later tasks
