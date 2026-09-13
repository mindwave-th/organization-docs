# Changelog

All notable changes to Mindwave Customer Interface will be documented in this file.

The format is based on [Keep a Changelog](https://keepachangelog.com/en/1.0.0/),
and this project adheres to [Semantic Versioning](https://semver.org/spec/v2.0.0.html).

## [Unreleased]

### Added
- Initialized `stagging` branch for environment-specific builds and staging deployments.

## [1.1.0] - 2026-03-14

### Added
- **Terms of Service page** (`/terms-of-service`) — full Thai content mirroring Privacy Policy layout with React Icons and card-based sections
- **Consent modal** — mandatory acceptance modal for authenticated users who haven't accepted ToS or Privacy Policy
  - Inline scrollable accordion sections (ToS + Privacy content displayed within modal)
  - Cannot be dismissed without accepting
  - Calls `POST /users/me/consent` on accept
- **Consent service** (`consentService.ts`) — API client for consent status check and acceptance
- ToS link in footer alongside existing Privacy Policy link

### Changed
- Authentication modal consent text now mentions both Terms of Service and Privacy Policy with inline links
- App.tsx: consent check runs after login, shows ConsentModal when `tos_accepted_at` or `privacy_accepted_at` is null

## [1.0.0] - 2026-03-09

### Added
- **Google OAuth login**: Google Sign-In button in authentication modal
  - Google Identity Services (GIS) loaded via script tag (no npm dependency)
  - `googleLogin()` in `authService` and `useAuth` hook
  - Google button renders with Thai locale and "หรือ" divider
  - `onGoogleLogin` prop wired from `App.tsx` → `Authentication` modal
- TypeScript declarations for Google Identity Services API (`src/types/google.d.ts`)
- `.env.example` with `VITE_API_URL` and `VITE_GOOGLE_CLIENT_ID`

### Changed
- `AuthUser.phone`: `string` → `string | null` (Google users may not have phone)
- `AuthUser.email`: added `string | null` field
