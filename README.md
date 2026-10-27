# TubeAgents — full-app version of the YouTube Agent Toolkit

Fourteen creator tools: the static site's 11, plus three vidIQ-style tools,
plus a real backend: Google sign-in, per-user memory, channel auto-fill,
ADK agent endpoints, and approve-gated publishing drafts. No tool was
rewritten: the browser engines are the same code as the static site,
verified against the Python originals.

## What the app does

| Piece | How it works |
|---|---|
| 14 tools | The 11 above, plus Keyword Explorer, Competitor Tracker, Best Time to Post — run in the browser, no key, no sign-in needed for the offline paths |
| Sign-in | "Sign in with Google" on the home page. Default scope is `youtube.readonly`. Tokens are stored server-side only, Fernet-encrypted; the browser never sees them |
| Auto-fill | Signed-in users get "Fill from my connected channel" on Audit (last 10 titles + dates) and Viral (videos + views). Snapshots are cached 24h: a full snapshot costs 3 quota units, repeat visits cost 0 |
| Memory | SQLite, per user: profile, channel snapshot, videos, comments, saved tool results, publish drafts. Every query filters on user id — a user only ever sees their own data |
| AI | `POST /api/ai/{script,audit,comment-reply,...}` runs the ADK agent pack. Needs `GEMINI_API_KEY`; without it the endpoint honestly returns "not configured" (HTTP 503) — never fake output |
| Publishing | Approve-gated only. `POST /api/publish/draft` stores a draft (no YouTube call). `POST /api/publish/approve` is the ONLY write path, and it runs only on an explicit approve call. There are no background jobs and no auto-posting, ever |

## vidIQ-style tools (12–14) — offline-first, no invented numbers

- **Keyword Explorer** (`/api/keywords/suggest`, `/api/keywords/door`, `/api/keywords/live`).
  Suggest reads public YouTube autocomplete — labelled "demand signals", never
  search volume. The door test scores pasted or searched top-10 results:
  small channels (under 10K subs), exact-phrase title matches, oldest/newest,
  median of known sub counts — verdict open door / tight / wall /
  inconclusive. Signed-in live search costs 101 units (search.list 100 +
  channels.list 1) and is cached 24h per phrase; repeat lookups cost 0.
- **Competitor Tracker** (`/api/competitors`, `/api/competitors/track`,
  `/api/competitors/resolve`). Sign-in required (tracking is per-user
  memory). Each snapshot stores subs, views, video count; the list shows
  deltas vs the previous snapshot. Handles can be resolved live (1 unit);
  unresolved handles and hand-typed numbers are stored and labelled
  "manual", never presented as API data.
- **Best Time to Post** (`/api/besttime/live`, `/api/besttime/paste`).
  Signed-in mode reads the YouTube Analytics API (views by hour, last 28
  days, own channel). Offline mode computes the same top 3 from pasted
  Studio "hour,views" rows. Empty input returns an honest empty result —
  hours are never fabricated.

## Run locally

```bash
cd ~/workspace/yt-app
python3 -m venv .venv
.venv/bin/pip install -r requirements.txt
cp .env.example .env   # fill in your values (never commit .env)
.venv/bin/python -m uvicorn backend.main:app --port 8000
```

Open http://localhost:8000. Without any keys the app still runs: all 11
tools work, `/api/health` reports `ai_configured: false,
oauth_configured: false`, and sign-in honestly reports "not configured".

Generate the two server secrets (never reuse across installs):

```bash
python3 -c "import secrets; print(secrets.token_hex(32))"   # SECRET_KEY
python3 -c "from cryptography.fernet import Fernet; print(Fernet.generate_key().decode())"  # FERNET_KEY
```

## Create the Google Cloud OAuth client (site owner, one time)

1. Google Cloud Console → create a project (or reuse one).
2. APIs & Services → Library → enable **YouTube Data API v3**.
3. APIs & Services → OAuth consent screen → User Type **External** →
   add the scopes your app requests (`youtube.readonly` now;
   `youtube.upload` only if you offer the publishing toggle) → add yourself
   as a test user while the app is unverified.
4. APIs & Services → Credentials → Create Credentials → **OAuth client ID**
   → Application type **Web application**.
5. Authorized JavaScript origins: your site origin, e.g.
   `http://localhost:8000` for local testing, and your production URL later.
6. Authorized redirect URIs: exactly
   `http://localhost:8000/api/auth/callback` locally, and
   `https://<your-domain>/api/auth/callback` in production.
7. Copy the Client ID + Client Secret into `.env` as
   `GOOGLE_OAUTH_CLIENT_ID` / `GOOGLE_OAUTH_CLIENT_SECRET` and restart.
   Creators then sign in with their OWN Google accounts; the site never
   sees anyone's password.

To enable publishing for your own account: sign in, turn ON the
"Enable publishing" toggle on the home page, and re-sign in when asked —
that second sign-in adds the `youtube.upload` scope. The toggle is OFF by
default and stays OFF until that re-sign-in succeeds.

## Deploy

**Cloudflare Pages alone cannot run this backend** — Pages serves static
files only, and this app needs a Python server for OAuth, tokens, and
memory. Pick one of these instead:

- **Render / Railway free tier (easiest):** create a Web Service from this
  folder, build `pip install -r requirements.txt`, start
  `python -m uvicorn backend.main:app --host 0.0.0.0 --port $PORT`.
  Set all `.env` values in the host's environment panel (never in the
  repo), then update the OAuth client's redirect URI + JS origin to the
  live URL and redeploy.
- **Any VPS:** same commands behind a reverse proxy (Caddy/nginx) with
  HTTPS; set the env vars in the service file, not in the repo.
- **Split option:** keep the static site on Cloudflare Pages for the
  tools, and point its `/api/*` calls at the backend host — this needs
  the frontend's `fetch("/api/...")` paths changed to the backend origin
  plus CORS. The single-host setup above is simpler; do the split only if
  you have a reason to.

## Tests (offline, no keys)

```bash
.venv/bin/python tests/test_app.py        # backend: 35 checks
node tests/test_engines_frontend.js       # frontend engines: 15 checks
```

Both exit non-zero on any FAIL. They use throwaway temp databases and
test-only secrets, and make no calls to Google or YouTube.

## Honest limits — read before promising anything

- **Full retention curve: unavailable via API.** Google's API does not
  expose it. Retention stays a YouTube Studio CSV upload, exactly as on
  the static site.
- **Unverified app warning + 100-user cap.** Until Google verifies the
  OAuth consent screen, new sign-ins see an "app isn't verified" warning
  and the project is capped at 100 users. Verification needs a privacy
  policy, a domain, and a review (weeks, not days).
- **Quota: 10,000 units/day per project.** A channel snapshot costs 3
  units; one video upload costs 1,600 units. Many creators uploading
  daily will hit the ceiling and need a quota increase request to Google.
- **Comment replies are draft-only in v1.** Posting a reply needs the
  `youtube.force-ssl` scope, which v1 does not request. Reply drafts stop
  at "approved" with an honest note; nothing is posted.
- **No real search volumes, ever.** YouTube's API does not expose them —
  not to this app, not to anyone. Autocomplete suggestions are demand
  signals only, and the door test is a heuristic over top-10 results, not
  a competition score. Any tool claiming exact YouTube search volume from
  the public API is inventing it.
- **Analytics needs sign-in.** Best Time to Post's live path reads the
  signed-in user's own Analytics; without sign-in it needs pasted Studio
  data, and with neither it says so instead of guessing hours.
- **AI live runs are untested in this build.** No Gemini key exists in
  this environment, so the keyed ADK path is implemented and key-gated
  but has never run end to end. The first keyed run is a verification
  step, not a claim.
- **Costs.** The static tools cost nothing. A hosted backend costs
  whatever the host charges, and Gemini usage is metered per the
  operator's Google plan. Nothing here is a promise of free-at-scale.
- **Security posture.** Tokens are Fernet-encrypted at rest and sessions
  are server-side invalidated on logout. The app never logs tokens, never
  puts secrets in page code, and never asks for or stores passwords.
