# CLAUDE.md

This file provides guidance to Claude Code (claude.ai/code) when working with code in this repository.

## Project Goal

This is a Nebius Academy assignment (Assignment 2 - RAG). The objective is to build, evaluate, and improve a RAG (Retrieval Augmented Generation) pipeline over the [FinanceBench dataset](https://huggingface.co/datasets/PatronusAI/financebench). All task instructions are in `Assignment_2_RAG.md`.

## Environment

- All work is done in a **single Jupyter notebook** that must contain both code and written answers (markdown cells).
- Dependencies to install: `langchain`, `langchain-community`, `faiss-cpu`, `sentence-transformers`, `pypdf`, `ragas`, `openai`, `pandas`, `openpyxl`, `deepseek` (or openai-compatible client).
- API access is via **Nebius Token Factory** — use an OpenAI-compatible client pointed at `https://api.studio.nebius.ai/v1/` with a `NEBIUS_API_KEY` placeholder.

## Key Models

| Role | Model |
|------|-------|
| Generation | `meta-llama/Llama-3.3-70B-Instruct` (via Nebius) |
| Embedding | `BAAI/bge-small-en-v1.5` (HuggingFace, local) |
| Judge (correctness + Ragas) | `deepseek-ai/DeepSeek-V3-0324` (via Nebius) |

## Dataset

- **Source:** `PatronusAI/financebench` from HuggingFace datasets
- **Filter:** Drop all rows where `question_type == "metrics-generated"`. Keep only `domain-relevant` and `novel-generated`.
- **Dead links:** Replace dead `doc_link` URLs with links to the [FinanceBench GitHub repo](https://github.com/patronusai/financebench).
- Key columns: `financebench_id`, `question_type`, `question`, `ground_truth`, `evidence` (contains `evidence_page_num`), `doc_name`, `company`, `doc_period`.

## Vector Store (FAISS)

- Use `PyPDFLoader` (one Document per page) for only the PDFs whose `doc_name` appears in the filtered dataset.
- Attach metadata per page: `doc_name`, `company`, `doc_period`, `page_number` (0-indexed, matching `evidence_page_num`).
- Split with `RecursiveCharacterTextSplitter(chunk_size=1000, chunk_overlap=150)`.
- Store with `BAAI/bge-small-en-v1.5` in a LangChain FAISS vector store; save to disk with `vectorstore.save_local(...)` so it persists across notebook restarts.

## Core RAG Function Signature

```python
answer_with_rag(query: str, k: int = 4) -> dict
# Returns: {"answer": str, "retrieved_chunks": list}
# Each chunk in retrieved_chunks must include doc_name and page_number metadata.
```

## Output Files (xlsx)

| File | Columns |
|------|---------|
| `assignment2_naive_generation` | financebench_id, question_type, question, naive_answer, ground_truth, verdict |
| `assignment2_run_and_compare` | financebench_id, question_type, question, naive_answer, RAG_answer, ground_truth |
| `assignment2_evaluation` | financebench_id, question, correctness, faithfulness, page_hit_at_k (one column per k) |
| `assignment2_improvement_cycles` | experiment, change, correctness, faithfulness, page_hit_at_k (one column per k) |

## Evaluation Metrics

- **Correctness:** LLM-as-judge (DeepSeek-V3-0324), binary verdict + one-sentence justification. Run on full dataset.
- **Faithfulness:** Ragas `Faithfulness` via `.score()` (synchronous). Wrap DeepSeek with `ragas.llms.llm_factory` + `AsyncOpenAI`. Run on first 20 examples only (sorted by `financebench_id`) — it is slow.
- **Retrieval hit-rate (page-hit@k):** For each question, retrieve top-k chunks and check if any chunk's `page_number` matches `evidence_page_num`. Report for k ∈ {1, 3, 5}. Count as a hit if *any* evidence page is retrieved (some questions have multiple evidence pages).

## Task 7 - Improvement Experiments

Run at least 3 experiments varying **one thing at a time** from the Task 6 baseline. Options:
1. Generation prompt
2. Generation model
3. k value in top-k retrieval
4. Chunk size (requires re-embedding; save each FAISS index under a different name)
5. Reranker: `BAAI/bge-reranker-base` — retrieve top-20 with FAISS, rerank to top-4 with cross-encoder

For faithfulness, reuse the same 20-question subsample across all experiments.
