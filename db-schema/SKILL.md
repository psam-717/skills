---
name: db-schema
description: >
  Designs, extends, and reviews database schemas for Python backend projects.
  Use this skill whenever the user wants to add a new table, add columns to an
  existing table, design relationships, create indexes, write a migration, or
  asks "how should I model this in the DB", "what columns do I need", "add a
  field to X", "create a table for Y", or "write a migration for Z". Also
  trigger when reviewing model files for correctness, consistency, or best
  practices. Works across different projects and tech stacks — always reads the
  project's existing models first to match its conventions before writing
  anything new.
---

# DB Schema Skill

Designs and extends database models for Python backend projects.
Adapts to each project's existing stack, style, and conventions.

---

## Step 1 — Discover the Project's Stack

Before designing anything, read the project's existing model files.
Look for them at common locations:

```
app/models/          # per-domain files or a single models.py
app/db/models/
src/models/
models.py
```

From the existing models, identify:

| Question | What to look for |
|---|---|
| ORM | SQLAlchemy? Tortoise? raw SQL? Which style — `Column()` or `Mapped[]`? |
| Database | PostgreSQL? SQLite? MySQL? |
| Primary keys | UUID? Integer auto-increment? Something else? |
| Migration tool | Alembic? Aerich? Manual? |
| Auth pattern | JWT tokens table? Session table? API keys? |
| Timestamp style | `server_default=func.now()`? Python-side `datetime.utcnow`? |
| Encryption | Any `LargeBinary` + IV pairs? Any hashed secrets? |
| Soft deletes | `deleted_at` / `is_deleted` columns? Or hard deletes? |
| Naming conventions | Table names plural/singular? Index naming pattern? |

**Match the project's existing style exactly.** If the project uses `Column()` style, don't introduce `Mapped[]`. If it uses integer PKs, don't switch to UUIDs. Consistency beats perfection.

---

## Step 2 — Map the Existing Schema

Before adding anything new, summarise what already exists:

- List all tables and their purpose (one line each)
- Note the primary key strategy
- Note all FK relationships and cascade rules
- Note any patterns used consistently (e.g. soft deletes, encryption, audit fields)

If the user has already shared their models, extract this from what they provided.
If not, ask them to share the relevant model file(s) or describe what's already there.

---

## Step 3 — Design the New Schema

### Clarifying questions (ask only what's genuinely unclear)

- What is this table/column for? What problem does it solve?
- Is this record owned by a user or another entity? (→ FK + cascade)
- Can records be updated after creation, or are they immutable? (→ timestamps)
- Will this table grow large? What's the primary lookup pattern? (→ index design)
- Does it store sensitive data? (→ encryption or hashing strategy)
- Should deletions be hard or soft?
- Is there a uniqueness constraint — globally, or scoped to a parent record?

### Design output

Produce the full model class(es) matching the project's existing style, then:

1. **Show the model code** — complete, copy-pasteable
2. **Explain key decisions** — why each column type, index, or constraint was chosen
3. **List what else needs updating** — reciprocal relationships, `__init__.py`, registration
4. **Show the migration** — or the command to generate it

---

## Universal Design Principles

These apply across all projects regardless of stack. Apply them unless the
existing codebase has an established pattern that contradicts them.

### Primary Keys
- UUIDs preferred for distributed systems, external-facing APIs, or security-sensitive data (no enumerable IDs)
- Integer auto-increment fine for internal/admin tables or simple projects
- Be consistent — don't mix strategies within a project

### Foreign Keys
- Always define `ondelete` behaviour explicitly: `CASCADE`, `SET NULL`, or `RESTRICT`
- `CASCADE` — child records are meaningless without the parent (e.g. user's posts)
- `SET NULL` — child records can exist orphaned (e.g. posts by a deleted author, if you want to keep them)
- `RESTRICT` — prevent deletion if children exist (e.g. a category with active products)

### Timestamps
- Use timezone-aware datetimes — never naive
- Prefer server-side defaults (`server_default=func.now()`) over Python-side (`datetime.utcnow`) — avoids clock skew
- Mutable rows: include both `created_at` and `updated_at`
- Immutable rows (tokens, audit logs, events): only `created_at` — no `updated_at`

### Sensitive Data
Two safe patterns — choose based on whether the raw value ever needs to be recovered:

| Need | Pattern |
|---|---|
| Never need raw value (passwords, tokens) | Hash with bcrypt/argon2/SHA-256; store only the hash |
| Need to recover raw value (encrypted secrets) | Encrypt with AES-256-GCM; store `encrypted_{thing}` + `iv` (12 bytes) |

Never store raw secrets, passwords, or tokens. Never store IVs without their ciphertext.

### Indexes
- Index every FK column (speeds up joins and cascade lookups)
- Index every column used in `WHERE` filters in frequent queries
- Composite indexes: column order matters — put the highest-cardinality or most-filtered column first
- Unique constraints: scope them correctly — `UNIQUE(user_id, name)` not just `UNIQUE(name)` if uniqueness is per-user
- Partial indexes: use `WHERE` conditions for flag-based tables (e.g. `WHERE revoked = false`) — keeps the index small and fast

### Boolean State Flags
- For single-use or revocable records (tokens, codes, invitations): use a `used`/`revoked` Boolean flag
- Mark as used/revoked on consumption — never delete the row; keep the audit trail
- Always pair with a partial index on the `false` case for fast active-record lookups

### Naming
Apply the project's existing conventions. If starting fresh, use these defaults:

| Thing | Convention |
|---|---|
| Table names | `snake_case`, plural (`user_sessions`, `audit_logs`) |
| Column names | `snake_case` |
| FK columns | `{parent_singular}_id` (`user_id`, `org_id`) |
| Index names | `ix_{table}_{columns}` (`ix_sessions_user_id`) |
| Encrypted columns | `encrypted_{thing}` paired with `{thing}_iv` or just `iv` |
| Hash columns | `{thing}_hash` (`token_hash`, `code_hash`) |
| Soft delete | `deleted_at` (nullable DateTime) preferred over `is_deleted` Boolean |

### String Lengths (PostgreSQL / SQLAlchemy)

| Use | Type |
|---|---|
| Names, labels, slugs | `String(255)` |
| Short codes, usernames | `String(64)` |
| Hex hashes (SHA-256) | `String(64)` |
| Emails, URLs | `String(255)` |
| Unbounded text | `Text` |
| Binary data | `LargeBinary` (or sized: `LargeBinary(12)` for IVs) |

---

## SQLAlchemy Reference Patterns

### Classic `Column()` style (SQLAlchemy < 2.0 or legacy projects)

```python
import uuid
from sqlalchemy import Boolean, Column, DateTime, ForeignKey, Index, String, Text
from sqlalchemy.dialects.postgresql import UUID
from sqlalchemy.orm import relationship
from sqlalchemy.sql import func

class Example(Base):
    __tablename__ = "examples"

    id = Column(UUID(as_uuid=True), primary_key=True, default=uuid.uuid4)
    owner_id = Column(UUID(as_uuid=True), ForeignKey("owners.id", ondelete="CASCADE"), nullable=False)
    name = Column(String(255), nullable=False)
    notes = Column(Text, nullable=True)
    is_active = Column(Boolean, nullable=False, default=True)

    created_at = Column(DateTime(timezone=True), server_default=func.now(), nullable=False)
    updated_at = Column(DateTime(timezone=True), server_default=func.now(), onupdate=func.now(), nullable=False)

    owner = relationship("Owner", back_populates="examples")

    __table_args__ = (
        Index("ix_examples_owner_name", "owner_id", "name", unique=True),
    )

    def __repr__(self) -> str:
        return f"<Example id={self.id} name={self.name!r}>"
```

### Modern `Mapped[]` style (SQLAlchemy 2.0+)

```python
import uuid
from datetime import datetime
from sqlalchemy import ForeignKey, Index, String, Text, func
from sqlalchemy.orm import Mapped, mapped_column, relationship

class Example(Base):
    __tablename__ = "examples"

    id: Mapped[uuid.UUID] = mapped_column(primary_key=True, default=uuid.uuid4)
    owner_id: Mapped[uuid.UUID] = mapped_column(ForeignKey("owners.id", ondelete="CASCADE"))
    name: Mapped[str] = mapped_column(String(255))
    notes: Mapped[str | None] = mapped_column(Text)
    is_active: Mapped[bool] = mapped_column(default=True)

    created_at: Mapped[datetime] = mapped_column(server_default=func.now())
    updated_at: Mapped[datetime] = mapped_column(server_default=func.now(), onupdate=func.now())

    owner: Mapped["Owner"] = relationship(back_populates="examples")

    __table_args__ = (
        Index("ix_examples_owner_name", "owner_id", "name", unique=True),
    )

    def __repr__(self) -> str:
        return f"<Example id={self.id} name={self.name!r}>"
```

---

## Alembic Migration Checklist

When using Alembic:

```bash
# Generate
alembic revision --autogenerate -m "add {description}"

# Review the generated file — always check:
# - Correct table/column names
# - Partial indexes (WHERE conditions) — autogenerate often misses these, add manually
# - ondelete cascade rules are present

# Apply and verify
alembic upgrade head
alembic check   # should report "No new upgrade operations detected"
```

Manually add partial indexes missed by autogenerate:
```python
# In upgrade():
op.create_index("ix_examples_active", "examples", ["owner_id"],
    postgresql_where=sa.text("is_active = true"))

# In downgrade():
op.drop_index("ix_examples_active", table_name="examples")
```

---

## Output Checklist

Before delivering schema output, verify:

- [ ] Matches the project's existing ORM style (Column vs Mapped, PK type, timestamp style)
- [ ] FK `ondelete` behaviour is explicitly set
- [ ] Every FK column has an index
- [ ] Timestamps are timezone-aware and server-side
- [ ] Sensitive data uses hash-or-encrypt pattern — no raw secrets
- [ ] Composite unique indexes are scoped correctly (e.g. per-user, not globally)
- [ ] Partial indexes added for any boolean flag columns
- [ ] `__repr__` includes the most useful identifying fields
- [ ] Reciprocal `relationship()` shown on the parent model
- [ ] Migration command or file included
- [ ] Any partial indexes that autogenerate would miss are called out explicitly