# ai-rag-cli

A local, CLI-first Retrieval-Augmented Generation (RAG) toolkit built on FAISS and [Ollama](https://ollama.com/), with a benchmarking script for comparing local LLM models. The scripts currently in this repository index PDF documents, but the pipeline itself is not PDF-specific — document-format support is expected to broaden over time (see `doc_embedding.py`). Originally prototyped with Cursor.

日本語版 README は [README.ja.md](./README.ja.md) をご覧ください。
Detailed usage instructions are in [MANUAL.md](./MANUAL.md) ([日本語版](./MANUAL.ja.md)).

## What this actually does (confirmed from source)

This repository embeds documents chunk-by-chunk with a multilingual sentence-transformer model, stores the vectors in a FAISS index, and answers questions from the command line by retrieving relevant chunks and passing them as context to a local Ollama model. `pdf_embedding.py` handles PDF ingestion today (extracting text per page); `doc_embedding.py` is a parallel pipeline aimed at documents more generally. Both feed the same FAISS/pickle storage layer defined in `rag_common.py`.

```
Document (PDF today; other formats via doc_embedding.py)
   │
   ▼
pdf_embedding.py / doc_embedding.py → text extraction, chunking (RecursiveCharacterTextSplitter),
   │                                    embedding, append to FAISS index
   ▼
document.index (FAISS)  +  document_chunks.pkl (chunk text / source / page)
   │
   ▼
pdf_search.py             → interactive CLI: embed query → FAISS search (top-k) →
   │                          build context-grounded prompt → ask Ollama
   ▼
Answer printed to terminal
```

`benchmark_ollama_models.py` runs a fixed set of questions (about `mobile_pm_jp.pdf`, a mobile printer command manual) across multiple Ollama models, timing each response and cross-referencing CPU/GPU utilization logged separately by [`ai-monitor-agent`](https://github.com/ai-advocate-jp/ai-monitor-agent).

> **Note:** An OpenAI-compatible API server for OpenWebUI integration (`best_of_server.py` / `best_of_search.py`) and the `monitor_*.py` embedding/search scripts exist in the working environment but are not yet documented here — this README will be updated once reviewed.

## Components

| File | Role |
|---|---|
| `rag_common.py` | Shared config (embedding model name, Ollama model name, default paths) and index/chunk load & save helpers |
| `pdf_embedding.py` | Extracts text per PDF page, splits into chunks (default 500 chars, 100 overlap), embeds with `EMBEDDING_MODEL_NAME`, appends to the FAISS index. Refuses to append if the existing index's embedding dimension doesn't match. |
| `pdf_search.py` | Interactive CLI loop: embeds the query, retrieves top-k chunks by L2 distance, builds a context-grounded Japanese prompt, and (unless `--no-llm`) sends it to Ollama for an answer |
| `benchmark_ollama_models.py` | Compares `qwen3:4b`, `qwen3:8b`, `gemma3:4b`, `gemma3:12b` on a fixed question set, recording latency, thinking-token length (for Qwen3's chain-of-thought), and CPU/GPU/VRAM/power stats pulled from `ai-monitor-agent`'s CSV logs |
| `best_of_server.py`, `best_of_search.py` | *(not yet reviewed — likely the OpenWebUI-facing API layer)* |
| `monitor_embedding.py`, `monitor_search.py`, `monitor_summary_embedding.py` | *(not yet reviewed)* |
| `doc_embedding.py` | *(not yet reviewed — general-document embedding pipeline)* |

## Configuration (`rag_common.py`)

- Embedding model: `sentence-transformers/paraphrase-multilingual-MiniLM-L12-v2`
- Default Ollama model: `gemma3:4b`
- Default index: `document.index` / default chunks: `document_chunks.pkl`

Changing the embedding model requires deleting the existing index/chunks files, since the vector dimension must match across the whole index.

## Requirements

- Python 3, virtual environment recommended
- [Ollama](https://ollama.com/) running locally with the target model(s) pulled (e.g. `ollama pull gemma3:4b`)
- Dependencies in `requirements.txt` (`faiss`, `sentence-transformers`, `pypdf`, `langchain-text-splitters`, `ollama`, `numpy`)

```bash
python -m venv env-pdf
source env-pdf/bin/activate
pip install -r requirements.txt
```

## Quick start

```bash
# 1. Build/append to the index from a PDF
python pdf_embedding.py path/to/document.pdf

# 2. Ask questions interactively
python pdf_search.py
```

See [MANUAL.md](./MANUAL.md) for all command-line options and the benchmarking workflow.

## Before making this public — things to check

- `benchmark_ollama_models.py` hardcodes `MONITOR_LOG_DIR = Path("/home/haraki/ai-monitor-agent2/logs")`. Consider making this an environment variable or CLI argument before publishing, since it currently bakes in a local username/path.
- Confirm no PDF content, index files, or chunk pickles are tracked (already excluded via `.gitignore`).

## Roadmap

This repository currently covers the **CLI version only**. A follow-up release will add OpenWebUI integration (an OpenAI-compatible API server), published separately once the server-side code has been reviewed and documented. Pulling and managing Ollama models is left to whoever runs this — no models are bundled or auto-downloaded by this repo.

## Status

Early-stage personal infrastructure project, prototyped with Cursor. Interfaces and file layout may change without notice.

## License

TBD.
