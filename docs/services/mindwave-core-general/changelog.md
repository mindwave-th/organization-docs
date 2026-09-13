# Changelog

All notable changes to Mindwave Core API will be documented in this file.

The format is based on [Keep a Changelog](https://keepachangelog.com/en/1.0.0/),
and this project adheres to [Semantic Versioning](https://semver.org/spec/v2.0.0.html).

## [3.4.0] - 2026-03-14

### Added
- **Terms of Service + Consent tracking system**
  - Migration 024: `tos_accepted_at` and `privacy_accepted_at` TIMESTAMPTZ columns on `users` table
  - `GET /users/me/consent` (auth) — check current consent status
  - `POST /users/me/consent` (auth) — accept ToS and/or Privacy Policy (timestamps set once, never overwritten)
  - `ConsentStatusResponse` and `AcceptConsentRequest` schemas in OpenAPI
- **Admin consent visibility**
  - "ยินยอม" (consent) column in admin user list table
  - ToS and Privacy acceptance timestamps in admin user detail modal

### Changed
- `UserResponse` now includes `tos_accepted_at` and `privacy_accepted_at` fields (returned from `GET /users/me`)
- `AdminUserResponse` and `UserDetailResponse` now include consent timestamps
- All user queries (`get_current_user`, `update_current_user`, `list_users`, `list_admin_users`, `get_admin_user`) updated to select consent columns
- API version bumped to 3.4.0

## [3.3.0] - 2026-03-09

### Added
- **AI generation config system**: Admin-configurable Gemini API parameters per AI type (venting, checkin, journaling)
  - `GET /admin/ai-gen-config` (admin+) — fetch all 3 AI generation configs
  - `PUT /admin/ai-gen-config/:type` (superadmin) — update generation config with validation
  - Parameters: model, temperature, topP, topK, maxOutputTokens, stopSequences, thinkingBudget
  - Stored in `system_config` table as JSONB (reuses existing pattern)
  - Migration 022: Seed 3 default configs (venting=creative, checkin=moderate, journaling=reflective)
- **Dynamic model selection**: Each AI type can use a different Gemini model
  - Default: `gemini-3-flash-preview` for all 3 AI types
  - Supported: gemini-3.1-pro-preview, gemini-3-flash-preview, gemini-3.1-flash-lite-preview, gemini-2.5-flash/pro/flash-lite
- **Admin UI for generation config** (`listening.html`):
  - Slider controls for temperature, top_p, top_k, max_output_tokens
  - Model dropdown with custom model input option
  - Stop sequences chip editor (max 5)
  - Thinking budget input
  - Thai tooltips explaining each parameter in detail
  - Superadmin-only visibility

### Changed
- `GeminiClient`: All 3 methods (`generate`, `generate_with_history`, `generate_stream`) now accept `Option<&GeminiGenParams>` for dynamic generation config
- `GeminiRequest`: Added `generationConfig` field (camelCase for Gemini API compatibility)
- Default Gemini model changed from `gemini-2.5-flash` to `gemini-3-flash-preview`
- 3 AI handlers (venting, checkin, journaling) now load and pass generation config from DB
- API version bumped to 3.3.0
- Moved Reflection config (moods/activities editor) from dashboard to การรับฟัง > AI กระจกส่องใจ (ตัวเปิด) tab
- Anonymized user identity in admin mind dump listings — shows "ผู้ใช้ #abc123" instead of username/phone

## [3.2.0] - 2026-03-09

### Added
- **Google OAuth login**: Second authentication channel via Google Identity Services
  - `POST /auth/google` — verify Google ID token, find/create/link user, return JWT
  - Account linking: lookup by `google_sub` → by `email` → create new user
  - Google-only users (no phone) fully supported
- **Migration 021**: `phone` nullable, added `email`, `auth_provider`, `google_sub` columns
  - `CHECK` constraint: user must have at least phone or email
  - Indexes on `email` and `google_sub`
- `GOOGLE_CLIENT_ID` env var in config
- `GoogleLoginRequest` schema in OpenAPI docs
- `.env.example` updated with `GEMINI_API_KEY` and `GOOGLE_CLIENT_ID`

### Changed
- `Claims.phone` and `AuthUser.phone`: `String` → `Option<String>` (backward compatible via `#[serde(default)]`)
- `Claims`, `AuthUser`, `UserInfo`, `UserResponse`, `AdminUserResponse`, `UserDetailResponse`: added `email: Option<String>`
- `generate_tokens()` refactored to accept `Option<&str>` for phone and email
- Admin user search now includes email field
- Admin user detail response includes email
- API version bumped to 3.2.0

## [3.1.0] - 2026-03-01

### Added
- **Adaptive prompt config system**: DB-backed adaptive prompts for mind dumps and reflections
  - `GET /config/prompts` (public) — fetch adaptive prompt config for frontend
  - `GET /admin/prompt-config` (admin) — get all prompt config sections
  - `PUT /admin/prompt-config/:section` (superadmin) — update prompt config section
  - Admin UI page (`/admin/prompt-config.html`) for managing adaptive prompts
  - Migration 017: Seed adaptive prompt configuration
- **5 standardized assessments** seeded in Thai (PHQ-9, GAD-7, PSS-10, OLBI, DASS-21)
  - Migration 018: Idempotent seed with `ON CONFLICT (type) DO UPDATE`
  - PSS-10 (Perceived Stress Scale) — 10 questions, reverse scoring on items 4/5/7/8
  - OLBI (Oldenburg Burnout Inventory) — 16 questions, exhaustion/disengagement subscales with averages
  - DASS-21 (Depression Anxiety Stress Scales) — 21 questions, x2 score multiplier, 3 subscales
- **Advanced scoring engine** in `calculate_scores()`:
  - Reverse scoring support (`reverse_items` + `reverse_max`)
  - Subscale average calculation (`{name}_avg`)
  - Score multiplier support (`score_multiplier` → `{name}_adjusted`)
- **Per-subscale interpretation** in `get_interpretation()`:
  - Subscale-level range matching with float comparison
  - Overall logic: "worst" severity (DASS-21) and "burnout" composite (OLBI)
  - Full range field passthrough (color, severity, advice, validation, symptoms)

### Fixed
- **Migration runner**: Replaced naive `split(';')` with string-aware SQL splitter that handles single-quoted strings, `$$`-dollar-quoted blocks, and `--` line comments
- **Admin sidebar**: Added missing "Adaptive Prompt Config" menu link to all admin pages (index, users, listening, assessments, mind-dumps)
- Removed stale conversations migrations (003/013) that caused re-run failures

### Changed
- API version bumped to 3.1.0
- OpenAPI description updated to reflect all 5 assessment types and adaptive prompts

## [3.0.0] - 2026-03-01

### Added
- **Reflection system**: Daily mood tracking with emoji moods, activities, and journaling
  - `GET /reflection/config` (public) — fetch mood/activity options
  - `POST /reflection` (auth) — save reflection entry
  - `GET /reflection` (auth) — list user's entries
  - `PATCH /reflection/:id` (auth) — update entry (follow-ups, NLP analysis)
- **AI Listener endpoints**: Gemini-powered passive AI companions
  - `POST /ai/checkin` (public) — mood check-in response
  - `GET /ai/journal` (public, SSE) — real-time journal companion stream
  - `POST /ai/venting` (auth) — venting listener with conversation history
- **NLP config**: DB-backed NLP keyword/pattern configuration
  - `GET /config/nlp` (public) — merged NLP config for frontend
- **Admin endpoints**:
  - AI response listing and prompt management
  - Reflection config management
  - NLP config management
  - Mind dump listing with insight summaries
- **Rate limiting**: Per-IP rate limits on public endpoints (10 req/min AI, 30 req/min config)
- **Public access logging**: Structured tracing for unauthenticated API requests
- **OpenAPI documentation**: All 15 new endpoints documented with utoipa annotations

### Changed
- Split reflection and AI routes into public (no auth) and protected (JWT required)
- `ai_checkin` and `ai_journal` accept optional authentication (anonymous usage allowed)
- API version bumped to 3.0.0

## [1.2.0] - 2026-02-15

### Added
- Admin conversations management:
  - `GET /api/v1/admin/conversations` - List all conversations with user info
  - `GET /api/v1/admin/conversations/{id}` - Get conversation details with messages
- Conversations admin page (`/admin/conversations.html`)
- Help documentation modal on assessments admin page
- Thai language translation for entire admin interface
- Mobile responsive design for admin interface
- Favicon for admin pages

### Fixed
- API error handling in admin pages (graceful fallback)
- Conversations page field mapping for API response

### Security
- Added encryption reminder banner on conversations page

## [1.1.0] - 2026-02-15

### Added
- Admin HTML interface served by Rust backend (`/admin/`)
- User management endpoints:
  - `GET /api/v1/admin/users` - List all users (paginated)
  - `GET /api/v1/admin/users/{id}` - Get user details with assessment history
  - `PATCH /api/v1/admin/users/{id}/role` - Update user role (superadmin only)
  - `PATCH /api/v1/admin/users/{id}/status` - Enable/disable user
- Analytics endpoints:
  - `GET /api/v1/admin/analytics/overview` - Dashboard statistics
  - `GET /api/v1/admin/analytics/assessments` - Assessment trends
- Admin response models for paginated users and analytics
- Migration 008: Seed assessments (PHQ-9, GAD-7)
- Migration 009: Add `is_active` column to users table
- Static file serving via tower-http for admin UI

### Fixed
- Docker image now includes static directory for admin UI

## [1.0.0] - 2026-02-14

### Added
- Initial Rust (Axum) API server implementation
- Authentication system:
  - `POST /api/v1/auth/login` - Phone-based login
  - `POST /api/v1/auth/register` - User registration (phone + username)
  - `POST /api/v1/auth/refresh` - Token refresh
- User management:
  - `GET /api/v1/users/me` - Get current user
  - `PATCH /api/v1/users/me` - Update current user
  - `GET /api/v1/users` - List users (admin only)
- Conversations:
  - `POST /api/v1/conversations/chat` - Chat with SI
  - `GET /api/v1/conversations` - List conversations
  - `GET /api/v1/conversations/{id}` - Get conversation
- Mind Dumps:
  - `POST /api/v1/mind-dumps` - Create mind dump
  - `GET /api/v1/mind-dumps` - List mind dumps
  - `GET /api/v1/mind-dumps/{id}` - Get mind dump
  - `DELETE /api/v1/mind-dumps/{id}` - Delete mind dump
- Assessments:
  - `GET /api/v1/assessments/types` - List assessment types
  - `GET /api/v1/assessments/types/{type}` - Get assessment config
  - `POST /api/v1/assessments/submit` - Submit assessment
  - `GET /api/v1/assessments/history` - Get assessment history
- Admin assessment management:
  - `GET /api/v1/admin/assessments` - List configs
  - `POST /api/v1/admin/assessments` - Create config
  - `PUT /api/v1/admin/assessments/{type}` - Update config
  - `DELETE /api/v1/admin/assessments/{type}` - Delete config
- System config management (superadmin)
- OpenAPI/Swagger documentation at `/swagger-ui`
- JWT authentication with role-based access control
- PostgreSQL database with SQLx migrations
- Docker support

### Changed
- Removed OTP authentication system (simplified to phone-only login)
- Removed API key system

### Fixed
- Multi-statement migrations handling
- SI response field alias ('message' for 'response')
- Rust 1.85 compatibility (pinned dependencies)
