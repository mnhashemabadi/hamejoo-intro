# Engineering note

## Overview

Hamejoo searches listings and shows the matches in one place. Listing detail stays on the original source.

## Stack

Declared in `apps/frontend/package.json`:

- Next.js ^14.2.35
- React 18.3.1
- TypeScript 5.7.3
- Tailwind CSS 3.4.17

Declared in `apps/backend/requirements.txt` (no versions pinned in that file):

- Python
- FastAPI
- Uvicorn
- SQLAlchemy
- psycopg
- Redis (`redis`)
- Pydantic Settings
- slowapi

Declared in `apps/crawler/requirements.txt` (no versions pinned in that file):

- Python
- Playwright
- HTTPX
- SQLAlchemy
- psycopg

Data stores confirmed in application code, not only in the requirements list:

- PostgreSQL full-text search (`to_tsvector`, `plainto_tsquery`) in `apps/backend/app/main.py` and `apps/backend/app/crud.py`; `ts_rank` on the search handler in `apps/backend/app/main.py`
- Redis reads and writes in `apps/backend/app/main.py`

## Specialties

- Search ranking: PostgreSQL `ts_rank` on listing text, with `ILIKE` matches, in `apps/backend/app/main.py`
- Web crawling: Playwright in `apps/crawler/app/main.py`
- Response cache: Redis in `apps/backend/app/main.py`

## Boundaries

This note covers the public search product and the libraries named above. It does not describe hosting or credentials.
