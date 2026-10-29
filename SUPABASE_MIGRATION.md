# YJ Studio — Supabase migration

This migration keeps the existing React/Vite UI and Hono API routes while moving persistent application data from Cloudflare D1 to Supabase PostgreSQL and uploaded media from Cloudflare R2 to a private Supabase Storage bucket. The API is run as a Node.js service (`server/index.ts`) so the existing parameterized SQL interface can continue to work without exposing a database secret to the browser.

## Required setup

1. Create a Supabase project and save its database connection string, project URL, and server-only secret key.
2. In Supabase SQL Editor, run `supabase/migrations/202610280001_initial_schema.sql` once. This creates the schema, initial defaults, role trigger, RLS protections, and private `yj-studio-media` bucket.
3. Copy `.env.supabase.example` to `.env` and fill in `SUPABASE_DB_URL`, `SUPABASE_URL`, and `SUPABASE_SECRET_KEY`. Use a database pooler connection appropriate for your host. Do not use `VITE_` prefixes for secrets, and never commit `.env`.
4. Install dependencies and build: `npm install`, `npm run build`.
5. Start the Node API and static frontend: `npm start`. Configure your hosting provider to use Node.js 20+ and the same build/start commands. The hosting service must support outbound TLS connections to the Supabase database and HTTPS to Supabase Storage.
6. Set `ALLOWED_ORIGINS` only if the public frontend origin differs from the API origin. Keeping the frontend and API on the same origin is preferred because the admin session cookie is HttpOnly and SameSite=Strict.

## Importing data

- `scripts/export-d1.py` exports a **local** D1 SQLite snapshot to JSON. Example:
  `python scripts/export-d1.py path/to/local.sqlite migration-data/d1-snapshot.json`
- After running the schema migration and setting `SUPABASE_DB_URL`, import with:
  `npm run db:import:snapshot -- migration-data/d1-snapshot.json`
- This import is idempotent and upserts matching primary keys, so customized source values replace starter defaults. Review counts and sample records before routing production traffic.
- A local `.wrangler` database is not the same as the live Cloudflare D1 database. Export the **remote production D1** separately using the Cloudflare D1 export tooling/dashboard, then convert its SQLite dump into the snapshot format before importing. Do not assume local data contains the production site's latest bookings/projects.
- Existing R2 media objects are not included in a D1 database export. Download/export those objects from the actual R2 bucket and upload them to the private `yj-studio-media` bucket, preserving their object keys so existing `/api/media/<key>` references keep working.

## Security and cutover checklist

- Keep `SUPABASE_SECRET_KEY` and `SUPABASE_DB_URL` server-only. Only the publishable key may be exposed to browser code; this migration does not need a browser Supabase client.
- App tables have RLS enabled and no `anon`/`authenticated` table grants. The private Storage bucket has no browser policies; uploads/downloads happen in the trusted Node API.
- Verify the owner/admin can sign in, role assignments remain present, session cookies are secure in production, and the audit log still records mutations.
- Test public portfolio/services, booking creation, admin project/service/campaign edits, settings, password changes, media uploads/downloads, Arabic/English text, and PWA install/offline behavior.
- Keep Cloudflare D1/R2 intact until data and media counts match and all smoke tests pass. This package cannot connect to or deploy into your Supabase account without credentials and explicit access; no remote data was changed by preparing these files.
