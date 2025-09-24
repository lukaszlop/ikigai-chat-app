# REST API Plan

## 1. Resources

- AuthSession: mocked login/logout and session inspection
- Config: public runtime configuration and client limits
- Conversation: implicit single conversation per authenticated session (no persistent storage)
- Message: user/assistant messages, generation (streaming and non-streaming), stop/retry/regenerate
- Attachment (ephemeral): files included with a message (images, txt, md), normalized on server
- Usage: daily budget tracking and enforcement (0.5 USD/day)
- Profile (optional, post-MVP): server persistence (MVP uses localStorage on client)
- Metrics (optional, dev-only): submit local metrics for a `/metrics` page
- Health: basic liveness/readiness

## 2. Endpoints

### AuthSession

- Method: POST

  - Path: /api/auth/login
  - Description: Mocked login with fixed credentials. Issues HttpOnly session cookie and returns session info.
  - Request JSON:
    {
    "email": "string",
    "password": "string"
    }
  - Response JSON:
    {
    "user": { "id": "string", "email": "string", "name": "string" },
    "session": { "id": "string", "issuedAt": "ISO-8601", "expiresAt": "ISO-8601" },
    "budget": { "limitUsd": 0.5, "usedUsd": 0, "remainingUsd": 0.5, "resetAt": "ISO-8601" }
    }
  - Success:
    - 200 OK (sets HttpOnly cookie)
  - Errors:
    - 400 Invalid payload
    - 401 Invalid credentials

- Method: POST

  - Path: /api/auth/logout
  - Description: Clears session cookie.
  - Request: none
  - Response JSON:
    { "ok": true }
  - Success:
    - 200 OK
  - Errors:
    - 401 Not authenticated (if no session)

- Method: GET
  - Path: /api/auth/session
  - Description: Returns current session and budget info.
  - Response JSON:
    {
    "authenticated": true,
    "user": { "id": "string", "email": "string", "name": "string" },
    "session": { "id": "string", "issuedAt": "ISO-8601", "expiresAt": "ISO-8601" },
    "budget": { "limitUsd": 0.5, "usedUsd": 0.12, "remainingUsd": 0.38, "resetAt": "ISO-8601" }
    }
  - Success:
    - 200 OK
  - Errors:
    - 401 Not authenticated

### Config

- Method: GET
  - Path: /api/config
  - Description: Read-only config for the client UI.
  - Response JSON:
    {
    "model": { "provider": "openai", "name": "gpt-4o-mini", "maxTokensText": 512, "maxTokensWithFiles": 768 },
    "limits": {
    "textMaxChars": 10000,
    "totalMaxChars": 30000,
    "attachmentsMaxPerMessage": 3,
    "attachmentMaxBytes": 5242880,
    "attachmentsTotalBytes": 15728640,
    "allowedMimeTypes": ["image/jpeg", "image/png", "text/plain", "text/markdown"]
    },
    "budget": { "dailyLimitUsd": 0.5 },
    "streaming": { "sse": true },
    "context": { "defaultWindow": 6 }
    }
  - Success:
    - 200 OK

### Conversation (implicit single conversation per session)

- Method: DELETE
  - Path: /api/chat
  - Description: Clear server-side ephemeral conversation context (if any). Client still owns UI history in sessionStorage.
  - Response JSON:
    { "ok": true }
  - Success:
    - 200 OK
  - Errors:
    - 401 Not authenticated

### Message

- Method: POST (non-streaming)

  - Path: /api/chat/messages
  - Description: Create a user message and synchronously return the full assistant reply. Processes attachments server-side.
  - Query params:
    - stream=false (default)
  - Request: multipart/form-data
    - fields:
      - text: string (0..10,000 chars)
      - context: string (optional JSON string)
        {
        "messages": [ { "role": "user|assistant|system", "text": "string" } ]
        }
      - files[]: up to 3 files (jpg, png, txt, md); ≤5 MB each; ≤15 MB total
    - server behavior:
      - Images: auto-orient, max dimension 2048 px, strip EXIF, convert to WEBP q≈80 (or PNG when text/transparency is detected)
      - txt/md: server reads content; contributes to totalMaxChars budget
  - Response JSON:
    {
    "userMessage": {
    "id": "string",
    "role": "user",
    "text": "string",
    "attachments": [
    {
    "id": "string",
    "kind": "image|text",
    "fileName": "string",
    "mimeType": "string",
    "sizeBytes": 0,
    "hash": "sha256",
    "image": { "width": 1200, "height": 800, "format": "webp|png" },
    "text": { "charCount": 1234, "preview": "first 120 chars" }
    }
    ],
    "createdAt": "ISO-8601"
    },
    "assistantMessage": {
    "id": "string",
    "role": "assistant",
    "text": "string",
    "status": "completed",
    "tokens": { "prompt": 0, "completion": 0, "total": 0 },
    "costUsd": 0.0012,
    "createdAt": "ISO-8601"
    },
    "usage": { "usedUsd": 0.12, "remainingUsd": 0.38, "limitUsd": 0.5 }
    }
  - Success:
    - 201 Created
  - Errors:
    - 400 Invalid form
    - 401 Not authenticated
    - 413 Payload too large
    - 415 Unsupported media type
    - 422 Validation failed (limits/char counts)
    - 429 Daily budget exceeded
    - 500 Internal error

- Method: POST (streaming, SSE)

  - Path: /api/chat/messages
  - Description: Create a user message and stream the assistant reply over SSE.
  - Headers:
    - Accept: text/event-stream
  - Query params:
    - stream=true
  - Request: multipart/form-data (same fields as non-streaming)
  - SSE Events:
    - event: "ready" data: { "messageId": "string" }
    - event: "delta" data: { "messageId": "string", "textDelta": "string" }
    - event: "usage" data: { "promptTokens": 0, "completionTokens": 0, "totalTokens": 0, "costUsd": 0.0005 }
    - event: "error" data: { "code": "string", "message": "string" }
    - event: "done" data: {
      "messageId": "string",
      "text": "full assistant text",
      "tokens": { "prompt": 0, "completion": 0, "total": 0 },
      "costUsd": 0.0012,
      "createdAt": "ISO-8601"
      }
  - Success:
    - 200 OK (SSE)
  - Errors:
    - Same as above (sent as SSE "error" then stream closed)

- Method: POST

  - Path: /api/chat/messages/{id}/stop
  - Description: Stop in-flight generation. Preserves partial content generated so far.
  - Request JSON:
    { "reason": "user_cancel" }
  - Response JSON:
    { "ok": true, "messageId": "string", "status": "stopped" }
  - Success:
    - 200 OK
  - Errors:
    - 404 Not found/in-flight
    - 409 Already finished
    - 401 Not authenticated

- Method: POST

  - Path: /api/chat/messages/{id}/regenerate
  - Description: Generate a new assistant reply for the same user message and attachments (server reuses cached-normalized attachments; TTL applies). Uses the same context by default.
  - Request JSON:
    { "stream": true|false, "context": { "messages": [ { "role": "user|assistant|system", "text": "string" } ] } }
  - Response:
    - Same shape as POST /api/chat/messages (streaming or non-streaming)
  - Success:
    - 201 Created (non-stream) or 200 OK (SSE)
  - Errors:
    - 404 Unknown message
    - 401 Not authenticated
    - 429 Daily budget exceeded

- Method: POST

  - Path: /api/chat/messages/{id}/retry
  - Description: Retry the last failed generation with identical inputs. Only allowed if prior status == error.
  - Request JSON:
    { "stream": true|false }
  - Response:
    - Same as regenerate
  - Success:
    - 201 Created / 200 OK
  - Errors:
    - 409 Not in error state
    - Others as above

- Method: GET (optional, future)
  - Path: /api/chat/messages
  - Description: List server-side messages for the implicit conversation (MVP may return empty or 204 when disabled).
  - Query params:
    - limit (default 50, max 200)
    - before (cursor id or ISO-8601 timestamp)
    - role (user|assistant)
    - status (completed|streaming|stopped|error)
    - sort (createdAt:asc|desc; default desc)
  - Response JSON:
    {
    "items": [ { "id": "string", "role": "user|assistant", "text": "string", "createdAt": "ISO-8601" } ],
    "pageInfo": { "nextCursor": "string|null", "hasMore": true }
    }
  - Success:
    - 200 OK; 204 No Content
  - Errors:
    - 401 Not authenticated

### Usage (Budget)

- Method: GET

  - Path: /api/usage
  - Description: Return today’s usage and whether sending is allowed.
  - Response JSON:
    {
    "date": "YYYY-MM-DD",
    "usedUsd": 0.12,
    "limitUsd": 0.5,
    "remainingUsd": 0.38,
    "willBlock": false,
    "resetAt": "ISO-8601"
    }
  - Success:
    - 200 OK

- Method: POST (dev-only)
  - Path: /api/usage/reset
  - Description: Reset daily usage (local testing only).
  - Response JSON:
    { "ok": true, "resetAt": "ISO-8601" }
  - Success:
    - 200 OK
  - Errors:
    - 403 Disabled in production

### Profile (optional, post-MVP)

- Method: GET

  - Path: /api/profile
  - Description: Fetch profile from server store (MVP: client-only localStorage; reserved for future).
  - Response JSON:
    { "name": "string", "email": "string", "avatarUrl": "string|null" }
  - Success:
    - 200 OK; 204 No Content

- Method: PUT
  - Path: /api/profile
  - Description: Update profile (name and avatar). Validate avatar type/size (jpg/png/webp; ≥256×256; ≤2 MB).
  - Request: multipart/form-data
    - name: string
    - avatar: file (optional)
  - Response JSON:
    { "ok": true, "profile": { "name": "string", "email": "string", "avatarUrl": "string|null" } }
  - Success:
    - 200 OK
  - Errors:
    - 400 Invalid
    - 415 Unsupported media type
    - 422 Validation failed

### Metrics (optional, dev-only)

- Method: POST
  - Path: /api/metrics
  - Description: Submit local metrics (TTFT, latency, upload success, error rate, cost/message). No PII.
  - Request JSON:
    {
    "type": "ttft|latency|error|upload|cost",
    "value": 0,
    "meta": { "route": "/api/chat/messages", "model": "gpt-4o-mini" }
    }
  - Response JSON:
    { "ok": true }
  - Success:
    - 202 Accepted
  - Errors:
    - 400 Invalid payload

### Health

- Method: GET
  - Path: /api/health
  - Description: Liveness/readiness check.
  - Response JSON:
    { "ok": true, "uptimeSec": 123.4 }
  - Success:
    - 200 OK

## 3. Authentication and Authorization

- Mechanism: Mocked credentials (email: test@example.com, password: password123). On success, issue a signed JWT stored in an HttpOnly, Secure, SameSite=Lax cookie (e.g., `ikigai_session`). Bearer token via Authorization header is also supported for tooling.
- Protection: All `/api/chat/*`, `/api/usage`, and (future) `/api/profile` require a valid session. Public: `/api/config`, `/api/health`.
- Session lifetime: 24 hours (configurable). No idle timeout in MVP.
- CSRF: For cookie-based auth, use SameSite=Lax and enforce Origin/Referer checks on state-changing requests. Optionally a double-submit CSRF token for added safety.

## 4. Validation and Business Logic

### Global

- Rate limiting: Per user/session and IP.
  - chat generation: 30 req/min (burst 10)
  - file uploads within message: 10 files/min, ≤15 MB/request
  - metrics: 60 req/min
  - Responses include `Retry-After` (when applicable) and `X-RateLimit-*` headers
- CORS: Restrict to localhost origins during dev.
- Error envelope (JSON): { "error": { "code": "string", "message": "string", "details": any } }

### Message creation (streaming and non-streaming)

- Text length: 0..10,000 chars (JS UTF-16 code units; newlines count as chars).
- Total content length (text + concatenated txt/md contents): ≤30,000 chars.
- Attachments: ≤3 files; allowed types: image/jpeg, image/png, text/plain, text/markdown; ≤5 MB/file; ≤15 MB total.
- Images normalization: auto-orient; strip EXIF; resize max(width,height) ≤ 2048; choose WEBP q≈80 unless transparency or text-heavy → PNG.
- txt/md handling: read UTF-8; include content in prompt under headers: "Attachment N: <fileName>"; counts toward total chars.
- Context window: client may provide up to the last ~6 messages via `context.messages`; server can fallback to any ephemeral in-flight cache if enabled. No persistent conversation storage in MVP.
- Output limits: maxTokens = 512 (no files) or 768 (with files).
- Streaming: Use SSE; send `ready`, `delta`, periodic `usage`, finalize with `done`.
- Stop: Maintain an AbortController keyed by messageId; `stop` sets status to `stopped` and closes stream.
- Retry: Allowed only when previous attempt ended in `error`.
- Regenerate: Produces a new assistant message for the same user input/attachments.
- Ephemeral storage: Normalized attachments cached server-side (tmp fs or memory) with TTL 24h to support regenerate/retry; never public; scoped to session.
- Budget enforcement: Before model call, estimate cost; if `usedUsd + estimated > 0.5`, return 429 with message "Daily budget exceeded (0.5 USD)." After completion, persist actual cost for the day.

### Usage (Budget)

- Counting: Use provider list prices for gpt-4o-mini with a safety margin (configurable). Track per user/session per local day (browser time). Reset at local midnight.
- Blocking: If remainingUsd ≤ 0, all chat endpoints return 429 until reset.

### Profile (optional)

- Name: 1..80 chars.
- Email: read-only equals login email.
- Avatar: jpg/png/webp; ≤2 MB; minimum 256×256; client performs 1:1 crop; server validates dimensions and type.

### Security

- Authentication: JWT (HS256) with rotation on login; validate on each protected route.
- Input validation: Strict MIME, file size, char counts; reject ambiguous encodings; sanitize filenames.
- Security headers: `Content-Security-Policy` (restrictive), `X-Content-Type-Options: nosniff`, `Referrer-Policy: strict-origin-when-cross-origin`.
- Transport: HTTPS in production (HTTP allowed on localhost for dev).
- Logging: Request IDs and minimal structured logs without PII.
