# Chat with Any Document

A Retrieval-Augmented Generation (RAG) application that allows a user to
upload a PDF, ask questions about its contents, and receive answers with
page-number references to the source document.

> **Project status:** Planning / early development. This README
> describes the intended MVP and proposed implementation. It does not
> claim that the endpoints, database, frontend, or deployment have
> already been completed.

## Table of contents

-   [1. Project overview](#1-project-overview)
-   [2. Problem statement](#2-problem-statement)
-   [3. MVP features](#3-mvp-features)
-   [4. Technology stack](#4-technology-stack)
-   [5. System architecture](#5-system-architecture)
-   [6. Main application flows](#6-main-application-flows)
-   [7. API design](#7-api-design)
-   [8. Data model](#8-data-model)
-   [9. Repository structure](#9-repository-structure)
-   [10. Local setup](#10-local-setup)
-   [11. Environment configuration](#11-environment-configuration)
-   [12. Development roadmap](#12-development-roadmap)
-   [13. Team work division](#13-team-work-division)
-   [14. Git and collaboration
    workflow](#14-git-and-collaboration-workflow)
-   [15. Testing strategy](#15-testing-strategy)
-   [16. Security and reliability
    considerations](#16-security-and-reliability-considerations)
-   [17. Current decisions and open
    questions](#17-current-decisions-and-open-questions)
-   [18. Definition of done](#18-definition-of-done)

------------------------------------------------------------------------

## 1. Project overview

**Chat with Any Document** is a document-question-answering application.
A user uploads a supported PDF and asks questions in natural language.
The application retrieves relevant passages from the uploaded document
and uses an AI language model to formulate an answer grounded in those
passages.

The intended response includes source-page information so that users can
locate and verify the supporting content in the original PDF.

### Intended users

The initial target user is anyone who needs to understand or search
information inside a PDF without manually reading the entire document.
Example use cases include students studying notes, readers reviewing
reports, and people working with manuals or other text-based documents.

These are intended use cases; the initial MVP is not designed as a
specialist legal, medical, or financial decision system.

## 2. Problem statement

Finding a specific answer in a long PDF can take time. A conventional
keyword search may miss relevant passages when the user's wording
differs from the document's wording.

The application aims to make document exploration conversational:

1.  Upload a PDF.
2.  Extract its text while retaining page numbers.
3.  Divide the text into manageable chunks.
4.  Convert chunks into embeddings and store them for semantic search.
5.  Retrieve the most relevant chunks when a question is asked.
6.  Send the question and retrieved context to Gemini.
7.  Return an answer with source-page references.

The application should not invent an answer when the uploaded document
does not provide enough supporting information.

## 3. MVP features

### Required for the first working version

-   Upload a text-based PDF.
-   Validate the uploaded file and handle invalid or unreadable
    documents.
-   Extract page-wise text and retain page numbers.
-   Split extracted text into chunks.
-   Generate embeddings for chunks.
-   Store and search vectors using ChromaDB.
-   Accept a question associated with a document.
-   Retrieve relevant chunks from that document.
-   Generate an answer using Gemini and retrieved context.
-   Return source-page references with the answer.
-   Display the upload and chat workflow in a frontend.

### Potential follow-up features

The following should be considered only after the core MVP works:

-   Chat history.
-   Multiple-document comparison or cross-document search.
-   OCR support for scanned/image-only PDFs.
-   User accounts and authentication.
-   Advanced document management.
-   Additional file formats.
-   Production-grade monitoring and administration.

These are not confirmed MVP requirements.

## 4. Technology stack

  Layer             Technology            Purpose
  ----------------- --------------------- ------------------------------------------
  Main language     Python                Backend and document/RAG logic
  API framework     FastAPI               Exposes backend endpoints
  ASGI server       Uvicorn               Runs the FastAPI application
  Validation        Pydantic              Validates structured request data
  PDF processing    PyMuPDF               Extracts text from PDF pages
  Vector database   ChromaDB              Stores embeddings and searchable chunks
  LLM               Gemini API            Generates answers from retrieved context
  Frontend          Vibe-coded frontend   Upload and chat user interface
  Version control   Git and GitHub        Collaboration and source history

### Technology decisions still to confirm

-   The exact embedding model and provider.
-   The exact Gemini model and Python SDK/version.
-   The frontend framework and hosting platform, if not already
    selected.
-   Whether chat history is part of the MVP.
-   The persistence strategy for uploaded PDFs and ChromaDB.
-   Hosting/deployment providers.

**Important:** FastAPI is the API/backend framework, not an AI model.
Gemini is the language model. ChromaDB is the vector database. The
frontend communicates with FastAPI; it should not connect directly to
Gemini or ChromaDB.

## 5. System architecture

High-level architecture:

``` text
                    FRONTEND
                       |
                  HTTP requests
                       |
                       v
                 FASTAPI BACKEND
                       |
             +---------+----------+
             |                    |
          UPLOAD                 CHAT
             |                    |
             v                    v
       PDF PROCESSING        RAG PIPELINE
             |                    |
       Extract text          Embed question
       page by page                |
             |                    v
       Create chunks          ChromaDB search
             |                    |
       Generate embeddings         |
             |                    |
             v                    v
          CHROMADB <---------- Relevant chunks
                                  |
                                  v
                              GEMINI API
                                  |
                                  v
                         Answer + page sources
                                  |
                                  v
                              FRONTEND
```

### Component responsibilities

**Frontend** - Lets the user select and upload a PDF. - Shows
upload/processing feedback. - Lets the user ask questions. - Displays
answers, errors, and source pages.

**FastAPI backend** - Receives frontend HTTP requests. - Validates
request data. - Coordinates document processing and RAG services. -
Returns structured JSON responses. - Keeps API credentials on the
server.

**PDF processing** - Reads the PDF page by page. - Extracts text. -
Splits text into chunks. - Preserves page and document metadata.

**Embedding and retrieval** - Converts document chunks and questions
into vectors using the same compatible embedding model. - Stores chunk
vectors and metadata in ChromaDB. - Retrieves relevant chunks for a
question. - Restricts retrieval to the selected document.

**Gemini** - Receives the user's question and retrieved document
context. - Generates an answer grounded in the supplied context. -
Should be instructed to say when the context does not contain enough
information.

## 6. Main application flows

### 6.1 PDF upload and indexing

``` text
User selects PDF
      |
      v
Frontend sends POST /upload
      |
      v
FastAPI validates the file
      |
      v
PDF text extracted page by page
      |
      v
Text divided into overlapping chunks
      |
      v
Chunks converted to embeddings
      |
      v
Chunks + vectors + metadata stored in ChromaDB
      |
      v
Backend returns document_id and status
```

The upload/indexing flow should run once for a document, rather than
reprocessing the entire PDF for every question.

### 6.2 Asking a question

``` text
User selects document and enters question
      |
      v
Frontend sends POST /chat
      |
      v
FastAPI validates document_id and question
      |
      v
Question converted to embedding
      |
      v
ChromaDB retrieves relevant chunks
(filtered by document_id)
      |
      v
Question + retrieved context sent to Gemini
      |
      v
Answer and source-page metadata returned
      |
      v
Frontend displays answer and page references
```

### 6.3 Example chat request

``` json
{
  "document_id": "abc123",
  "question": "Explain supervised learning."
}
```

### 6.4 Example chat response

``` json
{
  "answer": "Supervised learning is ...",
  "sources": [
    {
      "page": 12
    }
  ]
}
```

The response is an illustrative contract. The final schema may be
refined as the team implements the API.

## 7. API design

The following endpoints are planned in the project learning material.
Their final request/response schemas and status codes should be agreed
by the backend and frontend owners before implementation.

  Method     Endpoint           Intended responsibility
  ---------- ------------------ -----------------------------------------------
  `GET`      `/` or `/health`   Check whether the backend is running
  `POST`     `/upload`          Receive and process a PDF
  `POST`     `/chat`            Ask a question about a selected document
  `GET`      `/history`         Retrieve chat history, if included in the MVP
  `DELETE`   `/document`        Delete a document and its stored data

### Suggested chat request model

``` python
from pydantic import BaseModel

class ChatRequest(BaseModel):
    document_id: str
    question: str
```

Pydantic validates incoming structured data. For example, the chat
endpoint should receive a string document ID and a string question.

### HTTP methods

-   `GET`: retrieve information.
-   `POST`: send or create information.
-   `PUT`: replace/update information.
-   `PATCH`: partially update information.
-   `DELETE`: delete information.

### Common status codes

-   `200`: successful request.
-   `201`: resource created, where appropriate.
-   `400`: invalid request or upload.
-   `401`: unauthenticated request, if authentication is added.
-   `403`: request is not permitted.
-   `404`: requested document/resource not found.
-   `500`: unexpected server error.

Use specific and consistent error responses rather than returning
successful responses for failed operations.

## 8. Data model

A relational database has not yet been finalized in the project
discussions. The core RAG pipeline requires at least the following
logical records/metadata.

### Document metadata

  -----------------------------------------------------------------------
  Field                               Meaning
  ----------------------------------- -----------------------------------
  `document_id`                       Unique identifier for the uploaded
                                      document

  `filename`                          Original or display filename

  `status`                            Processing state, such as
                                      uploaded/processing/ready/failed

  `created_at`                        Upload timestamp, if tracked
  -----------------------------------------------------------------------

### Chunk metadata

  Field           Meaning
  --------------- -----------------------------------------------------
  `document_id`   Parent document identifier
  `chunk_id`      Unique chunk identifier
  `page_number`   Source PDF page
  `text`          Extracted chunk text
  `embedding`     Vector representation stored by the vector database

ChromaDB stores the searchable chunk records and associated metadata.
The precise collection layout, persistence configuration, and whether a
separate metadata database is needed remain implementation decisions.

### Chat history (optional)

If chat history is included, a logical message record could contain a
conversation ID, document ID, question, answer, timestamp, and source
references. This is a proposed extension, not a confirmed MVP schema.

## 9. Repository structure

Start with a simple structure and expand as the application grows. The
following is a proposed structure, not a claim that these files already
exist.

``` text
chat-with-any-document/
│
├── backend/
│   ├── main.py
│   ├── requirements.txt
│   ├── .env.example
│   ├── uploads/                 # Local development files; do not commit PDFs
│   ├── routes/                  # API endpoints (as project grows)
│   ├── services/                # PDF, embedding, retrieval, Gemini logic
│   ├── models/                  # Pydantic/data models
│   ├── database/                # ChromaDB setup and access
│   └── tests/
│
├── frontend/
│   └── ...                      # Frontend source
│
├── docs/                        # Architecture and team notes
├── .gitignore
└── README.md
```

The initial FastAPI learning exercise can be even smaller:

``` text
chat-with-document/
├── backend/
│   ├── main.py
│   ├── requirements.txt
│   └── uploads/
└── frontend/
```

Do not over-engineer the structure before the first endpoints and
PDF-processing flow work.

## 10. Local setup

### Prerequisites

Install:

-   Python (use a consistent version across the team; Python 3.11+ is a
    proposed baseline).
-   Git.
-   A code editor such as VS Code.
-   Node.js/npm if the chosen frontend requires it.
-   Access to the Gemini API when the AI integration phase begins.

### Clone the repository

``` bash
git clone <repository-url>
cd chat-with-any-document
```

### Create and activate a Python virtual environment

From the backend directory:

``` bash
cd backend
python -m venv venv
```

Windows:

``` bash
venv\Scripts\activate
```

macOS/Linux:

``` bash
source venv/bin/activate
```

### Install initial backend dependencies

For the first FastAPI exercise:

``` bash
pip install fastapi uvicorn
```

As the relevant phases begin, add the required dependencies, such as:

``` bash
pip install pydantic pymupdf chromadb
```

Install the selected Gemini SDK and embedding provider package once the
team confirms the model/provider. Keep verified dependencies in
`requirements.txt` so every teammate can reproduce the environment.

### Run the development server

If `main.py` contains the FastAPI instance named `app`:

``` bash
uvicorn main:app --reload
```

Open:

-   API root: `http://127.0.0.1:8000`
-   Interactive API docs: `http://127.0.0.1:8000/docs`

The `/docs` page can be used to test endpoints before the frontend is
connected.

### Minimal FastAPI starting point

``` python
from fastapi import FastAPI

app = FastAPI()

@app.get("/")
def home():
    return {
        "message": "Chat with Any Document API is running!"
    }
```

This is only a starting health/root endpoint. It does not implement PDF
processing or RAG.

## 11. Environment configuration

Keep credentials and machine-specific settings in a local `.env` file.
Do not commit `.env` to GitHub.

Suggested `.env.example` placeholders:

``` dotenv
GEMINI_API_KEY=replace_with_your_key
CHROMA_PATH=./chroma_data
UPLOAD_DIR=./uploads
MAX_UPLOAD_MB=20
FRONTEND_ORIGIN=http://localhost:3000
```

These names and the example size are proposed configuration values. The
code must load the variables it actually uses, and the team should agree
on the final names and limits.

**Never put a Gemini API key in frontend code or commit a real key.** If
a key is accidentally committed, revoke/rotate it.

## 12. Development roadmap

The proposed delivery plan has six phases across an estimated eight
weeks. The dates and durations are planning estimates and should be
adjusted to the team's actual weekly availability.

  -----------------------------------------------------------------------
  Phase                   Schedule                Main outcome
  ----------------------- ----------------------- -----------------------
  1\. Foundation          Week 1                  Repository,
                                                  environment, basic
                                                  FastAPI endpoints

  2\. PDF ingestion       Week 2                  Upload, page-wise
                                                  extraction, chunking

  3\. Semantic retrieval  Weeks 3--4              Embeddings, ChromaDB
                                                  indexing and retrieval

  4\. Answer generation   Week 5                  Gemini integration and
                                                  grounded answers

  5\. Product integration Week 6                  Frontend connected to
                                                  backend and RAG

  6\. Quality and release Weeks 7--8              Testing, deployment,
                                                  documentation and demo
  -----------------------------------------------------------------------

### Phase 1 --- Foundation

-   Agree on MVP scope, repository structure, endpoint contracts, and
    Git workflow.
-   Set up Python virtual environments and dependencies.
-   Build and run a minimal FastAPI application.
-   Create and test the initial root/health, upload placeholder, and
    chat placeholder endpoints.
-   Ensure every teammate can clone and run the project.

**Exit criterion:** all team members can run the backend and test its
endpoints using `/docs`.

### Phase 2 --- PDF ingestion

-   Learn and use PyMuPDF.
-   Validate uploaded PDFs.
-   Extract text page by page.
-   Handle empty, unreadable, and image-only PDFs.
-   Split text into chunks and preserve document/page metadata.

**Exit criterion:** a sample PDF can be converted into page-aware
chunks.

### Phase 3 --- Semantic retrieval

-   Select one embedding model/provider.
-   Generate embeddings for chunks.
-   Store chunks, vectors, and metadata in ChromaDB.
-   Embed a user question and retrieve the most relevant chunks.
-   Filter retrieval by document ID.

**Exit criterion:** test questions retrieve relevant passages from the
correct PDF with page metadata.

### Phase 4 --- Answer generation

-   Connect the backend to Gemini.
-   Construct prompts from the user's question and retrieved chunks.
-   Generate answers grounded in the provided context.
-   Return source-page references.
-   Handle insufficient context, timeouts, and API errors.

**Exit criterion:** supported questions receive grounded answers;
unsupported questions do not receive fabricated answers.

### Phase 5 --- Product integration

-   Connect upload and chat UI components to FastAPI.
-   Display upload progress, processing status, answers, loading states,
    and errors.
-   Display source-page references.
-   Test the complete browser workflow.

**Exit criterion:** a user can upload a PDF, ask a question, and see an
answer with page references in the application.

### Phase 6 --- Quality and release

-   Test normal, large, empty, invalid, and image-only PDFs.
-   Test relevant and irrelevant questions.
-   Verify page references and document-level retrieval isolation.
-   Test API failures and rate limits.
-   Configure persistent storage and deployment.
-   Finalize setup instructions, architecture notes, and a demo.

**Exit criterion:** the MVP passes agreed tests and another teammate can
run or use it by following the documentation.

## 13. Team work division

The proposed team structure divides the work into three departments,
each with three packages.

### Backend

-   **B1 --- Document Upload & Management:** upload endpoint,
    validation, storage, document IDs, deletion.
-   **B2 --- Chat API & Response Handling:** chat endpoint, request
    validation, AI service call, response/error formatting.
-   **B3 --- Data, Configuration & Reliability:** configuration, CORS,
    data models, shared errors, backend tests, optional history.

### Frontend

-   **F1 --- Document Upload Interface:** file selection, upload state,
    progress, document display, errors.
-   **F2 --- Chat Interface:** question input, message display, loading
    states, answers and source references.
-   **F3 --- App Layout & Integration:** app shell, navigation, document
    list, shared API client, responsive layout.

### AI / RAG

-   **A1 --- PDF Extraction & Chunking:** page-wise text extraction,
    chunking, metadata preservation.
-   **A2 --- Embeddings, ChromaDB & Retrieval:** embeddings, vector
    storage, similarity search, document filtering.
-   **A3 --- Gemini & Answer Generation:** prompt construction, answer
    generation, insufficient-context behavior, citations.

Each member is expected to take one package from each department. The
team has not yet confirmed the final assignment.

  Member     Backend          Frontend         AI/RAG
  ---------- ---------------- ---------------- ----------------
  Member 1   To be selected   To be selected   To be selected
  Member 2   To be selected   To be selected   To be selected
  Member 3   To be selected   To be selected   To be selected

### Work in parallel

Use mock inputs/outputs and agreed function contracts to reduce waiting:

-   A2 can test with sample chunks before A1 is complete.
-   A3 can test Gemini prompts with manually supplied passages before
    retrieval is connected.
-   B2 can use a mock AI service before A3 is integrated.
-   Frontend components can use sample JSON responses before real
    endpoints are ready.
-   F3 can establish shared layout/API utilities while F1 and F2 build
    their components.

Integration, testing, code review, deployment, documentation, and the
final presentation remain shared team responsibilities.

## 14. Git and collaboration workflow

Suggested branch pattern:

``` text
main
feature/pdf-upload
feature/chat-api
feature/pdf-extraction
feature/chromadb-retrieval
feature/gemini-integration
feature/upload-ui
feature/chat-ui
```

Recommended workflow:

1.  Pull the latest `main` branch.
2.  Create a feature branch for one task.
3.  Implement a small, focused change.
4.  Run relevant tests locally.
5.  Commit with a clear message.
6.  Open a pull request.
7.  Ask a teammate to review it.
8.  Resolve comments and merge.
9.  Pull the updated `main` branch before starting the next task.

Example:

``` bash
git checkout main
git pull origin main
git checkout -b feature/pdf-extraction

# Make changes and run tests
git add .
git commit -m "Add page-wise PDF text extraction"
git push -u origin feature/pdf-extraction
```

### Collaboration rules

-   Keep `main` runnable.
-   Do not commit `.env`, API keys, virtual environments, generated
    uploads, or local vector database files.
-   Use issues to record tasks, owners, and acceptance criteria.
-   Every pull request should explain what changed and how it was
    tested.
-   Review one another's code.
-   Demonstrate integrated progress regularly.
-   Document blockers and decisions so the whole team can follow them.

## 15. Testing strategy

Testing should happen throughout development, not only in the final
week.

### PDF processing

-   Valid text-based PDF.
-   Empty PDF or page with no extractable text.
-   Corrupted or invalid file.
-   Image-only/scanned PDF.
-   Long PDF and page-number accuracy.

### Retrieval

-   Question whose answer appears clearly in the PDF.
-   Question using different wording from the source.
-   Question whose answer is not in the PDF.
-   Retrieval from multiple documents without cross-document leakage.
-   Correct page metadata on retrieved chunks.

### Gemini and answer generation

-   Answer is supported by retrieved text.
-   Insufficient context produces an appropriate response.
-   API timeout, quota, and service errors are handled.
-   Source references correspond to retrieved chunks.

### API and frontend

-   Missing or malformed request fields.
-   Unknown document ID.
-   Upload and chat error states.
-   Loading and processing feedback.
-   Complete upload-to-answer workflow.

Create a small set of known questions and expected source pages. Use it
to check retrieval quality and citation correctness after changes.

## 16. Security and reliability considerations

-   Keep API keys on the backend and out of source control.
-   Validate file type and size; do not trust only the filename.
-   Generate safe server-side file names/IDs.
-   Avoid exposing server file paths in API responses.
-   Restrict retrieval to the selected document ID.
-   Handle model/provider errors and rate limits.
-   Decide how long uploaded PDFs and extracted content are retained.
-   Use persistent storage for ChromaDB and uploads in deployment;
    temporary hosting filesystems may be cleared on restart.
-   Do not log secrets or unnecessary document contents.
-   Treat document text as untrusted input when constructing prompts.

The initial MVP should support text-based PDFs. OCR, authentication, and
advanced security controls need separate scope and planning if required.

## 17. Current decisions and open questions

The following items must be confirmed by the team before or during
implementation:

-   [ ] Final MVP scope and supported PDF size/page limits.
-   [ ] Embedding model/provider.
-   [ ] Gemini model and SDK.
-   [ ] Chunk size, overlap, and retrieval top-k settings.
-   [ ] ChromaDB collection and persistence configuration.
-   [ ] Whether chat history is included in the MVP.
-   [ ] Whether users can upload more than one document in a session.
-   [ ] Frontend framework and final UI design.
-   [ ] Deployment provider and persistent storage strategy.
-   [ ] Document retention/deletion policy.
-   [ ] Final owner for each of the nine work packages.

Do not assume these choices are already finalized simply because they
appear as examples in this README.

## 18. Definition of done

The MVP can be considered complete when:

-   [ ] A user can upload a supported PDF.
-   [ ] The backend validates and processes the file.
-   [ ] PDF text is extracted page by page.
-   [ ] Chunks retain document and page metadata.
-   [ ] Embeddings are stored and searchable in ChromaDB.
-   [ ] Questions retrieve relevant chunks from the selected document
    only.
-   [ ] Gemini generates answers grounded in retrieved context.
-   [ ] Answers include useful source-page references.
-   [ ] The application handles unsupported questions and common errors.
-   [ ] The frontend supports the complete upload-to-answer flow.
-   [ ] Core tests pass and known limitations are documented.
-   [ ] Setup instructions work on another teammate's machine.
-   [ ] Deployment and persistent storage work as intended, if a hosted
    demo is required.
-   [ ] All team members can explain the complete application flow.

------------------------------------------------------------------------

## Getting started

The first practical step is to agree on the MVP and team package
assignments, create the shared GitHub repository, and complete the
Module 3 FastAPI exercise:

1.  Create a Python virtual environment.
2.  Install FastAPI and Uvicorn.
3.  Create `main.py` and a FastAPI app.
4.  Implement a root/health endpoint and a temporary chat endpoint.
5.  Create a Pydantic `ChatRequest` model.
6.  Run the server and test endpoints at `http://127.0.0.1:8000/docs`.
7.  Confirm that every teammate can run the same project locally.

Once this foundation is working, move to PDF processing (PyMuPDF) and
continue through the six development phases.

**Project principle:** build and understand the pipeline manually. Do
not introduce a framework such as LangChain at the beginning; the
learning plan is to understand the individual PDF processing, embedding,
retrieval, and generation steps first.
