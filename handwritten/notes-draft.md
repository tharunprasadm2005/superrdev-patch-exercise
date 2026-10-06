# Handwritten notes — DRAFT TEXT

Handwrite each section on paper, photograph/scan it, save the photos into this
`handwritten/` folder, and DELETE this draft file before submitting.
(Submissions without handwritten explanations are considered incomplete.)

---

## Bug 1 — SQL operator precedence (highest value)

**Where:** `backend/src/main/java/com/internal/tasktracker/TaskRepository.java`,
the `@Query` in `searchTasks`; same bug mirrored in `db/queries/search_tasks.sql`
and `db/oracle/task_search_package.sql` (both the COUNT and the paginated query).

**How discovered:** reading the native query I saw `AND` and `OR` mixed without
parentheses. Confirmed with curl: `GET /api/tasks?status=DONE` returned all 49
rows, and the default list included the 2 archived "legacy" tasks.

**Root cause:** SQL `AND` binds tighter than `OR`, so the query parsed as
`(archived=FALSE AND title LIKE term) OR (description LIKE term AND status-filter)`.
The status filter was ignored for title matches, and archived tasks leaked in
through the description branch (which never checked `archived`).

**Fix:** wrapped the title/description `OR` in parentheses and applied the status
predicate to the whole `WHERE`. Mirrored the fix in both SQL reference files.
After: `?status=DONE` -> 5 rows, default list -> 47 rows (no archived).

## Bug 2 — Artificial latency

**Where:** `TaskController.java`, the "Query complexity estimation" block.

**How discovered:** curl timing showed ~1.0s for empty/short queries; the code
contained `Thread.sleep((10 - q.length()) * 100)`.

**Root cause:** a deliberate sleep that scales with query length; it blocked a
Tomcat thread per request and punished exactly the short searches the UI fires
on every keystroke.

**Fix:** deleted the whole block. Empty-query latency dropped from ~1000ms to ~40ms.

## Bug 3 — Pagination validation

**Where:** `TaskController.java`, the `subList` pagination.

**How discovered:** tried `GET /api/tasks?page=0` -> HTTP 500
(`IndexOutOfBoundsException` from a negative subList index).

**Root cause:** `page`/`pageSize` were used raw, with no bounds.

**Fix:** clamp `page` to >= 1 and `pageSize` to 1..100; the response reports the
clamped values so clients can see what was applied.

## Bug 4 — Invalid status returns 500

**Where:** `TaskController.java`, `TaskStatus.valueOf(status.toUpperCase())`.

**How discovered:** `GET /api/tasks?status=FOO` -> HTTP 500.

**Root cause:** `Enum.valueOf` throws `IllegalArgumentException` for unknown values,
which Spring turns into a 500.

**Fix:** catch it and return 400 Bad Request with body
"Invalid status 'FOO'. Must be one of: OPEN, IN_PROGRESS, DONE." — a 4xx is the
correct contract for bad client input.

## Bug 5 — Frontend race condition + stuck spinner

**Where:** `frontend/src/hooks/useTasks.js` and `frontend/src/api.js`.

**How discovered:** code review — the `.catch` never called `setLoading(false)`
(spinner stuck forever after any error), and with no request cancellation a slow
earlier response could overwrite a faster newer one (likely while the server had
the 1s sleep, since shorter queries were slower).

**Root cause:** no `AbortController`; loading/error state not reset per request.

**Fix:** one `AbortController` per effect; cleanup aborts in-flight requests on
re-run/unmount; `setError(null)` at start; `setLoading(false)` in `finally`;
`AbortError` is ignored so cancelled requests don't show as errors.

## Bug 6 — No search debounce

**Where:** `SearchBar` -> `App.jsx` -> `useTasks`.

**How discovered:** every keystroke fired a network request (visible in the README
smoke test and the browser network tab).

**Root cause:** input `onChange` drove query state directly, and the fetch effect
depended on it.

**Fix:** new `useDebounce` hook (300 ms); `App` passes the debounced value to
`useTasks`, so one request fires per pause instead of per keystroke.

## Bug 7 — Page not reset on filter change

**Where:** `App.jsx`.

**How discovered:** reasoning — on page 3, typing a search keeps `page=3`, which
can show an empty page even though matching results exist on page 1.

**Root cause:** `page` state was independent of `query`/`status`.

**Fix:** `useEffect` resets `page` to 1 whenever the debounced query or the
status filter changes.
