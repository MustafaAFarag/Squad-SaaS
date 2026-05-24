# Squad — Phase 1 Design Spec
*Foundation + Envs + CI*
Date: 2026-05-24

---

## Product Context

**Squad** — lightweight multi-tenant team task manager SaaS. Multi-tenant, per-seat billing, auth flows, files, emails, observability. Phase 1 establishes the foundation only: scaffolding, database, auth wiring, error tracking, env config, CI.

---

## Design System

| Token | Value |
|-------|-------|
| Primary | `#7b39fc` (purple) |
| Dark | `#2b2344` (dark purple) |
| Font — UI/Nav | Manrope |
| Font — Buttons/Tags | Cabin |
| Font — Headlines | Instrument Serif |
| Font — Body | Inter |
| Style | Glassmorphism, video backgrounds, clean SaaS |

---

## Repository

- **GitHub:** `MustafaAFarag/Squad-SaaS` (SSH remote)
- **Package manager:** Bun (`bun.lockb`, replace `pnpm` with `bun` everywhere in master plan)
- **Runtime on Vercel:** Node (Bun is local dev/install only)

---

## File Structure

```
squad/
├── app/
│   ├── layout.tsx              # Root layout (fonts, Sentry)
│   ├── page.tsx                # Placeholder landing (env badge)
│   └── api/
│       └── health/
│           └── route.ts        # GET /api/health
├── prisma/
│   └── schema.prisma           # v1 schema
├── lib/
│   └── db.ts                   # Prisma client singleton
├── .env.local                  # Real secrets (gitignored)
├── .env.example                # Keys with empty values (committed)
└── .github/
    └── workflows/
        └── ci.yml              # lint + typecheck + build on PR
```

---

## Prisma Schema v1

```prisma
model User {
  id            String       @id @default(cuid())
  email         String       @unique
  name          String?
  emailVerified Boolean      @default(false)
  image         String?
  createdAt     DateTime     @default(now())
  updatedAt     DateTime     @updatedAt
  memberships   TeamMember[]
  tasks         Task[]
}

model Team {
  id        String       @id @default(cuid())
  name      String
  slug      String       @unique
  createdAt DateTime     @default(now())
  members   TeamMember[]
  tasks     Task[]
}

model TeamMember {
  id       String   @id @default(cuid())
  userId   String
  teamId   String
  role     Role     @default(MEMBER)
  joinedAt DateTime @default(now())
  user     User     @relation(fields: [userId], references: [id], onDelete: Cascade)
  team     Team     @relation(fields: [teamId], references: [id], onDelete: Cascade)
  @@unique([userId, teamId])
}

model Task {
  id          String     @id @default(cuid())
  title       String
  description String?
  status      TaskStatus @default(TODO)
  priority    Priority   @default(MEDIUM)
  teamId      String
  assigneeId  String?
  dueDate     DateTime?
  createdAt   DateTime   @default(now())
  updatedAt   DateTime   @updatedAt
  team        Team       @relation(fields: [teamId], references: [id], onDelete: Cascade)
  assignee    User?      @relation(fields: [assigneeId], references: [id])
}

enum Role        { OWNER ADMIN MEMBER VIEWER }
enum TaskStatus  { TODO IN_PROGRESS DONE CANCELLED }
enum Priority    { LOW MEDIUM HIGH URGENT }
```

---

## Environment Variables

```bash
# Database
DATABASE_URL=""

# BetterAuth
BETTER_AUTH_SECRET=""        # openssl rand -base64 32
BETTER_AUTH_URL=""           # http://localhost:3000 in dev

# Sentry
NEXT_PUBLIC_SENTRY_DSN=""
SENTRY_ORG=""
SENTRY_PROJECT=""
SENTRY_AUTH_TOKEN=""

# App
NEXT_PUBLIC_APP_URL=""
NEXT_PUBLIC_APP_ENV=""       # development | preview | staging | production
```

`.env.local` — real values, gitignored.
`.env.example` — keys only, committed as documentation.

---

## CI Pipeline

**File:** `.github/workflows/ci.yml`
**Trigger:** Every PR to `main`
**Steps:** `bun install` → `bun run lint` → `bun run typecheck` → `bun run build`
**Deploy:** Vercel GitHub integration handles preview deploys separately.

---

## Env Badge

Small pill in nav. Driven by `NEXT_PUBLIC_APP_ENV`:

| Value | Color |
|-------|-------|
| `development` | Blue |
| `preview` | Yellow |
| `staging` | Orange |
| `production` | Red |

---

## Health Check

`GET /api/health` → `200 { status: "ok", env: "development" }`

---

## Sentry Setup

Use wizard: `npx @sentry/wizard@latest -i nextjs --saas --org forward-bz --project squad-saas`

DSN: `https://66b0dd05ec6ba07a2f97744b1c1815d2@o4511446192291840.ingest.de.sentry.io/4511446193471568`

Wizard auto-wires: `instrumentation.ts`, source maps in `next.config.ts`, client/server init files.

---

## Exit Criteria

- [ ] `bun dev` runs, env badge visible at `localhost:3000`
- [ ] Prisma migrated against Neon EU Frankfurt, `prisma studio` shows all 4 tables
- [ ] Sentry captures a test error (visit `/sentry-example-page`)
- [ ] CI green on first PR
- [ ] `GET /api/health` returns 200

---

## Explicitly Out of Scope (Phase 1)

OAuth, email verification, file uploads, billing, Turnstile, feature flags, real marketing UI, Docker, Neon branching per PR. All deferred to their respective phases.
