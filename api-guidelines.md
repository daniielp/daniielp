# CRUD rules (incl. batch/bulk + rate limits)

## Global

* Prefer synchronous responses; use `202 Accepted` when work is asynchronous.
* For any **batch** (many requests) or **bulk** (many resources in one request) endpoint, **always** respond with **`207 Multi-Status`** and a per-item result body—**even if all parts succeed, all fail, or run asynchronously**. Clients **must** inspect each item’s result.
* Per-item result object:

  ```json
  {
    "id": "client-supplied-or-server-id",
    "status": "EXTENSIBLE_STATUS_STRING",
    "description": "Optional human-readable context/failure info"
  }
  ```

  Required fields: `id`, `status`. `description` is optional.

---

## CREATE (POST)

* **Single item**

  * `201 Created` with representation; include `Location` to new resource.
  * `202 Accepted` if async creation started (include how to track).
  * `409 Conflict` if duplicate/constraint conflict.
* **Batch/Bulk create**

  * `207 Multi-Status` with `items[]` results.
  * Each item `status` should reflect outcome, e.g., `CREATED`, `ACCEPTED`, `CONFLICT`, `FAILED_VALIDATION`, etc.
  * Do **not** collapse a fully successful batch into `200/201`.

## READ (GET)

* **Single item / collection**

  * `200 OK` with representation.
  * `404 Not Found` if missing.
* **Batch reads**

  * If supported, return `207 Multi-Status` with per-item `FOUND` / `NOT_FOUND` / `ERROR` statuses.

## UPDATE (PUT/PATCH)

* **Single item**

  * `200 OK` with updated representation, or `204 No Content` if no body.
  * `202 Accepted` if async update started.
  * `404 Not Found` if target missing; `409 Conflict` on version/ETag mismatch (if using concurrency control).
* **Batch/Bulk update**

  * `207 Multi-Status` with per-item statuses like `UPDATED`, `NO_CHANGE`, `ACCEPTED`, `NOT_FOUND`, `CONFLICT`, `FAILED_VALIDATION`.

## DELETE (DELETE)

* **Single item**

  * `204 No Content` on success.
  * `202 Accepted` if async deletion started.
  * `404 Not Found` if missing (idempotent deletes may also return `204`).
* **Batch/Bulk delete**

  * `207 Multi-Status` with per-item statuses like `DELETED`, `NOT_FOUND`, `ACCEPTED`, `FAILED`.

---

## 207 Multi-Status response shape (batch/bulk)

```json
{
  "items": [
    {
      "id": "string",
      "status": "EXTENSIBLE_STATUS_STRING",
      "description": "Optional human-readable detail"
    }
  ]
}
```

**Client rule:** Never assume overall success/failure from the HTTP code for batch/bulk. **Always** iterate `items` and act per item.

**Server rule:** Include actionable `description` when an item fails (validation errors, missing dependencies, etc.). Use a stable, documented set of status strings.

---

## Rate limiting (all endpoints)

* When a client exceeds allowed rate, return **`429 Too Many Requests`**.
* Include **either**:

  1. `Retry-After: <seconds>` (prefer seconds), **or**
  2. all three **X-RateLimit** headers on responses (ideally every response, not only 429):

     * `X-RateLimit-Limit: <max requests in window>`
     * `X-RateLimit-Remaining: <requests left in current window>`
     * `X-RateLimit-Reset: <seconds until window resets>` (note: **relative seconds**, not epoch time)
* Pick `Retry-After` for simpler throttling without per-tenant state; pick `X-RateLimit-*` when you track per-account windows and want clients to self-throttle proactively.

---

## Quick examples

**Bulk create (all succeed, but still 207):**

```
HTTP/1.1 207 Multi-Status
Content-Type: application/json

{
  "items": [
    {"id": "a1", "status": "CREATED"},
    {"id": "a2", "status": "CREATED"}
  ]
}
```

**Batch update (mixed outcomes):**

```
HTTP/1.1 207 Multi-Status
Content-Type: application/json

{
  "items": [
    {"id": "u1", "status": "UPDATED"},
    {"id": "u2", "status": "NOT_FOUND", "description": "No resource with id u2"}
  ]
}
```

**Rate limit exceeded:**

```
HTTP/1.1 429 Too Many Requests
Retry-After: 30
```

*or*

```
HTTP/1.1 429 Too Many Requests
X-RateLimit-Limit: 120
X-RateLimit-Remaining: 0
X-RateLimit-Reset: 25
```
