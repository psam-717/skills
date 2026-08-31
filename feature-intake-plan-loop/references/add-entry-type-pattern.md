# Adding a New Encrypted Entry Type to psamvault

This pattern emerged from adding Secure Notes to psamvault — a new encrypted entry type alongside existing site credentials and API keys. The same pattern can be reused for future entry types (TOTP, secure files, etc.).

## Architecture

psamvault has two repos:
- **psam_vault_backend** — FastAPI server (PostgreSQL, SQLAlchemy async, Alembic)
- **psamvault-cli** — Typer CLI client

Each entry type has a **paired implementation** across both repos.

---

## Backend Pattern (psam_vault_backend)

For each new entry type, create 6 pieces following the existing type's pattern exactly:

### 1. Model (`app/models/models.py`)
- Add `NoteEntry` class (or whatever name) inheriting `Base`
- Fields: `id` (UUID PK), `user_id` (FK → users.id, CASCADE), title/name (String, unique-per-user), plaintext metadata fields, `encrypted_blob` (LargeBinary), `iv` (LargeBinary[12]), timestamps
- Composite unique index on `(user_id, title)` — same pattern as `ApiKeyEntry`
- Add `relationship("NoteEntry", back_populates="owner")` on `User` model

### 2. Schemas (`app/schemas/note_schema.py`)
Mirror `api_key_schema.py` exactly:
- `Add*Request` — name/title fields + encrypted_blob + iv
- `Update*Request` — same fields all optional
- `*Response` — all fields including blob + iv
- `*ListItem` — list-safe version (no blob)
- `*ListResponse` — `entries: list[*ListItem]` + `total: int`
- `Delete*Response` — `detail: str`
- `*ExportResponse` — full entry for export

### 3. CRUD Controller (`app/controller/note_crud.py`)
Mirror `api_key_crud.py` exactly, replacing `ApiKeyEntry` → `NoteEntry`:
- `_normalise_title(name)` — strip + lowercase
- `_entry_to_response(entry)` — ORM → schema
- `_get_entry_for_user(db, name, user_id)` — single lookup
- `_decode_hex_fields(blob, iv)` — shared helper, returns `(bytes, bytes)` or 422
- `add_*` — check conflict, decode fields, create, commit, refresh
- `get_*` — lookup or 404
- `list_*` — count + paginated select, return `*ListResponse`
- `update_*` — lookup, blob+IV paired check, optional fields, update timestamps
- `delete_*` — lookup, delete, commit
- `export_*` — all entries with full blobs

### 4. Routes (`app/routes/notes.py`)
Mirror `api_key.py`:
- `APIRouter(prefix="/notes", tags=["notes"])`
- `get_authenticated_user` — same bearer token + `get_current_user` pattern
- POST (201) → add
- GET `` → list with limit/offset query params
- GET `/{title}` → get
- PUT `/{title}` → update
- DELETE `/{title}` → delete
- GET `/export/all` → export

### 5. Wire in `app/main.py`
- Import router from `.routes.notes`
- `app.include_router(notes_router)`

### 6. Migration  
- Create Alembic revision using the **actual current head** as `down_revision`  
  ```bash  
  # Check what the latest head actually is:  
  for f in migrations/versions/*.py; do  
    echo "$(basename $f): $(grep 'revision:' $f | head -1)"  
  done  
  # The head is whatever revision is NOT referenced as down_revision by any other file.  
  ```  
- `op.create_table(...)` with exact column types matching the model  
- `op.create_index(...)` for composite unique index  
- Test: `alembic upgrade head` (requires local DB)  

**Critical migration pitfall:** If the migration is parented to the wrong revision (a non-head ancestor), Alembic creates a fork with multiple heads and refuses to run `alembic upgrade head` with:  
```
FAILED: Multiple head revisions are present for given argument 'head'
```
Always verify the `down_revision` points to the one true head before publishing. If you already merged and deployed with the wrong parent, create a follow-up PR that changes just the `down_revision` (and `Revises` comment) to point to the actual head.

---

## Client Pattern (psamvault-cli)

### 1. Crypto helpers (`crypto.py`)
- `encrypt_note(key, content)` → same `AESGCM` pattern as `encrypt_api_key`
- `decrypt_note(key, encrypted_blob, iv)` → same pattern as `decrypt_api_key`
- Payload: `{"content": content, "category": category}` for single-blob encrypt

### 2. API client (`api_client.py`)
Mirror the API key functions:
- `add_note_entry(...)` → POST /notes
- `get_note_entry(...)` → GET /notes/{title}
- `list_note_entries(...)` → GET /notes
- `update_note_entry(...)` → PUT /notes/{title}
- `delete_note_entry(...)` → DELETE /notes/{title}
- Each follows the same `_call(token) / _refresh_and_retry` pattern

### 3. Command file (`command/note_commands.py`)
Each command mirrors `api_key_commands.py`:
- `note-add` — title, --content, --category, encrypt then POST
- `note-get` — fetch, decrypt, display
- `note-list` — fetch list items, display table
- `note-delete` — confirm then DELETE
- `note-update` — fetch, merge, re-encrypt, PUT. Handles three modes: re-encrypt on content change, re-encrypt on category change (preserving content), or just rename/add metadata.
- Forbidden chars validation on title (same as `_validate_entry_name`)

### 4. Wire in `main.py`
Two imports needed:
```python
from command.note_commands import app as note_app              # for grouped
from command.note_commands import note_add, note_get, ...      # for short aliases
```
- `app.add_typer(note_app, name="note", ...)` for grouped help
- Short-command aliases: `app.command("note-add")(note_add)`, etc.
- Add a line in the `help=` message describing the note subcommand

### 5. Update `list` to include the new type
- In `list_entries()`, add a third fetch + display section after sites and API keys
- Fetch notes: `api_client.list_note_entries(...)`, reload session between each fetch to avoid stale tokens
- Display with a `SECURE NOTES` header section matching the same table format
- Include notes in the `list` docstring and empty-state message

### 6. Update `search` to include the new type
- Add `_search_notes()` helper that mirrors `_search_credentials()` — calls `export_notes()`, decrypts each via `decrypt_note`, filters by title/content/category
- Update `search()` to fetch and search notes alongside sites and API keys (with session reload between each)
- Update the `total` count to include `len(note_results)`
- Add a `SECURE NOTES` display section in the results

### 7. Update `export` and `import` to include the new type
- **Export:** Fetch notes via `api_client.export_notes()`, decrypt via `decrypt_note` (import `decrypt_note` in export_command.py), append to `export_data["notes"]`
- **Import:** Read `export_data.get("notes", [])`, re-encrypt locally via `encrypt_note` (import `encrypt_note` in import_command.py), import via `api_client.add_note_entry()`
- Update summary counters and skip warnings in both commands

### 8. Tests — mirror existing test patterns for the new type

### Session reload pattern
When fetching data from multiple endpoints in the same function (list, search, export), reload the session between each API call. Each call may rotate tokens, and using stale tokens for the next call will fail:
```python
session = load_session()
# ... first API call ...
session = load_session()  # tokens may have rotated
# ... second API call ...
session = load_session()  # rotate again
# ... third API call ...
```
