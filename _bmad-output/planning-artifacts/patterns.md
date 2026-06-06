# ReadFlow — Implementation Patterns & Consistency Rules

> This document defines the patterns all AI coding agents and human contributors MUST follow when implementing ReadFlow. Its purpose is to prevent the "works in isolation, conflicts at integration" problem that arises when multiple agents make independent stylistic or structural decisions.
>
> When a pattern is not covered here, follow the nearest applicable rule and add the new pattern before merging.

---

## Pattern 1 — API Contract Conventions

### 1.1 Response Envelope

Every non-streaming JSON response from `readflow-api` uses this envelope:

**Success:**
```json
{
  "data": { ... },
  "meta": { "request_id": "uuid" }
}
```

**Error:**
```json
{
  "error": {
    "code": "QUOTA_EXCEEDED",
    "message": "Daily explain quota reached. Resets at UTC 00:00.",
    "detail": { "limit": 50, "resets_at": "2026-06-07T00:00:00Z" }
  },
  "meta": { "request_id": "uuid" }
}
```

**Rules:**
- Success response always wraps payload in `data` — never return a bare object or array
- `meta.request_id` is always present — injected by FastAPI middleware from `X-Request-ID` header (or generated if absent)
- `detail` is optional on errors; include only when structured context helps the client act on it
- HTTP status codes are authoritative — the envelope does not repeat the status
- Lists are returned as `{ "data": [...], "meta": { "total": N, "request_id": "..." } }`

**FastAPI implementation pattern:**
```python
# app/schemas/response.py
class SuccessResponse(BaseModel, Generic[T]):
    data: T
    meta: ResponseMeta

class ErrorResponse(BaseModel):
    error: ErrorDetail
    meta: ResponseMeta

class ErrorDetail(BaseModel):
    code: str          # SCREAMING_SNAKE_CASE
    message: str       # human-readable, safe to surface in UI
    detail: dict | None = None
```

**Next.js consumption pattern:**
```typescript
// src/lib/api-client.ts
type ApiSuccess<T> = { data: T; meta: { request_id: string } }
type ApiError = { error: { code: string; message: string; detail?: unknown }; meta: { request_id: string } }
type ApiResponse<T> = ApiSuccess<T> | ApiError

function isApiError(r: ApiResponse<unknown>): r is ApiError {
  return 'error' in r
}
```

---

### 1.2 Error Codes

Error codes are `SCREAMING_SNAKE_CASE` strings. Defined exhaustively here — agents must not invent new codes without adding them to this list.

| Code | HTTP Status | When |
|---|---|---|
| `UNAUTHORIZED` | 401 | Missing or invalid JWT |
| `FORBIDDEN` | 403 | Valid JWT but accessing another user's resource |
| `NOT_FOUND` | 404 | Resource does not exist for this user |
| `QUOTA_EXCEEDED` | 429 | Daily explain or monthly note quota exhausted |
| `PDF_UNSUPPORTED` | 422 | Uploaded file is scanned, encrypted, or not a PDF |
| `PDF_TOO_LARGE` | 422 | File exceeds 50 MB |
| `BEDROCK_ERROR` | 502 | AWS Bedrock call failed or timed out |
| `NOTE_JOB_FAILED` | 500 | Background note generation task failed |
| `VALIDATION_ERROR` | 422 | Request body failed Pydantic validation (FastAPI default) |
| `INTERNAL_ERROR` | 500 | Unhandled exception — never expose stack traces |

**Rule:** The `message` field must be safe to display directly in the UI. Never include internal details (stack traces, SQL errors, raw exception messages) in `message`. Use `detail` for structured debugging context visible only in dev mode.

---

### 1.3 SSE Event Format — Explain Stream

The `/explain/stream` endpoint returns `Content-Type: text/event-stream`. Events follow this exact format:

```
data: {"type": "token", "content": "nhật ký"}\n\n
data: {"type": "token", "content": " minh bạch"}\n\n
data: {"type": "token", "content": " chứng chỉ"}\n\n
data: {"type": "section", "section": "vi_complete", "content": "nhật ký minh bạch chứng chỉ — một cơ chế..."}\n\n
data: {"type": "section", "section": "en_context", "content": "Certificate Transparency (CT) is a framework..."}\n\n
data: {"type": "section", "section": "domain_tag", "content": "Security"}\n\n
data: {"type": "done"}\n\n
```

**Event types:**
| Type | When | Payload |
|---|---|---|
| `token` | Each streamed token | `{ content: string }` |
| `section` | Full section complete | `{ section: "vi_complete" \| "en_context" \| "domain_tag", content: string }` |
| `done` | Stream finished normally | `{}` |
| `error` | Bedrock error mid-stream | `{ code: string, message: string }` |

**FastAPI pattern:**
```python
# app/routers/explain.py
async def explain_stream_generator(text: str, domain: str):
    async for chunk in ai_service.explain_stream(text, domain):
        yield f"data: {chunk.model_dump_json()}\n\n"
    yield 'data: {"type": "done"}\n\n'

@router.post("/explain/stream")
async def explain_stream(req: ExplainRequest, user=Depends(get_current_user)):
    await quota_service.check_and_increment(user.id, QuotaType.EXPLAIN)
    return StreamingResponse(
        explain_stream_generator(req.selected_text, req.domain_hint),
        media_type="text/event-stream",
        headers={"X-Accel-Buffering": "no", "Cache-Control": "no-cache"},
    )
```

**Next.js consumption pattern:**
```typescript
// src/hooks/use-explain-stream.ts
export function useExplainStream() {
  return useMutation({
    mutationFn: async ({ selectedText, paperId }: ExplainRequest) => {
      const response = await fetch('/api/explain/stream', { method: 'POST', body: ... })
      const reader = response.body!.getReader()
      const decoder = new TextDecoder()
      // parse SSE events line by line, dispatch to state
    }
  })
}
```

**Rules:**
- `X-Accel-Buffering: no` header is mandatory — prevents Nginx from buffering the SSE stream
- The `error` event type terminates the stream; client must handle it and show an error state
- Client must not assume sections arrive in a specific order — parse by `section` field
- Quota check happens before the stream opens; a `429` HTTP status (not an SSE error event) signals quota exhaustion

---

### 1.4 Naming Conventions

**API endpoints:** Plural nouns, kebab-case, no trailing slash.
```
GET  /papers
POST /papers
GET  /papers/{paper_id}
POST /papers/{paper_id}/generate-note
POST /papers/{paper_id}/saved-explanations
GET  /note-jobs/{job_id}/status
POST /explain/stream
```

**Path parameters:** Always `{noun_id}` (snake_case). Never `{id}` alone — always contextual.

**Query parameters:** `snake_case`. Example: `?status=finished&page=1&page_size=20`.

**JSON field names (API ↔ frontend):** `snake_case` in all API responses. Next.js does NOT convert to camelCase — keep consistent with Python.

```json
{ "paper_id": "uuid", "created_at": "2026-06-06T10:00:00Z", "has_note": true }
```

**Date/time format:** ISO 8601 with UTC (`Z` suffix) in all API payloads. Never Unix timestamps.

---

## Pattern 2 — Database Migration Discipline

### 2.1 Migration Naming Convention

```
alembic revision --autogenerate -m "add_flashcards_table"
alembic revision --autogenerate -m "add_index_papers_user_id"
alembic revision --autogenerate -m "add_my_notes_to_smart_notes"
```

**Rules:**
- Name format: `verb_noun_context` in `snake_case`
- Verbs: `add`, `remove`, `alter`, `create`, `drop`, `rename`
- Always descriptive — `update_schema` is not acceptable
- Generated filename includes revision hash: `2026_06_06_1234abcd_add_flashcards_table.py`

### 2.2 Never Edit Existing Migrations

Once a migration has been applied to any environment (dev, prod), it is **immutable**. This is an absolute rule.

**If you made a mistake in a migration that has already run:**
1. Create a new migration that corrects the mistake
2. Never edit the existing file
3. If the migration has not yet run anywhere, it may be deleted and recreated

**Why:** Alembic tracks applied revisions by hash. Editing an applied migration creates a hash mismatch that corrupts the revision history and requires manual database intervention.

### 2.3 Always Additive Changes

v1 migrations only add — no `DROP COLUMN`, no `DROP TABLE`, no column renames that break existing queries.

**Permitted in v1:**
- `ADD COLUMN` (with default or nullable)
- `CREATE TABLE`
- `CREATE INDEX`
- `ALTER COLUMN` type widening (e.g., `VARCHAR(50)` → `TEXT`)

**Not permitted in v1 without a deprecation plan:**
- `DROP COLUMN` / `DROP TABLE`
- `ALTER COLUMN` that narrows type or changes semantics
- Column renames (breaks existing queries not yet updated)

### 2.4 CI Enforcement

GitHub Actions check in `readflow-api` pipeline:

```yaml
# .github/workflows/api-ci.yml
- name: Check no existing migrations were edited
  run: |
    # Fail if any file under alembic/versions/ that existed on main was modified
    git diff origin/main --name-only | grep "alembic/versions/" | while read f; do
      if git show origin/main:$f > /dev/null 2>&1; then
        echo "ERROR: Existing migration $f was edited. Create a new migration instead."
        exit 1
      fi
    done

- name: Verify migrations apply cleanly
  run: |
    alembic upgrade head
    alembic check  # Fails if autogenerate would produce changes (schema drift)
```

**`alembic check`** is the key enforcement: it regenerates the autogenerate diff and fails if the result is non-empty, meaning the models and DB are out of sync.

---

## Pattern 3 — Bedrock Call Abstraction

### 3.1 Rule: `ai_service.py` is the sole Bedrock caller

No router, no other service, no background task calls `boto3` (or `botocore`) directly. All AWS Bedrock interactions go through `app/services/ai_service.py`.

```
# CORRECT
from app.services.ai_service import ai_service
result = await ai_service.explain_stream(selected_text, domain)

# FORBIDDEN — never in routers or other services
import boto3
client = boto3.client("bedrock-runtime", ...)
```

### 3.2 `ai_service.py` Interface

```python
# app/services/ai_service.py

class AIService:
    def __init__(self, client: BedrockRuntimeClient):
        self._client = client

    async def explain_stream(
        self,
        selected_text: str,
        domain: str,
        paper_context: str | None = None,
    ) -> AsyncGenerator[SSEEvent, None]:
        """
        Streams bilingual explanation for selected_text.
        Yields SSEEvent objects (type: token | section | done | error).
        """
        ...

    async def generate_smart_note(
        self,
        paper_text: str,
        paper_title: str,
    ) -> SmartNoteResult:
        """
        Generates structured smart note from full paper text.
        Returns SmartNoteResult with what/why/how/key_terms fields.
        Raises BedrockServiceError on failure.
        """
        ...

# Singleton — injected via FastAPI dependency
ai_service = AIService(client=boto3.client("bedrock-runtime", region_name=settings.AWS_REGION))
```

### 3.3 Why this pattern

| Concern | How `ai_service.py` abstraction solves it |
|---|---|
| **Cost tracking** | All Bedrock calls in one file → add usage logging in one place |
| **Quota enforcement** | `quota_service` checks quota; `ai_service` executes; clear separation |
| **Mock injection for tests** | Tests inject a mock `AIService` via dependency override — no boto3 calls in test runs |
| **Model upgrades** | Haiku → Haiku v3, Sonnet → Sonnet v2 — change in one file, not scattered across codebase |
| **Error normalization** | `BedrockServiceError` wraps all boto3 exceptions — callers handle one exception type |
| **Prompt versioning** | All prompts live in `ai_service.py` or adjacent `prompts/` folder — never inline in routers |

### 3.4 Prompt Location

Prompts are defined as module-level constants in `app/services/prompts/`:

```
app/services/prompts/
├── explain_system.txt       # System prompt for bilingual explain
├── note_system.txt          # System prompt for smart note generation
└── __init__.py              # Loads and exposes prompt strings
```

**Rule:** Prompts are never f-strings inline in `ai_service.py` method bodies. They are loaded from the `prompts/` folder with variable substitution via `.format()` or a simple template loader. This makes prompt iteration visible in git diffs.

### 3.5 Test Pattern

```python
# tests/conftest.py
@pytest.fixture
def mock_ai_service():
    service = MagicMock(spec=AIService)
    service.explain_stream.return_value = async_generator([
        SSEEvent(type="section", section="vi_complete", content="giải thích test"),
        SSEEvent(type="done"),
    ])
    return service

@pytest.fixture
def app_with_mock_ai(mock_ai_service):
    app.dependency_overrides[get_ai_service] = lambda: mock_ai_service
    yield app
    app.dependency_overrides.clear()
```

---

## Pattern 4 — Environment Parity

### 4.1 docker-compose for Local Development

`readflow-api/docker-compose.yml` mirrors the k3s deployment as closely as possible. Same env var names, same service names, same port conventions.

```yaml
# readflow-api/docker-compose.yml
version: "3.9"
services:
  postgres:
    image: postgres:16-alpine
    environment:
      POSTGRES_DB: readflow
      POSTGRES_USER: readflow
      POSTGRES_PASSWORD: readflow_dev
    ports:
      - "5432:5432"
    volumes:
      - postgres_data:/var/lib/postgresql/data

  api:
    build: .
    ports:
      - "8000:8000"
    environment:
      DATABASE_URL: postgresql://readflow:readflow_dev@postgres:5432/readflow
      AWS_REGION: ap-southeast-1
      AWS_ACCESS_KEY_ID: ${AWS_ACCESS_KEY_ID}       # from .env.local, never hardcoded
      AWS_SECRET_ACCESS_KEY: ${AWS_SECRET_ACCESS_KEY}
      BEDROCK_EXPLAIN_MODEL: anthropic.claude-haiku-20240307-v1:0
      BEDROCK_NOTE_MODEL: anthropic.claude-sonnet-4-5
      JWT_SECRET_KEY: dev_jwt_secret_not_for_prod
      JWT_ALGORITHM: HS256
      GOOGLE_CLIENT_ID: ${GOOGLE_CLIENT_ID}
      GOOGLE_CLIENT_SECRET: ${GOOGLE_CLIENT_SECRET}
      S3_BUCKET_NAME: ${S3_BUCKET_NAME}
      FRONTEND_URL: http://localhost:3000
      ENVIRONMENT: development
    depends_on:
      - postgres
    volumes:
      - .:/app
    command: uvicorn app.main:app --host 0.0.0.0 --port 8000 --reload

volumes:
  postgres_data:
```

```yaml
# readflow-web/docker-compose.yml (or add to root compose)
  web:
    build: .
    ports:
      - "3000:3000"
    environment:
      NEXT_PUBLIC_API_URL: http://localhost:8000
      ENVIRONMENT: development
```

### 4.2 Env Var Naming Convention

All env vars follow `SCREAMING_SNAKE_CASE`. Same name in `.env.local` (dev), k3s Secret (prod), and `docker-compose.yml`.

| Variable | Used in | Dev value |
|---|---|---|
| `DATABASE_URL` | API | `postgresql://readflow:readflow_dev@postgres:5432/readflow` |
| `AWS_REGION` | API | `ap-southeast-1` |
| `AWS_ACCESS_KEY_ID` | API | from shell env (never committed) |
| `AWS_SECRET_ACCESS_KEY` | API | from shell env |
| `BEDROCK_EXPLAIN_MODEL` | API | `anthropic.claude-haiku-20240307-v1:0` |
| `BEDROCK_NOTE_MODEL` | API | `anthropic.claude-sonnet-4-5` |
| `JWT_SECRET_KEY` | API | `dev_jwt_secret_not_for_prod` |
| `S3_BUCKET_NAME` | API | `readflow-dev-pdfs` |
| `FRONTEND_URL` | API | `http://localhost:3000` |
| `NEXT_PUBLIC_API_URL` | Web | `http://localhost:8000` |
| `ENVIRONMENT` | Both | `development` \| `production` |

**Rules:**
- `NEXT_PUBLIC_` prefix required for any var accessed client-side in Next.js
- AWS credentials **never** appear in `docker-compose.yml` values — always reference from shell env via `${VAR}` and `.env.local` (git-ignored)
- k3s Secrets use the identical var names — Helm templates reference the same keys
- `ENVIRONMENT=development` enables OpenAPI docs at `/docs`, debug logging, and CORS from `localhost:3000`
- `ENVIRONMENT=production` disables OpenAPI docs, enables structured JSON logging, restricts CORS

### 4.3 `.env.local` Convention

```bash
# readflow-api/.env.local   ← git-ignored
AWS_ACCESS_KEY_ID=AKIA...
AWS_SECRET_ACCESS_KEY=...
GOOGLE_CLIENT_ID=...
GOOGLE_CLIENT_SECRET=...
S3_BUCKET_NAME=readflow-dev-pdfs
```

```bash
# readflow-web/.env.local   ← git-ignored
NEXT_PUBLIC_API_URL=http://localhost:8000
```

`.env.example` files (committed, no real values) document every required variable:
```bash
# readflow-api/.env.example
AWS_ACCESS_KEY_ID=          # IAM key with Bedrock + S3 permissions
AWS_SECRET_ACCESS_KEY=      # IAM secret
GOOGLE_CLIENT_ID=           # Google OAuth2 client ID
GOOGLE_CLIENT_SECRET=       # Google OAuth2 client secret
S3_BUCKET_NAME=             # e.g. readflow-dev-pdfs
```

### 4.4 Parity Rules

**Things that must match between local and k3s:**
- Same model IDs for Bedrock (no "use a cheaper model locally" shortcuts — Haiku is cheap enough)
- Same PostgreSQL version (16-alpine in docker-compose matches k3s StatefulSet image)
- Same Alembic migrations applied (`alembic upgrade head` runs on both)
- Same CORS configuration (controlled by `FRONTEND_URL` env var)

**Acceptable dev-only differences:**
- `JWT_SECRET_KEY` is a weak dev value locally; k3s uses a strong generated secret
- No EBS volume in local (docker volume instead)
- No TLS in local (plain HTTP on `localhost`)
- `--reload` flag in uvicorn (local only)

---

## Cross-Cutting Patterns

### Naming Conventions Summary

| Context | Convention | Example |
|---|---|---|
| Python files, modules | `snake_case` | `ai_service.py`, `quota_middleware.py` |
| Python classes | `PascalCase` | `AIService`, `SmartNoteResult` |
| Python functions/variables | `snake_case` | `generate_smart_note()`, `paper_id` |
| DB tables | `snake_case`, plural | `users`, `smart_notes`, `note_jobs` |
| DB columns | `snake_case` | `user_id`, `text_content`, `created_at` |
| API endpoints | `kebab-case`, plural nouns | `/papers`, `/note-jobs`, `/saved-explanations` |
| JSON field names | `snake_case` | `paper_id`, `created_at`, `has_note` |
| TypeScript files | `PascalCase` for components, `kebab-case` for utilities | `SmartNoteView.tsx`, `use-explain-stream.ts` |
| TypeScript types/interfaces | `PascalCase` | `SmartNote`, `ExplainRequest` |
| TypeScript variables | `camelCase` | `paperId`, `isLoading` |
| React Query keys | Array of strings | `['papers', paperId, 'note']` |
| CSS classes (Tailwind) | utility-first, no custom class names unless necessary | — |
| k3s namespaces | `kebab-case` | `readflow-dev`, `readflow-prod` |
| Git branches | `kebab-case` with prefix | `feat/explain-stream`, `fix/quota-reset` |

### Error Handling Pattern

**API (Python):**
```python
# All unhandled exceptions → INTERNAL_ERROR, no stack trace in response
@app.exception_handler(Exception)
async def global_exception_handler(request, exc):
    logger.error("Unhandled exception", exc_info=exc, extra={"request_id": ...})
    return JSONResponse(status_code=500, content={"error": {"code": "INTERNAL_ERROR", "message": "An unexpected error occurred."}, "meta": {...}})
```

**Frontend (TypeScript):**
```typescript
// React Query error is typed as ApiError — always check error.code before displaying
if (isApiError(response)) {
  if (response.error.code === 'QUOTA_EXCEEDED') {
    showQuotaWarning(response.error.message)
  } else {
    showGenericError(response.error.message)
  }
}
```

### Loading State Pattern

React Query manages all async state. Components use query/mutation status directly — no separate loading state in React context or Zustand.

```typescript
const { data, isLoading, isError, error } = useQuery({ ... })
// isLoading → show skeleton
// isError → show error state with error.message
// data → render content
```

No `useEffect` + `useState` pairs for data fetching. React Query only.

---

## Enforcement Checklist

All AI agents MUST follow these before marking a story complete:

- [ ] API response wrapped in `{ data, meta }` envelope
- [ ] Error uses a code from the defined `ErrorCode` list
- [ ] No `boto3` import outside `ai_service.py`
- [ ] New env vars added to both `docker-compose.yml` and `.env.example`
- [ ] New Alembic migrations follow `verb_noun_context` naming
- [ ] No existing migration files modified
- [ ] Tests use `mock_ai_service` fixture — no live Bedrock calls in unit tests
- [ ] JSON response fields use `snake_case`
- [ ] React Query used for all API data — no `useEffect` + `fetch` patterns
- [ ] SSE endpoint includes `X-Accel-Buffering: no` header
