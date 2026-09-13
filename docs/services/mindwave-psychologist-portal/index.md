# mindwave-psy-web

Frontend for the **psychologist** portal — mirrors mindwave-admin's shell,
design tokens, and auth/session conventions, talking to mindwave-core's
`/psy` routes (src/routes/psy) instead of `/admin`.

## Architecture
- Next.js 16 App Router, React 19, no CSS framework — the same hand-authored
  design-token + `ds/` component system as mindwave-admin, copied verbatim
  (`tokens/*.css`, `src/components/ds/*`, `ui_kits/psy/kit.css`) so the two
  portals stay visually consistent as one product family.
- **Backend**: lives inside `mindwave-core` (Rust/Axum), not a separate
  service — own tables (`psy_*`), own JWT secret/audience (`PsyClaims`,
  src/routes/psy/token.rs), same Postgres pools admin_core already has
  (`state.db` for psy_* tables, `state.user_db` for reading a patient's
  AI-chat log). See `mindwave-core/src/routes/psy/`.
- **Auth**: `src/lib/auth.ts` + `src/lib/api.ts`, same shape as mindwave-admin
  (localStorage token/refresh_token, silent refresh on 401, idle + absolute
  session timeouts via `AuthSentinel`, per-page RBAC via `PageGuard`) — just
  pointed at the psy portal's three roles (`psy_admin`, `psychologist`,
  `viewer`) instead of admin's four-tier staff model.
- **Live chat**: `src/lib/api.ts#chatSocketUrl` opens a browser WebSocket to
  `/psy/chat/ws/{conversation_id}` (token in the query string — the only way
  to authenticate a WS handshake from the browser). No SSE/polling fallback
  for delivery; `useLiveRefresh` only refreshes the conversation *list* in
  case a socket silently dropped.
- **Patient history**: `src/lib/patients-api.ts` — caseload + consultation
  notes (`psy_consultation`) plus the patient's AI-chat log, read directly
  from `user_core.user_chat` (same pattern as admin's moderation view).
  Degrades gracefully (`ai_chat_available: false`) if that query fails.

## Pages
- `/login` — username or email + password
- `/overview` — quick caseload/conversation stats
- `/patients` — caseload list → history drawer (consultations + AI chat)
- `/chat` — conversation list + live thread (WebSocket)
- `/account`, `/change-password`

## Run
    cp .env.local.example .env.local   # point NEXT_PUBLIC_CORE_API_URL at mindwave-core
    npm install
    npm run dev

Requires `mindwave-core` running (`cargo run`, port 8000 by default) with
`PSY_JWT_SECRET` set in its `.env` and at least one `psy_account` provisioned
via the admin console's Practitioners page (system = `psy`).
