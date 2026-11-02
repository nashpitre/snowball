# Snowball

A private mobile game queue. GitHub Pages serves the interface; a Cloudflare Worker holds Google credentials and uses a private D1 database. Google sign-in is restricted to the configured account. No ChatGPT session is required.

## Local startup

Use Node 22.13 or newer, then `npm ci`. Copy `.env.example` to `.dev.vars` and supply server credentials locally. Never commit that file. Run `npx wrangler d1 migrations apply snowball --local`, then `npm run dev:api`. For the interface, set `frontend/public/config.json` to your HTTPS backend and run `npx vite --config vite.pages.config.ts`. The production frontend is built with `npm run build:pages` into `docs/`.

## Hosting

Create a separate Cloudflare D1 database, set its ID in `wrangler.jsonc`, and apply `npx wrangler d1 migrations apply snowball --remote`. Store credentials with `npx wrangler secret put NAME` or secure standard input to `wrangler secret bulk`. Set APP_URL to the final Pages URL. Deploy with `npm run deploy:api`. Set the returned backend URL in `frontend/public/config.json`, rebuild, and publish `main` → `/docs` in GitHub Pages. `.nojekyll` preserves built assets.

The repository contains source and static assets only. Personal library data lives in D1. Google access/refresh tokens are encrypted using AES-GCM; TOKEN_ENCRYPTION_KEY must be 64 hexadecimal characters. Back up that key securely. Browser storage holds only an opaque revocable app session, never provider credentials. Sessions expire after seven days; sign out revokes the current session.

## Google authorization

Enable the Google Sheets API in your Google Cloud project. Create a Web OAuth client. Add the exact backend URL followed by `/api/google/callback` as an authorized redirect URI. Preserve any existing callback until migration is verified. Configure GOOGLE_CLIENT_ID, GOOGLE_CLIENT_SECRET, GOOGLE_REDIRECT_URI, GOOGLE_SPREADSHEET_ID, ALLOWED_USER_EMAIL, and TOKEN_ENCRYPTION_KEY as backend secrets. Sign in from Snowball and authorize spreadsheet editing. OAuth uses a server nonce cookie, one-time state, a short-lived browser-verifier-bound exchange, and an allowlisted verified Google email.

External OAuth apps in Testing can have seven-day refresh-token expiry when Sheets scopes are requested. Configure the Google consent audience and publishing status appropriately; revoked or expired authorization requires reconnecting. The spreadsheet stays private. The app does not add columns or change its layout. Sheet edits use conflict checks, targeted cell writes and read-back; failures remain in a durable outbox for retry.

## OpenAI

Set OPENAI_API_KEY, OPENAI_PARSE_MODEL and OPENAI_RESEARCH_MODEL on the backend after verifying account/model availability. Parsing uses structured proposals with application validation. Research uses sourced web search and cached estimates. Without those settings, supported explicit commands use a local parser and only existing sourced estimates are ranked. No claim of full AI interpretation is made until a real request is verified. Set PARSE_DAILY_LIMIT and RESEARCH_DAILY_LIMIT to control request counts; these are request budgets, not guaranteed monetary caps.

## Validation

`npm test` covers deterministic ranking, progress distinctions, source identity, ambiguity, write conflicts, research citation validation and token response handling. `python3 scripts/test-storage.py` exercises schema, idempotency, stale edits and Undo. `npx tsc --noEmit` checks types. Test real Google authorization, sheet import, a deliberately chosen reversible update and Undo before final cutover. Never mark a real game finished for a demonstration.

The previous hosted app remains available until standalone sign-in and sync are verified. No personal data or provider secrets are packaged in the frontend. Research is intentionally bounded; unknown games are not represented by invented estimates.
