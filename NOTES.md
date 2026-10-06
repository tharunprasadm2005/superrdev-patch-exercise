# NOTES

## Summary of changes
1. **SQL operator precedence** (`TaskRepository.java`, mirrored in `db/queries/search_tasks.sql` and `db/oracle/task_search_package.sql`): mixed `AND`/`OR` without parentheses, so the status filter only applied to description matches and archived tasks leaked into results (`?status=DONE` returned all 49 rows). Added parentheses around the `OR` group.
2. **Artificial latency** (`TaskController.java`): removed `Thread.sleep((10 - q.length()) * 100)` — empty queries took ~1s and tied up Tomcat threads.
3. **Pagination validation** (`TaskController.java`): `page=0` or negative `pageSize` caused `IndexOutOfBoundsException` (HTTP 500). Now clamped to `page>=1`, `pageSize` 1–100.
4. **Invalid status** (`TaskController.java`): `TaskStatus.valueOf` threw on unknown values (HTTP 500); now returns 400 with a clear message.
5. **Frontend request handling** (`useTasks.js`, `api.js`): no cancellation (stale responses could overwrite newer ones) and `loading` was never cleared on error (spinner stuck forever). Added `AbortController`, error reset, and `finally` cleanup.
6. **Debounce** (new `hooks/useDebounce.js`, wired in `App.jsx`): search fires 300 ms after the last keystroke instead of on every keystroke.
7. **Page reset** (`App.jsx`): changing query/status now returns to page 1 (previously could show an empty page).

## What I chose not to change
`System.out.println` logging (would move to SLF4J with more time), `@CrossOrigin` (redundant behind the Vite proxy, harmless), H2 console, entity `equals`/`hashCode`, Oracle ROWNUM pagination (correct pre-12c pattern).

## Biggest remaining risk
No authentication/authorization — anyone who can reach the API can read every task. Also `LOWER(col) LIKE` cannot use indexes, so search full-scans at scale, and in-memory H2 loses all data on restart.

## Tools used
Claude via opencode for code review, edits, and the verification battery (curl checks before/after); I reviewed every diff. Installed Temurin JDK 17 to run the backend.
