# FastAPI Production Rules

A set of conventions, patterns, and hard rules for building production-grade FastAPI projects.

## What this is

A single `SKILL.md` file you can drop into any LLM context (Claude, Cursor, Copilot, GPT) to generate FastAPI code that follows production standards — correct separation of concerns, structured logging, consistent error handling, multi-tenancy scoping, and more.

These aren't theoretical best practices. They come from building a B2B SaaS in production.

## How to use

**With any LLM:** paste or attach `SKILL.md` at the start of your conversation.

**With Cursor:** add `SKILL.md` to your project and reference it in `.cursorrules`.

**With Claude:** attach the file or paste its contents before asking for code.

## What it covers

- Project structure
- Router → Service → Model separation
- Dependency injection with `Annotated`
- Custom exceptions and global error handling
- Typed response wrappers
- Pagination
- Multi-tenancy and IDOR prevention
- Structured logging with structlog
- Rate limiting
- Background tasks
- Auth (JWT, refresh tokens, magic links)
- File upload validation
- Naming conventions
- Security checklist

## Stack

Examples use SQLAlchemy + Celery but every principle applies to any ORM or task queue.

## Related

[How I structure FastAPI projects in production — and why](https://bytay.dev/blog/how-i-structure-fastapi-projects) — the post explaining the reasoning behind these rules.

[The Python Toolchain I Use in Production: uv, ruff, and ty](https://bytay.dev/blog/python-toolchain-uv-ruff-ty/) - another post explaining why i use all the astral stack (uv, ruff and ty)
