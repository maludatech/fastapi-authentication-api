# FastAPI Authentication API

A production-style authentication REST API built with FastAPI, SQLAlchemy 2.0, and PostgreSQL (Neon). Implements registration, login, protected routes, and JWT access/refresh tokens with server-side revocation.

## Stack

- **FastAPI** — web framework
- **SQLAlchemy 2.0** — ORM
- **PostgreSQL (Neon)** — database
- **Alembic** — schema migrations
- **Pydantic v2** — request/response validation
- **bcrypt** — password hashing
- **python-jose** — JWT signing/verification
- **slowapi** — rate limiting

## Features

- Email/password registration and login
- Bcrypt-hashed passwords (never stored in plaintext, never returned in responses)
- Short-lived JWT access tokens (15 min default) + long-lived refresh tokens (7 days default)
- Refresh token rotation: every `/auth/refresh` call revokes the used token and issues a new pair, limiting the value of a leaked refresh token
- Refresh tokens are tracked server-side (by a `jti` claim, not the raw token) so they can be revoked — supports both single-session logout and logout-all-devices
- Rate limiting on `/auth/login` and `/auth/register` (5 requests/minute per IP)
- CORS configured for known frontend origins
- Global exception handler: unexpected errors return a generic `500` and are logged server-side, never leaking internals to the client

## Endpoints

| Method | Path                | Auth required | Description                                      |
|--------|---------------------|---------------|---------------------------------------------------|
| POST   | `/auth/register`    | No            | Create a new user                                  |
| POST   | `/auth/login`        | No            | Exchange credentials for an access/refresh pair    |
| GET    | `/auth/me`           | Yes (access)  | Return the current authenticated user              |
| POST   | `/auth/refresh`      | No (refresh token in body) | Rotate a refresh token for a new pair |
| POST   | `/auth/logout`       | Yes (access)  | Revoke a single refresh token                       |
| POST   | `/auth/logout-all`   | Yes (access)  | Revoke every active refresh token for the user      |
| GET    | `/health`            | No            | Health check                                        |

Interactive API docs are available at `/docs` (Swagger UI) once the server is running.

## Project structure

```
app/
├── main.py            # FastAPI app, middleware, global exception handler
├── config.py           # Typed settings loaded from .env (pydantic-settings)
├── database.py          # SQLAlchemy engine, session, declarative Base
├── security.py          # Password hashing + JWT create/decode helpers
├── dependencies.py       # get_current_user auth dependency
├── rate_limit.py         # slowapi limiter instance
├── models/               # SQLAlchemy models (User, RefreshToken)
├── schemas/               # Pydantic request/response schemas
└── routers/
    └── auth.py            # All /auth/* endpoints

alembic/                    # Migration environment and versions
```

## Setup

1. **Create a virtual environment and install dependencies:**

   ```bash
   python -m venv venv
   venv\Scripts\activate       # Windows
   pip install -r requirements.txt
   ```

2. **Configure environment variables.** Copy `.env.example` to `.env` and fill in real values:

   ```
   DATABASE_URL=postgresql+psycopg://user:password@host/dbname?sslmode=require
   SECRET_KEY=a-long-random-string
   JWT_ALGORITHM=HS256
   ACCESS_TOKEN_EXPIRE_MINUTES=15
   REFRESH_TOKEN_EXPIRE_DAYS=7
   RATE_LIMIT_STORAGE_URI=memory://
   CORS_ORIGINS=http://localhost:3000,http://localhost:5173
   ```

   `DATABASE_URL` must use the `postgresql+psycopg://` scheme (psycopg 3), not the bare `postgresql://` Neon gives by default.

3. **Run database migrations:**

   ```bash
   alembic upgrade head
   ```

4. **Start the dev server:**

   ```bash
   uvicorn app.main:app --reload --port 8000
   ```

   Visit `http://127.0.0.1:8000/docs` for interactive API docs.

## Notes on production readiness

- **Rate limiting** uses in-memory storage by default, which is only correct for a single running instance. For multi-instance/multi-worker deployments, set `RATE_LIMIT_STORAGE_URI` to a Redis URI — `slowapi` picks it up with no code changes.
- **Access tokens cannot be revoked early** — they're stateless JWTs valid until their signed expiry. `logout` and `logout-all` only revoke refresh tokens, so a leaked access token remains usable until it naturally expires (default 15 minutes). Shorten `ACCESS_TOKEN_EXPIRE_MINUTES` if a tighter window is needed.
- **Migrations** are managed with Alembic (`alembic revision --autogenerate -m "..."` after model changes, then `alembic upgrade head`).
