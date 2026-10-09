# Engineering note

## Overview

Hamejoo searches listings and shows the matches in one place. Listing detail stays on the original source.

## Stack

Web app:

- Next.js ^14.2.35
- React 18.3.1
- TypeScript 5.7.3
- Tailwind CSS 3.4.17

API (Python):

- FastAPI
- Uvicorn
- SQLAlchemy
- psycopg
- Redis
- Pydantic Settings
- slowapi

Crawler (Python):

- Playwright
- HTTPX
- SQLAlchemy
- psycopg

## Specialties

- Full-text search: PostgreSQL `to_tsvector` and `plainto_tsquery` with the `simple` configuration for Persian text, `ts_rank` for relevance, and `ILIKE` for partial matches
- Response cache: Redis, keyed by a SHA-256 of the search parameters
- Web crawling: Playwright, with hash-based deduplication and a capped job count per round
- Rate limiting on read routes: slowapi
