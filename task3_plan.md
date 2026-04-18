# Task 3 — Embed Documents: Implementation Plan

**Goal:** Build a FAISS vector store over FinanceBench PDFs that Tasks 4–7 will reuse, following the spec in `Assignment_2_RAG.md`.

**Location:** Cells appended to `assignemnt2_rag_ilay_nisim_boris_kogan.ipynb`, after the trailing empty markdown cell (`13fb4035`) at the end of the Task 2 discussion.

---

## Assignment requirements (for reference)

1. Load PDFs with `PyPDFLoader` — one Document per page. Only the PDFs for `doc_name` values in the filtered dataset.
2. Attach metadata **before** splitting: `doc_name`, `company`, `doc_period`, `page_number` (0-indexed, matching `evidence_page_num`).
3. Split with `RecursiveCharacterTextSplitter(chunk_size=1000, chunk_overlap=150)`. Chunks inherit page metadata.
4. Store in a LangChain FAISS vector store using `BAAI/bge-small-en-v1.5` from HuggingFace.
5. Save index with `vectorstore.save_local(...)` so we don't re-embed on restart.
6. Pick 2–3 questions, retrieve top-k, and check: right `doc_name`, evidence-text overlap, right `page_number`.
7. Short markdown with observations (user requested I leave conclusions empty for now).

---

## Cell plan (7 new cells)

### Cell 1 — markdown header

Short intro re-stating what Task 3 does and the artifact produced (`faiss_chunk1000/`).

### Cell 2 — code: download PDFs only for needed `doc_name`s

- Build `needed_docs = sorted(df["doc_name"].unique())` (42 docs based on Task 1 output).
- Create local `pdfs/` folder.
- For each doc, skip if cached; otherwise `urllib.request.urlretrieve` from `https://raw.githubusercontent.com/patronus-ai/financebench/main/pdfs/{name}.pdf` (matches the fixed-up `doc_link` we already set in dataset prep).
- Track and print any missing docs — don't silently swallow 404s.

### Cell 3 — code: load + attach metadata

- Import `PyPDFLoader` from `langchain_community.document_loaders`.
- For each available PDF: `PyPDFLoader(path).load()` → list of page-level `Document`s (PyPDFLoader sets `page` metadata 0-indexed).
- **Overwrite** `doc.metadata` with the four required fields only (`doc_name`, `company`, `doc_period`, `page_number`). Reading `company` and `doc_period` from the `doc_meta` lookup built in dataset prep.
- Cast `page_number` to `int` to avoid numpy-int surprises downstream.
- Print total page-documents loaded and a metadata sample.

### Cell 4 — code: split

- `RecursiveCharacterTextSplitter(chunk_size=1000, chunk_overlap=150).split_documents(docs_per_page)`.
- LangChain propagates parent metadata to chunks automatically → page metadata carries through.
- Print chunk count and a sample.

### Cell 5 — code: embed + FAISS + save

- Embedder: `HuggingFaceEmbeddings(model_name=EMBED_MODEL_NAME, model_kwargs={"device":"cpu"}, encode_kwargs={"normalize_embeddings": True})`.
- `FAISS_DIR = "faiss_chunk1000"` — the `chunk1000` suffix mirrors Task 7's convention so we can add `faiss_chunk500`, `faiss_chunk2000` later without renaming.
- If the folder exists, `FAISS.load_local(..., allow_dangerous_deserialization=True)` (trusted local artifact).
- Otherwise `FAISS.from_documents(chunks, embeddings)` → `save_local(FAISS_DIR)`.
- Print vector count for sanity.

### Cell 6 — code: retrieval sanity check on 2–3 questions

- Pick 3 `financebench_id`s (e.g., `00005`, `00080`, `00206`) to cover domain-relevant answers we already reasoned about in Task 1's discussion.
- For each, retrieve top-`K=5`. For each retrieved chunk, report:
  - `doc_name` match vs expected `row["doc_name"]`
  - `page_number` vs expected `row["evidence_page_nums"]`
  - First ~120 chars of chunk text, plus the dataset's first evidence text (truncated) for visual comparison
- Compute two summary signals per question: "chunks from correct doc / K" and "any chunk on an evidence page (yes/no)".
- This surfaces whether failures are at the doc level, the page level, or a text-overlap level — useful framing for Task 7.

### Cell 7 — markdown: observations placeholder

Leave structure + headings for the three questions the assignment asks, with the bodies blank. User will fill after viewing the cell 6 output.

---

## Design notes / gotchas

- **`allow_dangerous_deserialization=True`** is required to `load_local` a FAISS index that contains a pickled docstore. Safe here because the file is one we just wrote.
- **PyPDFLoader `page` metadata is already 0-indexed**, matching `evidence_page_num`. No off-by-one correction needed — just rename the field to `page_number`.
- **Embedding normalization:** BGE models expect normalized embeddings for cosine similarity. Set `normalize_embeddings=True` in `encode_kwargs`.
- **Metadata overwrite vs update:** I overwrite rather than update so PyPDFLoader's extra fields (`source`, `page_label`, etc.) don't leak into downstream prompts.
- **Deterministic doc list:** `sorted(df["doc_name"].unique())` — stable print ordering across reruns.
- **No re-embedding on restart:** the `if FAISS_DIR exists` guard is the save/load contract from the assignment.
- **Dead-link handling:** we already rewrote `doc_link` in dataset prep to point at the FinanceBench GitHub raw URLs; Task 3 just consumes that.

## Dependencies

Already installed in the setup cell: `langchain`, `langchain-community`, `faiss-cpu`, `sentence-transformers`, `pypdf`. No new installs needed.
