# GitRAG:Chat with Any Repository, Locally

> **Production-grade local RAG system for chatting with any code repository.**

GitRAG ingests a local repository, builds a semantic + structural index using AST-aware chunking and hybrid search, and supports conversational multi-turn querying(all fully offline).

---

## Architecture

### Design Decision

**Selected Architecture: Microkernel + Plugin.** Each component (loader, chunker, embedder, retriever, generator) implements an abstract interface and is swappable without re-plumbing the pipeline. This avoids the rigidity of a monolith while keeping the system simple enough to run locally.

### Data Flow

```
Repository Files
      │
      ▼
File Loader (language detection, binary filtering)
      │
      ▼
AST Chunker (tree-sitter) ── Fallback: Text Chunker
      │
      ├──▶ Vector Store   (ChromaDB, cosine similarity)
      ├──▶ BM25 Store     (rank_bm25, code-aware tokenization)
      └──▶ Call Graph     (networkx, import + symbol resolution)

User Query
      │
      ▼
Intent Classification ──▶ Retrieval Strategy Selection
      │
      ▼
Query Reformulation (for follow-up queries)
      │
      ▼
Hybrid Retrieval
      ├── Dense:  Vector similarity search
      ├── Sparse: BM25 keyword search
      └── Reciprocal Rank Fusion (RRF)
      │
      ▼
Cross-Encoder Reranking
      │
      ▼
Multi-Hop Graph Expansion (flow trace queries)
      │
      ▼
Context Compression + Deduplication
      │
      ▼
LLM Generation (context-grounded, citations enforced)
```

### AST-Aware Chunking. Why?

Naïve fixed-size chunking (e.g. 512-token windows with overlap) is fundamentally broken for code:

1. **Semantic boundary violation** — A 512-token window can split a function in half, losing the connection between signature and body.
2. **Context loss** — The chunk loses import context, class membership, and docstrings.
3. **Redundant overlap** — Sliding-window overlap wastes embedding space on duplicated content.
4. **No structural metadata** — Fixed chunks can't carry symbol names, file paths, or dependency references.
5. **Cross-language inconsistency** — Code structure varies by language; fixed windows ignore this.

AST-aware chunking extracts **function, class, and method boundaries** directly from the parse tree, preserving complete function bodies as atomic units, docstrings attached to their parent symbols, import context for dependency resolution, and structural metadata (symbol kind, parent class, line numbers).

### Hybrid Search over Pure Vector. Why?

Pure vector search fails on exact symbol names ("Where is `calculateTaxRate` defined?" — vector search may miss the exact token match), rare identifiers with poor embedding coverage, and short queries like "AutoSaveService" that have weak semantic signal. BM25 handles exact token matching perfectly. Hybrid search combines the semantic understanding of vectors with the precision of keyword search, then uses Reciprocal Rank Fusion to merge the ranked lists without requiring score normalization.

### Security Analysis

GitRAG runs **Semgrep** static analysis on every repository at ingest time. Semgrep scans all source files against a set of security rules and flags potential vulnerabilities like SQL injection, hardcoded secrets, insecure deserialization, and more.

The findings are attached to the relevant code chunks in the index. When a security-related question like "are there any vulnerabilities in this code?" or "is this endpoint secure?" is asked, GitRAG combines the Semgrep findings with the retrieved code context to give a security-aware answer grounded in the actual code.

If no vulnerabilities are found, the ingest summary will show `0 security findings`.

---

## Hallucination Mitigation

GitRAG uses five layers of hallucination control:

1. **Low temperature (0.1)** — Reduces creative/hallucinated responses. The LLM stays close to its input context.
2. **Strict system prompt** — The LLM is explicitly instructed: *"Answer using ONLY the code provided in context. If the answer is not in the provided code, say so clearly."*
3. **Citation requirement** — The prompt requires the model to reference actual function names, variable names, and line numbers from the context.
4. **Context grounding** — Only retrieved, verified code chunks from the ingested repo are passed to the LLM. The model cannot draw from general training knowledge.
5. **Cross-encoder reranking** — A separate reranker model scores all retrieved chunks for relevance before they reach the LLM, ensuring the most relevant actual code is prioritized over loosely related chunks.

---

## Prerequisites

- [Docker Desktop](https://www.docker.com/products/docker-desktop/) installed and running
- **8GB RAM minimum** (6GB absolute minimum with smaller models)
- **10GB free disk space**
- Git installed

**Windows only — increase WSL 2 memory:**

Create or edit `C:\Users\<YourName>\.wslconfig`:
```ini
[wsl2]
memory=6GB
processors=4
```
Then run `wsl --shutdown` and restart Docker Desktop.

---

## Quick Start

```bash
# 1. Clone
git clone https://github.com/pranaviii29/GitRAG.git
cd GitRAG

# 2. Create your .env file (see Configuration section below for all variables)

# 3. Start all services
docker compose up -d

# 4. Pull a model (first time only)
docker exec gitrag_ollama ollama pull llama3.2:1b

# 5. Clone a repo to analyze
git clone https://github.com/user/myproject repos/myproject

# 6. Open http://localhost — ingest /app/repos/myproject — start chatting
```

First run downloads Docker images and the LLM model (~1–5GB depending on model). All subsequent runs start in under 30 seconds with no downloads.

---

## Services

| Service | Port | Description |
|---|---|---|
| `gitrag_nginx` | 80 | Reverse proxy — main entry point |
| `gitrag_frontend` | 3000 | React UI |
| `gitrag_backend` | 8000 | FastAPI RAG pipeline |
| `gitrag_ollama` | 11434 | Local LLM runtime |

Open `http://localhost` in your browser.

## API Endpoints

| Method | Path | Description |
|---|---|---|
| `POST` | `/api/ingest` | Index a repository |
| `POST` | `/api/query` | Query with conversation support |
| `DELETE` | `/api/memory/{session_id}` | Clear conversation history |
| `GET` | `/api/health` | Health check |

---

## Configuration

Create a `.env` file in the project root with the following variables:

```env
# LLM
OLLAMA_HOST=http://ollama:11434
LLM_MODEL=llama3.2:1b

# Embedding & Reranking (downloaded once, cached permanently)
EMBEDDING_MODEL=BAAI/bge-base-en-v1.5
RERANKER_MODEL=BAAI/bge-reranker-base

# Storage paths
CHROMA_PATH=/app/data/chroma
MEMORY_PATH=/app/data/memory
REPOS_PATH=/app/data/repos
SEMGREP_RULES=gitrag/semgrep_ext/rules/

# API
API_HOST=0.0.0.0
API_PORT=8000
API_WORKERS=1
CORS_ORIGINS=http://localhost:3000,http://frontend:3000

# Retrieval
VECTOR_TOP_K=20
BM25_TOP_K=20
RERANK_TOP_K=5
GRAPH_HOPS=1

# Flow tracer
FLOW_MAX_DEPTH=6
FLOW_MAX_CHILDREN=10

# Generation
GEN_MAX_TOKENS=2048
GEN_TEMPERATURE=0.1
GEN_CONTEXT_LIMIT=6000

# Logging
LOG_LEVEL=INFO
LOG_FORMAT=json
```

### Recommended Models by Hardware

| Hardware | Model | Command | Size |
|---|---|---|---|
| CPU only / low RAM | `llama3.2:1b` | `ollama pull llama3.2:1b` | 1.3 GB |
| CPU with 8GB+ RAM | `llama3.2` | `ollama pull llama3.2` | 2 GB |
| NVIDIA GPU | `llama3` | `ollama pull llama3` | 4.7 GB |
| NVIDIA GPU (code-optimized) | `codellama:7b` | `ollama pull codellama:7b` | 3.8 GB |
| AMD GPU (Linux + ROCm) | `llama3.2` | `ollama pull llama3.2` | 2 GB |

To switch models:
```bash
docker exec gitrag_ollama ollama pull llama3.2
# Edit .env: LLM_MODEL=llama3.2
docker exec gitrag_ollama ollama rm llama3.2:1b   # free up space
docker restart gitrag_backend
```

---

## Project Structure

```
gitrag/
├── backend/
│   ├── Dockerfile
│   ├── pyproject.toml             # Python dependencies
│   └── gitrag/
│       ├── core/
│       │   ├── config.py          # Settings from .env
│       │   ├── pipeline.py        # Master orchestrator
│       │   ├── types.py           # CodeChunk, Intent, etc.
│       │   └── exceptions.py
│       ├── ingest/
│       │   ├── loader.py          # Repository file discovery
│       │   ├── language.py        # Language detection
│       │   └── filters.py         # Binary/ignore filtering
│       ├── chunking/
│       │   ├── ast_chunker.py     # tree-sitter AST-aware chunking
│       │   └── text_chunker.py    # Fallback for docs/config
│       ├── embeddings/
│       │   └── local.py           # Sentence-transformers embedder
│       ├── index/
│       │   ├── vector_store.py    # ChromaDB vector index
│       │   ├── bm25_store.py      # BM25 sparse index
│       │   └── graph_store.py     # Dependency graph (networkx)
│       ├── retrieval/
│       │   ├── hybrid.py          # Hybrid retriever orchestrator
│       │   ├── fusion.py          # Reciprocal Rank Fusion
│       │   └── reranker.py        # Cross-encoder reranking
│       ├── query/
│       │   ├── intent.py          # Intent classification
│       │   ├── reformulator.py    # Follow-up query reformulation
│       │   └── multi_hop.py       # Graph-based context expansion
│       ├── memory/
│       │   └── conversation.py    # Multi-turn conversation state
│       ├── generation/
│       │   ├── llm.py             # Ollama LLM client
│       │   ├── prompts.py         # System prompts per intent type
│       │   └── context.py         # Context compression + assembly
│       ├── flow_tracer/
│       │   ├── orchestrator.py    # Flow trace entry point
│       │   ├── call_graph.py      # Call graph builder
│       │   ├── deep_tracer.py     # Recursive execution tracer
│       │   ├── resolver.py        # Symbol resolution
│       │   └── narrative.py       # Human-readable trace formatter
│       ├── semgrep_ext/
│       │   ├── runner.py          # Semgrep static analysis runner
│       │   ├── tagger.py          # Tag chunks with findings
│       │   └── rules/
│       │       └── security.yaml  # Custom security rules
│       └── api/
│           └── server.py          # FastAPI routes
├── frontend/
│   ├── Dockerfile
│   └── src/
│       ├── components/            # React UI components
│       ├── services/api.ts        # Backend API client
│       ├── store/                 # State management
│       └── hooks/                 # React hooks
├── nginx/
│   └── nginx.conf                 # Reverse proxy config
├── docker-compose.yml
├── .gitignore
└── README.md
```

---

## Persistent Storage

GitRAG uses Docker named volumes so data survives `docker compose down`:

| Volume | Contents | Effect if deleted |
|---|---|---|
| `ollama_data` | Ollama LLM models | Must re-pull model (~1–5GB) |
| `app_data` | ChromaDB vectors, BM25 index, conversation memory | Must re-ingest repo |

**Never run `docker compose down -v`** — the `-v` flag deletes all volumes.

---

## Memory Strategy

Three-tier conversation memory keeps multi-turn chats coherent:

1. **Short-term buffer**: Last N turns kept verbatim.
2. **Rolling summary**: Older turns compressed and fed as context prefix.
3. **Context window optimization**: Summary + recent turns fit within the LLM's token budget; oldest turns dropped first when budget is exceeded.

Query reformulation detects follow-up queries (pronouns, short queries referencing prior context) and prepends context from prior turns to create standalone queries before retrieval.

---

## Supported Languages

AST-aware chunking is supported for: Python, JavaScript, TypeScript, Java, Go, Rust, C, C++, C#, Ruby, PHP.

All other file types fall back to the text chunker, which splits on logical boundaries rather than fixed token windows.

---

## Troubleshooting

**Response times out**

Increase the nginx timeout in `nginx/nginx.conf`:
```nginx
proxy_read_timeout 600s;
```
Then: `docker restart gitrag_nginx`

**"model requires more system memory than is available"**

Increase Docker/WSL memory. Edit `.wslconfig` and set `memory=6GB`, then `wsl --shutdown` and restart Docker Desktop.

**Backend stuck on "reranker_loading"**

The reranker model is downloading. Monitor with `docker stats gitrag_backend --no-stream`. If NET I/O is increasing, it's downloading. If stuck: `docker restart gitrag_backend`

**"Repo not found" when ingesting**

Clone your repo into the `repos/` folder and enter the container path `/app/repos/<foldername>`, not a Windows path.

**Free up disk space**
```bash
docker builder prune   # clears build cache
docker system prune    # clears unused containers/images
```

---

## Future Implementations

GitRAG works end-to-end but here are some future improvements planned:

- [ ] **Smarter Indexing** — Right now, every ingest re-processes the entire repository from scratch. The next step is content-hash based diffing — only files that actually changed since the last ingest get re-chunked and re-embedded. For large repos this would cut ingest time from minutes to seconds.

- [ ] **Better Embedding for Code** — The current embedding model (`BAAI/bge-base-en-v1.5`) is a general-purpose text model. It works well, but a model trained specifically on source code (like `jina-embeddings-v2-base-code`) would give meaningfully better retrieval for things like variable names, function signatures, and code idioms.

- [ ] **Multi-Repository Support** — Currently GitRAG indexes one repo at a time. The plan is to support indexing multiple repos simultaneously with a namespace-aware vector store, supporting questions that span across projects.

- [ ] **Ingest from GitHub URL** — Instead of manually cloning a repo into `repos/`, paste a GitHub URL directly into the UI and GitRAG clones and ingests it automatically.

- [ ] **CLI Interface** — A lightweight command-line tool for terminal-based querying, useful for scripting, automation, and developers who prefer not to open a browser.

- [ ] **Chat Export** — Export the current conversation as a markdown file, useful for turning a code exploration session into documentation or a PR review summary.

---

## License

MIT
