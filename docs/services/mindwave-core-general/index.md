# Mindwave Core

Rust (Axum) API server that acts as a gateway for customer-interface, proxying AI requests to synthetic-intelligent and managing users, conversations, and assessment configs in PostgreSQL.

## Architecture

```
customer-interface (React :3000) --> mindwave-core (Rust :8080) --> synthetic-intelligent (Python :8000)
                                            |
                                            v
                                     PostgreSQL (Railway)
```

## Prerequisites

- Rust 1.75+
- PostgreSQL database
- (Optional) Docker for containerized deployment

## Setup

1. Copy environment variables:
```bash
cp .env.example .env
# Edit .env with your database credentials
```

2. Run migrations (automatic on startup, or manually):
```bash
# Migrations run automatically when the server starts
```

3. Build and run:
```bash
cargo build --release
cargo run
```

4. Access API docs:
```
http://localhost:8080/swagger-ui/
```

## API Endpoints

| Group | Endpoints |
|-------|-----------|
| Auth | POST /api/v1/auth/login, /register, /refresh |
| Users | GET/PATCH /api/v1/users/me, GET /api/v1/users (admin) |
| API Keys | POST/GET /api/v1/api-keys, DELETE /api/v1/api-keys/:id |
| Conversations | POST /api/v1/conversations/chat, GET /api/v1/conversations |
| Mind Dumps | CRUD /api/v1/mind-dumps |
| Assessments | GET /api/v1/assessments/types, POST /submit, GET /history |
| Admin | CRUD /api/v1/admin/assessments, /system-config |

## Docker

```bash
docker build -t mindwave-core .
docker run -p 8080:8080 --env-file .env mindwave-core
```

## Environment Variables

| Variable | Description | Default |
|----------|-------------|---------|
| DATABASE_URL | PostgreSQL connection string | required |
| HOST | Server host | 0.0.0.0 |
| PORT | Server port | 8080 |
| JWT_SECRET | JWT signing secret | required |
| SI_BASE_URL | Synthetic Intelligent API URL | http://localhost:8000 |
| SUPER_ADMIN_PHONE | Phone for superadmin auto-role | +66000000000 |
