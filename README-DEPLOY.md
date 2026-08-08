# SPARK Desk — Deployment Guide (Vercel)

## Required environment variables

Set these in **Vercel → Project → Settings → Environment Variables**, for
the **Production** environment (not just Preview/Development — a variable
set only for Preview will not exist when your production domain runs).

| Variable | Required? | What happens if missing |
|---|---|---|
| `DATABASE_URL` | **Yes** | App falls back to local SQLite, which **crashes on first write** — Vercel's deployed filesystem is read-only except `/tmp`. Must be a real Postgres connection string (Neon, Supabase, Vercel Postgres, etc.), e.g. `postgresql://user:pass@host:5432/dbname`. |
| `SECRET_KEY` | **Yes** | Without it, a new random key is generated on every cold start. Since sessions/CSRF tokens are signed with this key, users hitting a different serverless instance mid-session get silently logged out or see "session expired" (400) errors on form submits. Generate one with `python3 -c "import secrets; print(secrets.token_hex(32))"`. |
| `ADMIN_PASSWORD` | Recommended | Without it, the admin account falls back to a default password (`Admin@1234`) — a real security risk in production. |
| `GOOGLE_CLIENT_ID` / `GOOGLE_CLIENT_SECRET` | Only if using Google login | Without these, Google OAuth login won't work (email/password login still will). |
| `GEMINI_API_KEY` | Optional | Only used for an image-validation feature; app runs fine without it. |
| `VERCEL` | Set automatically by Vercel | Don't set manually — Vercel injects this itself, and the app uses it to detect that `/tmp` (not the project folder) must be used for temporary file storage. |

## The Vercel routing config

`vercel.json` must include a rewrite rule sending every path to `app.py` —
without it, Vercel has no rule telling it to route `/welcome`, `/admin/*`,
`/static/*`, etc. to your Flask app, and most routes will 404 even on an
otherwise-successful deploy:

```json
{
  "functions": { "app.py": { "maxDuration": 30 } },
  "rewrites": [{ "source": "/(.*)", "destination": "/app.py" }]
}
```

## Important limitation: uploaded photos are not persistent

On Vercel, complaint photo uploads are saved to `/tmp`, which is
**ephemeral** — files can disappear once the serverless instance recycles,
and a later request may land on a different instance that never had the
file. This doesn't crash anything, but an uploaded photo may 404 later.
For real persistent photo storage, integrate object storage (Vercel Blob,
S3, or Cloudinary) and store the returned URL instead of a local filename.
This is a real feature gap, not a bug — flagging it so it isn't a surprise.

## Google OAuth redirect URI (if using Google login)

The app builds its own redirect URI dynamically at runtime
(`url_for("google_authorize", _external=True)`), but Google will reject
the login attempt unless the *exact* resulting URL is pre-registered in
**Google Cloud Console → Credentials → your OAuth Client → Authorized
redirect URIs** — e.g. `https://your-app.vercel.app/google/authorize`.
This is configured in Google's console, not in this repo.

## Verifying a deployment

1. Visit a non-root page directly, e.g. `/welcome` — should redirect to
   login, not 404. Confirms the rewrite rule is working.
2. Log in as admin, wait ~60 seconds, then submit any form (e.g. edit a
   warden). If it fails with "session expired," `SECRET_KEY` isn't set
   correctly.
3. Check Vercel → your deployment → Functions → logs for the startup
   lines starting with `[HOSTEL APP]` — they'll tell you directly if
   `DATABASE_URL`, `SECRET_KEY`, or `ADMIN_PASSWORD` weren't picked up.
