# SangamAI — System Architecture

## 1. High-Level Overview

SangamAI is a multi-modal AI assistant platform that allows users to upload and interact with:
- **PDF documents** (via RAG - Retrieval Augmented Generation)
- **YouTube videos** (transcript extraction + RAG)
- **CSV/Excel data** (via Pandas DataFrame agent for analysis)

The system has **two interfaces**:
1. **FastAPI REST API** (`server/main.py`) — served by Uvicorn, consumed by the Next.js client
2. **Streamlit UI** (`server/app.py`) — legacy self-contained web UI

Both interfaces share the same backend modules and Firebase infrastructure.

---

## 2. Technology Stack

| Layer | Technology |
|-------|-----------|
| **Frontend Client** | Next.js 14 (App Router), React, TypeScript, Tailwind CSS |
| **Server API** | FastAPI, Uvicorn |
| **Legacy UI** | Streamlit |
| **Authentication** | Firebase Auth (client-side JS SDK + server-side Admin SDK) |
| **Database** | Google Cloud Firestore (NoSQL document store) |
| **Vector Store** | FAISS (Facebook AI Similarity Search) |
| **Embeddings** | HuggingFace `all-MiniLM-L6-v2` (local, free) |
| **LLM Provider** | OpenRouter API (unified gateway for multiple models) |
| **RAG Framework** | LangChain (LCEL pipelines) |
| **Data Analysis** | Pandas + LangChain Experimental Pandas Agent |
| **PDF Processing** | PyPDFLoader (LangChain community) |
| **YouTube** | `youtube-transcript-api` |
| **Secrets** | `python-dotenv` (.env files) |

---

## 3. Project Directory Structure

```
SangamAI/
├── client/                          # Next.js frontend
│   ├── app/
│   │   ├── layout.tsx               # Root layout
│   │   ├── page.tsx                 # Landing/home page
│   │   ├── login/                   # Login page (client-side Firebase Auth)
│   │   ├── chat/                    # Chat interface (API calls to FastAPI)
│   │   └── profile/                 # User profile settings
│   ├── lib/
│   │   ├── firebase.ts              # Firebase JS SDK initialization
│   │   ├── auth-context.tsx         # React context for auth state
│   │   └── api.ts                   # Fetch wrappers for FastAPI endpoints
│   ├── .env                         # NEXT_PUBLIC_FIREBASE_* vars
│   └── public/                      # Static assets
│
├── server/                          # FastAPI backend
│   ├── main.py                      # App entry, Firebase init, router registration
│   ├── app.py                       # Streamlit entry (legacy)
│   ├── middleware.py                 # Auth dependency (verify Firebase ID token)
│   ├── .env                         # FIREBASE_SERVICE_ACCOUNT, CORS_ORIGINS, FIREBASE_API_KEY
│   ├── serviceAccount.json          # Firebase Admin SDK private key
│   ├── requirements.txt             # Python dependencies
│   │
│   ├── modules/                     # Core business logic
│   │   ├── database.py              # Firestore operations (CRUD for vectorstores, PDFs, chat, CSV)
│   │   ├── llm.py                   # LLM factory (OpenRouter via ChatOpenAI)
│   │   ├── rag.py                   # RAG pipeline (load, chunk, embed, retrieve)
│   │   ├── chains.py                # LCEL chain assembly (condense → retrieve → answer)
│   │   ├── memory.py                # Chat memory builder from Firestore history
│   │   ├── prompts.py               # Prompt templates (QA, condense, YouTube summary)
│   │   ├── agents.py                # Pandas DataFrame agent for CSV queries
│   │   └── auth.py                  # Firebase Auth helpers (register, login, username)
│   │
│   ├── routes/                      # FastAPI route handlers
│   │   ├── auth.py                  # POST /api/auth/register
│   │   ├── upload.py                # POST /api/upload/{pdf,youtube,csv}
│   │   ├── chat.py                  # POST /api/chat/message, GET/DELETE history
│   │   ├── files.py                 # GET/DELETE /api/files/, GET pdf
│   │   └── profile.py               # GET/PUT /api/profile/*, API key management
│   │
│   └── views/                       # Streamlit page renderers (legacy)
│       ├── login.py
│       ├── chat.py
│       └── profile.py
```

---

## 4. Data Model (Firestore)

```
users/{user_id}                      # User profile document
├── email: string
├── username: string
├── api_key: string (OpenRouter key)
└── files/{file_name}                # Each uploaded/processed file
    ├── file_name: string
    ├── content_type: "pdf" | "youtube" | "csv"
    ├── created_at: timestamp
    ├── total_chunks: int            # For vectorstores
    ├── total_size: int              # For vectorstores
    │
    ├── chunks/{chunk_id}            # FAISS vectorstore chunks (binary)
    │   ├── data: bytes
    │   └── chunk_id: int
    │
    ├── pdf_raw/{chunk_id}           # Raw PDF bytes for viewer
    │   ├── data: bytes
    │   └── chunk_id: int
    │
    ├── messages/{auto_id}           # Chat history
    │   ├── role: "user" | "assistant"
    │   ├── content: string
    │   └── timestamp: server timestamp
    │
    └── (for CSV) dataframe: bytes   # Pickled Pandas DataFrame
```

---

## 5. Authentication Flow

### Client-Side (Firebase JS SDK)
- User signs in via email/password using Firebase Auth REST API or JS SDK
- Firebase returns a **custom ID token** (JWT)
- Token is stored in memory/React state
- All API requests include `Authorization: Bearer <token>` header

### Server-Side (Firebase Admin SDK)
- FastAPI middleware (`get_current_user`) extracts the Bearer token
- Calls `auth.verify_id_token(token)` to validate and get the `uid`
- `uid` is injected as `user_id` into route handlers via FastAPI `Depends()`

### Registration
- `POST /api/auth/register` uses Firebase Admin SDK `auth.create_user()` to create the user
- Also creates an initial Firestore document at `users/{uid}` with empty `api_key` and `username`

---

## 6. Content Processing Pipelines

### 6a. PDF Pipeline
1. User uploads PDF via multipart form data
2. Server saves to temp file, runs `PyPDFLoader` to extract text per page
3. `RecursiveCharacterTextSplitter` splits into ~1000-char chunks with 200-char overlap
4. `HuggingFaceEmbeddings` (all-MiniLM-L6-v2) converts chunks to vectors
5. `FAISS.from_documents()` builds the vector index
6. Vector index serialized to bytes, chunked into <700KB pieces, stored in Firestore `chunks/` subcollection
7. Raw PDF bytes also stored in `pdf_raw/` for in-browser viewer

### 6b. YouTube Pipeline
1. User pastes YouTube URL
2. `extract_video_id()` parses the URL
3. `YouTubeTranscriptApi.fetch()` retrieves the transcript
4. Transcript text is chunked and embedded (same as PDF step 3-5)
5. Vectorstore saved to Firestore with `content_type: "youtube"`
6. Video ID becomes `file_name` prefix: `youtube_{video_id}`

### 6c. CSV Pipeline
1. User uploads CSV file
2. `pd.read_csv()` parses it into a DataFrame
3. DataFrame is pickled and stored directly in Firestore under `dataframe` field
4. Frontend shows preview (head 10 rows)

---

## 7. RAG Chat Flow (PDF/YouTube)

```
User Question
     │
     ▼
[Condense Question Step] ── if chat history exists
     │  Uses condense prompt + LLM
     │  "Given chat history, rewrite question as standalone"
     ▼
[Retriever Step]
     │  FAISS similarity search (k=3 chunks)
     ▼
[QA Step]
     │  Stuff chunks + question into QA prompt
     │  LLM generates answer with citations
     ▼
Answer + Source Chunks
     │
     ▼
Persist to Firestore messages/
```

---

## 8. CSV Chat Flow (Pandas Agent)

```
User Question
     │
     ▼
[Pandas Agent]
     │  LangChain Experimental create_pandas_dataframe_agent
     │  Agent generates Python code, executes against DataFrame
     │  Can produce plots via matplotlib
     ▼
Natural Language Answer
     │
     ▼
Persist to Firestore messages/
```

---

## 9. Caching Strategy

- **Vectorstore cache** (`_vs_cache` in `routes/chat.py`): In-memory dict with 600s TTL
  - Key: `"{user_id}:{file_name}"`
  - Stores deserialized FAISS index to avoid repeated Firestore reads
  - Invalidated on file delete

- **Embeddings model**: `@lru_cache(maxsize=1)` singleton — loaded once per server process

---

## 10. LLM Integration

- **Provider**: OpenRouter (`https://openrouter.ai/api/v1`)
- **Library**: LangChain `ChatOpenAI` with custom `base_url`
- **Models available**: Gemini 2.5 Flash, Gemini 3 Flash Preview, Claude Sonnet 4.5, Claude 3.7 Sonnet, GPT-5.2, GPT-4o-mini, Grok 4.1 Fast, Grok 4 Fast
- **User API key**: Stored in Firestore per user, sent with every chat request
- **Temperature**: 0 (deterministic) for all modes

---

## 11. Environment Variables

### Server (`server/.env`)
| Variable | Purpose |
|----------|---------|
| `FIREBASE_SERVICE_ACCOUNT` | Path to Firebase Admin SDK JSON key |
| `FIREBASE_CREDENTIALS` | Alternative: JSON string of credentials |
| `CORS_ORIGINS` | Comma-separated allowed origins |
| `FIREBASE_API_KEY` | Firebase Web API key (for client-side REST calls) |

### Client (`client/.env`)
| Variable | Purpose |
|----------|---------|
| `NEXT_PUBLIC_FIREBASE_API_KEY` | Firebase JS SDK API key |
| `NEXT_PUBLIC_FIREBASE_AUTH_DOMAIN` | Firebase auth domain |
| `NEXT_PUBLIC_FIREBASE_PROJECT_ID` | Firebase project ID |
| `NEXT_PUBLIC_FIREBASE_STORAGE_BUCKET` | Firebase Storage bucket |
| `NEXT_PUBLIC_FIREBASE_MESSAGING_SENDER_ID` | Firebase messaging sender |
| `NEXT_PUBLIC_FIREBASE_APP_ID` | Firebase app ID |

---

## 12. Key Design Decisions

1. **Dual interface**: FastAPI for the Next.js SPA, Streamlit for rapid prototyping. Both share the same `modules/`.
2. **Stateless server**: No server-side sessions. Chat history lives in Firestore, reconstructed on each request via `build_memory_from_history()`.
3. **Client provides API key**: OpenRouter keys are stored per-user in Firestore, avoiding server-side billing complexity.
4. **Firestore as blob store**: Vectorstores and raw PDFs are serialized and stored directly in Firestore documents/subcollections (chunked to respect 1MB limit).
5. **Local embeddings**: Uses free HuggingFace model instead of paid embedding APIs.