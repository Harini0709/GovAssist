# Exam Compass

Exam Compass is a locally runnable exam-preparation coach: student accounts and profiles, exam discovery and tracking, a dynamically dated study plan, persisted quizzes and progress, notifications, weekly summaries, and a contextual chat with document upload and retrieval.

The included exam catalogue and practice questions are **illustrative sample data**, not current official recruitment notices. Eligibility matching uses the catalogue's per-exam minimum qualification, degree rules, age limits, and state/category/gender/percentage constraints when those values are actually present. Missing profile facts needed for age eligibility do not produce age-limited recommendations. This is a first-pass shortlist, not an official eligibility decision. A separate official-source field marks schedules taken from an official notice/calendar; all other date claims remain reported or unannounced. Always confirm eligibility, dates, application links, and notices with the recruiting authority.

## Requirements

- Python 3.10 or later
- Node.js 20 or later
- npm

## Backend setup (Windows)

From the project root:

```powershell
cd backend
python -m venv venv
venv\Scripts\activate
pip install -r requirements.txt
Copy-Item .env.example .env
uvicorn main:app --reload
```

The API is available at `http://localhost:8000`; interactive Swagger documentation is at `http://localhost:8000/docs`. The database is created and seeded on first start.

## Frontend

In another terminal, from the project root:

```powershell
cd frontend
npm install
npm run dev
```

Open `http://localhost:5173`. Copy `frontend/.env.example` to `frontend/.env` and set `VITE_API_URL` if the API is hosted somewhere other than `http://localhost:8000/api`.

## Environment variables

Copy the root `.env.example` to `.env` and set `SECRET_KEY` to a long, random value before deployment. If left empty for local development, the backend creates and persists a private random key at `backend/.secret_key`. The backend loads the root `.env` (or `backend/.env`) on startup. Never commit secrets or the generated key.

- `DATABASE_URL`: SQLite URL for local use; the data layer uses SQLite-compatible SQL and can be adapted to PostgreSQL.
- `SECRET_KEY`: signs the expiring bearer tokens.
- `CORS_ORIGINS`: comma-separated allowed frontend origins.
- `AI_PROVIDER`, `AI_API_KEY`, `AI_MODEL`, `AI_BASE_URL`: optional OpenAI-compatible chat-completions configuration. Without these, core features continue to work in offline chat mode. For Groq image understanding, use `AI_PROVIDER=groq`, your private key as `AI_API_KEY`, `qwen/qwen3.8-27b` as `AI_MODEL`, and `https://api.groq.com/openai/v1` as `AI_BASE_URL`. Check [Groq's vision model documentation](https://console.groq.com/docs/vision) for current model availability.
- `UPLOAD_DIR`: private local upload storage.

The backend reads `.env`, not `.env.example`. Copy the example to `.env`, add the AI settings there, and restart the backend. Never put a real key in an example file or commit it. PDF questions use extracted text; image analysis additionally requires a vision-capable model. Configured-provider errors are reported instead of silently switching to offline chat. Generic chat works without AI, but uploaded-document and image understanding returns a clear configuration error until a provider is configured.

## Database and data

SQLite database initialization, schema creation, and clearly unverified sample catalogue/question seeding run automatically at API startup. By default the database is `backend/exam_compass.db`. Keep the database and `backend/uploads/` private; they may contain personal profiles and documents.

Uploaded PDF, DOCX, and text files are extracted, stored per user, chunked, and indexed in SQLite FTS for relevant chat retrieval. Image uploads are safely stored; OCR/vision analysis is not enabled without a configured vision-capable AI integration. Max upload size is 10 MB.

## Data and feature boundaries

- The bundled exam catalogue and question pool are illustrative, unverified seed data. The PYQ-style pool is not represented as a verified historical paper, and the mock test is a short practice set rather than a full official-length exam.
- Exam discovery includes a maintained starter catalogue spanning 10th-, 12th-, diploma-, graduate-, engineering-, and postgraduate-level routes across SSC, railways, postal, banking, UPSC, defence, research, and public-sector recruitment. Profile education, degree, branch, age, category, and state are applied by the existing match endpoint. Students pursuing a degree match using their highest completed qualification; final-year eligibility is enabled only on an exam record that explicitly allows it and when the profile identifies the current graduating year. State-dependent recruitment families require a state in the profile; each actual state's open notification must be added separately when verified. Seeded catalogue additions are applied idempotently on startup without deleting user data.
- The dashboard lists every exam tracked by the signed-in student and displays its application deadline and exam date/window with the schedule's reported/official status. Schedule values that are not available from a confirmed authority remain unannounced; follow the linked official notice/site for updates.
- The exam table is migrated idempotently to store job/post title, minimum qualification, degree/branch/status restrictions, explicit final-year allowance, percentage and gender constraints, category rules, date/status/window metadata, source type, and last verification date. Exact schedules are not inferred from catalogue additions: records without reviewed dates remain unannounced. Recruitment-specific degree, branch, age, language, physical, marks, and category rules must still be verified against the current official notice. The catalogue is expandable rather than an exhaustive live register of every central and state vacancy; there is no unified official feed or automatic notice importer configured.
- A student pursuing a degree can call `POST /api/exams/match?stage=after_graduation` to request a separate advisory shortlist. The default `eligible_now` stage remains the existing frontend behavior. Future shortlist results are conditional and require rechecking age and the actual notice at graduation.
- The SSC 2026–27 tentative calendar is linked directly from SSC's official PDF. Its month windows remain `TENTATIVE`, not exact exam dates. Where an older secondary report conflicts with that calendar, the UI labels the cycle and source separately; it does not mix an SSC GD 2026 report into the calendar's SSC GD 2027 cycle. The saved calendar record was checked on 06 Oct 2026 and links to the primary source.
- An administrator can update an exam schedule using `PUT /api/admin/exams/{exam_id}/schedule`. Updates require an HTTPS source on the exam's listed official host, an allowed status, and validated ISO dates. Exact `ANNOUNCED` or past `COMPLETED` dates, tentative windows, application timing, official URLs, source type, and verification timestamp are persisted. This app does not automatically poll recruiting websites; unknown dates remain unknown until an official schedule is reviewed and entered.
- A practice set returns up to 10 questions. Questions are added idempotently to existing local databases, and previously answered questions are skipped until the available pool is exhausted.
- No live government notice/date feed is configured. Exact-date reminders run only for tracked records explicitly marked verified; tentative windows do not produce exact countdowns or reminders.
- When an official exam date is unavailable, the planner uses a clearly labeled 90-day personal target unless the user has set their own. If a saved personal target has passed, the planner refreshes it to a new 90-day target and rebuilds pending tasks while preserving completed tasks. Video links search YouTube for the current exam, subject, and topic; they are search results, not vetted or endorsed individual videos.
- SSC CGL 2026 dates shown as "reported" are sourced from the linked Testbook report, not verified official fields. Tier I's reported window is 30 September–30 October 2026, the reported application last date is 25 June 2026, and Tier II is described as tentative for December 2026. SSC GD 2026 dates shown as "reported" are sourced from the linked Sreedhar's CCE article: applications are reported as 1–31 December 2025 and CBT as 27 April–30 May 2026. These secondary-source dates are not marked verified in the catalogue; confirm them with SSC. Passed reported windows are labeled closed/passed in exam cards. Pending SSC CGL plan tasks after 30 October are automatically adapted to Tier II foundation practice; completed tasks are preserved.
- When an exam has no official date and the user has not set a personal target date, the study planner uses a clearly labeled 90-day personal planning target. This does not change or imply an official exam date.
- Document retrieval currently uses per-user SQLite full-text search and chunking. FAISS/pgvector embeddings are not bundled.
- Image uploads are sent to the configured chat-completions endpoint only when it is enabled; choose a vision-capable model. There is no local OCR.
- Admin exam creation is available through an admin-protected API endpoint; a separate admin web console and broader exam-editing workflow are not included.

## Architecture

```text
React + TypeScript + Tailwind (frontend)
             │ REST / bearer auth
FastAPI (backend/main.py)
   ├── SQLite: users, profiles, exams, plans, tasks, attempts, chats, documents
   ├── `backend/ai/`: matching, study/adaptive planning, readiness, chat, file extraction and FTS retrieval
   └── Optional AI provider + per-user FTS document retrieval
```

Student passwords are PBKDF2-HMAC-SHA256 hashed with per-password salts. API routes scope personal records to the authenticated user. The minimal admin exam-create endpoint requires a user whose database role is explicitly set to `admin`; regular registrations never grant this role. Do not expose this local SQLite/admin setup as a production deployment without proper operational hardening.

## Main API routes

- `POST /api/auth/register`, `POST /api/auth/login`
- `GET/PUT /api/users/profile`
- `GET /api/exams`, `GET /api/exams/{id}`, `POST /api/exams/match`
- `POST/GET /api/tracker`, `DELETE /api/tracker/{exam_id}`
- `POST /api/study-plan/generate`, `GET /api/study-plan/{exam_id}`, `POST /api/study-plan/update`
- `GET /api/quiz`, `POST /api/quiz/submit`; `/api/pyq` and `/api/mock` routes are also available
- `GET /api/progress`, `GET /api/progress/weekly`, `GET /api/readiness`
- `GET /api/notifications`, `POST /api/notifications/read`
- `PUT /api/admin/exams/{exam_id}/schedule` (admin only; records official-source schedule updates)
- `POST /api/chat`, `GET /api/chat/history`, `GET /api/chat/{session_id}`
- `POST /api/upload/{image|pdf|document}`

## Tests

Run the backend integration tests from `backend/`:

```powershell
python -m unittest -v test_api
```

Build-check the frontend from `frontend/`:

```powershell
npm run build
```

## Troubleshooting

- **Frontend cannot reach API:** start the backend on port 8000 and keep the frontend origin in `CORS_ORIGINS`.
- **`python` or `npm` is not found:** install Python/Node.js and reopen the terminal so PATH refreshes.
- **Invalid login:** passwords require at least eight characters at registration; use the email address in lowercase/uppercase as entered.
- **No exam dates shown:** the bundled catalogue intentionally has no fabricated dates. Official live notices have not been imported.
- **AI chat is general:** configure the optional provider settings in the backend process and restart it. Core features do not require an API key.
- **Uploaded PDF has no extracted text:** scanned/image-only PDFs need OCR, which is not bundled. Text-based PDFs and DOCX/TXT are supported.
- **Reset local development data:** stop the backend and remove only `backend/exam_compass.db` when you explicitly want to clear local accounts and progress. Back up user data first.

## ZIP-ready handoff

The project is self-contained in this directory. Exclude local `.env` files, virtual environments, `node_modules`, SQLite databases, and user uploads from any distributable ZIP. Include `.env.example`, source, and this README.
