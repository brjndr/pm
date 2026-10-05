# Code review

Full-project review of the PM MVP: FastAPI backend, Next.js frontend, Docker packaging, tests, and documentation.

## Verification performed

| Check | Result |
|---|---|
| `pytest backend` from repo root (as documented in PLAN.md) | FAILS: `ModuleNotFoundError: No module named 'app'` (5 collection errors) |
| `pytest` from `backend/` | 20 passed, 3 skipped |
| `pytest tests/test_chat_api.py` in isolation | FAILS: `sqlite3.OperationalError: no such table: users`; also creates a 0-byte `backend/data/pm.db` in the working tree |
| `npm run test:unit` | Not run: `frontend/node_modules` is not installed |

Scope reviewed: `backend/app/**`, `backend/tests/**`, `frontend/src/**`, `frontend/tests/**`, `Dockerfile`, `.dockerignore`, `scripts/**`, `docs/**`, `AGENTS.md`, `CLAUDE.md`, `README.md`.

---

## Blockers

### 1. All data is destroyed on every container restart

`scripts/start-mac.sh`, `scripts/start-linux.sh`, and `scripts/start-windows.ps1` run `docker rm -f pm-app` followed by `docker run` with no volume mount. The SQLite file lives at `/app/backend/data/pm.db` in the container's writable layer, so every start wipes the board.

This contradicts PLAN Part 7 ("Kanban data persists across reloads") and the README.

Fix: add `-v pm-data:/app/backend/data` and `-e PM_DB_PATH=/app/backend/data/pm.db` to all three start scripts.

### 2. The documented test command is wrong and no pytest configuration exists

PLAN.md line 150 documents `PYTHONPATH=/absolute/path/to/repo pytest backend`. The `app` package lives at `backend/app`, so `backend` must be on the path, not the repo root. There is no `conftest.py`, `pytest.ini`, `pyproject.toml`, or `backend/tests/__init__.py`.

CLAUDE.md gets this right (run from `backend/`), but README documents no backend test command at all.

Fix: add `backend/tests/conftest.py` or `[tool.pytest.ini_options] pythonpath = ["."]` so the suite runs from the repo root.

### 3. Cross-test state leak

`test_ai_actions.py`, `test_board_api.py`, and `test_chat_structured_api.py` each assign `os.environ["PM_DB_PATH"]` inside `_make_client` with no teardown. `test_chat_api.py` uses a module-level `TestClient(app)` and never calls `init_db()`, so it passes only because an alphabetically earlier test happens to leave an initialized database behind. In isolation it fails, and it writes to the real `backend/data/pm.db`.

Fix: use `monkeypatch.setenv` plus a shared fixture, and drop the module-level client.

### 4. Column rename fires one PATCH per keystroke

`KanbanColumn.tsx:50` calls `onRename` on every `onChange`, which calls `page.tsx:101` and issues an unawaited `updateColumn` request per character. Requests can land out of order. Clearing the field sends `""`, which fails `ColumnUpdate`'s `min_length=1` with a 422 and surfaces a spurious board error.

Fix: hold local input state, commit on blur or Enter, and debounce.

### 5. The entire UI is replaced by a loading screen on every mutation

`refreshBoard` sets `isLoading`, and `page.tsx:290` gates the whole render on `isLoading || !board`. Adding, deleting, or moving a card swaps the board and the chat sidebar for "Loading your board" until the refetch resolves. Additionally, `handleAddCard` never applies an optimistic local insert, so PLAN Part 7's "add optimistic UI" claim does not hold.

Fix: gate only the initial load on `isLoading`; use a separate refetching flag for background updates.

### 6. Unhandled exceptions from malformed AI output return 500 instead of 502

`ai.py` calls `int(action.columnId)` and `int(action.cardId)` in four places with no guard. If the model returns `"col-2"` or `"Backlog"` instead of `"2"`, the result is an unhandled `ValueError` and a 500 with a stack trace. `StructuredChatOutput.model_validate(data)` raises `pydantic.ValidationError` with the same outcome. `main.py` registers no exception handler.

Fix: treat every model-output problem as a 502.

### 7. Static catch-all has a path traversal hole and shadows the API

`static.py:42`, route `/{full_path:path}`:

- `GET /api/does-not-exist` returns 200 with `index.html` instead of 404. The catch-all is registered after the API router but matches every unmatched path.
- `STATIC_DIR / full_path` has no containment check. A request for `/../../etc/passwd` resolves outside the static root and is returned by `FileResponse`.

Fix: resolve and assert `is_relative_to(STATIC_DIR)`, and exclude an `api` prefix from the catch-all.

---

## Significant

### 8. "Structured output" is prompt-only

`docs/ai-structured-output.json` defines a proper JSON Schema, but `call_openrouter` never sends `response_format`. Enforcement is entirely via the prose contract in `build_structured_messages`. The `parse_structured_output` fallback that scrapes from the first `{` to the last `}` hides malformed output rather than failing loudly.

Fix: send `response_format: { type: "json_schema", ... }` and delete the scraping.

### 9. AI action failures are silent

`apply_actions` uses `continue` for every unknown column or card id and returns `None`. The assistant replies that it made a change while nothing changed, and neither the model nor the UI is informed.

Fix: return applied and failed actions and surface them.

### 10. OpenRouter errors are undiagnosable, with no retry

`detail="OpenRouter returned an error"` discards the upstream body, status, and request id. There is no backoff for 429 or 5xx responses.

Fix: log the response body at minimum, and add retry with backoff.

### 11. Unbounded prompt growth

Every turn sends the full board JSON plus the entire unbounded history. The client never truncates `chatMessages`.

Fix: cap both the history length and the board payload.

### 12. No authentication whatsoever

`get_username` trusts the `X-User` header verbatim. The password check at `page.tsx:69` is client-side only; the backend never validates it, and `users.password_hash` is never written or read. Any caller can read and mutate any user's board with `curl -H "X-User: anyone"`.

Acceptable for a localhost MVP, but this needs an explicit note in the README before the app leaves localhost.

### 13. No CORS, despite `NEXT_PUBLIC_API_BASE`

`api.ts:31` reads `NEXT_PUBLIC_API_BASE`, which implies cross-origin use, but the app registers no CORS middleware. `next dev` on port 3000 talking to port 8000 will fail.

Fix: add CORS or remove the env var.

### 14. Double-prefixed IDs

`toBoardData` already produces `card.id === "card-5"`, then `KanbanCard` and `SortableContext` prepend `card-` again. Verified:

```
backend id 5       -> card-5
KanbanCard testid  -> card-card-5
column testid      -> column-col-1
```

The result is internally consistent so it works, but it is confusing and the Playwright selectors depend on the mangling.

Fix: prefix once at the dnd boundary rather than in two places.

### 15. Fragile Playwright selectors

`tests/kanban.spec.ts:12` filters columns with `input[aria-label="Column title"][value="Backlog"]`, an attribute selector matched against a live controlled-input property.

Fix: use `filter({ hasText })` or a `toHaveValue` assertion.

### 16. E2E tests mutate the real board with no reset

`tests/kanban.spec.ts` adds "Playwright card" and moves a seeded card to Review, with no seeding or teardown. Repeated runs accumulate state.

### 17. No coverage thresholds, despite the documentation claiming them

`vitest.config.ts` defines a coverage reporter but no `thresholds`, and `test:unit` does not pass `--coverage`. Untested code includes `lib/api.ts` (notably `toBoardData` and the id-prefix round trip), `KanbanColumn`, `NewCardForm`, `KanbanCard`, `findCardLocation`, and the prefix helpers.

---

## Minor

| # | Issue |
|---|---|
| 18 | `database.get_db` (database.py:78) is dead code, duplicated by `dependencies.get_db`. |
| 19 | `lib/kanban.ts` `initialData` duplicates `config.py` `INITIAL_COLUMNS`. Two sources of truth for the same seed data; `initialData` is now only used by unit tests. |
| 20 | `STATIC_DIR` is resolved at import time in both `main.py` and `static.py`, so `PM_STATIC_DIR` changes after import are ignored. This is why tests monkeypatch module globals. `main.py`'s explicit `/_next` and `/static` mounts are also redundant with the catch-all. |
| 21 | Doc drift: `frontend/AGENTS.md` still says "in-memory state only" and omits `ChatSidebar` and `api.ts`; `CLAUDE.md:58` describes `main.py` as a "single file containing all routes, database setup, and AI integration"; `docs/kanban-schema.json` declares `idx_columns_board_position` and `idx_cards_column_position` that `init_db` never creates; `docs/DB_MODEL.md` documents a `PRAGMA user_version` migration system that does not exist; `backend/AGENTS.md` lists `POST /api/columns/{id}` (POST is on `/api/columns`) and omits `test_chat_structured_api.py` and `test_schema.py`. |
| 22 | `pytest` and `pytest-cov` are installed into the runtime image. Consider a dev extra or a multi-stage build. |
| 23 | `start-windows.ps1` runs `docker run` in the foreground, so Playwright's `webServer` step blocks for the whole test run unless a container is already up. The Mac and Linux scripts have the same issue. |
| 24 | `apiFetch` sets `Content-Type: application/json` on GET requests, and the `status === 204` branch is dead code; no endpoint returns 204. |
| 25 | `page.tsx:150` `handleMoveCard` accepts `_overId` and ignores it. |
| 26 | Three separate error channels (`error`, `boardError`, `chatError`). `refreshBoard` clears `boardError` on entry, so a failed mutation's message is wiped by its own recovery refetch. |
| 27 | Accessibility: the Remove button sits inside the element carrying `{...listeners}`, so the entire card including the button is a drag handle. No `KeyboardSensor` is configured, so cards cannot be moved by keyboard. |
| 28 | `"No details yet."` is injected by the frontend (`page.tsx:120`) rather than the backend, so AI-created and API-created cards behave inconsistently. |
| 29 | No `typecheck` script. `tsconfig.json` is `strict: true` but nothing ever runs `tsc --noEmit`. |
| 30 | `.dockerignore` omits `.env`. Nothing leaks today because the Dockerfile copies `backend` and `frontend` explicitly, but it is one `COPY .` away from baking the API key into an image. |

---

## Suggested order of work

1. Mount a volume in the start scripts (#1). Everything else is moot without persistence.
2. Add `backend/tests/conftest.py` and pytest configuration so the suite runs from the repo root (#2, #3).
3. Fix the static catch-all: containment check plus exclude `/api` (#7).
4. Debounce column rename; separate initial-load from refetch loading (#4, #5).
5. Guard `int()` and `model_validate`, return 502, and report failed actions (#6, #9).
6. Send a real `response_format` and remove the JSON scraping (#8).
7. Bring the documentation back in sync (#17, #21).
