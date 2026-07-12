# URL Shortener

A small URL shortener service built to learn core system design concepts: caching, rate limiting, and running a multi-service app with Docker. This is a **learning project**, not a production system — it favors clarity over hardening.

## Why this exists

I built this to get hands-on with a few system design ideas that come up constantly in interviews and real backend work:

- How a cache sits in front of a database to absorb read traffic
- How to protect an API from abuse with rate limiting
- How a relational database can be the durable source of truth while a faster store handles the hot path
- How to wire multiple services (app, cache, database) together with Docker Compose

## Architecture

**Write path — `POST /shorten`**

```
client --> Flask app --> PostgreSQL (INSERT long_url, get auto id)
                               |
                               v
                    base62-encode id -> short_code
                               |
                               v
                    Redis SET short_code -> long_url
                               |
                               v
                    return { short_url, short_code } to client
```

The long URL is inserted into Postgres first, which assigns it an auto-incrementing `id`. That `id` is base62-encoded into a short code (e.g. `1` → `"1"`, `62` → `"10"`), and the `short_code -> long_url` mapping is written into Redis so future reads are fast.

**Read path — `GET /<short_code>`** (cache-aside)

```
client --> Flask app --> Redis GET short_code
                               |
                    +----------+----------+
                    | HIT                 | MISS
                    v                     v
             return long_url       PostgreSQL SELECT long_url
             (302 redirect)         (base62-decode short_code -> id)
                                          |
                                          v
                                   Redis SET short_code -> long_url
                                          |
                                          v
                                   return long_url (302 redirect)
```

Redis is checked first. On a cache hit, the app redirects immediately without touching Postgres. On a cache miss, the short code is base62-decoded back into the row id, Postgres is queried, the result is written back into Redis so the next request for that code is a cache hit, and then the client is redirected.

## Redis cache

Redis is used two ways in this project:

1. **Read-through cache for redirects** — `short_code -> long_url` mappings are cached on write and on cache miss, so repeat visits to the same short link never hit Postgres.
2. **Rate limit counters** — see below.

## Rate limiting

`POST /shorten` is rate-limited per client IP using a fixed-window counter stored in Redis:

- Each request increments a `rate_limit:<ip>` key.
- The key expires after `window` seconds (default: 60s), which resets the count.
- If a client exceeds `max_requests` (default: 10) within that window, the API responds with `429 Rate limit exceeded. Try again later.`

The increment-and-expire happens inside a single Redis pipeline so the counter update is atomic. This is implemented as a reusable `@rate_limit(...)` decorator in `app/app.py`, so it could be applied to other routes with different limits.

## PostgreSQL

Postgres is the durable source of truth. The schema (`app/init.sql`) is intentionally minimal:

```sql
CREATE TABLE IF NOT EXISTS urls (
    id SERIAL PRIMARY KEY,
    long_url TEXT NOT NULL,
    created_at TIMESTAMP DEFAULT CURRENT_TIMESTAMP
);
```

The `id` is what gets base62-encoded into the public short code — so short codes are never stored directly, they're derived from the row id on the fly. Redis can be flushed at any time and the app still works correctly (just slower, since every read becomes a cache miss until it's repopulated).

## Docker

The whole stack runs via `docker-compose.yml` as three services:

| Service | Image / Build | Purpose |
|---|---|---|
| `web` | built from `app/Dockerfile` (Python 3.12-slim + Flask) | the API |
| `db` | `postgres:16-alpine` | durable storage, initialized with `app/init.sql` |
| `redis` | `redis:7-alpine` | cache + rate limit counters |

`web` waits on `db` and `redis` passing their healthchecks (`pg_isready`, `redis-cli ping`) before starting, so the app never boots against a database or cache that isn't ready yet. Postgres data persists in a named volume (`postgres_data`) across restarts.

## Running it locally

```bash
docker compose up --build
```

The API will be available at `http://localhost:5000`.

### Shorten a URL

```bash
curl -X POST http://localhost:5000/shorten \
  -H "Content-Type: application/json" \
  -d '{"url": "https://example.com/some/very/long/path"}'
```

```json
{ "short_url": "http://localhost:5000/1", "short_code": "1" }
```

### Follow a short URL

```bash
curl -i http://localhost:5000/1
```

Redirects (`302`) to the original long URL.

### Health check

```bash
curl http://localhost:5000/health
```

## Notes on scope

This is a learning project, so a few things are deliberately simplified rather than production-hardened:

- Credentials are plaintext defaults in `docker-compose.yml`, fine for local use, not for deployment.
- Rate limiting is per-IP only (no auth, no per-user limits).
- There's no HTTPS, no input validation on the submitted URL, and no expiry/eviction policy for cached entries.

## Stack

Flask · Redis · PostgreSQL · Docker Compose · Python 3.12
