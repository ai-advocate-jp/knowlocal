# KnowLocal

[日本語](./README.md) | [English](./README.en.md)

A RAG (Retrieval-Augmented Generation) platform that ingests internal PDF, Word, Excel, and PowerPoint files and answers questions with a local LLM.
Document text never leaves the company LAN — search and answers stay fully on-prem, end to end.

> **This repository publishes the program itself only.**
> No sample data or reference materials are included. Bring your own documents.

Details: https://www.aiadvocate.jp/knowlocal

## Features

- Hybrid retrieval: semantic search (pgvector) + keyword search (BM25), fused with RRF
- Reranking with BGE-reranker-v2-m3, then MMR for diverse, deduplicated grounding
- Multiple local LLMs (via Ollama: qwen3 / gemma3, etc.) answer from the same grounding; the one with the best grounding-consistency score is adopted automatically
- Searchable across the same platform as monitoring logs (ai-monitor-agent3)
- Retrieval accuracy is measured with a real metric (MRR), not guesswork
- Measured example (743command.pdf, 17 questions): vector search only MRR 0.146 → hybrid + reranking **0.618**
  ※ Numbers vary by document and question set. Measure with your own department's evaluation set before rolling out.

## Supported formats

PDF / Word / Excel / PowerPoint / CSV / Markdown / TXT

## Overall flow

```
PDF/Word/Excel/PPT/CSV/MD/TXT
        ↓
  doc_embedding.py (text extraction & splitting)
        ↓
PostgreSQL+pgvector   figure_rag/ColPali (optional visual index)
        ↓
        User's question
        ↓
Semantic search (pgvector)  ×  Keyword search (BM25)
        ↓
      RRF hybrid fusion
        ↓
   Rerank (BGE-reranker-v2-m3)
        ↓
   MMR selects grounding
        ↓
   Ollama (qwen3 / gemma3)
        ↓
  Grounded answer → Open WebUI
```

※ Documents and questions are never sent to an external cloud (aside from the one-time model download).

## Requirements

Recommended (current production-equivalent) and minimum/alternative configurations. Production use assumes an NVIDIA GPU.

| Item | Recommended | Minimum / alternative |
|---|---|---|
| OS | Linux / Windows, both supported | Either works |
| GPU | NVIDIA GPU recommended | CPU-only works but impractically slow |
| VRAM | 16GB (figures + best-of) | 8GB (small models) |
| LLM | Ollama (GPU) | LM Studio also works |
| DB | PostgreSQL + pgvector | Docker or native install |
| UI | Open WebUI (optional) | CLI only also works |
| Monitoring | ai-monitor-agent3 (optional) | Document RAG alone is fine too |
| Python | 3.12 | — |

Not designed for public internet exposure (built for the company LAN).

## Setup

### Linux

```bash
git clone https://github.com/ai-advocate-jp/knowlocal.git
cd knowlocal

# pgvector (Docker)
docker compose up -d

# Python environment
python3 -m venv .venv
source .venv/bin/activate
pip install -r requirements.txt

# Pull local LLM models
ollama pull qwen3:8b gemma3:12b

# Start the server
uvicorn best_of_server:app --host 0.0.0.0 --port 8100
```

### Windows (works without Docker)

- Install PostgreSQL + pgvector natively
- Install Ollama for Windows + the latest NVIDIA driver
- Install the CUDA build of torch first, then `pip install -r requirements.txt`

See `MANUAL.md`, chapter 8, for details.

## Ingesting and embedding documents

1. `doc_embedding.py` reads target documents (PDF / Word / Excel / PowerPoint / CSV / Markdown / TXT) and extracts/splits text.
   - Chunking: `chunk_size=500` / `overlap=100`
   - Splits prefer blank line → line break → 。 → 、, never mid-sentence, with a 100-character overlap so information at a boundary isn't lost.
2. Figures can be added later with `--figures-only` without disturbing the existing text index.
3. Embedding model: `paraphrase-multilingual-MiniLM-L12-v2` (Japanese-aware)
   - Internal documents go to PostgreSQL + pgvector; monitoring logs go to FAISS + pickle.
4. No API restart is needed after ingestion (hot reload).

```bash
python doc_embedding.py --source ./documents
```

## How retrieval and answering work

1. Semantic search (pgvector) and keyword search (BM25) are fused with RRF
2. Reranked with BGE-reranker-v2-m3 (cross-encoder)
3. MMR removes duplicates and selects diverse grounding
4. Multiple models pulled into Ollama (e.g. `qwen3:8b`, `gemma3:12b`) each answer from the same grounding; the one with the best grounding-consistency score is adopted automatically
5. The adopted model name, score, and source (document/page) are shown alongside the answer

Pass `--show-all` on the CLI to inspect every model's answer (more models = more time).

Notes from measured accuracy:
- BM25 is strong for documents dense with command names and part numbers
- Reranking only helps once it's combined with hybrid search
- "Just add vector search and accuracy goes up" is a myth

## Access control & operations

- Roles (viewer / staff / admin), document ACLs, and audit logging are implemented
- `auth_required` defaults to `false`. Cross-department operating design and full SSO rollout are still ahead

## Constraints

- Production use assumes an NVIDIA GPU (CPU-only works but is slow)
- On 16GB VRAM, be careful running figure search and image generation at the same time
- Manuals with many near-identical commands can sometimes point to the wrong page
  → Mitigate by showing multiple sources, letting users ask for a specific page ("what's the procedure on p.X?"), or ingesting chapter by chapter

## How it compares

Dify, LangChain, and LM Studio aren't competitors — they sit on a different layer. KnowLocal wins on retrieval accuracy, explainability, and staying fully local; it's weaker on non-engineer UI, agents, and connector breadth. The accuracy know-how here can be ported onto Dify or LangChain too.

## License

This project is released under the [MIT License](./LICENSE).

## Disclaimer

This tool is provided as-is. Handling of any documents you ingest is your own responsibility.

## Related projects

- [AI Monitor Agent](https://www.aiadvocate.jp) (GPU/system monitoring agent, written in Rust)

## Developed by

Across Systems Corporation / [AI Advocate](https://www.aiadvocate.jp)
