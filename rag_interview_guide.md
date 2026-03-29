# RAG Technical Interview Guide (E-commerce Agent)

This guide provides interview-ready answers based on the implementation in the `ecommerce-agent` repository.

## 🚀 Why RAG?

### Why RAG instead of Fine-tuning?
- **Knowledge Freshness**: Fine-tuning is static. RAG allows immediate updates by simply swapping markdown files in `data/policies/`.
- **No Retraining Cost**: Fine-tuning requires expensive GPU hours; RAG only requires a quick re-indexing of documents.
- **Attribution (Citations)**: RAG provides clear citations (e.g., `03-returns-perishables-food.md`), which is critical for support auditability. Fine-tuning is a "black box."
- **Data Privacy**: RAG can filter data at retrieval time based on user permissions, whereas fine-tuning bakes all data into the model weights.

### When should you NOT use RAG?
- When the task requires learning a *style*, *persona*, or *complex logic* that isn't documented (Fine-tuning is better for this).
- When the context is very small (just use the prompt).
- When the data is highly interconnected and requires global reasoning (e.g., "Summarize the sentiment of 1000 reviews").

### Difference between RAG and traditional search?
- **Traditional Search (BM25)**: Keyword-based. If you search "spoiled food," it might miss "melted cookies."
- **RAG (Semantic)**: Uses embeddings to understand *meaning*. It retrieves context based on intent, then uses an LLM to generate a natural response instead of just providing links.

---

## 📊 Data Collection & Preprocessing

### Source & Format
- **Source**: Local file system (`data/policies/`).
- **Format**: Markdown (`.md`).
- **Handling**: Used `Path.glob("*.md")` to iterate and `Document` objects from LangChain to store content and metadata (filename).

### Data Quality (Noise Removal)
- **Deduplication**: Handled by maintaining unique filenames and a single source of truth in the `data/` directory.
- **Normalization**: Handled during chunking by `RecursiveCharacterTextSplitter`, which respects structural boundaries like paragraphs and lists.

---

## ✂️ Chunking Strategy

### Chunk Size & Overlap
- **Selection**: `CHUNK_SIZE = 500` characters, `CHUNK_OVERLAP = 100`.
- **Reasoning**: 500 characters (~100-150 tokens) is small enough for granular retrieval but large enough to contain a full policy rule.
- **Overlap**: 100 characters ensure that context (like "Exceptions to this rule:") isn't cut off at the boundary between chunks.

### Tradeoffs
- **Large Chunks**: Better context, but more "noise" (irrelevant text) and higher prompt costs.
- **Small Chunks**: Precise retrieval, but might lose necessary surrounding context.

---

## 🔢 Embeddings

### Model: `sentence-transformers/all-MiniLM-L6-v2`
- **Why?**: Extremely efficient, low latency, and runs locally on CPU/GPU without API costs. It's a gold standard for "lightweight" semantic search.
- **Dimensions**: 384.
- **Latency vs Accuracy**: Offers a great balance; it handles the e-commerce retrieval tasks (refunds, shipping) with high precision while being much faster than larger models like `e5-large-v2`.

---

## 🗄️ Vector Database

### Database: FAISS (Facebook AI Similarity Search)
- **Why?**: It's a library, not a managed service (like Pinecone). Perfect for local development and high-performance in-memory search.
- **Indexing**: Uses exact similarity search (Flat index) by default for this scale, ensuring perfect recall.
- **SQL vs Vector**: Vector DBs allow "nearest neighbor" search in high-dimensional space, which SQL can't do efficiently without specialized extensions (like pgvector).

---

## 🔍 Retrieval & Query Embedding

### Search Method
- **Similarity Search (Cosine Similarity)**: Ranks chunks by how close they are to the query embedding in vector space.
- **Top-K (k=3)**: I chose `k=3` because e-commerce policies are usually concise. Three chunks provide enough context without exceeding LLM token limits or introducing confusion.

### Process
1. Query ("My food arrived spoiled") is embedded using the *same* `all-MiniLM-L6-v2` model.
2. FAISS finds the top 3 closest vectors.
3. Chunks are passed to the generator.

---

## 🧠 Prompt Construction

### Template Structure
- **Strict Roles**: "You are a strict e-commerce customer support assistant."
- **Formatting Rules**: Forces a specific format (Classification, Decision, Rationale, etc.).
- **Grounding**: Instructs the LLM to only use provided policies and *never* assume facts.

---

## ⚠️ Hallucination Handling

### Techniques Used
1. **Citation Enforcement**: LLM must list the exact source file (e.g., `returns-policy.md`).
2. **Confidence-based Fallback**: If the LLM output is malformed (missing sections), a Python-based rule engine (`is_valid_response`) intercepts and generates a safe "grounded" response based on keywords.
3. **Low Temperature (0.2)**: Reduces randomness and "creativity."

---

## 📈 Evaluation

### Methodology (`evaluate.py`)
- **Dataset**: 20 hand-crafted tickets covering standard cases, exceptions (perishables), conflicts, and out-of-scope queries.
- **Metrics**:
    - **Citation Coverage Rate**: Measures how often the agent correctly cites a policy.
    - **Escalation Rate**: Specifically checks if "Conflict" or "Not-in-policy" cases correctly trigger an escalation.
- **Tools**: Custom implementation using Python/JSON-based reporting.
