# YJ Studio — Supabase migration status

> **Current migration target:** Supabase PostgreSQL + a private Supabase Storage bucket. The React/Vite design and existing Hono API route contract are preserved; the API now runs through the Node.js adapter in `server/`. Follow [`SUPABASE_MIGRATION.md`](./SUPABASE_MIGRATION.md) for environment setup, schema creation, database snapshot import, media migration, and cutover validation. The Cloudflare deployment notes below describe the previous hosting setup and should be treated as rollback/reference documentation only.

# YJ Studio

A bilingual, responsive website and installable PWA for YJ Studio, backed by a Cloudflare Worker API, D1 database, and R2 media bucket. The visual identity follows the supplied logo: graphite and black surfaces, brushed silver details, and cyan waveform accents. The original logo and resized PWA icons are in `public/`.

The admin area is available at `/admin`. There is no public admin registration. The initial administrator is created with the local or remote Wrangler provisioning command described below.

## Stack

- React 19 and TypeScript, built with Vite
- Hono API running on Cloudflare Workers
- Cloudflare D1 for relational data and versioned SQL migrations
- Cloudflare R2 for private media objects served through an authorization-aware Worker route
- PWA web manifest and a service worker that caches the public app shell and static assets only; `/api/*` is always network-only

Prices, services, projects, campaigns, studio settings, bookings, audit events, and admin sessions are stored in D1. Uploaded images, audio, and video are stored in R2. Public API responses contain only published portfolio content, active services, live campaigns, and public studio settings. Booking records and draft content require an admin session.

## Local development on Windows

Use Node.js 20.19 or newer. PowerShell may block the `npm.ps1` shim; use `npm.cmd` for the commands below.

1. Install dependencies:

   ```powershell
   npm.cmd install
   Copy-Item .dev.vars.example .dev.vars
   ```

2. Apply local D1 migrations and start the Worker API plus Vite:

   ```powershell
   npm.cmd run dev
   ```

   The public site is served at `http://127.0.0.1:5173`; Vite proxies `/api` to the local Worker on port 8787. Wrangler keeps its local D1 and R2 state under `.wrangler/`.

3. In a second terminal, provision the first local administrator. Set a 12 character or longer password in the environment so it is not saved in the project or command history:

   ```powershell
   $env:ADMIN_USERNAME = 'studio-admin'
   $secureAdminPassword = Read-Host 'Choose a strong admin password' -AsSecureString
   $adminPasswordPointer = [Runtime.InteropServices.Marshal]::SecureStringToBSTR($secureAdminPassword)
   try {
     $env:ADMIN_PASSWORD = [Runtime.InteropServices.Marshal]::PtrToStringBSTR($adminPasswordPointer)
     npm.cmd run admin:create
   } finally {
     [Runtime.InteropServices.Marshal]::ZeroFreeBSTR($adminPasswordPointer)
     Remove-Item Env:ADMIN_USERNAME, Env:ADMIN_PASSWORD -ErrorAction SilentlyContinue
     Remove-Variable secureAdminPassword, adminPasswordPointer -ErrorAction SilentlyContinue
   }
   ```

   Open `http://127.0.0.1:5173/admin` and sign in. The local database must have migrations applied first. The provisioning command is idempotent for an existing username and never prints the password.

4. Run the quality checks:

   ```powershell
   npm.cmd run typecheck
   npm.cmd run lint
   npm.cmd test
   npm.cmd run build
   npm.cmd run smoke
   ```

The local admin credentials and uploaded files live in the local Wrangler state, not browser storage. A reset of local Wrangler state removes them.

## Features

- Public portfolio with category filters, search, project media, one shared audio player, video playback, and YouTube embeds.
- D1-driven service prices, hourly session length, current discount scheduling, and a booking form with a database price quote.
- Server-side booking validation and overlap checks against pending and confirmed appointments; admin status, note, and rescheduling controls.
- Campaign management for studio promotions and partner announcements, with placement, scheduling, media, CTA links, view and click counts.
- Project drafts, publish and feature flags, order, tags, credits, and image/audio/video/YouTube media.
- Studio copy, logo, contact details, WhatsApp, social links, SEO fields, and administrator-managed upload formats and size limits.
- Editable navigation labels, section visibility, maintenance announcements, and social sharing image metadata.
- Configurable in-dashboard booking alerts. They do not send email, WhatsApp or push messages.
- Responsive admin overview, metrics, activity trail, audit log, and password update.
- English and Arabic language toggle, `dir=rtl` layout, accessible form labels, keyboard focus styling, reduced-motion support, and offline app shell.

## Security notes

- Passwords are PBKDF2-SHA-256 hashes with a random per-password salt and 310,000 iterations.
- Admin sessions use random opaque tokens stored as SHA-256 hashes in D1 and sent only in `HttpOnly`, `SameSite=Strict` cookies (`Secure` in production).
- Every private API path checks the session and administrator role server-side. Writes require an allowed `Origin` and the app request header. Login attempts are rate-limited per hashed client address.
- JSON records use Zod validation and prepared D1 statements. Allowed media types and size limits are checked against D1 settings on every upload. R2 object keys are random, and media requests are checked against published project or active campaign references unless an admin session is present.
- The service worker does not cache API responses, booking records, admin pages as data, or authentication material. It provides the cached public app shell when offline.
- Booking alerts are shown only inside the signed-in dashboard. No email, WhatsApp, or push delivery integration is configured.
- Set strong account passwords and keep `.dev.vars`, Wrangler credentials, and any deployment secrets out of source control. Public admin registration and hardcoded credentials are not included.

## Cloudflare setup and deployment

Production resources are **not created or deployed by this repository**. Provision and verify the target Cloudflare account and environment before continuing.

1. Authenticate Wrangler with the intended Cloudflare account:

   ```powershell
   npx wrangler login
   ```

2. Create or identify a dedicated production D1 database and R2 bucket. Confirm their names and IDs in Cloudflare. Update `wrangler.jsonc` with the verified D1 `database_id` and R2 bucket name. The checked-in all-zero D1 ID is a local development placeholder and is not deployable.

3. Review the Worker name, custom domain, account, D1 database ID, bucket, and environment before making any remote changes. Keep production variables and credentials in Cloudflare or Wrangler secret storage, not in frontend files.

4. Apply the reviewed D1 migrations to the production database:

   ```powershell
   npm.cmd run db:migrate:remote
   ```

5. Create the first production administrator from an authorized machine after verifying the remote database target:

   ```powershell
   $env:ADMIN_USERNAME = 'studio-admin'
   $secureAdminPassword = Read-Host 'Choose a strong admin password' -AsSecureString
   $adminPasswordPointer = [Runtime.InteropServices.Marshal]::SecureStringToBSTR($secureAdminPassword)
   try {
     $env:ADMIN_PASSWORD = [Runtime.InteropServices.Marshal]::PtrToStringBSTR($adminPasswordPointer)
     npm.cmd run admin:create -- --remote
   } finally {
     [Runtime.InteropServices.Marshal]::ZeroFreeBSTR($adminPasswordPointer)
     Remove-Item Env:ADMIN_USERNAME, Env:ADMIN_PASSWORD -ErrorAction SilentlyContinue
     Remove-Variable secureAdminPassword, adminPasswordPointer -ErrorAction SilentlyContinue
   }
   ```

6. Review and deploy only after the account, D1, R2, and domain configuration are correct:

   ```powershell
   npm.cmd run deploy
   ```

The `deploy` script builds the static site and invokes `wrangler deploy`; it does not create Cloudflare resources or run production migrations. R2 objects are not public bucket URLs. The Worker serves them through the media authorization route.

## Backups and restore planning

Before a production change, export the target D1 database to a protected backup directory and verify the file before proceeding:

```powershell
New-Item -ItemType Directory -Force .\backups | Out-Null
npx wrangler d1 export yj-studio --remote --output .\backups\yj-studio-YYYYMMDD.sql
```

Protect that SQL file as customer data: it contains booking contact details and administrator hashes. Store a separate, access-controlled copy of R2 objects. D1 export does not include R2 media. For a restore, use a new or verified target database, review the backup and migration state, apply the migrations, then restore the SQL export with Wrangler's D1 execute command. Restore media from the separately secured R2 copy and verify project references. Do not test a restore against the live database. Cloudflare backup retention and R2 versioning/replication depend on the account plan and bucket configuration; confirm those settings in Cloudflare before relying on them.

## Local checks and acceptance review

`npm.cmd run typecheck`, `npm.cmd run lint`, `npm.cmd test`, and `npm.cmd run build` cover TypeScript, source lint, schema validation, and the production frontend bundle. The live flow can be checked locally by provisioning a local admin, uploading project media, publishing a project, changing a service price, creating a campaign, and submitting overlapping booking requests. No production data or resources are touched by local development.

This project does not contain media uploads, sample bookings, or fabricated portfolio entries. YouTube/video playback depends on valid authorized links or uploaded media. Actual end-to-end local checks require the D1/R2 dev server and first-admin setup above; production login and delivery remain dependent on the account resources and environment variables the owner configures.
