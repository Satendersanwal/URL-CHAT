<p align="center">
  <img src="docs/images/hero.png" alt="URL-CHAT — Turn webpages into conversations" width="100%" />
</p>

<p align="center">
  <strong>Paste a link. Ask a question. Inspect the source.</strong><br />
  A full-stack retrieval-augmented generation app for exploring webpage content through conversation.
</p>

<p align="center">
  <img alt="React 18" src="https://img.shields.io/badge/React-18-61DAFB?style=flat-square&amp;logo=react&amp;logoColor=white" />
  <img alt="Express" src="https://img.shields.io/badge/Express-API-222222?style=flat-square&amp;logo=express" />
  <img alt="Google Gemini" src="https://img.shields.io/badge/Google-Gemini-8E75B2?style=flat-square&amp;logo=googlegemini&amp;logoColor=white" />
  <img alt="LangChain.js" src="https://img.shields.io/badge/LangChain-JavaScript-1C3C3C?style=flat-square" />
  <img alt="MongoDB Atlas" src="https://img.shields.io/badge/MongoDB-Atlas-47A248?style=flat-square&amp;logo=mongodb&amp;logoColor=white" />
</p>

<p align="center">
  <a href="#preview">Preview</a> ·
  <a href="#quick-start">Quick start</a> ·
  <a href="#architecture">Architecture</a> ·
  <a href="#api-reference">API reference</a> ·
  <a href="#deployment">Deployment</a>
</p>

---

## Why URL-CHAT?

A long webpage can contain the answer you need without making it easy to find. URL-CHAT turns that page into a searchable conversation: it extracts readable text, retrieves relevant passages, and asks Gemini to answer using those passages.

The result is a focused reading companion for articles, documentation, project pages, and other publicly accessible HTML content.

> **Designed for grounded answers.** The model is instructed to use retrieved page content and acknowledge missing information. Source excerpts make answers easier to check; they are not a guarantee that every generated statement is correct.

## Preview

![URL-CHAT desktop interface with a source library, AI-generated answer, and source controls](docs/images/app-preview.png)

*URL-CHAT interface showing a conversation about Satender Sanwal's public GitHub profile.*

### A simple workflow

| Step | You do | URL-CHAT does |
| --- | --- | --- |
| **01 · Add** | Paste a webpage URL. | Extracts readable content and splits it into overlapping passages. |
| **02 · Ask** | Ask a specific question about the page. | Retrieves relevant passages and generates a contextual answer. |
| **03 · Verify** | Expand the sources beneath the answer. | Shows the text chunks supplied to the model. |
| **04 · Continue** | Ask a follow-up or return to a saved source. | Uses recent conversation context and loads stored history when available. |

**Example questions**

- “What are the main ideas in this article?”
- “Which technologies does this project use?”
- “Explain the installation steps from this page.”
- “What does it say about deployment?” — after discussing the same project.

## Features

| Feature | What is implemented |
| --- | --- |
| **Webpage ingestion** | Axios fetches HTML; Cheerio removes common navigation, script, style, and form elements. |
| **Semantic retrieval** | Gemini embeddings with MongoDB Atlas Vector Search, or a process-local memory vector store. |
| **Context-aware follow-ups** | Up to three previous user/assistant exchanges help rewrite a follow-up into a standalone retrieval query. |
| **Source visibility** | Expandable source excerpts accompany answers that have retrieved context. |
| **Persistent conversations** | MongoDB stores source records and per-URL chat history when connected. |
| **Fallback retrieval** | Keyword overlap over cached chunks is used when vector retrieval fails or returns no matching passages. |
| **Responsive interface** | Dark theme, Markdown answers, resizable desktop sidebar, and a mobile source drawer. |
| **Source management** | Select indexed pages and remove a page together with its vectors and chat history. |
| **API foundations** | CORS configuration, Helmet headers, compression, request logging, rate limiting, and a health route. |

## Tech stack

| Layer | Technologies | Responsibility |
| --- | --- | --- |
| Frontend | React 18, Vite 6, Tailwind CSS 3 | Source library, chat interface, loading states, responsive layout |
| UI utilities | Lucide React, react-markdown, remark-gfm | Icons and rich answer rendering |
| API client | Axios | Calls the Express endpoints through one shared client |
| Backend | Node.js, Express 4, ES modules | Validation, routing, orchestration, and HTTP lifecycle |
| Retrieval | LangChain.js | Chunking, embeddings, vector stores, prompts, and model calls |
| AI | Google Gemini | Embeddings, follow-up rewriting, and answer generation |
| Storage | MongoDB, Mongoose, Atlas Vector Search | Source metadata, messages, and persistent vectors |
| Extraction | Axios, Cheerio | Fetching and cleaning HTML |

## Architecture

```mermaid
flowchart TD
    UI["React interface"] --> API["Express API"]
    API --> RAG["RAG service"]
    RAG --> WEB["Public webpage"]
    RAG --> GEMINI["Gemini API"]
    RAG --> STORE{"Storage mode"}
    STORE --> ATLAS["MongoDB Atlas"]
    STORE --> MEMORY["Process memory"]
    ATLAS --> RECORDS["Documents and chat history"]
    ATLAS --> VECTORS["Chunks and vector index"]
```

The backend separates HTTP concerns from retrieval logic:

| Component | Role |
| --- | --- |
| [Routes](server/src/routes/ragRoutes.js) | Maps endpoint paths to controller functions. |
| [Controller](server/src/controllers/ragController.js) | Validates requests and formats responses. |
| [RAG service](server/src/services/ragService.js) | Scrapes, chunks, indexes, retrieves, rewrites questions, generates answers, and manages storage. |
| [Database connection](server/src/config/db.js) | Connects to MongoDB and supports a memory-only fallback. |
| [API client](client/src/services/api.js) | Centralizes frontend requests and the API base URL. |

### What happens when you ask a question?

```mermaid
sequenceDiagram
    participant U as User
    participant A as API
    participant R as RAG service
    participant S as Storage
    participant G as Gemini
    U->>A: Question and selected URL
    A->>R: Validate and query
    R->>S: Load recent conversation
    opt Prior conversation exists
        R->>G: Rewrite question for retrieval
        G-->>R: Standalone search question
    end
    R->>S: Retrieve relevant passages
    alt Vector retrieval has no usable result
        R->>R: Rank cached chunks by keyword overlap
    end
    R->>G: Context, history, and original question
    G-->>R: Answer
    R->>S: Save question, answer, and sources
    R-->>A: Answer and source excerpts
    A-->>U: Render conversation
```

*If no context can be retrieved, the service returns a “not found” response without making the answer-generation call.*

## Inside the RAG pipeline

### 1. Ingestion

1. **Fetch:** request the page with a 15-second timeout.
2. **Clean:** remove common non-content elements; prefer text from `main`, then `article`, then `body`. Reject extracted text shorter than 50 characters.
3. **Chunk:** split text with a target size of **1,000 characters** and **200 characters of overlap**.
4. **Attach metadata:** record the URL, page title, chunk index, and total chunk count.
5. **Embed and index:** replace previously indexed vectors for that URL and add the new chunks to the active vector store.
6. **Save the source:** upsert MongoDB metadata, or store a process-local record.

Chunks are also cached in memory for keyword fallback. If embedding or indexing fails, ingestion can still save a source record and retain this fallback cache; a successful ingestion response alone does not prove that Atlas vectors were written.

### 2. Follow-up understanding

A question such as “Who wrote it?” lacks enough context for useful retrieval. When chat history exists, the service asks Gemini to turn it into a standalone question.

- Uses the last **three exchanges**.
- Limits each message in the rewriting transcript to **800 characters**.
- Uses **temperature 0** for rewriting.
- Falls back to the original question if rewriting fails or produces an unsuitable result.
- Preserves the user's original question in the displayed and stored conversation.

### 3. Retrieval and generation

- Retrieve up to **15** vector matches across the store.
- Filter those matches to the selected URL, then keep up to **4** passages.
- If no usable vector result remains, rank cached chunks by keyword overlap.
- Combine the passages into labeled context blocks.
- Generate an answer with **temperature 0.2**, recent conversation messages, and a prompt that restricts factual claims to the retrieved context.
- Return the answer and source excerpts, then persist both conversation roles.

These are implementation settings, not benchmark results or guarantees of retrieval accuracy.

## Quick start

### Prerequisites

- **Node.js 22.12+** and npm. The server manifest declares Node 18+, but the locked root development runner requires Node 22+; use the newer runtime for the complete project.
- Git.
- A Google API key with access to the configured Gemini chat and embedding models.
- Optional: a MongoDB Atlas database with Vector Search for persistence.

### 1. Clone and install

```bash
git clone https://github.com/Satendersanwal/URL-CHAT.git
cd URL-CHAT

npm ci
npm ci --prefix server
npm ci --prefix client
```

The first command installs root tooling, including `concurrently`. The other two install the server and client dependencies from their lockfiles.

### 2. Configure the server

Create `server/.env`:

```dotenv
GOOGLE_API_KEY=replace_with_your_google_api_key
PORT=5000
NODE_ENV=development
CLIENT_URL=http://localhost:5173

# Optional: enables persistent documents, messages, and vectors.
# MONGO_URI=mongodb+srv://USER:PASSWORD@CLUSTER/urlchat

# Optional model overrides; verify access in your Google project.
# GOOGLE_CHAT_MODEL=models/your_available_chat_model
# GOOGLE_EMBEDDING_MODEL=models/your_available_embedding_model
```

The current code defaults to `models/gemini-3.6-flash` for chat and `models/gemini-embedding-001` for embeddings. These are repository defaults, not a promise of model availability for every account. Check the [official Gemini model catalog](https://ai.google.dev/gemini-api/docs/models) and override them when needed.

Keep `GOOGLE_API_KEY` on the server. Do not put it in a `VITE_` variable or commit an actual `.env` file.

### 3. Start development

```bash
npm run dev
```

| Service | Default address |
| --- | --- |
| Frontend | http://localhost:5173 |
| Backend | http://localhost:5000 |
| Health check | http://localhost:5000/health |

Vite forwards `/api` requests to port `5000`, so no client environment file is needed for the default local setup. If you change the backend port, also update [the Vite proxy target](client/vite.config.js).

### 4. Try a conversation

Open the frontend, add a publicly accessible article with readable text, wait for ingestion, and ask a question about that page. Expand the source excerpts to compare the answer with the retrieved text.

Without MongoDB, sources, messages, and cached vectors are lost when the backend process stops. Gemini access is still required.

## Environment variables

| Variable | Location | Required? | Purpose / default |
| --- | --- | --- | --- |
| `GOOGLE_API_KEY` | `server/.env` | Yes | Gemini API access; startup exits if missing. |
| `GOOGLE_CHAT_MODEL` | Server | No | Chat and question-rewrite model; defaults to `models/gemini-3.6-flash`. |
| `GOOGLE_EMBEDDING_MODEL` | Server | No | Embedding model; defaults to `models/gemini-embedding-001`. |
| `MONGO_URI` | Server | For persistence | MongoDB connection URI; otherwise use memory-only storage. |
| `PORT` | Server | No | Backend port; defaults to `5000`. |
| `NODE_ENV` | Server | No | Defaults to `development`; production enables static client hosting when the build exists. |
| `CLIENT_URL` | Server | For split hosting | Allowed frontend origins, comma-separated; defaults to `http://localhost:5173`. |
| `VITE_API_URL` | `client/.env` or frontend build environment | For split hosting | Full API base URL **including `/api`**; defaults to `/api`. |

## MongoDB Atlas setup

For persistent vector retrieval:

1. Configure a database user and network access for your backend.
2. Put a connection URI with the intended database name in `server/.env`.
3. Run the included index setup script from the server directory:

```bash
cd server
node scripts/setupVectorIndex.js
cd ..
```

The script creates `chunks` if needed, then creates `vector_index` with this definition:

```json
{
  "fields": [
    {
      "type": "vector",
      "path": "embedding",
      "numDimensions": 3072,
      "similarity": "cosine"
    }
  ]
}
```

Wait until the index is queryable before relying on semantic search. The script skips an index that already exists; it does not migrate its definition.

If you change the embedding model or output dimensions, update the index definition to match and re-index your sources. See [MongoDB's Vector Search index documentation](https://www.mongodb.com/docs/atlas/atlas-vector-search/vector-search-type/).

### Storage behavior

| Data | MongoDB connected | Memory-only mode |
| --- | --- | --- |
| Indexed source metadata | `documents` collection | Process-local map |
| Conversation messages | `chatmessages` collection | Process-local array |
| Successfully indexed vectors | `chunks` collection | Memory vector store |
| Keyword fallback chunks | Process-local cache | Process-local cache |
| Survives restart? | Persisted records and vectors do | No |

The keyword fallback cache is **not rehydrated from MongoDB** at startup. If vector retrieval is unavailable after a restart, re-ingesting a source rebuilds its fallback chunks.

## API reference

All `/api` routes share a configured limit of **30 requests per minute per client IP**.

| Method | Endpoint | Input | Main response |
| --- | --- | --- | --- |
| `POST` | `/api/ingest` | `{ "url": "https://..." }` | Source ID, title, URL, chunk count, creation time |
| `POST` | `/api/chat` | `{ "url": "https://...", "question": "..." }` | `answer` and `sources` |
| `GET` | `/api/documents` | None | `{ success, documents }` |
| `GET` | `/api/chat/history` | `?url=<encoded-url>` | `{ success, history }` |
| `DELETE` | `/api/documents/:id` | Source ID in path | Deletion status; removes source, vectors, and chat history |
| `GET` | `/health` | None | Process status, environment, timestamp, service name |

### Example requests

The examples below use Bash-style quoting. On Windows PowerShell, use `Invoke-RestMethod` or adapt quoting for `curl.exe`.

```bash
curl -X POST http://localhost:5000/api/ingest \
  -H "Content-Type: application/json" \
  -d '{"url":"https://example.com"}'

curl -X POST http://localhost:5000/api/chat \
  -H "Content-Type: application/json" \
  -d '{"url":"https://example.com","question":"What does this page explain?"}'
```

Use the normalized URL returned by ingestion for later chat and history requests.

<details>
<summary><strong>Illustrative chat response shape</strong></summary>

```json
{
  "answer": "An answer generated from the retrieved page passages.",
  "sources": [
    {
      "pageContent": "A retrieved excerpt from the selected webpage.",
      "metadata": {
        "url": "https://example.com",
        "title": "Example page",
        "chunkIndex": 0,
        "totalChunks": 3
      }
    }
  ]
}
```

This illustrates the response schema; it is not a recorded model response.

</details>

## Project guide

| Path | What to explore |
| --- | --- |
| `package.json` | Root install, development, build, and production scripts |
| `client/src/App.jsx` | Active source, messages, ingestion state, and sidebar resizing |
| `client/src/components/` | Header, source list, ingestion form, chat, status stepper, source accordions |
| `client/src/services/api.js` | Shared Axios instance and endpoint helpers |
| `client/vite.config.js` | Frontend dev server and backend proxy |
| `server/src/index.js` | Middleware, CORS, startup, static hosting, health route, and shutdown |
| `server/src/config/db.js` | MongoDB connection and DNS retry behavior |
| `server/src/routes/ragRoutes.js` | REST endpoint definitions |
| `server/src/controllers/ragController.js` | Request validation and response formatting |
| `server/src/services/ragService.js` | Retrieval and generation pipeline |
| `server/src/models/` | Source and chat-message schemas |
| `server/scripts/setupVectorIndex.js` | Atlas vector index setup |
| `docs/images/` | README banner and actual app screenshot |

## Scripts

Run these from the repository root unless noted.

| Command | Purpose |
| --- | --- |
| `npm run install:all` | Installs server and client dependencies; root dependencies must be installed separately. |
| `npm run dev` | Starts the backend watcher and Vite concurrently. |
| `npm run build` | Builds the frontend into `client/dist`. |
| `npm start` | Builds the frontend, then starts Express in production mode. |
| `npm run start:server-only` | Starts the production backend without rebuilding the frontend. |
| `npm run preview --prefix client` | Previews an existing Vite build; configure a reachable API separately. |

## Deployment

### Option A: one Node service

The Express server can serve `client/dist` and the API from the same origin.

1. Install root, server, and client dependencies with the quick-start commands.
2. Configure server environment variables in the host's environment settings.
3. Build with `npm run build`.
4. Start with `npm run start:server-only`.
5. Check `/health` and open the frontend at the service origin.

Keep `VITE_API_URL` unset so frontend requests use `/api` on the same origin. Keep `cross-env` installed: the production startup script uses it even though it is listed under server development dependencies.

### Option B: separate frontend and backend

| Setting | Frontend static host | Backend Node host |
| --- | --- | --- |
| Project root | `client` | `server` |
| Install | `npm ci` | `npm ci --include=dev` |
| Build | `npm run build` | None required |
| Output | `dist` | Running Node service |
| Start | Host serves static assets | `npm run start:prod` |
| Environment | `VITE_API_URL=https://YOUR-BACKEND/api` | Google key, optional MongoDB URI, model overrides, `CLIENT_URL=https://YOUR-FRONTEND` |

Frontend environment variables are embedded at build time. Rebuild after changing `VITE_API_URL`. Use your own frontend and backend deployment URLs.

## Troubleshooting

| Symptom | What to check |
| --- | --- |
| `concurrently` is missing | Install root dependencies with `npm ci`, not only client/server dependencies. |
| Engine-version warning | Use Node 22.12+ for the root development tooling. |
| Startup exits immediately | Check that `GOOGLE_API_KEY` is set in the backend's environment. |
| Model not found or permission denied | Verify the configured model names and API-key access; use the model override variables. |
| Frontend calls return 404 | For split hosting, `VITE_API_URL` must end in `/api`. |
| Browser reports a CORS error | Set `CLIENT_URL` to the exact frontend origin and restart the backend. |
| Ingestion works but semantic search does not | Inspect backend logs, embedding access, `vector_index` readiness, and dimensions. |
| Sources disappear after restarting | The backend may be running in memory-only mode; check MongoDB connection logs. |
| Page has insufficient readable text | Try a public HTML page with meaningful server-rendered text. |
| API returns 429 | The local rate limit or upstream model quota may have been reached. |
| Port 5000 is occupied | Stop the other process or change `PORT` and the Vite proxy target together. |

## Current boundaries

- **HTML pages, not a browser crawler:** the scraper does not execute page JavaScript, authenticate to sites, crawl linked pages, or parse uploaded PDFs.
- **Per-URL conversations, not private accounts:** authentication and user-level ownership are not implemented. Source and history endpoints share the backend's store.
- **Retrieval trade-off:** URL filtering happens after the top 15 vector results are fetched; this can miss relevant chunks for one page in a large mixed collection.
- **Fallback trade-off:** keyword ranking can return the first chunks even when every overlap score is zero.
- **No token streaming:** the UI waits for the complete API response.
- **Illustrative progress:** ingestion steps advance on frontend timers rather than live backend progress events.
- **Operational hardening remains:** unrestricted public hosting should add authentication, URL-fetch restrictions against private network targets, and stronger input validation.
- **Health is process-level:** `/health` does not verify successful Gemini requests or Atlas index readiness.

## Possible next steps

These are future ideas, not shipped features:

- [ ] URL filtering inside vector retrieval, plus a relevance evaluation set.
- [ ] Authentication and per-user source ownership.
- [ ] Server-sent events for streamed answers and actual ingestion progress.
- [ ] Durable fallback retrieval after restarts.
- [ ] PDF ingestion and optional JavaScript-rendered page extraction.
- [ ] Automated API and retrieval regression tests.

---

<p align="center">
  <strong>Read less. Ask better questions. Check the source.</strong>
</p>
