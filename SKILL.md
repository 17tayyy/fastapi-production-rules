# FastAPI Production Rules

Production-grade conventions for FastAPI projects. Follow these strictly when generating or reviewing code. Examples use SQLAlchemy + Celery but every principle applies regardless of ORM or task queue.

---

## Project Structure

```
app/
├── core/
│   ├── config.py           # Settings via pydantic-settings
│   ├── dependencies.py     # Type aliases: CurrentUser, CurrentAdmin, CurrentOwner
│   ├── constants.py        # Values that never change between environments
│   ├── db/database.py      # DB engine and session factory
│   ├── security/auth.py    # JWT, password hashing, magic links
│   ├── http/exceptions.py  # AppError base class and subclasses
│   ├── http/responses.py   # DataResponse, PaginatedResponse, MessageResponse
│   ├── logging/logger.py   # structlog setup
│   ├── limiter.py          # Rate limiter singleton
│   ├── storage.py          # Storage abstraction (local/S3)
│   └── mail/mailer.py      # Email sending
│
└── [domain]/               # tickets/, auth/, users/, billing/, etc.
    ├── router.py           # HTTP endpoints only
    ├── service.py          # Business logic — pure Python, no FastAPI imports
    ├── model.py            # SQLAlchemy models
    └── schemas.py          # Pydantic I/O schemas
```

---

## Rule 1 — Router → Service → Model

The router is an HTTP translator. It validates input, calls a service, and returns a response. Nothing else.

The service is pure Python. It contains all business logic and never imports FastAPI, HTTPException, Request, or Depends.

```python
# CORRECT — router.py
@router.post("/tickets")
async def create_ticket(
    data: TicketCreate,
    user: CurrentUser,
    db: AsyncSession = Depends(get_db),
) -> DataResponse[TicketRead]:
    ticket = await ticket_service.create_ticket(db, data, user)
    return DataResponse(data=TicketRead.model_validate(ticket))
```

```python
# CORRECT — service.py (no FastAPI imports)
async def create_ticket(db: AsyncSession, data: TicketCreate, user: User) -> Ticket:
    ticket = Ticket(
        title=data.title,
        organization_id=user.organization_id,
        created_by=user.id,
    )
    db.add(ticket)
    await db.commit()
    await db.refresh(ticket)
    logger.info("ticket.created", ticket_id=str(ticket.id))
    return ticket
```

```python
# WRONG — logic in router
@router.post("/tickets")
async def create_ticket(data: TicketCreate, user: CurrentUser, db: AsyncSession = Depends(get_db)):
    ticket = Ticket(title=data.title, organization_id=user.organization_id)
    db.add(ticket)
    await db.commit()  # DB logic belongs in service
    return {"id": str(ticket.id)}
```

**Never in a router:**
- Direct database access
- Business logic or conditionals
- Private helper functions (`_something`)
- Manual data mappings (put these in schemas)

---

## Rule 2 — Use Annotated for Dependencies

```python
# dependencies.py
CurrentUser = Annotated[User, Depends(get_current_user)]
CurrentAdmin = Annotated[User, Depends(require_admin)]
CurrentOwner = Annotated[User, Depends(require_owner)]
CurrentSubscription = Annotated[User, Depends(require_subscription)]
DbSession = Annotated[AsyncSession, Depends(get_db)]
```

```python
# CORRECT — clean, defined once, easy to change
async def create_ticket(data: TicketCreate, user: CurrentUser, db: DbSession):
    ...

# WRONG — verbose, repeated everywhere
async def create_ticket(data: TicketCreate, user: User = Depends(get_current_user), db: AsyncSession = Depends(get_db)):
    ...
```

`Annotated` attaches metadata FastAPI reads at startup for dependency injection. Define once, use everywhere. If the dependency changes, update it in one place.

---

## Rule 3 — Custom Exceptions, Never HTTPException in Services

Services raise domain exceptions. A global handler converts them to HTTP responses.

```python
# core/http/exceptions.py
class AppError(Exception):
    status_code: int = 400
    code: str = "bad_request"

    def __init__(self, message: str) -> None:
        self.message = message
        super().__init__(message)

class UnauthorizedError(AppError):
    status_code = 401
    code = "unauthorized"

class ForbiddenError(AppError):
    status_code = 403
    code = "forbidden"

class NotFoundError(AppError):
    status_code = 404
    code = "not_found"

class ConflictError(AppError):
    status_code = 409
    code = "conflict"

class RateLimitError(AppError):
    status_code = 429
    code = "rate_limit_exceeded"
```

```python
# main.py — register handler once
@app.exception_handler(AppError)
async def app_error_handler(request: Request, exc: AppError):
    return JSONResponse(
        status_code=exc.status_code,
        content={"error": {"code": exc.code, "message": exc.message}},
    )
```

```python
# service.py — raise domain exceptions
async def get_ticket(db: AsyncSession, ticket_id: UUID, org_id: UUID) -> Ticket:
    ticket = await db.get(Ticket, ticket_id)
    if not ticket or ticket.organization_id != org_id:
        raise NotFoundError("Ticket")
    return ticket
```

**Error response format — always consistent:**
```json
{ "error": { "code": "not_found", "message": "Ticket not found" } }
```

**Never expose internals:**
```python
# WRONG
raise AppError(f"SELECT failed on tickets WHERE id={ticket_id}")

# CORRECT
raise NotFoundError("Ticket")
```

---

## Rule 4 — Typed Responses

Define response wrappers and use them consistently across all endpoints.

```python
# core/http/responses.py
class DataResponse(BaseModel, Generic[T]):
    data: T

class PaginatedResponse(BaseModel, Generic[T]):
    data: list[T]
    total: int
    skip: int
    limit: int

class MessageResponse(BaseModel):
    message: str
```

```python
# Always typed, always consistent
@router.get("/tickets/{ticket_id}")
async def get_ticket(...) -> DataResponse[TicketRead]: ...

@router.get("/tickets")
async def list_tickets(...) -> PaginatedResponse[TicketRead]: ...

@router.delete("/tickets/{ticket_id}")
async def delete_ticket(...) -> MessageResponse: ...
```

---

## Rule 5 — Pydantic Schemas

```python
# Read schema — include model_config for ORM compatibility
class TicketRead(BaseModel):
    id: UUID
    title: str
    status: TicketStatus
    priority: TicketPriority
    created_at: datetime
    model_config = {"from_attributes": True}

# Create schema
class TicketCreate(BaseModel):
    title: str
    priority: TicketPriority = TicketPriority.MEDIUM
    description: str | None = None

# Update schema — all fields optional, use exclude_unset=True
class TicketUpdate(BaseModel):
    title: str | None = None
    status: TicketStatus | None = None
    priority: TicketPriority | None = None
```

**Never include sensitive fields in Read schemas:** `password_hash`, `token_hash`, internal IDs not meant for clients.

---

## Rule 6 — Pagination on Every List Endpoint

```python
@router.get("/tickets")
async def list_tickets(
    user: CurrentUser,
    db: DbSession,
    skip: int = Query(default=0, ge=0),
    limit: int = Query(default=20, ge=1, le=100),
) -> PaginatedResponse[TicketRead]:
    tickets, total = await ticket_service.list_tickets(
        db, org_id=user.organization_id, skip=skip, limit=limit
    )
    return PaginatedResponse(
        data=[TicketRead.model_validate(t) for t in tickets],
        total=total,
        skip=skip,
        limit=limit,
    )
```

```python
# service.py
async def list_tickets(
    db: AsyncSession, org_id: UUID, skip: int, limit: int
) -> tuple[list[Ticket], int]:
    query = select(Ticket).where(Ticket.organization_id == org_id)
    total = await db.scalar(select(func.count()).select_from(query.subquery()))
    tickets = (await db.execute(query.offset(skip).limit(limit))).scalars().all()
    return list(tickets), total
```

---

## Rule 7 — Multi-tenancy: Every Query Scoped by Organization

Every database query must include `organization_id`. No exceptions. A missing scope is an IDOR vulnerability.

```python
# CORRECT
result = await db.execute(
    select(Ticket).where(
        Ticket.id == ticket_id,
        Ticket.organization_id == user.organization_id,
    )
)

# WRONG — security bug, any user can access any ticket
result = await db.execute(select(Ticket).where(Ticket.id == ticket_id))
```

For stricter isolation requirements, consider per-tenant database schemas or separate databases. For most SaaS applications consistent query scoping is sufficient — if applied without exceptions.

---

## Rule 8 — Structured Logging with structlog

```python
# core/logging/logger.py
import structlog

def get_logger():
    return structlog.get_logger()
```

```python
# In any module
from app.core.logging.logger import get_logger
logger = get_logger()

# CORRECT — domain.action with key-value context
logger.info("ticket.created", ticket_id=str(ticket.id), org_id=str(org_id))
logger.warning("auth.login_failed", email=email, reason="invalid_password")
logger.error("billing.webhook_failed", event_type=event, error=str(e))

# WRONG — formatted strings, no structure
logger.info(f"Ticket {ticket.id} created")
print(f"Login failed for {email}")
```

**Event naming convention:** `domain.action` — `ticket.created`, `auth.login_failed`, `billing.payment_failed`

**Bind request context for traceability:**
```python
# middleware
structlog.contextvars.bind_contextvars(request_id=str(uuid.uuid4()))
```

Every log line in that request now carries `request_id` automatically.

**Never log:**
- Passwords or password hashes
- Raw tokens or secrets
- LLM response content
- File contents or binary data
- PII beyond what's strictly necessary

**Minimum events to log:**
- `auth.login_success`, `auth.login_failed`, `auth.magic_link_sent`
- `ticket.created`, `ticket.updated`, `ticket.deleted`
- `billing.checkout_created`, `billing.payment_failed`, `billing.subscription_activated`

---

## Rule 9 — Rate Limiting

Apply rate limits to all public or sensitive endpoints.

```python
from app.core.limiter import limiter

@router.post("/auth/login")
@limiter.limit("10/minute")
async def login(request: Request, data: LoginSchema) -> DataResponse[TokenResponse]:
    ...
```

**Minimum rate limits:**
- `POST /auth/login` — 10/min
- `POST /auth/register` — 5/min
- `POST /auth/magic-link` — 5/min
- `POST /organizations/invite` — 5/min
- `POST /ai/chat` — 20/min per user

---

## Rule 10 — Background Tasks

Use an async-native task queue (ARQ, Taskiq) for async stacks. Use Celery for sync stacks. Either way, the rules are the same.

```python
# Enqueue from service or router
await task_queue.enqueue("process_knowledge_file", node_id=str(node.id))

# Task definition
async def process_knowledge_file(node_id: str) -> None:
    logger.info("knowledge.indexing_started", node_id=node_id)
    # work here
    logger.info("knowledge.indexing_done", node_id=node_id)
```

**Rules:**
- Tasks must be **idempotent** — safe to retry if they fail
- Never enqueue a task from inside another task
- Always log start and end of every task
- Pass primitive types as arguments (str, int), not ORM objects

---

## Rule 11 — SQLAlchemy Models

Every model must have `created_at` and `updated_at`.

```python
class Ticket(Base):
    __tablename__ = "tickets"

    id: Mapped[UUID] = mapped_column(primary_key=True, default=uuid4)
    title: Mapped[str]
    status: Mapped[TicketStatus] = mapped_column(default=TicketStatus.OPEN)
    priority: Mapped[TicketPriority] = mapped_column(default=TicketPriority.MEDIUM)
    organization_id: Mapped[UUID] = mapped_column(index=True)
    created_by: Mapped[UUID]
    assigned_to: Mapped[UUID | None] = mapped_column(nullable=True)
    created_at: Mapped[datetime] = mapped_column(default=lambda: datetime.now(timezone.utc))
    updated_at: Mapped[datetime] = mapped_column(
        default=lambda: datetime.now(timezone.utc),
        onupdate=lambda: datetime.now(timezone.utc),
    )
```

Always index `organization_id`, `status`, and any field used in frequent filters.

---

## Rule 12 — Auth (JWT)

**JWT payload:**
```json
{ "sub": "user_id", "org": "organization_id", "role": "role" }
```

**Refresh tokens:**
- Store hashed (SHA-256) in the database
- Set TTL via DB-level expiry or a scheduled cleanup
- Rotate on every use
- Web clients: `httpOnly` cookie, never response body
- Mobile clients: response body

**Magic links:**
- Store SHA-256 hash with TTL
- Mark `used=True` after first use — never reusable
- Purpose field: `"register"` (30 min TTL) | `"invite"` (7 days TTL)

---

## Rule 13 — File Uploads

```python
ALLOWED_EXTENSIONS = {".jpg", ".jpeg", ".png", ".webp", ".pdf", ".docx"}
MAX_SIZE_MB = 10

async def validate_upload(file: UploadFile) -> None:
    ext = Path(file.filename).suffix.lower()
    if ext not in ALLOWED_EXTENSIONS:
        raise ForbiddenError(f"File type not allowed: {ext}")
    if file.size > MAX_SIZE_MB * 1024 * 1024:
        raise AppError(f"File too large, max {MAX_SIZE_MB}MB")
```

- Validate extension against an allowlist — never trust `Content-Type`
- Save with a UUID filename, never the original name (path traversal risk)
- Never serve files without authentication

---

## Rule 14 — Constants vs Config

```python
# constants.py — never changes between environments
ALLOWED_AVATAR_EXTENSIONS = {".jpg", ".jpeg", ".png", ".webp"}
MAX_AVATAR_SIZE_MB = 2
INVITE_TOKEN_EXPIRE_DAYS = 7
MAGIC_LINK_EXPIRE_MINUTES = 30

# config.py — environment-specific values and secrets
class Settings(BaseSettings):
    secret_key: SecretStr
    database_url: str
    redis_url: str
    stripe_secret_key: SecretStr
    smtp_host: str
```

---

## Rule 15 — Naming Conventions

| Element | Convention | Example |
|---|---|---|
| Files | `snake_case.py` | `ticket_service.py` |
| Classes | `PascalCase` | `TicketService` |
| Functions/variables | `snake_case` | `create_ticket` |
| Constants | `UPPER_SNAKE_CASE` | `MAX_FILE_SIZE_MB` |
| DB tables | `snake_case` plural | `tickets`, `ai_chat_threads` |
| Endpoints | `kebab-case` | `/api/v1/tickets/new` |
| Log events | `domain.action` | `ticket.created` |
| Task names | `snake_case` verb-noun | `process_knowledge_file` |

---

## Rule 16 — Async Rules

Never use synchronous I/O inside async functions.

```python
# WRONG — blocks the event loop
async def upload_file(file: UploadFile):
    content = file.read()

# CORRECT
async def upload_file(file: UploadFile):
    content = await file.read()

# WRONG — CPU-bound blocks the event loop
async def process(text: str):
    result = heavy_cpu_function(text)

# CORRECT
async def process(text: str):
    result = await asyncio.to_thread(heavy_cpu_function, text)
```

---

## Rule 17 — No Over-Engineering

**Do not use:**
- Repository pattern if you already have an ORM — the ORM is your abstraction layer. Adding a repository on top is writing your own ORM on top of an ORM. Only use it with raw SQL.
- Abstract base classes for services — services are not interchangeable at runtime. Plain functions are enough.
- Event buses or pub/sub systems — use a task queue (ARQ, Celery).
- DI containers — FastAPI `Depends()` is sufficient.
- Generic CRUD base classes — each domain has specific logic that doesn't generalize cleanly.

**Use:**
- Plain async functions in service files
- Direct ORM calls in services
- FastAPI `Depends()` with `Annotated` type aliases
- Task queue only when a job would block the request

---

## Rule 18 — Never Use print()

```python
# WRONG
print(f"Ticket created: {ticket.id}")

# CORRECT
logger.info("ticket.created", ticket_id=str(ticket.id))
```

---

## Security Checklist — Before Every Endpoint

- [ ] Query scoped by `organization_id`
- [ ] Sensitive fields excluded from response schema
- [ ] Rate limiting applied if public or sensitive
- [ ] Subscription check applied if behind paywall
- [ ] Errors raised as `AppError` subclasses
- [ ] No passwords, tokens, or PII in logs
- [ ] `updated_at` updated on mutations
- [ ] File uploads validated (extension + size)
- [ ] Background tasks are idempotent

---

## Complete CRUD Example

```python
# model.py
class Ticket(Base):
    __tablename__ = "tickets"
    id: Mapped[UUID] = mapped_column(primary_key=True, default=uuid4)
    title: Mapped[str]
    status: Mapped[TicketStatus] = mapped_column(default=TicketStatus.OPEN)
    priority: Mapped[TicketPriority] = mapped_column(default=TicketPriority.MEDIUM)
    organization_id: Mapped[UUID] = mapped_column(index=True)
    created_by: Mapped[UUID]
    created_at: Mapped[datetime] = mapped_column(default=lambda: datetime.now(timezone.utc))
    updated_at: Mapped[datetime] = mapped_column(
        default=lambda: datetime.now(timezone.utc),
        onupdate=lambda: datetime.now(timezone.utc),
    )

# schemas.py
class TicketCreate(BaseModel):
    title: str
    priority: TicketPriority = TicketPriority.MEDIUM
    description: str | None = None

class TicketUpdate(BaseModel):
    title: str | None = None
    status: TicketStatus | None = None
    priority: TicketPriority | None = None

class TicketRead(BaseModel):
    id: UUID
    title: str
    status: TicketStatus
    priority: TicketPriority
    created_at: datetime
    model_config = {"from_attributes": True}

# service.py
async def create_ticket(db: AsyncSession, data: TicketCreate, user: User) -> Ticket:
    ticket = Ticket(
        title=data.title,
        priority=data.priority,
        organization_id=user.organization_id,
        created_by=user.id,
    )
    db.add(ticket)
    await db.commit()
    await db.refresh(ticket)
    logger.info("ticket.created", ticket_id=str(ticket.id))
    return ticket

async def get_ticket(db: AsyncSession, ticket_id: UUID, org_id: UUID) -> Ticket:
    result = await db.execute(
        select(Ticket).where(Ticket.id == ticket_id, Ticket.organization_id == org_id)
    )
    ticket = result.scalar_one_or_none()
    if not ticket:
        raise NotFoundError("Ticket")
    return ticket

async def update_ticket(db: AsyncSession, ticket_id: UUID, org_id: UUID, data: TicketUpdate) -> Ticket:
    ticket = await get_ticket(db, ticket_id, org_id)
    update_data = data.model_dump(exclude_unset=True)
    for field, value in update_data.items():
        setattr(ticket, field, value)
    await db.commit()
    await db.refresh(ticket)
    logger.info("ticket.updated", ticket_id=str(ticket.id))
    return ticket

async def delete_ticket(db: AsyncSession, ticket_id: UUID, org_id: UUID) -> None:
    ticket = await get_ticket(db, ticket_id, org_id)
    await db.delete(ticket)
    await db.commit()
    logger.info("ticket.deleted", ticket_id=str(ticket.id))

async def list_tickets(db: AsyncSession, org_id: UUID, skip: int, limit: int) -> tuple[list[Ticket], int]:
    query = select(Ticket).where(Ticket.organization_id == org_id)
    total = await db.scalar(select(func.count()).select_from(query.subquery()))
    tickets = (await db.execute(query.offset(skip).limit(limit))).scalars().all()
    return list(tickets), total

# router.py
@router.post("/tickets")
async def create_ticket(data: TicketCreate, user: CurrentUser, db: DbSession) -> DataResponse[TicketRead]:
    ticket = await ticket_service.create_ticket(db, data, user)
    return DataResponse(data=TicketRead.model_validate(ticket))

@router.get("/tickets/{ticket_id}")
async def get_ticket(ticket_id: UUID, user: CurrentUser, db: DbSession) -> DataResponse[TicketRead]:
    ticket = await ticket_service.get_ticket(db, ticket_id, user.organization_id)
    return DataResponse(data=TicketRead.model_validate(ticket))

@router.patch("/tickets/{ticket_id}")
async def update_ticket(ticket_id: UUID, data: TicketUpdate, user: CurrentUser, db: DbSession) -> DataResponse[TicketRead]:
    ticket = await ticket_service.update_ticket(db, ticket_id, user.organization_id, data)
    return DataResponse(data=TicketRead.model_validate(ticket))

@router.delete("/tickets/{ticket_id}")
async def delete_ticket(ticket_id: UUID, user: CurrentUser, db: DbSession) -> MessageResponse:
    await ticket_service.delete_ticket(db, ticket_id, user.organization_id)
    return MessageResponse(message="Ticket deleted")

@router.get("/tickets")
async def list_tickets(
    user: CurrentUser,
    db: DbSession,
    skip: int = Query(default=0, ge=0),
    limit: int = Query(default=20, ge=1, le=100),
) -> PaginatedResponse[TicketRead]:
    tickets, total = await ticket_service.list_tickets(db, user.organization_id, skip, limit)
    return PaginatedResponse(
        data=[TicketRead.model_validate(t) for t in tickets],
        total=total,
        skip=skip,
        limit=limit,
    )
```
