# SangamAI — Detailed Application Flows

This document traces every flow end-to-end, including function calls, data transformations, API requests/responses, Firestore paths, and UI state changes. Nothing is omitted.

---

## Table of Contents

1. [Server Startup](#1-server-startup)
2. [Client Initialization & Auth Guard](#2-client-initialization--auth-guard)
3. [Registration Flow](#3-registration-flow)
4. [Login Flow](#4-login-flow)
5. [Logout Flow](#5-logout-flow)
6. [PDF Upload Flow](#6-pdf-upload-flow)
7. [YouTube Upload Flow](#7-youtube-upload-flow)
8. [CSV Upload Flow](#8-csv-upload-flow)
9. [RAG Chat Flow (PDF/YouTube)](#9-rag-chat-flow-pdfyoutube)
10. [CSV Chat Flow (Pandas Agent)](#10-csv-chat-flow-pandas-agent)
11. [Chat History Management](#11-chat-history-management)
12. [File Management (List, Delete, View PDF)](#12-file-management-list-delete-view-pdf)
13. [Profile Management](#13-profile-management)
14. [Streamlit Legacy Flow](#14-streamlit-legacy-flow)

---

## 1. Server Startup

**Entry point:** `uvicorn main:app --reload ...`

**File:** `server/main.py`

```
Line-by-line execution:
────────────────────────────────────────────────────────
1  import os, json, contextlib.asynccontextmanager
2  from fastapi import FastAPI
3  from fastapi.middleware.cors import CORSMiddleware
4  import firebase_admin
5  from firebase_admin import credentials
6  from dotenv import load_dotenv

11 load_dotenv()
   → Reads server/.env file into os.environ
   → Variables available: FIREBASE_SERVICE_ACCOUNT, CORS_ORIGINS, FIREBASE_API_KEY

14 def _init_firebase():
     ┌─────────────────────────────────────────────────────┐
     │ Purpose: Initialize Firebase Admin SDK exactly once  │
     │ Idempotent: calling get_app() raises ValueError if   │
     │ not initialized, which triggers init.                │
     └─────────────────────────────────────────────────────┘
16   try:
17     firebase_admin.get_app()
       → If already initialized → return silently (no-op)
18   except ValueError:
       → No existing app → must initialize
19     cred_path = os.getenv("FIREBASE_SERVICE_ACCOUNT", "serviceAccount.json")
       → Reads path from env, default "serviceAccount.json"
20     if os.path.exists(cred_path):
       → Check if file exists on disk
21       cred = credentials.Certificate(cred_path)
         → Parses JSON, creates Certificate credential object
22     else:
       → File not found → fall back to env variable
24       raw = os.getenv("FIREBASE_CREDENTIALS", "{}")
         → Read JSON string from env, default "{}" if missing
25       cred_data = json.loads(raw)
         → Parse JSON string to dict
26       cred_data["private_key"] = cred_data["private_key"].replace("\\n", "\n")
         → Fix: JSON escaping converts real newlines to literal \n
         → Replace escaped \n with actual newline characters
         → ⚠ THIS IS WHERE THE ORIGINAL ERROR OCCURRED:
           If FIREBASE_CREDENTIALS is "{}", cred_data = {}
           cred_data["private_key"] → KeyError because dict is empty
27       cred = credentials.Certificate(cred_data)
28     firebase_admin.initialize_app(cred)
       → Initialize default Firebase app with credentials
       → After this, firebase_admin.get_app() will succeed

31 @asynccontextmanager
32 async def lifespan(app: FastAPI):
   → FastAPI lifespan context manager: startup → yield → shutdown
34   _init_firebase()
     → Call our init function (see above)
   (NOTE: in the original traceback this was line 34)
35   from modules.rag import get_embeddings
36   get_embeddings()
     → @lru_cache(maxsize=1) decorated
     → Loads HuggingFace all-MiniLM-L6-v2 model into memory
     → First call downloads model (~100MB), subsequent calls return cached
     → This is the "pre-warm" step - avoids cold-start penalty on first chat
37   yield
     → App is now ready to serve requests
     → (shutdown code after yield would run on shutdown)

41 app = FastAPI(title="SangamAI API", version="1.0.0", lifespan=lifespan)
   → Create FastAPI app with our lifespan

44-50 CORS Middleware
   allow_origins = os.getenv("CORS_ORIGINS", "http://localhost:3000").split(",")
   → Default allows Next.js dev server on port 3000
   → allow_credentials=True → cookies/auth headers allowed
   → allow_methods=["*"], allow_headers=["*"] → permissive

53-63 Router registration
   from routes.auth import router as auth_router
   from routes.upload import router as upload_router
   from routes.chat import router as chat_router
   from routes.files import router as files_router
   from routes.profile import router as profile_router
   app.include_router(...) → Mount at prefix paths

68-77  MODEL_OPTIONS list
80-87  Health and models endpoints (no auth required)
────────────────────────────────────────────────────────
```

**Server environment checklist at startup:**
- `server/.env` must exist with valid `FIREBASE_SERVICE_ACCOUNT` path
- `serviceAccount.json` (or custom path) must exist
- Firebase project must be active and credentials valid
- Firestore must be enabled in Firebase Console
- HuggingFace model download requires internet on first run

---

## 2. Client Initialization & Auth Guard

**Entry:** User navigates to any route in Next.js app

**File:** `client/app/layout.tsx` (root layout wrapping all pages)

```
Flow:
────────────────────────────────────────────────────────
1. Root layout renders <AuthProvider> from auth-context.tsx

2. AuthProvider mounts:
   ┌──────────────────────────────────────────────────────┐
   │ useEffect on mount:                                   │
   │   unsub = onAuthStateChanged(auth, callback)          │
   │   → Firebase JS SDK listens for auth state changes    │
   │   → On initial load, checks persisted session         │
   │     (IndexedDB / localStorage via Firebase SDK)       │
   │   → callback(user) fires with User | null             │
   │   → setUser(u), setLoading(false)                    │
   │   → Returns unsubscribe function                      │
   └──────────────────────────────────────────────────────┘

3. Individual pages use useAuth() hook:
   const { user, loading } = useAuth()

4. Auth guard pattern (used in login/page.tsx and chat/page.tsx):
   useEffect(() => {
     if (!loading && !user) router.replace("/login")
     if (!loading && user) router.replace("/chat")
   }, [user, loading, router])
   → If loading: show nothing (spinner/null return)
   → If no user: redirect to /login
   → If user exists: allow access to /chat or /profile

5. Loading state handling:
   if (authLoading || !user) return null
   → Prevents flash of logged-in content before auth state resolves
```

---

## 3. Registration Flow

**Entry:** User clicks "Create Account" tab on login page

**File:** `client/app/login/page.tsx` (lines 46-59)

### Step-by-step:

```
CLIENT SIDE:
────────────────────────────────────────────────────────
1. User fills: email, password, display_name (optional)

2. handleRegister(e) fires on form submit:
   setRegLoading(true)

3. await register(regEmail, regPassword, regUsername)
   ┌─────────────────────────────────────────────────────┐
   │ client/lib/api.ts  line 22-28:                      │
   │   POST ${API}/api/auth/register                     │
   │   headers: {"Content-Type": "application/json"}     │
   │   body: JSON.stringify({                           │
   │     email: regEmail,                               │
   │     password: regPassword,                         │
   │     username: regUsername                          │
   │   })                                                │
   └─────────────────────────────────────────────────────┘

   NOTE: No Authorization header - registration is unauthenticated

4. On success:
   toast.success("Account created! Sign in to continue.")
   setTab("signin")        → Switch UI to Sign In tab
   setEmail(regEmail)      → Pre-fill email in sign-in form

5. On error:
   toast.error(error message)
   → "Email already in use" if email taken
   → Generic error for other failures

SERVER SIDE:
────────────────────────────────────────────────────────
POST /api/auth/register hits routes/auth.py

1. FastAPI parses JSON body → RegisterRequest model
   email: str, password: str, username: str = ""

2. @router.post("/register"):
   try:
     user = auth.create_user(email=req.email, password=req.password)
       → Firebase Admin SDK creates user in Firebase Auth
       → Returns UserRecord with uid, email, etc.
   
     db = firestore.client()
     db.collection("users").document(user.uid).set({
       "email": req.email,
       "api_key": "",           # Empty OpenRouter key
       "username": req.username # May be empty string ""
     })
       → Creates Firestore document at: users/{uid}
       → This is the user profile document
   
     return {"uid": user.uid, "message": "Account created successfully"}
   
   except auth.EmailAlreadyExistsError:
     raise HTTPException(409, "Email already in use")
   
   except Exception as e:
     raise HTTPException(400, str(e))

3. Response flows back to client as JSON:
   { "uid": "abc123...", "message": "Account created successfully" }
```

**Firestore state after registration:**
```
users/
└── {uid}/
    ├── email: "user@example.com"
    ├── api_key: ""
    └── username: "optional_name"
```

---

## 4. Login Flow

**Entry:** User clicks "Sign In" button

**File:** `client/app/login/page.tsx` (lines 32-44)

### Step-by-step:

```
CLIENT SIDE:
────────────────────────────────────────────────────────
1. User fills: email, password

2. handleSignIn(e) fires on form submit:
   setLoading(true)

3. await signInWithEmailAndPassword(auth, email, password)
   ┌─────────────────────────────────────────────────────┐
   │ client/lib/firebase.ts:                             │
   │   Imports signInWithEmailAndPassword from           │
   │   firebase/auth                                      │
   │   → Calls Firebase Auth REST API internally         │
   │   → POST to identitytoolkit.googleapis.com          │
   │   → Sends email, password, returnSecureToken=true  │
   │   → On success: returns UserCredential             │
   │     containing: user object with uid, email, etc.  │
   │     AND idToken (JWT, expires in 1 hour)           │
   └─────────────────────────────────────────────────────┘

4. On success:
   toast.success("Welcome back!")
   router.push("/chat")
   → Navigates to /chat page

5. Auth state change fires:
   onAuthStateChanged callback receives User object
   → useAuth() now returns { user: User, loading: false }
   → /chat page auth guard passes, renders chat UI

6. All subsequent API calls include token:
   headers() function in api.ts:
     const token = await auth.currentUser?.getIdToken()
     h["Authorization"] = `Bearer ${token}`
   → Token is auto-included in every API request

SERVER SIDE (when /chat page loads):
────────────────────────────────────────────────────────
1. api.getProfile(), api.getApiKey(), api.listFiles() called
   → Each has Authorization: Bearer {firebase_id_token}

2. middleware.py: get_current_user(request):
   header = request.headers.get("Authorization", "")
   → Extracts "Bearer abc123..."
   token = header.split("Bearer ", 1)[1]
   
   if not token: raise 401
   
   try:
     decoded = auth.verify_id_token(token)
       → Firebase Admin SDK verifies JWT signature
       → Checks token not expired
       → Checks token issued for this Firebase project
       → Returns decoded payload: { uid, email, aud, ... }
     return decoded["uid"]
       → uid is injected as user_id parameter in route

   except Exception as e:
     raise HTTPException(401, f"Invalid token: {e}")

3. Route handler receives user_id:
   @router.get("/")
   async def get_profile(user_id: str = Depends(get_current_user)):
     → user_id = "abc123..." from verified token
     → Uses it to query Firestore
```

**Key insight — dual auth systems:**
- Registration: Server creates user via Admin SDK (`auth.create_user`)
- Login: Client authenticates via Firebase JS SDK REST API
- Server validates client's token via Admin SDK (`auth.verify_id_token`)
- Neither system uses passwords directly on the server for login

---

## 5. Logout Flow

**Triggered in two places:**
- Streamlit sidebar: `views/chat.py` line 118-120
- Next.js sidebar: `client/app/chat/page.tsx` line 234-237

### Step-by-step:

```
CLIENT SIDE:
────────────────────────────────────────────────────────
1. User clicks "Logout" button

2. await signOut(auth)
   → Firebase JS SDK signs out
   → Clears persisted session (IndexedDB/localStorage)
   → Fires onAuthStateChanged(auth, null)

3. Auth context updates:
   setUser(null)
   → useAuth() now returns { user: null, loading: false }

4. Auth guard triggers:
   useEffect sees user=null
   → router.replace("/login")
   → Redirects to login page

5. Server-side session:
   → Stateless: no server session to clear
   → Future requests without valid token → 401
```

---

## 6. PDF Upload Flow

**Entry:** User selects file in upload modal → clicks "Process & Save"

**Client file:** `client/app/chat/page.tsx` `handlePdf()` (line 772-782)

### Client Side:

```
1. User selects PDF file via <input type="file" accept=".pdf">

2. handlePdf(file) called:
   setUploading(true)
   
3. await api.uploadPdf(file)
   ┌─────────────────────────────────────────────────────────────┐
   │ client/lib/api.ts line 88-98:                               │
   │   const fd = new FormData()                                  │
   │   fd.append("file", file)  ← actual File object             │
   │   const token = await auth.currentUser?.getIdToken()        │
   │   POST ${API}/api/upload/pdf                                │
   │   headers: { Authorization: `Bearer ${token}` }             │
   │     → Note: NO Content-Type header (browser sets           │
   │       multipart/form-data with boundary automatically)      │
   │   body: fd (FormData - binary file data)                    │
   └─────────────────────────────────────────────────────────────┘

4. On success:
   onDone({ file_name: res.file_name, content_type: "pdf", created_at: "" })
   → Adds file to sidebar list
   → Auto-selects the new file
   → Closes upload modal
   → Shows success toast

SERVER SIDE:
────────────────────────────────────────────────────────
POST /api/upload/pdf → routes/upload.py

1. FastAPI dependency: user_id = Depends(get_current_user)
   → Token extracted, verified, uid returned

2. @router.post("/pdf"):
   file: UploadFile = File(...)  ← multipart form data parsed
   user_id: str

3. tempfile.NamedTemporaryFile(delete=False, suffix=".pdf"):
   → Creates temp file path: C:\Users\...\AppData\Local\Temp\tmpXXXXX.pdf
   
4. content = await file.read()
   → Reads entire uploaded file into memory as bytes
   
5. tmp.write(content)
   → Writes bytes to temp file on disk

6. splits = load_and_split_pdf(tmp_path)
   ┌─────────────────────────────────────────────────────────────┐
   │ modules/rag.py line 41-55:                                  │
   │   loader = PyPDFLoader(file_path)                           │
   │     → PyPDFLoader from langchain_community                 │
   │     → Opens PDF, extracts text per page                     │
   │     → Returns list of Document objects:                    │
   │       [Document(page_content="text...", metadata={         │
   │         "source": tmp_path,                                 │
   │         "page": 0                                           │
   │       }), ...]                                              │
   │                                                             │
   │   splitter = _get_text_splitter(1000, 200)                 │
   │     → RecursiveCharacterTextSplitter:                      │
   │       chunk_size=1000 chars, overlap=200 chars             │
   │       separators=["\n\n", "\n", ". ", " ", ""]            │
   │       → Tries to split at paragraph boundaries first       │
   │       → Falls back to sentence, word, char                 │
   │                                                             │
   │   return splitter.split_documents(docs)                    │
   │     → Returns list of ~1000-char Document chunks          │
   └─────────────────────────────────────────────────────────────┘

7. vectorstore = create_vectorstore(splits)
   ┌─────────────────────────────────────────────────────────────┐
   │ modules/rag.py line 130-132:                                │
   │   return FAISS.from_documents(splits, get_embeddings())    │
   │                                                             │
   │   get_embeddings() (cached):                                │
   │     → HuggingFaceEmbeddings(model_name="all-MiniLM-L6-v2") │
   │     → First call: downloads ~100MB model from HF Hub       │
   │     → Returns embedding pipeline object                     │
   │                                                             │
   │   FAISS.from_documents:                                    │
   │     → Each Document's page_content → embedding vector      │
   │     → Builds FAISS index (IVF flat, default)               │
   │     → In-memory vector store                                │
   └─────────────────────────────────────────────────────────────┘

8. save_vectorstore_to_firestore(user_id, file.filename, vectorstore, "pdf")
   ┌─────────────────────────────────────────────────────────────┐
   │ modules/database.py line 15-61:                             │
   │                                                             │
   │   pkl = vectorstore.serialize_to_bytes()                   │
   │     → Serializes entire FAISS index to bytes               │
   │     → Could be several MB for large documents              │
   │                                                             │
   │   CHUNK_SIZE = 700 * 1024 = 716800 bytes (~700KB)         │
   │   num_chunks = ceil(len(pkl) / CHUNK_SIZE)                │
   │                                                             │
   │   db = _get_db() → firestore.client()                      │
   │                                                             │
   │   # Save metadata                                           │
   │   doc_ref = db.collection("users")                         │
   │     .document(user_id)                                      │
   │     .collection("files")                                    │
   │     .document(file.filename)                                │
   │   doc_ref.set({                                             │
   │     "file_name": file.filename,                             │
   │     "content_type": "pdf",                                  │
   │     "total_chunks": num_chunks,                             │
   │     "total_size": len(pkl),                                 │
   │     "created_at": firestore.SERVER_TIMESTAMP               │
   │   })                                                        │
   │                                                             │
   │   # Save chunks                                             │
   │   for i in range(num_chunks):                              │
   │     start = i * CHUNK_SIZE                                 │
   │     end = start + CHUNK_SIZE                               │
   │     chunk_data = pkl[start:end]                            │
   │     doc_ref.collection("chunks")                           │
   │       .document(str(i))                                    │
   │       .set({                                               │
   │         "data": chunk_data,  ← binary in Firestore        │
   │         "chunk_id": i                                      │
   │       })                                                    │
   │                                                             │
   │   Firestore path created:                                  │
   │   users/{user_id}/files/{file_name}/                       │
   │     ├── metadata document                                  │
   │     └── chunks/                                             │
   │         ├── 0/ (data: bytes, chunk_id: 0)                 │
   │         ├── 1/ (data: bytes, chunk_id: 1)                 │
   │         └── ...                                              │
   └─────────────────────────────────────────────────────────────┘

9. save_pdf_bytes(user_id, file.filename, content)
   ┌─────────────────────────────────────────────────────────────┐
   │ modules/database.py line 100-110:                           │
   │   Same chunking logic (700KB chunks)                        │
   │   doc_ref.update({                                          │
   │     "pdf_size": len(raw_bytes),                             │
   │     "pdf_chunks": num_chunks                                │
   │   })                                                        │
   │   for each chunk:                                           │
   │     doc_ref.collection("pdf_raw")                          │
   │       .document(str(i))                                    │
   │       .set({"data": chunk_data, "chunk_id": i})            │
   │                                                             │
   │   Firestore path:                                           │
   │   users/{user_id}/files/{file_name}/                       │
   │     └── pdf_raw/                                            │
   │         ├── 0/ (data: raw PDF bytes)                       │
   │         └── ...                                              │
   └─────────────────────────────────────────────────────────────┘

10. os.unlink(tmp_path)
    → Deletes the temporary file from disk

11. return {"message": "...", "file_name": file.filename}
    → Sent back to client
```

**Firestore state after PDF upload:**
```
users/{user_id}/files/{file_name}/
├── (doc) file_name: "mydoc.pdf"
├── (doc) content_type: "pdf"
├── (doc) total_chunks: 5
├── (doc) total_size: 3456789
├── (doc) created_at: <timestamp>
├── (doc) pdf_size: 2345678
├── (doc) pdf_chunks: 4
├── chunks/
│   ├── 0/ (data: <bytes>, chunk_id: 0)
│   ├── 1/ (data: <bytes>, chunk_id: 1)
│   └── ...
└── pdf_raw/
    ├── 0/ (data: <bytes>, chunk_id: 0)
    └── ...
```

---

## 7. YouTube Upload Flow

**Entry:** User pastes URL, clicks "Process & Save"

**Client file:** `client/app/chat/page.tsx` `handleYoutube()` (line 784-795)

### Client Side:

```
1. User pastes YouTube URL in input field

2. handleYoutube():
   setUploading(true)
   
3. await api.uploadYoutube(ytUrl.trim())
   ┌─────────────────────────────────────────────────────────────┐
   │ client/lib/api.ts line 100-107:                              │
   │   POST ${API}/api/upload/youtube                            │
   │   headers: { "Content-Type": "application/json" }           │
   │   body: JSON.stringify({ url: ytUrl })                      │
   └─────────────────────────────────────────────────────────────┘

4. On success: same as PDF (onDone callback)

SERVER SIDE:
────────────────────────────────────────────────────────
POST /api/upload/youtube → routes/upload.py

1. user_id = Depends(get_current_user)

2. class YouTubeRequest(BaseModel): url: str

3. @router.post("/youtube"):
   video_id = extract_video_id(req.url)
   ┌─────────────────────────────────────────────────────────────┐
   │ modules/rag.py line 60-75:                                  │
   │   regex: r'(?:youtube\.com/watch\?v=|                    │
   │             youtu\.be/|                                     │
   │             youtube\.com/embed/)([a-zA-Z0-9_-]{11})'      │
   │   → Matches standard YouTube URL formats                   │
   │   → Returns 11-character video ID                          │
   │   → Examples:                                              │
   │     https://www.youtube.com/watch?v=dQw4w9WgXcQ           │
   │       → "dQw4w9WgXcQ"                                      │
   │     https://youtu.be/dQw4w9WgXcQ                          │
   │       → "dQw4w9WgXcQ"                                      │
   └─────────────────────────────────────────────────────────────┘
   
   if not video_id:
     raise HTTPException(400, "Invalid YouTube URL")

4. vectorstore, transcript = process_youtube(req.url)
   ┌─────────────────────────────────────────────────────────────┐
   │ modules/rag.py line 155-169:                                │
   │   transcript_text, metadata = load_youtube_transcript(url) │
   │                                                             │
   │   load_youtube_transcript:                                  │
   │     video_id = extract_video_id(url)                       │
   │     ytt_api = YouTubeTranscriptApi()                       │
   │     transcript = ytt_api.fetch(video_id)                   │
   │       → Calls youtube-transcript-api library               │
   │       → Returns list of Transcript snippet objects:        │
   │         [{"text": "Hello world", "start": 0.0,            │
   │          "duration": 2.5}, ...]                            │
   │     transcript_text = " ".join(snippet.text for ...)       │
   │       → Concatenates all snippet texts with spaces         │
   │                                                             │
   │   splits = chunk_text_documents(transcript_text,           │
   │     metadata={"video_id": video_id, "source": url},       │
   │     chunk_size=1000, chunk_overlap=200)                    │
   │     → Creates Document(page_content=transcript,           │
   │                        metadata={"video_id": "abc",        │
   │                                  "source": "https://..."}) │
   │     → Splits with same RecursiveCharacterTextSplitter     │
   │                                                             │
   │   vectorstore = create_vectorstore(splits)                 │
   │     → Same FAISS embedding process as PDF                  │
   │                                                             │
   │   return vectorstore, transcript_text                      │
   └─────────────────────────────────────────────────────────────┘

5. file_name = f"youtube_{video_id}"
   → e.g., "youtube_dQw4w9WgXcQ"

6. save_vectorstore_to_firestore(user_id, file_name, vectorstore, "youtube")
   → Same chunk-and-save process as PDF

7. return {"message": f"Video {video_id} transcript indexed", "file_name": file_name}
```

**Key difference from PDF:**
- No raw bytes saved (no pdf_raw subcollection)
- Content type is "youtube"
- file_name has "youtube_" prefix
- Source metadata includes original URL

---

## 8. CSV Upload Flow

**Entry:** User selects CSV file, clicks "Process & Save"

**Client file:** `client/app/chat/page.tsx` `handleCsv()` (line 797-807)

### Client Side:

```
1. User selects CSV via <input type="file" accept=".csv">

2. handleCsv(file):
   setUploading(true)
   
3. await api.uploadCsv(file)
   ┌─────────────────────────────────────────────────────────────┐
   │ client/lib/api.ts line 109-119:                              │
   │   const fd = new FormData()                                  │
   │   fd.append("file", file)                                    │
   │   POST ${API}/api/upload/csv                                │
   │   headers: { Authorization: `Bearer ${token}` }             │
   │   body: fd                                                  │
   └─────────────────────────────────────────────────────────────┘

4. On success: receives shape and columns in response
   → onDone updates file list

SERVER SIDE:
────────────────────────────────────────────────────────
POST /api/upload/csv → routes/upload.py

1. user_id = Depends(get_current_user)

2. raw = await file.read()
   → Reads entire CSV into memory as bytes

3. df = pd.read_csv(io.BytesIO(raw))
   → Pandas parses CSV from bytes buffer
   → Returns DataFrame
   → Infers dtypes, headers, handles delimiters

4. save_dataframe_to_firestore(user_id, file.filename, df)
   ┌─────────────────────────────────────────────────────────────┐
   │ modules/database.py line 195-221:                            │
   │   df_bytes = pickle.dumps(dataframe)                        │
   │     → Serializes entire DataFrame to bytes                  │
   │     → Pickle format (Python-specific)                       │
   │                                                             │
   │   db.collection("users")                                    │
   │     .document(user_id)                                      │
   │     .collection("files")                                    │
   │     .document(file_name)                                    │
   │     .set({                                                  │
   │       "file_name": file_name,                               │
   │       "content_type": "csv",                                │
   │       "size": len(df_bytes),                                │
   │       "shape": dataframe.shape,     ← (rows, cols)         │
   │       "columns": list(df.columns),  ← column names         │
   │       "dataframe": df_bytes,        ← pickled DF           │
   │       "created_at": firestore.SERVER_TIMESTAMP              │
   │     })                                                      │
   │                                                             │
   │   Note: If DataFrame > 1MB, this will FAIL because         │
   │   Firestore document limit is 1MB. For large CSVs,        │
   │   chunking would be needed (not currently implemented).    │
   └─────────────────────────────────────────────────────────────┘

5. return {
   "message": f"{file.filename} uploaded",
   "file_name": file.filename,
   "shape": list(df.shape),      # e.g., [1500, 25]
   "columns": list(df.columns),  # e.g., ["date", "sales", "region"]
   "preview": df.head(10).to_dict(orient="records")
     # First 10 rows as array of dicts for frontend table
}
```

**Firestore state after CSV upload:**
```
users/{user_id}/files/{file_name}/
└── (doc) file_name: "sales.csv"
    ├── content_type: "csv"
    ├── size: 45678
    ├── shape: [1500, 25]
    ├── columns: ["date", "sales", "region", ...]
    ├── dataframe: <pickled bytes>
    └── created_at: <timestamp>
```

---

## 9. RAG Chat Flow (PDF/YouTube)

**Entry:** User selects a file, types a question, hits Enter

**Client file:** `client/app/chat/page.tsx` `handleSend()` (line 165-187)

### Client Side:

```
1. Pre-conditions checked:
   if (!input.trim() || !selectedFile || !apiKey || sending) return
   → Question must be non-empty
   → A file must be selected
   → User must have an OpenRouter API key
   → Not already sending

2. UI updates:
   setInput("")                    → Clear input field
   setMessages(prev => [...prev, { role: "user", content: question }])
                                  → Add user message to chat UI immediately
   setSending(true)                → Show loading/thinking state
   setLastSources([])              → Clear previous source chunks
   setShowSources(false)           → Collapse sources panel

3. API call:
   const response = await api.sendMessage(selectedFile, question, apiKey, selectedModel)
   ┌─────────────────────────────────────────────────────────────┐
   │ client/lib/api.ts line 146-159:                              │
   │   POST ${API}/api/chat/message                              │
   │   headers: {                                                │
   │     "Content-Type": "application/json",                     │
   │     "Authorization": `Bearer ${firebase_id_token}`          │
   │   }                                                         │
   │   body: JSON.stringify({                                    │
   │     file_name: "mydoc.pdf",         ← selected file        │
   │     question: "What is this about?", ← user question       │
   │     api_key: "sk-or-v1-...",        ← OpenRouter key       │
   │     model: "google/gemini-2.5-flash" ← selected model      │
   │   })                                                        │
   └─────────────────────────────────────────────────────────────┘

4. On response:
   setMessages(prev => [...prev, { role: "assistant", content: response.answer }])
   → Add AI answer to chat UI
   
   if (response.sources?.length > 0):
     setLastSources(response.sources)
     → Store source chunks for expandable "Source chunks" panel

5. On error:
   const msg = err instanceof Error ? err.message : "Failed to get response"
   toast.error(msg)
   setMessages(prev => prev.slice(0, -1))
   → Remove the optimistic user message (revert)

6. Finally:
   setSending(false)
   → Hide thinking animation

SERVER SIDE:
────────────────────────────────────────────────────────
POST /api/chat/message → routes/chat.py

1. user_id = Depends(get_current_user)
   → Verify Firebase ID token → get uid

2. FastAPI parses body → ChatRequest model:
   file_name: str
   question: str
   api_key: str
   model: str = "google/gemini-2.5-flash"

3. Look up file metadata:
   db = firestore.client()
   file_ref = db.collection("users").document(user_id)
                  .collection("files").document(req.file_name)
   file_doc = file_ref.get()
   
   if not file_doc.exists:
     raise HTTPException(404, "File not found")
   
   content_type = file_doc.to_dict().get("content_type", "pdf")
   → Could be "pdf", "youtube", or "csv"

4. llm = get_llm(req.api_key, req.model)
   ┌─────────────────────────────────────────────────────────────┐
   │ modules/llm.py:                                             │
   │   return ChatOpenAI(                                        │
   │     base_url="https://openrouter.ai/api/v1",               │
   │     api_key=api_key,                                        │
   │     model=model_name,                                       │
   │     temperature=0,                                          │
   │     max_tokens=1024,                                       │
   │   )                                                         │
   │   → Creates LangChain LLM wrapper                          │
   │   → All future calls go through OpenRouter                  │
   └─────────────────────────────────────────────────────────────┘

5. Branch by content type:

   ┌─ CSV MODE ──────────────────────────────────────────────┐
   if content_type == "csv":
     df = load_dataframe_from_firestore(user_id, req.file_name)
       → pickle.loads(data["dataframe"])
       → Reconstructs original Pandas DataFrame in memory
     
     if df is None:
       raise HTTPException(404, "DataFrame not found")
     
     agent = create_pandas_agent_chain(llm, df, verbose=False)
       ┌─────────────────────────────────────────────────────┐
       │ modules/agents.py line 12-46:                       │
       │   create_pandas_dataframe_agent(                    │
       │     llm=llm,                                        │
       │     df=dataframe,                                   │
       │     verbose=False,                                  │
       │     allow_dangerous_code=True,  ← executes Python  │
       │     handle_parsing_errors=True                      │
       │   )                                                 │
       │   → Creates ReAct-style agent                        │
       │   → Agent has tools: python_repl_ast,               │
       │     pandas DataFrame access                          │
       │   → Agent will generate Python code, run it,        │
       │     observe output, iterate                          │
       └─────────────────────────────────────────────────────┘
     
     answer = ask_dataframe_question(agent, req.question)
       ┌─────────────────────────────────────────────────────┐
       │ modules/agents.py line 49-66:                       │
       │   result = agent.invoke({"input": question})        │
       │   return result.get("output", str(result))          │
       │                                                     │
       │ Behind the scenes:                                  │
       │   1. Agent receives question                         │
       │   2. LLM decides to write Python code               │
       │   3. Code executed via Python REPL                  │
       │   4. Result fed back to LLM                         │
       │   5. Loop until answer ready                        │
       │   6. Return final output string                     │
       └─────────────────────────────────────────────────────┘
     
     source_chunks = []    ← Empty for CSV

   ┌─ RAG MODE ─────────────────────────────────────────────┐
   else (pdf or youtube):
     embeddings = get_embeddings()
       → Returns cached HuggingFace embeddings model
     
     vectorstore = _get_vectorstore_cached(user_id, file_name, embeddings)
       ┌─────────────────────────────────────────────────────┐
       │ routes/chat.py line 31-40:                          │
       │   key = f"{user_id}:{file_name}"                    │
       │   if key in _vs_cache:                              │
       │     vs, ts = _vs_cache[key]                         │
       │     if time.time() - ts < 600:  # 600s TTL         │
       │       return vs                                     │
       │   vs = load_vectorstore_from_firestore(...)         │
       │   _vs_cache[key] = (vs, time.time())               │
       │   return vs                                         │
       │                                                     │
       │ load_vectorstore_from_firestore:                    │
       │   meta = doc_ref.get()                              │
       │   num_chunks = meta["total_chunks"]                 │
       │   full_pkl = b""                                   │
       │   for i in range(num_chunks):                      │
       │     chunk = doc_ref.collection("chunks")           │
       │                  .document(str(i)).get()           │
       │     full_pkl += chunk["data"]                      │
       │   return FAISS.deserialize_from_bytes(             │
       │     embeddings, full_pkl,                           │
       │     allow_dangerous_deserialization=True           │
       │   )                                                 │
       └─────────────────────────────────────────────────────┘
     
     if not vectorstore:
       raise HTTPException(404, "Vectorstore not found")
     
     retriever = get_retriever(vectorstore)
       → vectorstore.as_retriever(search_type="similarity", search_kwargs={"k": 3})
       → Returns top-3 most similar chunks per query
     
     history = load_chat_history(user_id, req.file_name)
       ┌─────────────────────────────────────────────────────┐
       │ modules/database.py line 155-172:                   │
       │   msgs_ref = db.collection("users")                 │
       │     .document(user_id)                              │
       │     .collection("files")                            │
       │     .document(file_name)                            │
       │     .collection("messages")                         │
       │     .order_by("timestamp")                          │
       │   return [                                          │
       │     {"role": doc["role"], "content": doc["content"]}│
       │     for doc in msgs_ref.stream()                    │
       │   ]                                                  │
       │   → Returns list sorted oldest-first               │
       │   → e.g., [                                         │
       │     {"role": "user", "content": "What is X?"},     │
       │     {"role": "assistant", "content": "X is..."},   │
       │     {"role": "user", "content": "Tell me more"}    │
       │   ]                                                  │
       └─────────────────────────────────────────────────────┘
     
     memory = build_memory_from_history(history)
       ┌─────────────────────────────────────────────────────┐
       │ modules/memory.py line 12-49:                       │
       │   messages = []                                      │
       │   i = 0                                              │
       │   while i < len(chat_messages) - 1:                │
       │     if msg[i]["role"] == "user" and                │
       │        msg[i+1]["role"] == "assistant":            │
       │       messages.append(HumanMessage(content=...))   │
       │       messages.append(AIMessage(content=...))      │
       │       i += 2                                         │
       │     else:                                            │
       │       i += 1                                         │
       │   # Keep last k pairs                                │
       │   if len(messages) > 2*k:                           │
       │     messages = messages[-(2*k):]                    │
       │   return messages                                    │
       │                                                     │
       │   → Pairs user+assistant messages                   │
       │   → Converts to LangChain message objects           │
       │   → Keeps last 8 pairs (16 messages) by default    │
       └─────────────────────────────────────────────────────┘
     
     chain = build_conversational_chain(llm, retriever, memory)
       ┌─────────────────────────────────────────────────────┐
       │ modules/chains.py line 34-89:                       │
       │   ConversationalRAGChain(llm, retriever, history)   │
       │                                                     │
       │   __init__:                                         │
       │     self.condense_prompt = get_condense_question_prompt() │
       │     self.condense_chain = (                         │
       │       condense_prompt | llm | StrOutputParser()     │
       │     )                                               │
       │     # LCEL pipe: prompt → LLM → string parser      │
       │                                                     │
       │     self.qa_prompt = get_qa_prompt()               │
       │     self.qa_chain = (                              │
       │       qa_prompt | llm | StrOutputParser()          │
       │     )                                               │
       └─────────────────────────────────────────────────────┘
     
     result = ask_question(chain, req.question)
       ┌─────────────────────────────────────────────────────────┐
       │ modules/chains.py line 52-76:                           │
       │                                                         │
       │ STEP 1 - Condense question (if history exists):        │
       │   if self.chat_history:                                 │
       │     standalone_question = self.condense_chain.invoke({ │
       │       "chat_history": _format_chat_history(history),   │
       │       "question": question                             │
       │     })                                                 │
       │   else:                                                 │
       │     standalone_question = question                      │
       │                                                         │
       │   _format_chat_history converts:                       │
       │     [HumanMessage("Hi"), AIMessage("Hello!")]          │
       │     → "Human: Hi\nAssistant: Hello!"                   │
       │                                                         │
       │   Condense prompt sends to LLM:                        │
       │     "Given chat history, rewrite question as           │
       │      standalone that captures full context"             │
       │   → If first message: returns original                 │
       │   → If follow-up: "it" → "the transformer architecture"│
       │                                                         │
       │ STEP 2 - Retrieve:                                      │
       │   docs = self.retriever.invoke(standalone_question)    │
       │     → FAISS similarity search on embedded chunks       │
       │     → Returns top-3 Document objects                   │
       │     → Each has page_content + metadata (page, source)  │
       │                                                         │
       │ STEP 3 - Generate answer:                               │
       │   answer = self.qa_chain.invoke({                      │
       │     "context": _format_docs(docs),  ← joined chunks   │
       │     "question": standalone_question                    │
       │   })                                                   │
       │     → QA prompt:                                       │
       │       "You are SangamAI, a helpful AI assistant.      │
       │        Use the following pieces of retrieved          │
       │        context to answer the user's question.          │
       │        Context: {context}                              │
       │        Question: {question}"                           │
       │     → LLM generates answer using retrieved context     │
       │     → StrOutputParser extracts string from response    │
       │                                                         │
       │   return {"answer": answer, "source_documents": docs}  │
       └─────────────────────────────────────────────────────────┘

6. Source chunk extraction:
   source_chunks = []
   for doc in result["source_documents"]:
     chunk_text = doc.page_content[:200].strip()
     page = doc.metadata.get("page", None)
     source = doc.metadata.get("source", None)
     source_chunks.append({ text, page, source })
   → Truncated to 200 chars for UI display
   → Includes page number (PDF) or video URL (YouTube)

7. Persist both messages:
   save_chat_message(user_id, req.file_name, "user", req.question)
   save_chat_message(user_id, req.file_name, "assistant", answer)
   ┌─────────────────────────────────────────────────────────────┐
   │ modules/database.py line 138-152:                           │
   │   msgs_ref = db.collection("users")                        │
   │     .document(user_id)                                     │
   │     .collection("files")                                   │
   │     .document(file_name)                                   │
   │     .collection("messages")                                │
   │   msgs_ref.add({                                           │
   │     "role": role,        ← "user" or "assistant"          │
   │     "content": content,  ← message text                   │
   │     "timestamp": firestore.SERVER_TIMESTAMP               │
   │   })                                                       │
   │   → Auto-generates document ID                            │
   │   → Messages accumulate over time                          │
   └─────────────────────────────────────────────────────────────┘

8. Return response:
   {
     "answer": "Based on the document...",
     "sources": [
       {"text": "The transformer architecture...", "page": 3, "source": null},
       ...
     ]
   }
```

**Thinking animation on client:**

While waiting for response, client shows animated terminal-style thinking indicator:

```
thinkingSteps = [
  { label: "Parsing query",         tag: "PARSE"  },
  { label: "Vectorizing input",     tag: "EMBED"  },
  { label: "Searching document chunks", tag: "SEARCH" },
  { label: "Ranking relevant passages",  tag: "RANK"   },
  { label: "Generating response",   tag: "GEN"    },
]

→ Cycles through phases every 1.8 seconds
→ Current phase pulses with "_" animation
→ Completed phases show green "+" + "done"
→ Progress bar advances proportionally
→ This is purely cosmetic (doesn't reflect actual progress)
```

---

## 10. CSV Chat Flow (Pandas Agent)

**Same entry point:** `handleSend()` → `api.sendMessage()`

**Server-side branch difference:**

```
In routes/chat.py, the "csv" branch:

1. df = load_dataframe_from_firestore(user_id, req.file_name)
   → pickle.loads(data["dataframe"])
   → Reconstructs DataFrame

2. agent = create_pandas_agent_chain(llm, df, verbose=False)
   ┌─────────────────────────────────────────────────────────────┐
   │ modules/agents.py:                                          │
   │   create_pandas_dataframe_agent(                            │
   │     llm=llm,                                                │
   │     df=dataframe,                                           │
   │     verbose=False,                                          │
   │     allow_dangerous_code=True,                              │
   │     handle_parsing_errors=True                              │
   │   )                                                         │
   │                                                             │
   │   → Agent uses ReAct pattern:                               │
   │     Thought → Action → Observation → Thought ...           │
   │                                                             │
   │   → Tools available to agent:                               │
   │     - python_repl_ast: Execute Python code                 │
   │     - df (pandas DataFrame) injected as context            │
   │                                                             │
   │   → LLM writes code like:                                   │
   │     df.groupby('region')['sales'].mean()                   │
   │   → Code executed, result returned to LLM                  │
   │   → LLM formats natural language answer                    │
   └─────────────────────────────────────────────────────────────┘

3. answer = ask_dataframe_agent(agent, req.question)
   → agent.invoke({"input": question})
   → Returns {"output": "The average sales by region are..."}

4. source_chunks = []  ← No sources for CSV

5. Persist messages (same as RAG)

6. Return { answer, sources: [] }
```

**Client display:**
- Shows DataFrame preview in expander
- Chat bubbles same as RAG but no source chunks panel
- AI response may include formatted tables, numbers, etc.

---

## 11. Chat History Management

### Load History

**Trigger:** When user selects a file (useEffect in chat/page.tsx)

```
CLIENT:
────────────────────────────────────────────────────────
1. selectedFile changes → useEffect fires (line 126-149)

2. api.getChatHistory(selectedFile)
   GET /api/chat/{file_name}/history

3. setMessages(data.messages)
   → Populates chat UI with full history

SERVER:
────────────────────────────────────────────────────────
@router.get("/{file_name}/history")
async def get_history(file_name: str, user_id: str = Depends(get_current_user)):
  messages = load_chat_history(user_id, file_name)
  return {"messages": messages}

modules/database.py line 155-172:
  msgs_ref = db.collection("users").document(user_id)
                .collection("files").document(file_name)
                .collection("messages")
                .order_by("timestamp")
  return [{"role": doc["role"], "content": doc["content"]} for doc in msgs_ref.stream()]
  → Streams all documents in order
  → Oldest message first (chronological)
```

### Clear History

**Trigger:** User clicks "Clear History" button

```
CLIENT:
────────────────────────────────────────────────────────
1. handleClear():
   await api.clearChatHistory(selectedFile)
   → DELETE /api/chat/{file_name}/history
2. setMessages([])
   → Clears chat UI
3. toast.success("History cleared")

SERVER:
────────────────────────────────────────────────────────
@router.delete("/{file_name}/history")
async def delete_history(file_name: str, user_id: str = Depends(get_current_user)):
  clear_chat_history(user_id, file_name)

modules/database.py line 175-187:
  msgs_ref = db.collection(...).collection("messages")
  for doc in msgs_ref.stream():
    doc.reference.delete()
  → Firestore has no collection-level delete
  → Must iterate and delete each document individually
  → If 1000 messages, 1000 separate delete operations
```

---

## 12. File Management (List, Delete, View PDF)

### List Files

**Trigger:** Chat page loads (useEffect on user)

```
CLIENT:
────────────────────────────────────────────────────────
1. api.listFiles()
   GET /api/files
   → Authorization: Bearer {token}

2. setFiles(data.files)
   → Populates sidebar file list

SERVER:
────────────────────────────────────────────────────────
@router.get("/")
async def list_files(user_id: str = Depends(get_current_user)):
  files_ref = db.collection("users").document(user_id).collection("files")
  files = []
  for doc in files_ref.stream():
    data = doc.to_dict()
    files.append({
      "file_name": doc.id,
      "content_type": data.get("content_type", "pdf"),
      "created_at": str(data.get("created_at", "")),
    })
  return {"files": files}
  → Returns all documents in user's files collection
```

### Delete File

**Trigger:** User clicks trash icon on file in sidebar

```
CLIENT:
────────────────────────────────────────────────────────
1. handleDeleteFile(fileName):
   await api.deleteFile(fileName)
   → DELETE /api/files/{file_name}

2. setFiles(prev => prev.filter(f => f.file_name !== fileName))
   → Remove from sidebar

3. If deleted file was selected:
   setSelectedFile(null)
   setMessages([])
   setLastSources([])
   → Clear chat area

SERVER:
────────────────────────────────────────────────────────
@router.delete("/{file_name}")
async def delete_file(file_name: str, user_id: str = Depends(get_current_user)):
  db = firestore.client()
  doc_ref = db.collection("users").document(user_id)
                .collection("files").document(file_name)
  
  doc = doc_ref.get()
  if not doc.exists:
    raise HTTPException(404, "File not found")
  
  # Delete sub-collections
  for sub in ("chunks", "messages", "pdf_raw"):
    for child in doc_ref.collection(sub).stream():
      child.reference.delete()
  → Delete all chunks, messages, raw PDF data
  
  doc_ref.delete()
  → Delete the parent document
  
  _invalidate_cache(user_id, file_name)
  → Remove from in-memory vectorstore cache
  → Prevents stale data if file is re-uploaded

  return {"message": f"{file_name} deleted"}
```

### View PDF

**Trigger:** PDF toggle button clicked (if PDF loaded)

```
CLIENT:
────────────────────────────────────────────────────────
1. If file is PDF:
   api.getPdfUrl(selectedFile)
   → Returns: `${API}/api/files/${fileName}/pdf?token=${token}`

2. setPdfUrl(url)
   → Set as iframe src

3. iframe renders:
   <iframe src={pdfUrl} className="w-full h-full" />
   → Browser fetches PDF directly from server

4. PDF toggle button controls showPdf state
   → showPdf=false: PDF panel hidden, full-width chat
   → showPdf=true: Split view (50% PDF | 50% chat)

SERVER:
────────────────────────────────────────────────────────
@router.get("/{file_name}/pdf")
async def get_pdf(file_name: str, user_id: str = Depends(get_current_user)):
  pdf_bytes = load_pdf_bytes(user_id, file_name)
  
  if not pdf_bytes:
    raise HTTPException(404, "PDF not found")
  
  return Response(
    content=pdf_bytes,
    media_type="application/pdf",
    headers={"Content-Disposition": f'inline; filename="{file_name}"'}
  )
  → Sends raw PDF bytes
  → inline disposition shows in browser
  → Browser's built-in PDF viewer handles rendering

load_pdf_bytes (modules/database.py line 113-129):
  meta = doc_ref.get()
  num_chunks = meta.get("pdf_chunks", 0)
  full = b""
  for i in range(num_chunks):
    chunk_doc = doc_ref.collection("pdf_raw").document(str(i)).get()
    if chunk_doc.exists:
      full += chunk_doc.to_dict()["data"]
  return full
  → Reassembles raw PDF from chunks
```

---

## 13. Profile Management

**Entry:** User navigates to /profile

### Get Profile

```
CLIENT:
────────────────────────────────────────────────────────
1. api.getProfile()
   GET /api/profile

2. setUsername(profile.username)
   setUserEmail(profile.email)
   → Display in profile page and sidebar

SERVER:
────────────────────────────────────────────────────────
@router.get("/")
async def get_profile(user_id: str = Depends(get_current_user)):
  doc = db.collection("users").document(user_id).get()
  data = doc.to_dict()
  return {
    "uid": user_id,
    "email": data.get("email", ""),
    "username": data.get("username", ""),
    "has_api_key": bool(data.get("api_key")),
    "api_key_hint": f"...{data['api_key'][-8:]}" if data.get("api_key") else None,
  }
  → Never returns full API key (only last 8 chars hint)
```

### Update Username

```
CLIENT:
────────────────────────────────────────────────────────
1. User types new display name, clicks "Save Name"

2. api.updateUsername(newUsername)
   PUT /api/profile/username
   body: { username: newUsername }

SERVER:
────────────────────────────────────────────────────────
class UpdateUsername(BaseModel): username: str

@router.put("/username")
async def update_username(body: UpdateUsername, user_id: str = Depends(get_current_user)):
  db.collection("users").document(user_id).update({"username": body.username.strip()})
  return {"message": "Username updated"}
```

### API Key Management

```
CLIENT (profile page):
────────────────────────────────────────────────────────
1. Input: OpenRouter API key (password-masked)

2. api.updateApiKey(newKey)
   PUT /api/profile/api-key
   body: { api_key: newKey }

SERVER:
────────────────────────────────────────────────────────
@router.put("/api-key")
async def update_api_key(body: UpdateApiKey, user_id: str = Depends(get_current_user)):
  db.collection("users").document(user_id).update({"api_key": body.api_key})
  return {"message": "API key updated"}

Get API key (when loading chat page):
@router.get("/api-key")
async def get_api_key(user_id: str = Depends(get_current_user)):
  doc = db.collection("users").document(user_id).get()
  return {"api_key": doc.to_dict().get("api_key", "")}
  → Returns FULL key (stored encrypted at rest by Firebase)
  → Frontend stores in state, sends with every chat message
```

---

## 14. Streamlit Legacy Flow

**Entry:** `streamlit run server/app.py`

**File:** `server/app.py` (note: imports `streamlit` not FastAPI)

### Startup:

```
1. Firebase init (same as FastAPI but from st.secrets):
   try: firebase_admin.get_app()
   except ValueError:
     cred_data = dict(st.secrets["firebase"])
     cred_data["private_key"] = cred_data["private_key"].replace('\\n', '\n')
     cred = credentials.Certificate(cred_data)
     firebase_admin.initialize_app(cred)
   → Reads from Streamlit secrets (.streamlit/secrets.toml)
   → NOT from .env file

2. Session state initialization:
   st.session_state.user_id = None        # Auth state
   st.session_state.vectors = None        # Loaded vectorstore
   st.session_state.dataframe = None      # Loaded DataFrame
   st.session_state.active_file = None    # Currently selected file
   st.session_state.page = "chat"         # Current page

3. Routing:
   if st.session_state.user_id:
     # Show loader on first login
     if "_app_ready" not in st.session_state:
       st.set_page_config(...)
       inject_theme()
       st.markdown(loader_html)
       st.session_state._app_ready = True
       import time; time.sleep(8)       ← 8 second artificial delay
       st.rerun()
     
     if st.session_state.page == "profile":
       show_profile_page()
     else:
       show_chat_page()
   else:
     st.session_state.pop("_app_ready", None)  # Reset flag
     show_login_page()
```

### Streamlit Chat Page:

**File:** `server/views/chat.py`

```
1. _render_sidebar():
   → Direct Firebase reads (no FastAPI)
   → DB: get username, API key
   → Model selector, API key input
   → Profile and logout buttons

2. _render_upload_tab():
   → PDF: Streamlit file_uploader → save locally → load_and_split_pdf → create_vectorstore → save_vectorstore_to_firestore
   → YouTube: URL input → process_youtube → save_vectorstore_to_firestore
   → CSV: file_uploader → pd.read_csv → save_dataframe_to_firestore
   → All processing happens IN THE BROWSER (Streamlit reruns script on each interaction)

3. _render_chat_tab(api_key, model_name):
   File selection → load from Firestore → chat interface
   
   For CSV mode:
   → Uses pandas agents directly (no separate /api/chat endpoint)
   → UI: st.chat_message() for bubbles
   → Direct: agent.invoke({"input": query})
   
   For RAG mode:
   → get_embeddings(), create vectorstore from cache
   → Uses memory.get_memory() (IN-MEMORY per session, not Firestore)
   → NOTE: Different from FastAPI which uses build_memory_from_history()
```

**Key difference from FastAPI flow:**
- Streamlit is stateful within a single session
- FastAPI is stateless, reconstructs everything from Firestore each request
- Streamlit chat history lives in `modules/memory.py` (in-memory buffer)
- FastAPI chat history lives in Firestore `messages/` subcollection

---

## Appendix: Complete Data Flow Diagrams

### A. End-to-End: PDF Upload → Chat

```
User Browser                    FastAPI Server                    Firestore         FAISS/Embeddings
    │                               │                               │                   │
    │──POST /api/upload/pdf────────>│                               │                   │
    │   (multipart: file.pdf)       │                               │                   │
    │                               │──1. PyPDFLoader──────────────>│                   │
    │                               │   (extract text per page)     │                   │
    │                               │<──2. [Document, ...]──────────│                   │
    │                               │                               │                   │
    │                               │──3. TextSplitter─────────────>│                   │
    │                               │   (1000 chars / 200 overlap) │                   │
    │                               │<──4. [Document chunks]───────│                   │
    │                               │                               │                   │
    │                               │──5. HuggingFaceEmbedding────>│                   │
    │                               │   (all-MiniLM-L6-v2)        │                   │
    │                               │<──6. Vector embeddings───────│                   │
    │                               │                               │                   │
    │                               │──7. FAISS.from_documents()   │                   │
    │                               │   (build in-memory index)    │                   │
    │                               │<──8. FAISS vectorstore───────│                   │
    │                               │                               │                   │
    │                               │──9. serialize_to_bytes()     │                   │
    │                               │   (FAISS binary format)      │                   │
    │                               │                               │                   │
    │                               │──10. Chunk into 700KB parts  │                   │
    │                               │                               │                   │
    │                               │──11. Save to Firestore──────>│                   │
    │                               │   users/{uid}/files/{name}/  │                   │
    │                               │   ├── metadata doc           │                   │
    │                               │   └── chunks/{i}: data       │                   │
    │                               │                               │                   │
    │                               │──12. Save raw PDF bytes─────>│                   │
    │                               │   users/{uid}/files/{name}/  │                   │
    │                               │   └── pdf_raw/{i}: data      │                   │
    │                               │                               │                   │
    │<─13. {"file_name": "x.pdf"}───│                               │                   │
    │                               │                               │                   │
    │                               │                               │                   │
User types question                │                               │                   │
    │                               │                               │                   │
    │──POST /api/chat/message──────>│                               │                   │
    │   {file_name, question,       │                               │                   │
    │    api_key, model}            │                               │                   │
    │                               │                               │                   │
    │                               │──14. Load vectorstore chunks──>│                  │
    │                               │    (download all from         │                   │
    │                               │     chunks/ subcollection)    │                   │
    │                               │<──15. FAISS binary────────────│                   │
    │                               │                               │                   │
    │                               │──16. deserialize_from_bytes()│                   │
    │                               │   with embeddings model       │                   │
    │                               │<──17. In-memory vectorstore──│                   │
    │                               │                               │                   │
    │                               │──18. similarity_search(q)────>│                  │
    │                               │    (via FAISS → top-3)       │                   │
    │                               │<──19. [Doc, Doc, Doc]────────│                   │
    │                               │                               │                   │
    │                               │──20. LLM API call            │                   │
    │                               │   (OpenRouter)                │                   │
    │                               │   Prompt: context + question  │                   │
    │                               │<──21. "Based on the doc..."──│                   │
    │                               │                               │                   │
    │                               │──22. Save to Firestore───────>│                   │
    │                               │   messages/{auto_id}:        │                   │
    │                               │   {role, content, timestamp} │                   │
    │                               │                               │                   │
    │<─23. {answer, sources}────────│                               │                   │
    │                               │                               │                   │
```

### B. End-to-End: CSV Upload → Chat

```
User Browser                    FastAPI Server                    Firestore         Pandas DF
    │                               │                               │                   │
    │──POST /api/upload/csv────────>│                               │                   │
    │   (multipart: data.csv)       │                               │                   │
    │                               │──1. pandas.read_csv()         │                   │
    │                               │   (parse CSV from bytes)      │                   │
    │                               │<──2. DataFrame───────────────│                   │
    │                               │                               │                   │
    │                               │──3. pickle.dumps(df)          │                   │
    │                               │                               │                   │
    │                               │──4. Save to Firestore───────>│                   │
    │                               │   users/{uid}/files/{name}/  │                   │
    │                               │   └── doc: {content_type:csv,│                   │
    │                               │        shape, columns,        │                   │
    │                               │        dataframe: <bytes>}   │                   │
    │                               │                               │                   │
    │<─5. {file_name, shape,        │                               │                   │
    │     columns, preview}─────────│                               │                   │
    │                               │                               │                   │
    │                               │                               │                   │
User: "Average sales by region?"   │                               │                   │
    │                               │                               │                   │
    │──POST /api/chat/message──────>│                               │                   │
    │   {file_name, question,       │                               │                   │
    │    api_key, model}            │                               │                   │
    │                               │                               │                   │
    │                               │──6. pickle.loads(data)        │                   │
    │                               │   ← Reconstruct DataFrame     │                   │
    │                               │                               │                   │
    │                               │──7. create_pandas_agent_chain │                   │
    │                               │   LLM + DataFrame agent       │                   │
    │                               │                               │                   │
    │                               │──8. agent.invoke({input: q})  │                   │
    │                               │   ReAct loop:                 │                   │
    │                               │   - LLM writes Python code    │                   │
    │                               │   - Execute python_repl_ast   │                   │
    │                               │   - df.groupby('region')[...] │                   │
    │                               │   - Result → LLM → answer    │                   │
    │                               │<──9. "The average sales..."───│                   │
    │                               │                               │                   │
    │                               │──10. Save messages to FS────>│                   │
    │                               │                               │                   │
    │<─11. {answer, sources: []}────│                               │                   │
    │                               │                               │                   │
```

---

## Summary of Key Patterns

| Pattern | Where | Why |
|---------|-------|-----|
| Bearer token auth | All FastAPI routes | Stateless, works with Firebase ID tokens |
| Firestore as blob store | Vectorstores, PDFs, DataFrames | Avoids separate storage service (S3/GCS) |
| 700KB chunking | Binary blobs in Firestore | Firestore document limit is 1MB |
| In-memory vectorstore cache | routes/chat.py `_vs_cache` | Avoid repeated Firestore deserialization |
| LRU cache on embeddings | modules/rag.py `get_embeddings()` | Load model once per process |
| Client-provided LLM key | Stored in Firestore per user | Avoid server billing, user controls costs |
| LCEL chains | modules/chains.py | Modern LangChain, composable, testable |
| Firestore chat history | messages/ subcollection | Persistent across server restarts |
| Stateless server | No sessions | Easy horizontal scaling |
| Dual UI (FastAPI + Streamlit) | Both in server/ | FastAPI for production, Streamlit for dev |