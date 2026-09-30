# DocuMind — System Design

1. Overview

DocuMind is an AI-powered knowledge assistant that allows users to upload
internal documents and ask questions about their contents.

The system uses Retrieval-Augmented Generation (RAG) to retrieve relevant
document passages before generating an answer. Answers are grounded only
in retrieved document content and include citations to the source document
and passage.

The system supports document upload, background document processing,
semantic retrieval, question answering, authentication, question history,
and document-level access control 


2. Requirements

# Functional Requirements

- Users can create an account and log in.
- Passwords are securely hashed.
- Authentication is token-based using JWT.
- Users can upload PDF and plain text/Markdown documents.
- Uploaded documents are processed asynchronously in a background worker.
- Documents have queued, processing, ready, and failed states.
- Users can list and delete their own documents.
- Users can ask questions across their ready documents.
- Users can optionally restrict a question to selected documents.
- Answers include citations to the source document and passage.
- Question history is stored and can be retrieved.
- The system provides usage information such as token usage and latency.
- The system provides a health endpoint.
- The system applies per-user rate limiting to the question endpoint.
- The system provides a minimal web interface.
- Interactive API documentation is available through OpenAPI/Swagger.


# AI Requirements

- Document content is split into chunks before embedding.
- Embeddings are used for semantic retrieval.
- Retrieved passages are provided to the language model as context.
- Answers must be grounded in retrieved document content.
- The system must clearly refuse to answer when the documents do not
  contain the requested information.
- Uploaded documents are treated as untrusted data.
- Instructions contained inside uploaded documents must not be treated
  as system instructions.
- The system includes a prompt-injection test document.
- The system includes a repeatable evaluation set with at least 15 questions.


# 3. Assumptions

- Each user can access only documents that they own.
- Only PDF and plain text/Markdown files are supported.
- A sensible maximum upload size will be enforced.
- Only documents with a `ready` status can be used for question answering.
- Document processing may take time and therefore runs asynchronously.
- The vector store is used only for semantic retrieval and is not treated
  as the source of user/account data.
- API keys and other secrets are provided through environment variables
  and are never committed to the repository.
- The initial version is designed for a small number of users and documents.

  # 4. Architecture

# High-Level Architecture

![DocuMind High-Level Architecture](architecture.png)

DocuMind follows a client-server architecture with a separate background
processing worker for document ingestion.

The React client communicates with the FastAPI backend over HTTPS using
JWT-based authentication. The backend handles authentication, document
management, question answering, access control, rate limiting, and API
requests.

Uploaded documents are stored in persistent file storage. The FastAPI
backend creates a document record and places an ingestion job into Redis.
A background worker consumes the job, extracts text, splits the document
into chunks, generates embeddings, and stores the chunks and embeddings in
PostgreSQL with pgvector.

For question answering, the backend converts the user's question into an
embedding and performs semantic similarity search against the stored
document embeddings. The most relevant chunks are then supplied to the
LLM as context. The LLM generates an answer based only on the retrieved
content, and the backend returns the answer together with citations and
usage information.

Document Ingestion Flow
The user uploads a supported PDF or plain text/Markdown file.
FastAPI validates the file and stores it in persistent file storage.
FastAPI creates a document record with status queued.
FastAPI places an ingestion job into Redis.
The background worker consumes the job and changes the document status
to processing.
The worker extracts text from the document.
The extracted text is divided into smaller chunks.
An embedding model converts each chunk into a vector representation.
The chunks and their embeddings are stored in PostgreSQL with pgvector.
The document status is changed to ready.
If processing fails, the status is changed to failed and the failure
reason is recorded.
Question Answering Flow
The authenticated user submits a question.
FastAPI validates the request and checks the user's access permissions.
The question is converted into an embedding.
pgvector performs semantic similarity search against the user's
accessible document chunks.
The most relevant chunks are selected as context.
FastAPI constructs a prompt containing the question and retrieved
document passages.
The prompt is sent to the LLM.
The LLM generates an answer using only the supplied document context.
The backend returns the answer together with citations.
Token usage, latency, and other available usage information are recorded.
The question and answer are stored in question history


## 5. API Contract

All document and question endpoints require a valid JWT access token.
Authentication endpoints do not require a token.

Authentication
POST /auth/register

Creates a new user account.

Request

{
  "username": "example_user",
  "email": "user@example.com",
  "password": "password"
}

Response

{
  "message": "User registered successfully"
}
POST /auth/login

Authenticates a user and returns a JWT access token.

Request

{
  "email": "user@example.com",
  "password": "password"
}

Response

{
  "access_token": "jwt-token",
  "token_type": "bearer"
}
Documents
POST /documents

Uploads a PDF or plain text/Markdown document.

Request

Multipart form-data containing the document file.

Response

{
  "id": 1,
  "filename": "company_policy.pdf",
  "status": "queued"
}
GET /documents

Returns the authenticated user's documents.

Response

[
  {
    "id": 1,
    "filename": "company_policy.pdf",
    "status": "ready"
  }
]
GET /documents/{document_id}

Returns information about a document owned by the authenticated user.

DELETE /documents/{document_id}

Deletes the document and its associated chunks and embeddings.

Response

{
  "message": "Document deleted successfully"
}
Questions
POST /questions

Asks a question across the user's ready documents or selected documents.

Request

{
  "question": "How many annual leave days are provided?",
  "document_ids": []
}

Response

{
  "answer": "Employees receive 18 annual leave days.",
  "citations": [
    {
      "document": "company_policy.pdf",
      "page": 12,
      "passage": "Employees receive 18 annual leave days..."
    }
  ],
  "usage": {
    "tokens": 500,
    "latency_ms": 1200,
    "estimated_cost": 0.001
  }
}
GET /questions/history
Returns the authenticated user's previous questions and answers.

Health
GET /health

Reports the health of required system dependencies including the database,
vector store, and background worker/queue.


## 6. Data Model

DocuMind uses PostgreSQL as the primary relational database and pgvector
for storing and searching document embeddings.

### 6.1 Users

Stores registered user accounts.

| Field         | Type      | Description              |
| ------------- | --------- | ------------------------ |
| id            | UUID      | Unique user identifier   |
| username      | VARCHAR   | User's username          |
| email         | VARCHAR   | User's email address     |
| password_hash | VARCHAR   | Securely hashed password |
| created_at    | TIMESTAMP | Account creation time    |

### 6.2 Documents

Stores metadata and processing status for uploaded documents.

| Field         | Type      | Description                          |
| ------------- | --------- | ------------------------------------ |
| id            | UUID      | Unique document identifier           |
| user_id       | UUID      | Owner of the document                |
| filename      | VARCHAR   | Original filename                    |
| file_type     | VARCHAR   | PDF, TXT, or Markdown                |
| file_path     | VARCHAR   | Location of the stored file          |
| file_size     | INTEGER   | File size                            |
| status        | VARCHAR   | queued, processing, ready, or failed |
| error_message | TEXT      | Processing failure reason, if any    |
| created_at    | TIMESTAMP | Upload time                          |
| updated_at    | TIMESTAMP | Last status update                   |

Relationship:

```text
User 1 ──────────── N Documents
```

A user can own multiple documents, but each document belongs to exactly
one user.

### 6.3 Document Chunks

Stores the extracted and chunked content of each document together with
its embedding.

| Field       | Type      | Description                |
| ----------- | --------- | -------------------------- |
| id          | UUID      | Unique chunk identifier    |
| document_id | UUID      | Parent document            |
| chunk_index | INTEGER   | Position of the chunk      |
| content     | TEXT      | Extracted chunk text       |
| page_number | INTEGER   | Source page when available |
| embedding   | VECTOR    | Semantic embedding         |
| created_at  | TIMESTAMP | Creation time              |

Relationship:

```text
Document 1 ──────────── N Document Chunks
```

### 6.4 Questions

Stores question history and generated answers.

| Field          | Type      | Description                         |
| -------------- | --------- | ----------------------------------- |
| id             | UUID      | Unique question identifier          |
| user_id        | UUID      | User who asked the question         |
| question       | TEXT      | User's question                     |
| answer         | TEXT      | Generated answer                    |
| citations      | JSONB     | Source citations                    |
| tokens_used    | INTEGER   | Token usage when available          |
| latency_ms     | INTEGER   | Request latency                     |
| estimated_cost | DECIMAL   | Estimated model cost when available |
| created_at     | TIMESTAMP | Question time                       |

Relationship:

```text
User 1 ──────────── N Questions
```

### 6.5 Data Isolation

Every document and question is associated with a user ID.

All document and question queries will include the authenticated user's
ID so that a user cannot access another user's data by guessing a
document or question ID.

---

## 7. Technology Choices

### Frontend — React

React will be used to build the minimal web interface.

Reasons:

* Component-based UI development.
* Familiar ecosystem.
* Easy integration with REST APIs.
* Suitable for the required upload, status, question, and citation views.

### Backend — FastAPI

FastAPI will be used as the API backend.

Reasons:

* Python has a strong ecosystem for document processing and AI/ML.
* FastAPI provides automatic OpenAPI/Swagger documentation.
* Type validation can be implemented using Pydantic.
* It supports asynchronous API operations and is lightweight for this
  project.

### Database — PostgreSQL

PostgreSQL will store users, documents, chunks, and question history.

Reasons:

* Reliable relational database.
* Strong support for transactions and constraints.
* Suitable for user/document relationships.
* Works with pgvector for vector similarity search.

### Vector Store — pgvector

pgvector will be used for storing document embeddings and performing
semantic similarity searches.

Reasons:

* Keeps relational data and vector data in one database.
* Reduces infrastructure complexity.
* Supports similarity search needed for the RAG pipeline.

### Background Queue — Redis

Redis will be used as the queue backend for document processing jobs.

Reasons:

* Lightweight and easy to run with Docker Compose.
* Suitable for short-lived background jobs.
* Separates upload requests from expensive document processing.

### Background Worker — Celery

Celery will process document ingestion jobs asynchronously.

The worker will perform:

```text
Read file
   ↓
Extract text
   ↓
Chunk text
   ↓
Generate embeddings
   ↓
Store chunks and embeddings
   ↓
Update document status
```

### Document Processing — PyMuPDF and Python Text Processing

PyMuPDF will be used for extracting text from PDF documents.

Plain text and Markdown files will be read using standard Python file
handling.

### Authentication — JWT

JWT-based authentication will be used for API authorization.

Passwords will never be stored directly. Only secure password hashes
will be stored in PostgreSQL.

### LLM and Embedding Services

The LLM and embedding components will be accessed through configurable
providers using environment variables.

The application will keep provider-specific configuration outside the
source code so that the provider or model can be changed without
changing the core RAG architecture.

No API keys will be committed to the repository.

### Containerization — Docker

Docker and Docker Compose will be used to provide a reproducible local
environment containing the API, worker, database, Redis, and frontend.

### Testing — Pytest

Pytest will be used for unit and integration tests.

### CI — GitHub Actions

GitHub Actions will run linting, tests, and Docker image builds on
pull requests and pushes to the main branch.

---

## 8. Key Trade-offs

### 8.1 PostgreSQL + pgvector vs Separate Vector Database

A separate vector database such as Qdrant could provide dedicated vector
search capabilities.

For the initial version, pgvector is preferred because it allows
relational data and embeddings to be stored together.

This reduces the number of services that need to be configured and
deployed.

The trade-off is that a dedicated vector database may provide better
specialized functionality at larger scale.

### 8.2 Background Worker vs Synchronous Processing

Document processing is intentionally separated from the upload request.

If extraction and embedding were performed during the upload request,
large documents could cause long request times or timeouts.

The queue and worker add infrastructure complexity, but they satisfy the
required asynchronous ingestion behavior and make the API more reliable.

### 8.3 Local Persistent File Storage vs Object Storage

The initial deployment will use persistent file storage for uploaded
documents rather than introducing a separate object-storage service.

This keeps the first version simpler.

For a larger production deployment, object storage such as S3-compatible
storage could be introduced to improve scalability and durability.

### 8.4 Chunking Strategy

Documents will be divided into moderately sized chunks with a small
overlap between neighboring chunks.

The initial chunking parameters will be selected as a baseline and then
evaluated as part of the required retrieval experiment.

The exact parameters and evaluation results will be documented in
`EVALUATION.md`.

### 8.5 Top-K Retrieval

The system will retrieve a limited number of the most relevant chunks
rather than passing an entire document to the LLM.

This reduces prompt size, latency, and cost while keeping the retrieved
context focused.

The selected top-K value will be treated as an experimental parameter
and evaluated.

### 8.6 Streaming

Streaming responses are not part of the initial required implementation.

A normal request/response flow will be implemented first.

Streaming can be added later as an optional improvement if the required
functionality is complete.

---

## 9. Security Considerations

### Authentication

Protected endpoints require a valid JWT.

### Password Security

Passwords will be hashed using a secure password-hashing algorithm.
Plain-text passwords will never be stored.

### Authorization

Every document access will be checked against the authenticated user's
ID.

For example:

```text
User A
  ↓
GET /documents/123
  ↓
Check document.user_id == authenticated_user.id
  ↓
Allow / Reject
```

This prevents users from accessing another user's documents by changing
an ID in the request.

### Prompt Injection

Uploaded documents are considered untrusted data.

Retrieved document content will be explicitly treated as context rather
than instructions.

The system prompt will instruct the LLM not to follow instructions
contained inside retrieved documents.

A dedicated prompt-injection document will be included in the test and
evaluation set.

### Rate Limiting

The question endpoint will have per-user rate limiting to reduce the
risk of excessive LLM usage and unexpected costs.

### Secrets

API keys, database passwords, JWT secrets, and other sensitive
configuration will be supplied through environment variables.

A `.env.example` file will contain variable names and example values
without real secrets.

### Request Tracking

Requests will have a request ID that can be included in structured logs
to help trace failures across the API and background processing system.

---

## 10. Design Changes

This document represents the initial design created before the main
implementation.

Any significant architectural changes made during development will be
recorded here with the reason for the change.

### Initial Design

* React frontend
* FastAPI backend
* PostgreSQL with pgvector
* Redis queue
* Celery background worker
* Persistent file storage
* Embedding-based semantic retrieval
* LLM-based answer generation
* JWT authentication

### Changes During Implementation

No implementation-driven design changes have been made yet.

This section will be updated if the implementation requires changes to
the architecture, data model, API contract, or technology choices.
