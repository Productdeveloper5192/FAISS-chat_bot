# FAISS PDF Chatbot (Groq)

Upload any PDF and ask questions about it — the app chunks and embeds the document into a local FAISS vector index on the fly, then answers questions grounded in the retrieved chunks using Groq's hosted Llama 3.3 70B.

## What It Does

- **Upload a PDF** through the Streamlit UI — no pre-processing required.
- **Builds a fresh FAISS index per upload**: extracts text (`PyPDF2`), splits it into overlapping chunks, embeds each chunk locally with `sentence-transformers/all-MiniLM-L6-v2`, and writes the index to disk.
- **Answers questions about the uploaded document** via a LangChain `RetrievalQA` chain: retrieve the most relevant chunks, then have Groq's `llama-3.3-70b-versatile` answer from them.
- **Re-indexes automatically** when a different file is uploaded, tracked via Streamlit session state so re-asking a question against the same file doesn't rebuild the index.

## End-to-End Flow

```
 User uploads a PDF via Streamlit
        │
        ▼
 extract_text_from_pdf()  — PyPDF2 reads every page into one text blob
        │
        ▼
 create_faiss_vector_store()
   - RecursiveCharacterTextSplitter: 1000-char chunks, 200-char overlap
   - each chunk embedded with sentence-transformers/all-MiniLM-L6-v2
   - FAISS.from_texts() builds the index, saved to ./faiss_index
        │
        ▼
 build_qa_chain()
   - loads the FAISS index back off disk
   - wraps it as a retriever
   - ChatGroq(model="llama-3.3-70b-versatile") + a "stuff" QA chain
     combine retrieved chunks into the model's context
        │
        ▼
 User asks a question in the text input
        │
        ▼
 RetrievalQA.run(question)
   - retriever pulls the most relevant chunks from FAISS
   - Groq generates an answer grounded in those chunks
        │
        ▼
 Answer displayed in the UI
```

Re-uploading a different PDF re-runs the whole pipeline and replaces the index; asking another question against the same PDF reuses the already-built chain from session state.

## Tech Stack

| Layer | Technology |
|---|---|
| UI | Streamlit |
| PDF parsing | PyPDF2 |
| Chunking | LangChain `RecursiveCharacterTextSplitter` |
| Embeddings | `sentence-transformers/all-MiniLM-L6-v2` (local, via `langchain-huggingface`) |
| Vector store | FAISS (local, `faiss-cpu`) |
| Generation | Groq — `llama-3.3-70b-versatile` (via `langchain-groq`) |

## Project Layout

```
chat_bot.py                       Main Streamlit app — upload, index, and QA
requirements.txt                  Dependencies
Rama_krishna.pdf                  Sample PDF for trying the app out
Vector_db_chatbot.ipynb           Earlier prototype notebook (local Ollama/llama3.2
                                   instead of Groq; hardcoded local file path — kept
                                   for reference, superseded by chat_bot.py)
.devcontainer/                    VS Code dev container config
```

## Setup

### 1. Install dependencies
```bash
pip install -r requirements.txt
```

### 2. Add your Groq API key
Create `.streamlit/secrets.toml` (copy from `.streamlit/secrets.toml.example`) with:
```toml
GROQ_API_KEY = "your_groq_api_key_here"
```
Get a key from [console.groq.com](https://console.groq.com/keys). This file is gitignored and should never be committed.

### 3. Run it
```bash
streamlit run chat_bot.py
```
Open the URL Streamlit prints (typically `http://localhost:8501`), upload a PDF, and start asking questions about it.

## Security Note

A previous commit had a live Groq API key hardcoded directly in `chat_bot.py` and duplicated in a tracked `secrets.toml`. Both have been fixed to read from `.streamlit/secrets.toml` via `st.secrets` (gitignored) instead — **if you're reusing this repo, rotate that key**, since anything committed to a public repo's history should be treated as compromised even after removal.
