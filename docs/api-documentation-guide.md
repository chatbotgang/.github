# API Documentation Guide

How every Crescendo Lab service documents its HTTP API.

This guide is organization-wide. It applies to any repository in `chatbotgang` or
`crescendolab-open` that serves an HTTP API — product backends (`rubato`, `cantata`,
`interlude`, `dolce`, `bebop`), platform services (`medley`, `overture`, `harmony`,
`arioso`, `modulation`), and anything else a second team has to call.

---

## A note on the word "OpenAPI"

The term is overloaded here, and mixing the two meanings is the fastest way to write a
confusing document:

| Term | Meaning |
|------|---------|
| **OpenAPI Specification** (OAS) | The industry format for describing HTTP APIs, currently 3.x. The machine-readable file this guide asks you to produce. |
| **MAAC OpenAPI** | Our customer-facing public API product, served under `/maac/openapi/v2/...` and authenticated with a customer API key. A *product*, not a format. |

When you mean the format, write "the OpenAPI spec" or "the schema." When you mean the
product, write "MAAC OpenAPI" or "the public API." Never write "the OpenAPI" alone.

---

## Principles

1. **The spec ships with the code.** An endpoint that is merged without documentation is
   an unfinished endpoint. Reviewers should treat a missing or stale schema the same way
   they treat a missing test.
2. **Generate, don't transcribe.** Derive the schema from the handlers, serializers, and
   type definitions that actually run. Hand-written endpoint tables drift within weeks and
   then actively mislead.
3. **One source of truth per service.** Each service commits exactly one spec file. If a
   customer-facing document exists elsewhere (Help Center, developer portal), it is
   generated from — or reviewed against — that file, never maintained in parallel.
4. **Document the contract, not the implementation.** Callers need request shape, response
   shape, status codes, errors, and limits. They do not need your table names, queue
   topology, or which service you call downstream.
5. **Accuracy beats consistency.** Where an existing service's behavior differs from the
   recommended defaults below, document what it really does. Then open a ticket if the
   difference is worth closing. Never document the convention you wish you had.

---

## What every endpoint must state

A reader should be able to make a correct first call without reading your source or asking
in Slack. That means:

- **Summary** — one line, imperative ("Submit a batch send request").
- **Description** — what it does, when to use it, and what it does *not* do.
- **Auth** — which scheme, which scopes or permissions, which roles.
- **Parameters** — path, query, header, and body fields, each with type, whether it is
  required, constraints (length, range, enum, format), and a default when one exists.
- **Success responses** — status code and body schema for each. `202` means the work is
  queued; say explicitly how the caller learns the outcome.
- **Error responses** — every status code a caller can reasonably hit, with the stable
  machine-readable error code for each. See [Errors](#errors).
- **At least one realistic example** — a `curl` request and the response it returns. Use
  plausible values; never `string`, `foo`, or a real customer's data.
- **Limits** — rate limit, payload size caps, batch size caps, pagination ceiling.
- **Idempotency** — whether a retry is safe, and which key makes it safe.

For asynchronous and long-running endpoints, also state the terminal states, the expected
latency to reach them, and how a caller distinguishes "still running" from "finished with
no result."

---

## Where documentation lives

```
<service>/
├── docs/
│   └── api/
│       ├── openapi.yaml       # generated; committed
│       └── CHANGELOG.md       # human-written; one entry per contract change
└── README.md                  # links to both, plus the local docs UI URL
```

Conventions:

- **Commit the generated spec.** It makes contract changes visible in the diff, which is
  the single highest-value thing this guide asks for. A reviewer who sees
  `openapi.yaml` untouched in a PR that adds a field knows something is wrong.
- **Serve it in non-production environments.** Swagger UI, ReDoc, or Scalar at `/docs`,
  with the raw document at `/openapi.json`. Whether it is exposed in production depends on
  whether the API is public.
- **Regenerate in CI and fail on drift.** Run the generator, then `git diff --exit-code`
  on the spec path. This is the only mechanism that reliably keeps the file honest.

---

## Per-stack recipes

### Go + Gin — `cantata`, `legato`, `dolce`, `modulation`

Use [`swaggo/swag`](https://github.com/swaggo/swag) annotations on handlers, and keep
request and response structs in a package that is exported for generation.

```go
// ListConversations returns a page of conversations for an organization.
//
// @Summary      List conversations
// @Description  Returns conversations ordered by last activity, newest first.
// @Tags         conversations
// @Produce      json
// @Param        org_id  path   int     true   "Organization ID"
// @Param        cursor  query  string  false  "Opaque cursor from a previous page"
// @Param        limit   query  int     false  "Page size, 1-100"  default(20)
// @Success      200  {object}  ListConversationsResponse
// @Failure      400  {object}  ErrorResponse
// @Failure      403  {object}  ErrorResponse
// @Router       /orgs/{org_id}/conversations [get]
func (h *Handler) ListConversations(c *gin.Context) { ... }
```

Generate with `swag init --parseDependency --output docs/api`, and wire the check into CI.

### Django + DRF — `rubato`, `sforzando`, `ardito`

Use [`drf-spectacular`](https://drf-spectacular.readthedocs.io/). Serializers carry most of
the schema already; `@extend_schema` fills in what introspection cannot infer.

```python
@extend_schema(
    summary="Send a direct message",
    description="Queues a message for a single recipient. Delivery is asynchronous.",
    request=DirectMessageRequestSerializer,
    responses={
        202: SendAcceptedSerializer,
        400: ErrorSerializer,
        429: ErrorSerializer,
    },
    examples=[
        OpenApiExample(
            "LINE text message",
            value={"recipient_id": "U4af4980629...", "messages": [{"type": "text", "text": "Hi"}]},
            request_only=True,
        )
    ],
)
def post(self, request): ...
```

Generate with `./manage.py spectacular --file docs/api/openapi.yaml`.

### Django Ninja — `interlude`

The schema is derived from the Pydantic models and the `response` mapping, so the work is
keeping those honest and adding prose.

```python
@router.post(
    "/messages",
    response={202: SendAccepted, 400: ErrorOut},
    summary="Submit a batch send request",
    description="Accepts up to 500 recipients per request.",
)
def send_messages(request, payload: SendRequest): ...
```

### FastAPI — `bebop`, `Epistola`, `whatsapp-extension`, `wccs-extension`

Same shape. Declare every non-2xx response explicitly; FastAPI will not infer the ones your
exception handlers raise.

```python
@router.post(
    "/messages",
    status_code=202,
    response_model=SendAccepted,
    responses={400: {"model": ErrorOut}, 429: {"model": ErrorOut}},
    summary="Submit a batch send request",
)
async def send_messages(payload: SendRequest) -> SendAccepted: ...
```

Dump the schema in CI rather than relying on the live app:
`python -c "import json, app; print(json.dumps(app.api.openapi()))"`.

---

## Conventions

These are the defaults for **new** endpoints and new services. Existing services keep their
established style — a service that mixes two conventions is worse than one that is
consistently old-fashioned. When you deviate, say so in the endpoint description.

### Paths and naming

- Plural resource nouns: `/conversations`, `/messages`, `/members`.
- Nest only one level deep; beyond that, use a query parameter.
- Match the surrounding service's casing. MAAC's public API uses `snake_case` segments and
  Django's trailing slash (`/openapi/v1/direct_message/multicast/`); Go services generally
  do not use a trailing slash. Do not "fix" one to look like the other.
- Field names in JSON bodies are `snake_case` across all stacks, including the ones served
  to TypeScript frontends.

### Versioning and deprecation

- Version in the path: `/v1/`, `/v2/`. Versions coexist; `v2` shipping does not remove `v1`.
- Adding an optional field or a new endpoint is not a version bump. Removing a field,
  renaming one, tightening validation, or changing a status code is.
- Mark superseded operations `deprecated: true` in the spec, and say in the description
  what replaces them and when the endpoint is scheduled to go away.
- Announce a removal date, return it in the `Sunset` header, and record it in the service's
  API `CHANGELOG.md`.

### Errors

Recommended envelope for new services:

```json
{
  "error": {
    "code": "recipient_not_found",
    "message": "No recipient matches the supplied recipient_id.",
    "details": [{ "field": "recipient_id", "issue": "unknown" }]
  }
}
```

- `code` is stable, `snake_case`, and safe for callers to branch on. `message` is for
  humans and may change at any time — say so in the docs.
- Document the error codes as a table in the operation description or a shared `ErrorCode`
  enum in the spec. A list of bare HTTP statuses is not documentation.
- Never put customer PII, API keys, or internal hostnames in an error message.

### Pagination, filtering, sorting

- Prefer cursor pagination: `cursor` and `limit` in, `next_cursor` (null on the last page)
  out. Treat the cursor as opaque and say so.
- Offset pagination is acceptable for small, bounded collections. Document the maximum
  offset.
- Document the default and maximum `limit`, the default sort order, and every filter the
  endpoint accepts. Undocumented filters get used, then break.

### Authentication

- Customer-facing APIs use a per-organization API key. Document where the customer gets it,
  which header carries it, and what an invalid or revoked key returns.
- Internal service-to-service calls use the platform's JWT flow via `arioso`. Document the
  required scopes.
- Show the auth header in every example, with an obviously fake key.

### Rate limits

State the limit, the window, the scope it applies to (per key, per organization, per
endpoint), and what the caller gets when it trips — status code, error code, and whether a
`Retry-After` header is returned. Quotas that exist only in someone's head cause incidents.

### Asynchronous work and webhooks

A `202` is a promise, and the documentation has to explain how it is kept:

- Name the identifier returned by the accepting call and the endpoint that reads status
  from it.
- List every terminal state, including the failure ones, and what each means.
- For webhooks, document the full payload schema, the delivery guarantee (at-least-once
  means the receiver must be idempotent — say it), the retry schedule, and how the receiver
  verifies the signature. Include one complete sample payload per event type.

---

## Consumers

Our frontends generate clients from the spec — Zodios plus TanStack Query in `Grazioso`,
`Zeffiroso`, `Polifonia`, and `Vivace`. That has two consequences worth internalizing:

- **A sloppy schema becomes a sloppy client.** A response typed as a bare object produces
  `any` on the other side, and the frontend loses every guarantee the types were for.
  Model response bodies precisely, including nullability.
- **Publish the spec as a build artifact.** Frontend repos should consume a versioned,
  published document rather than a file copied by hand into the repo.

Before merging a breaking change, check who is generating against you and give them a
release ahead of the removal.

---

## Review checklist

Paste into the PR description for any change that touches an HTTP contract:

```markdown
- [ ] `docs/api/openapi.yaml` regenerated and committed
- [ ] Summary and description written for each new or changed operation
- [ ] All parameters carry type, required flag, and constraints
- [ ] Every reachable error status documented with its stable error code
- [ ] At least one realistic request/response example
- [ ] Rate limits, payload caps, and pagination bounds stated
- [ ] Breaking change? -> version bump, deprecation note, `CHANGELOG.md` entry,
      consumers notified
- [ ] Customer-facing? -> Help Center article filed or updated
```

---

## Customer-facing documentation

Public API changes are not done when the spec merges. They are done when the customer can
find them:

1. Merge the spec change with the code.
2. Add a `CHANGELOG.md` entry — date, endpoints affected, breaking or not, migration note.
3. File or update the Help Center article; the public API is documented there alongside the
   rest of MAAC and CAAC.
4. Tell the CSM team what changed, in customer language, before the release reaches them.

---

## Related

- [Awesome Crescendo Lab](https://github.com/chatbotgang/awesome-cresclab) — index of all
  repositories, with the owning team and tech stack for each service
- [OpenAPI Specification 3.1](https://spec.openapis.org/oas/latest.html)
