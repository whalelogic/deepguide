# Local Embeddings + Search with Ollama (Python)

This guide shows how to:
1) run Ollama locally
2) generate embeddings for your documents
3) store them on disk
4) query them later using cosine similarity

No external vector DB required (everything is local).

---

## Prerequisites

- Python 3.9+
- Ollama installed and running locally

### Install Ollama

Follow the official instructions: https://ollama.com

After installing, start Ollama (it usually runs as a background service).

Verify it’s running:

```bash
ollama --version
```

---

## Choose and pull an embeddings model

Ollama can serve embedding models. A commonly used one is:

- `nomic-embed-text`

Pull it:

```bash
ollama pull nomic-embed-text
```

You can confirm it exists:

```bash
ollama list
```

---

## Create a Python environment

```bash
python -m venv .venv
source .venv/bin/activate  # macOS/Linux
# .venv\Scripts\activate   # Windows

pip install --upgrade pip
pip install ollama numpy
```

Notes:
- `ollama` is the Python client for your local Ollama server.
- `numpy` is used for cosine similarity and basic math.

---

## How embeddings work (quick mental model)

- An embedding is a vector (list of floats) representing the meaning of text.
- Similar meanings produce similar vectors.
- For search:
  1) embed each document chunk
  2) embed the query text
  3) find the document vectors closest to the query vector (cosine similarity)

---

## Example: embed documents and save to disk

Create `embed_and_index.py`:

```python
import json
from pathlib import Path

import numpy as np
import ollama


MODEL = "nomic-embed-text"
INDEX_PATH = Path("embeddings_index.json")


def embed(text: str) -> list[float]:
    # Ollama Python client call:
    # response = { "embedding": [ ... floats ... ] }
    resp = ollama.embeddings(model=MODEL, prompt=text)
    return resp["embedding"]


def chunk_text(text: str, max_chars: int = 800) -> list[str]:
    """
    Minimal chunker to keep the demo simple.
    For real workloads, chunk by tokens, sentences, or semantic boundaries.
    """
    text = text.strip()
    if not text:
        return []
    return [text[i : i + max_chars] for i in range(0, len(text), max_chars)]


def main():
    # Example documents (replace with files, DB rows, etc.)
    docs = [
        {
            "id": "doc-1",
            "title": "Getting started",
            "text": "Ollama lets you run models locally. You can generate embeddings for semantic search.",
        },
        {
            "id": "doc-2",
            "title": "Vector search",
            "text": "To query embeddings, embed the user question and compare with cosine similarity.",
        },
        {
            "id": "doc-3",
            "title": "Unrelated",
            "text": "Bananas are rich in potassium and are a popular fruit.",
        },
    ]

    index = {"model": MODEL, "items": []}

    for d in docs:
        chunks = chunk_text(d["text"])
        for chunk_i, chunk in enumerate(chunks):
            vec = embed(chunk)
            index["items"].append(
                {
                    "doc_id": d["id"],
                    "title": d["title"],
                    "chunk_index": chunk_i,
                    "text": chunk,
                    "embedding": vec,
                }
            )

    INDEX_PATH.write_text(json.dumps(index, indent=2), encoding="utf-8")
    print(f"Wrote {len(index['items'])} embedding(s) to {INDEX_PATH}")


if __name__ == "__main__":
    main()
```

Run it:

```bash
python embed_and_index.py
```

This writes an `embeddings_index.json` file containing:
- document chunk text
- its embedding vector
- metadata (doc id, title, etc.)

---

## Example: query the index (cosine similarity)

Create `query_index.py`:

```python
import json
from pathlib import Path

import numpy as np
import ollama


MODEL = "nomic-embed-text"
INDEX_PATH = Path("embeddings_index.json")


def embed(text: str) -> np.ndarray:
    resp = ollama.embeddings(model=MODEL, prompt=text)
    return np.array(resp["embedding"], dtype=np.float32)


def cosine_similarity(a: np.ndarray, b: np.ndarray) -> float:
    # Avoid division by zero
    denom = (np.linalg.norm(a) * np.linalg.norm(b))
    if denom == 0.0:
        return 0.0
    return float(np.dot(a, b) / denom)


def main():
    if not INDEX_PATH.exists():
        raise SystemExit(
            f"Missing {INDEX_PATH}. Run `python embed_and_index.py` first."
        )

    index = json.loads(INDEX_PATH.read_text(encoding="utf-8"))

    # Optional: check the stored model matches what you're using now.
    stored_model = index.get("model")
    if stored_model != MODEL:
        print(f"Warning: index built with model={stored_model}, querying with model={MODEL}")

    query = input("Query: ").strip()
    if not query:
        raise SystemExit("Empty query.")

    q_vec = embed(query)

    scored = []
    for item in index["items"]:
        d_vec = np.array(item["embedding"], dtype=np.float32)
        score = cosine_similarity(q_vec, d_vec)
        scored.append((score, item))

    scored.sort(key=lambda x: x[0], reverse=True)

    top_k = 5
    print("\nTop matches:\n")
    for rank, (score, item) in enumerate(scored[:top_k], start=1):
        print(f"{rank}. score={score:.4f}  doc_id={item['doc_id']}  title={item['title']}")
        print(f"   text={item['text']}\n")


if __name__ == "__main__":
    main()
```

Run it:

```bash
python query_index.py
```

Try queries like:
- `how do I query embeddings?`
- `semantic search with ollama`
- `potassium fruit`

---

## Common troubleshooting

### 1) “Connection refused” / cannot reach Ollama
Make sure the Ollama service is running. If you installed the desktop app, open it once.
Then retry.

### 2) Model not found
Pull it first:

```bash
ollama pull nomic-embed-text
```

### 3) Results are low quality
- Chunk your docs more carefully (by sentence/paragraph, overlap chunks).
- Ensure you embed *chunks* (not entire long documents).
- Consider a different embeddings model available in Ollama.

---

## Next steps (optional upgrades)

- **Chunking**: add overlap (e.g., 100–200 chars) and chunk by paragraphs/sentences.
- **Persistence**: store embeddings in SQLite instead of JSON for faster loads.
- **RAG**: after retrieving top chunks, pass them to an LLM (also via Ollama) to generate an answer with citations.

---