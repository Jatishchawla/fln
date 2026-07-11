# Contributing Guide — FLN Assessment Platform

Read this **before writing any code**. It exists so we all build the same way and
avoid the mistakes that made the first prototype hard to maintain.

> **Golden rule:** the frontend never contains business logic. All logic lives in
> the backend API. The frontend only calls the API and shows the result.

---

## 1. Project layout

```
mvp/
├── backend/    # Express + TypeScript + MongoDB (Mongoose) — the ONLY source of truth
├── frontend/   # React + Vite + TypeScript — a thin client that calls the API
└── docs/       # curriculum + reference material
```

### Backend is feature-first (one folder per domain feature)
```
backend/src/
├── config/     env, database connection
├── core/       framework glue: middleware (auth, validate, error), errors, utils
├── shared/     types, enums, mongoose plugin  (NO database, NO business logic)
├── domain/     pure curriculum logic (question generator, level mapping) — NO database
└── modules/    the app — each feature is a self-contained folder:
    └── <feature>/
        ├── <feature>.model.ts        Mongoose schema
        ├── <feature>.service.ts      business logic (the real work)
        ├── <feature>.controller.ts   thin HTTP handlers
        ├── <feature>.routes.ts       endpoints + role guards
        └── <feature>.validators.ts   zod request schemas
```

### Frontend is feature-based
```
frontend/src/
├── api/         typed API client (adds the JWT header)
├── context/     auth/session
├── components/  reusable UI (Modal, Layout, ...)
└── pages/       one screen per role/feature
```

---

## 2. The rules (non-negotiable)

1. **No business logic in the frontend.** No scoring, no generation, no data rules.
   If you're tempted to compute something in React, it belongs in a backend service.
2. **Layers flow one way:** `routes → controller → service → model`.
   - **Controllers are thin** — read the request, call a service, return the result. No logic.
   - **Services do the work** — all business logic lives here.
   - **Models are owned by their module** — only that module's service imports its model.
3. **`domain/` is pure** — no database, no Express, no imports from `modules/`.
   Just functions in, data out (easy to test).
4. **`shared/` is types/enums only** — no logic, no I/O.
5. **Always validate input** with a zod schema in `<feature>.validators.ts`. Never
   trust `req.body`.
6. **Never leak secrets or internals** — no passwords in responses, mask the
   Aadhar/Birth-Certificate id, use `id` (not `_id`) in API output.
7. **One question = one competency/level.** Keep the curriculum mapping intact.

---

## 3. How to add a new feature (step by step)

Say you're adding **"tickets"**:

1. Create `backend/src/modules/tickets/`.
2. Add `ticket.model.ts` (Mongoose schema, apply the shared `basePlugin`).
3. Add `ticket.service.ts` — the logic (create, list, etc.).
4. Add `ticket.validators.ts` — zod schemas for the request bodies.
5. Add `ticket.controller.ts` — thin handlers calling the service.
6. Add `ticket.routes.ts` — endpoints + `authenticate` + `requireRole(...)`.
7. Mount it in `backend/src/routes.ts`: `router.use('/tickets', ticketRoutes)`.
8. Add matching calls to `frontend/src/api/client.ts` and a page/component.
9. **Verify it end-to-end** (see §6), then open a PR.

You should be able to add a feature by adding **one folder** — if you find yourself
editing five unrelated files, stop and ask.

---

## 4. Coding standards

- **TypeScript everywhere.** No `any` unless truly unavoidable (explain in a comment).
- **Naming:** files `kebab-case` (`ticket.service.ts`), types/classes `PascalCase`,
  variables/functions `camelCase`.
- **Small files, single responsibility.** If a file passes ~250 lines, split it.
- **Comments explain _why_, not _what_.** The code already says what.
- **Errors:** throw the typed helpers (`badRequest`, `notFound`, `forbidden`, ...)
  from `core/errors`. Never send raw 500s for expected cases.
- **Async:** wrap route handlers in `asyncHandler` so errors reach the error middleware.
- **No console.log spam** in committed code (a startup log or two is fine).

---

## 5. Git workflow

- **Never commit directly to `main`.** Branch first:
  `feat/<short-name>`, `fix/<short-name>`, `refactor/<short-name>`.
- **Commit per logical change**, not per file and not per day. Each commit should
  build/run green on its own. A few small commits a day is normal.
- **Conventional Commits** message format:
  ```
  feat(students): add roll-number uniqueness check
  fix(auth): reject expired JWT instead of 500
  refactor(baseline): extract pdf rendering into its own service
  docs: update README run steps
  ```
  Types: `feat` `fix` `refactor` `docs` `test` `chore`.
- **Open a Pull Request** for review. Keep PRs focused and small.
- **Never commit:** `node_modules/`, `.env`, `dist/`, secrets, API keys.
  These are gitignored — don't force-add them.
- If `git status` shows thousands of files, your `.gitignore` is broken — fix it,
  don't commit them.

---

## 6. Before you push — verify it works

Don't rely on "it compiles." Actually run the flow you changed.

```bash
# backend
cd mvp/backend
npm install
npm run typecheck        # must pass, zero errors
npm run dev              # boots with MONGODB_URI=memory (no DB install needed)

# frontend (second terminal)
cd mvp/frontend
npm install
npm run dev
```

Then click through the actual feature in the browser (http://localhost:5173).
For backend-only changes, hit the endpoint with `curl` and confirm the response.

Demo logins: `teacher@fln.org` / `Teach@123`, `superadmin@fln.org` / `Admin@123`.

---

## 7. Definition of Done (checklist for every PR)

- [ ] Logic is in a backend **service**, not a controller or the frontend.
- [ ] New feature is a self-contained **module** folder.
- [ ] Request bodies are **validated** with zod.
- [ ] `npm run typecheck` passes on the backend.
- [ ] Ran the flow end-to-end and it works.
- [ ] No secrets, `node_modules`, or `.env` in the diff.
- [ ] Commits follow Conventional Commits; PR is focused.

---

## 8. Common mistakes to avoid (learned the hard way)

| Don't | Do |
|---|---|
| Put scoring/generation in React | Put it in a backend service |
| Duplicate logic in frontend + backend | One source of truth (backend) |
| One giant 2000-line file | Small, single-purpose modules |
| Skip validation ("I'll trust the UI") | Validate every request with zod |
| `git add .` including `node_modules` | Rely on `.gitignore`; add only your files |
| Commit straight to `main` | Branch → PR → review |
| "It compiles, ship it" | Run the actual flow first |

Questions? Ask before building the wrong thing — it's cheaper than a rewrite.
