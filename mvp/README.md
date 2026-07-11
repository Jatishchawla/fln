# FLN Assessment Platform

Industry-structured MERN implementation of the FLN Assessment & Personalized
Worksheet Platform. This pass covers the **Super Admin** and **Teacher** roles.

```
mvp/
├── backend/   # Express + TypeScript + MongoDB (Mongoose) REST API
├── frontend/   # React + Vite + TypeScript SPA (talks to the real API)
└── docs/     # curriculum + workflow reference
```

## Quick Start

**Prerequisites:** Node.js 18+ (`node -v`). No database install needed — the
backend can run an embedded MongoDB for local dev.

Open **two terminals**.

**Terminal 1 — backend**
```bash
cd mvp/backend
cp .env.example .env      # default MONGODB_URI=memory → zero-setup embedded DB
npm install
npm run dev               # → http://localhost:4000
```
Expected: `Using in-memory MongoDB` → `MongoDB connected` → `FLN server running`.
A demo Super Admin + Teacher are seeded on first boot.

**Terminal 2 — frontend**
```bash
cd mvp/frontend
npm install
npm run dev               # → http://localhost:5173  (proxies /api → backend)
```

Open **http://localhost:5173** and log in (see [Demo accounts](#demo-accounts)).

### Database options
The backend reads `MONGODB_URI` from `mvp/backend/.env`:

| Value | Use |
|---|---|
| `memory` | **Default.** Embedded MongoDB, no install/network. Data resets on restart — great for dev/demo. |
| `mongodb://127.0.0.1:27017/fln` | A locally installed MongoDB (persistent). |
| `mongodb+srv://...` | MongoDB Atlas (cloud, persistent). If the SRV form fails with a DNS error, use Atlas's non-`srv` `mongodb://host1,host2,.../` connection string. |

> First `npm run dev` in `memory` mode downloads a small MongoDB binary once (needs
> internet that first time only); afterwards it runs fully offline.

---

## Why this is a rewrite of the earlier prototype

The earlier prototype worked but had structural problems typical of a first
build. This version fixes them:

| Problem in the prototype | Fixed here |
|---|---|
| Core logic ran in the **frontend** (a `fetch` interceptor + `localStorage` faked the backend) | All logic is in a real Express API; the client only calls it |
| Question generation + evaluation **duplicated** in frontend and backend | Single source of truth in `backend/src/domain` |
| God files (2700-line component, 2200-line data file) | Small, single-responsibility modules |
| Token = raw email string | JWT + bcrypt, role middleware |
| No layering | `routes → controllers → services → models`, with validation + central error handling |

The interns' genuinely good work — the **59-level curriculum question generator**
and the **weakest-level placement** logic — was kept and moved into the backend
domain layer as the single source of truth.

## Backend (`backend/`)

**Feature-first (modular monolith)** — grouped by domain module so the app scales
to the full 7-role platform: adding a feature means adding one folder under
`modules/`, not touching six shared folders.

```
backend/src/
├── config/                 env + Mongo connection
├── core/                   framework-level, cross-cutting
│   ├── middleware/         auth, validate (zod), errorHandler
│   ├── errors/             HttpError + helpers
│   └── utils/              asyncHandler
├── shared/                 types + enums + Mongoose base plugin (no I/O)
├── domain/                 pure curriculum logic: levelGenerator (59 levels) + level mapping
├── modules/                one self-contained folder per domain feature
│   ├── auth/               auth.{routes,controller,service,validators}
│   ├── users/              user.model
│   ├── students/           student.{model,service,controller,routes,validators}
│   ├── baseline/           worksheet.model + baseline/pdf/zip services + routes/validators
│   └── evaluation/         evaluation.{model,service}
├── routes.ts               mounts every module router
├── app.ts
├── server.ts
└── seed.ts
```

**Conventions that keep it scalable:** controllers stay thin; business logic lives
in services; a module's model is owned by that module; `domain/` is pure (no DB,
no module imports); `core/` is framework glue; `shared/` is types/enums only.

### Run
```bash
cd mvp/backend
cp .env.example .env         # then set MONGODB_URI (Atlas or local)
npm install
npm run dev                  # http://localhost:4000  (API at /api)
```
Seeds a demo Superadmin + Teacher on first boot.

## Frontend (`frontend/`)

```
frontend/src/
├── api/         typed fetch client (JWT in Authorization header)
├── context/     auth/session context
├── components/  Layout, Modal, Student/Evaluate/Report modals
└── pages/       Login, SuperAdminDashboard, TeacherDashboard
```

### Run
```bash
cd mvp/frontend
npm install
npm run dev                  # http://localhost:5173 (proxies /api → :4000)
```

## Demo accounts

| Role        | Email                | Password    |
|-------------|----------------------|-------------|
| Super Admin | `superadmin@fln.org` | `Admin@123` |
| Teacher     | `teacher@fln.org`    | `Teach@123` |

## Teacher flow

1. **Students** — add / edit / delete students (masked Aadhar id).
2. **Generate Baseline** — one click builds a per-student baseline paper
   (previous-class syllabus) from the 59-level curriculum.
3. **Download & Print** — downloads a ZIP of one A4 PDF per pending student.
4. **Upload Answers** — paste/upload each student's answer JSON.
5. **Evaluate** — scores against the answer key and places the student at the
   correct FLN level (weakest-level mapping), with a full report.

## API summary

| Method | Path | Role | Purpose |
|--------|------|------|---------|
| POST | `/api/auth/login` | public | Login → JWT |
| GET | `/api/auth/me` | any | Current user |
| POST/GET | `/api/auth/teachers` | superadmin | Create / list teachers |
| GET/POST/PUT/DELETE | `/api/students[...]` | teacher | Student CRUD |
| POST | `/api/baseline/generate` | teacher | Generate for pending students |
| GET | `/api/baseline/status` | teacher | Status summary |
| GET | `/api/baseline/download` | teacher | ZIP of test PDFs |
| POST | `/api/baseline/evaluate/:studentId` | teacher | Evaluate → assign level |
| GET | `/api/baseline/report/:studentId` | teacher | Evaluation report |
