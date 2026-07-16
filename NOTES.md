# Xonacatla.com — Project Notes

Session record: Supabase-backed gallery + admin panel build (July 9, 2026) and
auto-pause incident fix (July 16, 2026). Written by Claude Code.

## Architecture

Static site on **GitHub Pages** (this repo, `main` branch, custom domain
xonacatla.com via CNAME) backed by **Supabase** project **Xonacatla-Demo**
(ref `vyarqorszxrsvpappcju`, us-east-1, free tier).

| Piece | What it does |
|---|---|
| `index.html` | Public bilingual site. All text hydrates at load from the `content` table (hardcoded ES/EN strings remain as offline fallbacks). Gallery uploads: canvas-downscale to max 1000px JPEG → `gallery` storage bucket → `photos` row with `status='pending'`. Grid shows only `status='approved'`, newest first. |
| `admin.html` | Client admin panel (not linked from the site). Supabase email/password login. Approve/reject pending photos (reject deletes file + row). Edit Spanish text blocks; edits flip `en_stale=true` ("traducción pendiente" badge). |
| `.github/workflows/keepalive.yml` | Pings Supabase REST Mon/Thu so the free project isn't paused for inactivity. |
| `SYNC.md` | Developer loop for translating stale English text (`en_stale=true` rows). |

## Supabase schema (applied via migrations)

- `photos(id, storage_path, caption, uploader_name, status, created_at)` —
  `status in ('pending','approved','rejected')`, default `pending`.
- `content(key, es, en, en_stale, updated_at)` — 70 rows keyed by the
  `data-i18n` attributes in index.html. Trigger `content_mark_stale` flips
  `en_stale=true` when `es` changes without `en`.
- Storage bucket `gallery` — public read, images only (jpeg/png/webp), 2 MB max.
  Bucket `site` (fondo video + poster) untouched.

### Security model (RLS)

- The anon/publishable key in the HTML is safe to expose; RLS is the boundary.
- anon: INSERT photos only with `status='pending'`; SELECT only approved;
  read-only on `content`; upload-only to `gallery` bucket.
- Moderation (update/delete photos, edit content, delete storage files) requires
  a logged-in user whose email is in the `public.is_admin()` allowlist:
  `jpmartinez789@yahoo.com`, `ejmartinez.889@gmail.com`. Random sign-ups get no
  rights. To add an admin: update `is_admin()` AND create the auth user.
- Admin passwords are NOT in this repo. They were issued in the build session;
  change them via "Cambiar contraseña" in admin.html.

## Keep-alive (after the July 16 auto-pause)

The project was auto-paused July 16 after 7 idle days (the workflow hadn't been
merged to `main`, where scheduled workflows must live). Two independent
keep-alives now run:

1. GitHub Action (`keepalive.yml`): Mon/Thu 08:17 UTC. Note GitHub disables
   scheduled workflows after ~60 days without repo activity — respond to
   GitHub's warning emails, or any commit resets the clock.
2. `pg_cron` job `supabase-keepalive` inside the database: Tue/Fri 03:43 UTC,
   pings the project's own REST API via `pg_net` (migration
   `keepalive_db_self_ping`). Independent of GitHub.

If the project pauses anyway: Supabase dashboard → project → Restore. Data is
retained for 90 days after a pause.

## Verified end-to-end (July 16, against production)

- Live site serves the new code; old localStorage/demo code gone.
- Anonymous pending upload → invisible publicly → admin approve → public →
  admin reject/delete → gone (file and row). MIME limit rejects non-images (415).
- Non-admin authenticated accounts blocked from all moderation.
- Content edit propagates to the public endpoint and flips `en_stale`.
- Both admin logins return valid tokens; both keep-alives registered and firing.

## Known follow-ups

- Delete unused edge function `site` (dashboard only; no API access from here).
- Optional: enable "leaked password protection" (dashboard → Auth settings).
- Optional enhancement discussed: a "Fotos publicadas" section in admin.html to
  remove already-published photos (delete policies already permit it).
- Clients should change their temporary passwords in admin.html.
