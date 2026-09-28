# AI Interview Verifier

Proctored online interviews: candidates pick a position, consent to monitoring, pass a
camera-based identity gate, and are interviewed under supervision — while a recruiter
reviews the evidence afterwards. **The system never produces an automatic verdict**
(NFR-SAFE-01): flags, scores and reports exist to assist a human reviewer only.

**Stack:** FastAPI · PostgreSQL · OpenCV YuNet / MediaPipe / InsightFace (backend) —
React 19 · TypeScript · Vite · Tailwind CSS (frontend)

---

## How it works

1. **Register / login** — self-service candidate accounts; console access is role-gated.
2. **Pick a position** — choose a job posting; each posting brings its own questions
   (topped up from the shared question bank to 5 per interview).
3. **Disclosure & consent** — the candidate reads exactly what is monitored and must
   accept it before anything starts (NFR-PRIV-02).
4. **Identity gate** — reference face enrolment, liveness (blink / head turn), then a
   verification that **re-checks every second** during the interview. No verified
   identity → no interview (FR-ID-01..06).
5. **Monitored interview** — 5 timed questions while the app records a timestamped
   event timeline: focus/visibility, clipboard copy, print, page exit, fullscreen,
   keyboard activity (counts/shortcuts/typed text), mic level (never recorded),
   camera lifecycle, and identity results.
6. **Human review** — the recruiter opens the session, timeline, answers, identity
   history and the generated security report, and decides.

---

## Features

### Candidate
- Position board (9 seeded demo postings, all editable in the admin console)
- Consent/disclosure screen before camera or monitoring starts
- Identity verification with liveness and a per-second re-check; a frozen, paused or
  stopped camera makes identity *unverifiable* (amber) rather than falsely passing
- Per-question timers, overall duration from the posting, answer self-review at the end

### Monitoring (advisory only)
- **Identity:** every verify outcome — pass, no-face, multi-face — stored as a
  timestamped event for review; condensed logging avoids per-second row spam
- **Keyboard tracer:** keystroke counts, shortcut/devtools/page-search attempts, and
  characters typed into the interview's own inputs (documented cross-tab limits)
- **Browser:** focus loss, hidden tab, clipboard copy/paste snippets, print, leaving
  the page — classified `review` / `high_review`
- **Audio:** microphone level and silence detection only — **no audio is ever
  recorded, stored or uploaded**
- **External-resource / AI-assistant signals:** copy-out / paste-back / away-time
  correlated into "possible lookup" evidence, always labelled *not proof of cheating*
- **Incidents:** repeated flagged signals grouped for faster review

### Admin console (`/admin/login`)
- CRUD for job postings (title, department, location, description, duration) and their
  questions (MCQ / text / coding, with model answers and time limits)
- Interview list with flag summaries; per-session detail (identity, answers, timeline)
- One-click **security report** per interview
- Roles: `candidate` (default), `recruiter` (review only), `admin` (review + postings)

### Safety, privacy & security
- No automatic pass/fail anywhere — grading is a quiz score, reports say "for review"
- Server-side clock on all events keeps the shared timeline trustworthy (NFR-TRACE-01)
- JWT auth; review/admin endpoints enforce roles server-side (NFR-SEC-02) *and* in the UI;
  login and face-verify are throttled (sliding window → `429` + `Retry-After`)
- CORS restricted to the configured frontend origin
- Consent and privacy notices shown before monitoring and inside the dashboards

---

## Project structure

```
ai-interview-verifier/
├── README.md
├── pytest.ini                  # test suite config (run pytest from this folder)
├── docs/
│   └── SRS-TRACEABILITY.md     # requirement → implementation → tests matrix
├── backend/
│   ├── app/
│   │   ├── main.py             # FastAPI app, CORS, routers, /health, model warmup
│   │   ├── api/                # auth · face · interview · review · postings
│   │   ├── core/               # config (.env), security (JWT + role gates), database
│   │   ├── models/             # user · face · interview · posting (SQLAlchemy)
│   │   └── services/           # face_service, review_service, event_classification,
│   │                           # question_bank, grading, user_service
│   ├── tests/                  # pytest integration suite (13 tests)
│   ├── requirements.txt
│   └── requirements-dev.txt    # pytest + requests
└── frontend/
    └── src/
        ├── pages/              # Login, Register, Positions, Start, Enroll, Interview,
        │                       # Complete, Recruiter, Admin, AdminLogin, Postings
        ├── components/         # Webcam, FaceEnrollCheck, ProtectedRoute, ui, AppHeader
        ├── hooks/              # useKeyboardTracer, useAudioMonitor, useCursorTracker, useAuth
        ├── services/           # axios client + api modules (auth/interview/postings/review)
        └── lib/                # session storage helpers, event labels, utils
```

---

## Getting started

### Prerequisites
- **Python 3.11+** (3.13 tested) and **PostgreSQL 14+**
- **Node.js 20+** (npm)
- A webcam and microphone for the identity/monitoring flow

### 1. Database

```bash
psql -U postgres -c "CREATE DATABASE interview_verifier;"
```

### 2. Backend

```bash
cd backend
python -m venv venv
venv\Scripts\activate          # macOS/Linux: source venv/bin/activate
pip install -r requirements.txt
uvicorn app.main:app --reload --port 8000
```

On startup the backend:
- creates tables (plus small in-code migrations for older databases),
- **seeds 9 demo job postings** when the postings table is empty,
- grants the console role to every email in `REVIEWER_EMAILS`,
- downloads/initialises the InsightFace + MediaPipe models on first run.

Configuration lives in `backend/.env` (see the table below); `.env` ships with working
dev defaults.

### 3. Frontend

```bash
cd frontend
npm install
npm run dev                    # http://localhost:5173
```

### 4. Sign in

- **Candidates:** register at `/register`, then pick a position.
- **Admin console:** `/admin/login` — the seeded admin is `kafeel@gmail.com`
  (via `REVIEWER_EMAILS`); candidates who visit the console are rejected.

---

## Configuration (`backend/.env`)

| Key | Purpose | Dev default |
|---|---|---|
| `DATABASE_URL` | PostgreSQL DSN | `postgresql://postgres@127.0.0.1:5432/interview_verifier` |
| `SECRET_KEY` | JWT signing key — **change for production** | dev placeholder |
| `ALGORITHM` | JWT algorithm | `HS256` |
| `ACCESS_TOKEN_EXPIRE_MINUTES` | Session lifetime | `43200` (30 days) |
| `FRONTEND_URL` | Only allowed CORS origin | `http://localhost:5173` |
| `REVIEWER_EMAILS` | Comma-separated emails granted console access on startup | `kafeel@gmail.com` |
| `DEBUG` | SQL echo + verbose logging | `true` |
| `API_V1_PREFIX` | API mount point | `/api/v1` |
| `LOGIN_RATE_LIMIT` / `LOGIN_RATE_WINDOW_SECONDS` | Login attempts per client IP + email before `429` + `Retry-After` (0 disables) | `10` / `60` |
| `VERIFY_RATE_LIMIT` / `VERIFY_RATE_WINDOW_SECONDS` | `/face/verify` calls per candidate before `429` — sized for the per-second re-check (~60/min) | `100` / `60` |

---

## API

Base URL `http://127.0.0.1:8000/api/v1` — interactive docs at **`/docs`**, health at **`/health`**.

| Group | Endpoints |
|---|---|
| Auth | `POST /auth/register` · `POST /auth/login` · `GET /auth/me` |
| Face | `POST /face/enroll` · `POST /face/verify` · `POST /face/liveness` · `GET /face/reference/{id}` · `GET /face/events/{interview_id}` |
| Interview | `POST /interview/start` · `GET /interview/{id}` · `POST /interview/{id}/answer` · `POST /interview/{id}/event` · `POST /interview/{id}/end` |
| Postings | `GET /postings` (candidate) · `GET/POST /admin/postings` · `PUT/DELETE /admin/postings/{id}` · `POST/PUT/DELETE /admin/postings/{id}/questions[...]` |
| Review | `GET /review/interviews` · `GET /review/interviews/{id}` · `GET /review/interviews/{id}/report` (recruiter/admin) |

---

## Testing

The backend has an **integration pytest suite** (22 tests covering posting CRUD +
question draw, identity-absence recording, one-identity-per-candidate enrolment,
monitoring/report evidence, the shuffled draw (FR-Q-03), and login/verify rate
limiting):

```bash
# start the backend first - the suite talks to the running server
cd ai-interview-verifier
backend\venv\Scripts\python -m pytest           # -v for details
```

- Skips with a clear reason if uvicorn isn't running; `AIV_BASE_URL` overrides the target.
- Creates throwaway accounts/postings and removes everything afterwards, even on failure.

Frontend checks:

```bash
cd frontend
npm run lint                   # oxlint
npm run build                  # tsc + vite build
```

---

## Troubleshooting

- **Port 8000 busy / stale server on Windows:** asyncio's `SO_REUSEADDR` lets a second
  uvicorn bind the same port and silently serve old code. Check
  `netstat -ano | findstr ":8000" | findstr LISTENING` and stop orphaned PIDs, then
  restart uvicorn once. If edits don't take effect despite `--reload`, the WatchFiles
  reloader is stuck (seen twice on Windows) — kill the uvicorn supervisor + worker PIDs
  and start it once; running **without** `--reload` and restarting by hand after backend
  edits avoids it entirely.
- **Identity stays "pending":** the browser has no camera permission (or the frame is
  stale/stopped) — presence is unverifiable by design; allow the camera and retry.
- **Another person passes the identity check:** accounts enrolled before the replace
  fix held several *stacked* reference faces, and `/face/verify` matches the **best** of
  them — so each person passed against their own row (a second identity sits ~0.30 cosine
  from the first, far from the 0.55 threshold, but it was still on file). Re-enrol once
  on `/enroll`: an account now stores exactly one face, any other person scores ~0.3,
  fails, and the interview shows the **"Person not identified"** notice.
- **First run is slow:** InsightFace/MediaPipe model files are downloaded once.
- **Registration rejected for `.test` addresses:** the validator blocks reserved
  special-use domains — use a real TLD such as `.com`.

## Known gaps

See [docs/SRS-TRACEABILITY.md](docs/SRS-TRACEABILITY.md) — frontend unit tests,
an incident-correlation fixture, and formal database migrations are not in place yet.
