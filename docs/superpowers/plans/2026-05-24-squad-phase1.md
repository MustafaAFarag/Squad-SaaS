# Squad Phase 1 — Foundation + Envs + CI Implementation Plan

> **For agentic workers:** REQUIRED SUB-SKILL: Use superpowers:subagent-driven-development (recommended) or superpowers:executing-plans to implement this plan task-by-task. Steps use checkbox (`- [ ]`) syntax for tracking.

**Goal:** Scaffold Squad with Prisma + Neon, BetterAuth (email/password), Sentry, env config, health check, GitHub Actions CI, and a landing page matching the design system (Manrope/Cabin/Instrument Serif/Inter, purple `#7b39fc`).

**Architecture:** Next.js 16 App Router monolith, no `src/` directory. Prisma talks to Neon EU Frankfurt. BetterAuth sessions via cookies. Sentry wired via wizard. Landing page is a static marketing surface — no auth required.

**Tech Stack:** Next.js 16, Bun, Tailwind v4, Prisma 6, BetterAuth, Sentry, GitHub Actions, Neon Postgres

---

## File Map

| File | Action | Responsibility |
|------|--------|----------------|
| `app/layout.tsx` | Modify | Load Google Fonts, wire Sentry, set metadata |
| `app/globals.css` | Modify | Design tokens (colors, fonts via CSS vars) |
| `app/page.tsx` | Modify | Landing page (all marketing sections) |
| `app/api/health/route.ts` | Create | GET /api/health |
| `app/api/auth/[...all]/route.ts` | Create | BetterAuth catch-all route |
| `lib/auth.ts` | Create | BetterAuth instance config |
| `lib/auth-client.ts` | Create | BetterAuth browser client |
| `lib/db.ts` | Create | Prisma client singleton |
| `prisma/schema.prisma` | Create | User, Team, TeamMember, Task models |
| `components/navbar.tsx` | Create | Transparent nav with logo, links, env badge |
| `components/env-badge.tsx` | Create | Colored pill showing current env |
| `components/hero.tsx` | Create | Full-screen video hero section |
| `components/features.tsx` | Create | Feature highlights section |
| `components/how-it-works.tsx` | Create | 3-step workflow section |
| `components/pricing.tsx` | Create | Pricing tiers section |
| `components/testimonials.tsx` | Create | Social proof section |
| `components/cta.tsx` | Create | Bottom CTA section |
| `components/footer.tsx` | Create | Footer with links |
| `.env.local` | Create | Real secrets (gitignored) |
| `.env.example` | Create | Keys with empty values (committed) |
| `.gitignore` | Modify | Add `.env.local` if missing |
| `.github/workflows/ci.yml` | Create | lint + typecheck + build on PR |
| `next.config.ts` | Modify | Sentry source maps (wizard handles this) |

---

## Task 1: Env Files + .gitignore

**Files:**
- Create: `.env.local`
- Create: `.env.example`
- Modify: `.gitignore`

- [ ] **Step 1: Create `.env.example`**

```bash
# Database
DATABASE_URL=""

# BetterAuth
BETTER_AUTH_SECRET=""
BETTER_AUTH_URL=""

# Sentry
NEXT_PUBLIC_SENTRY_DSN=""
SENTRY_ORG=""
SENTRY_PROJECT=""
SENTRY_AUTH_TOKEN=""

# App
NEXT_PUBLIC_APP_URL=""
NEXT_PUBLIC_APP_ENV=""
```

- [ ] **Step 2: Create `.env.local` with real values**

```bash
DATABASE_URL="postgresql://neondb_owner:npg_ljIEALgT3r2o@ep-sparkling-fire-alper2hp-pooler.c-3.eu-central-1.aws.neon.tech/neondb?sslmode=require&channel_binding=require"

BETTER_AUTH_SECRET=""
BETTER_AUTH_URL="http://localhost:3000"

NEXT_PUBLIC_SENTRY_DSN="https://66b0dd05ec6ba07a2f97744b1c1815d2@o4511446192291840.ingest.de.sentry.io/4511446193471568"
SENTRY_ORG="forward-bz"
SENTRY_PROJECT="squad-saas"
SENTRY_AUTH_TOKEN=""

NEXT_PUBLIC_APP_URL="http://localhost:3000"
NEXT_PUBLIC_APP_ENV="development"
```

For `BETTER_AUTH_SECRET`, run in terminal and paste the output:
```bash
openssl rand -base64 32
```

- [ ] **Step 3: Verify `.gitignore` has `.env.local`**

Open `.gitignore`. If `.env.local` is not present, add this block:
```
# env
.env.local
.env*.local
```

- [ ] **Step 4: Commit `.env.example` only**

```bash
git add .env.example .gitignore
git commit -m "chore: add env example and gitignore"
```

---

## Task 2: Prisma Setup + Schema

**Files:**
- Create: `prisma/schema.prisma`
- Create: `lib/db.ts`

- [ ] **Step 1: Install Prisma**

```bash
bun add prisma @prisma/client
bunx prisma init --datasource-provider postgresql
```

This creates `prisma/schema.prisma` and adds `DATABASE_URL` to `.env` (ignore that file — we use `.env.local`).

- [ ] **Step 2: Replace `prisma/schema.prisma` content**

```prisma
generator client {
  provider = "prisma-client-js"
}

datasource db {
  provider = "postgresql"
  url      = env("DATABASE_URL")
}

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

enum Role {
  OWNER
  ADMIN
  MEMBER
  VIEWER
}

enum TaskStatus {
  TODO
  IN_PROGRESS
  DONE
  CANCELLED
}

enum Priority {
  LOW
  MEDIUM
  HIGH
  URGENT
}
```

- [ ] **Step 3: Run migration against Neon**

```bash
bunx prisma migrate dev --name init
```

Expected output: `✔ Generated Prisma Client` + migration applied. If it fails with SSL error, verify `DATABASE_URL` in `.env.local` has `?sslmode=require`.

- [ ] **Step 4: Verify in Prisma Studio**

```bash
bunx prisma studio
```

Open `http://localhost:5555`. You should see: `User`, `Team`, `TeamMember`, `Task` tables. Close studio when done.

- [ ] **Step 5: Create `lib/db.ts`**

```typescript
import { PrismaClient } from "@prisma/client";

const globalForPrisma = globalThis as unknown as { prisma: PrismaClient };

export const db =
  globalForPrisma.prisma ??
  new PrismaClient({
    log: process.env.NODE_ENV === "development" ? ["error", "warn"] : ["error"],
  });

if (process.env.NODE_ENV !== "production") globalForPrisma.prisma = db;
```

- [ ] **Step 6: Commit**

```bash
git add prisma/ lib/db.ts
git commit -m "feat: add prisma schema v1 and db client"
```

---

## Task 3: BetterAuth Setup

**Files:**
- Create: `lib/auth.ts`
- Create: `lib/auth-client.ts`
- Create: `app/api/auth/[...all]/route.ts`

- [ ] **Step 1: Install BetterAuth**

```bash
bun add better-auth
```

- [ ] **Step 2: Create `lib/auth.ts`**

```typescript
import { betterAuth } from "better-auth";
import { prismaAdapter } from "better-auth/adapters/prisma";
import { db } from "./db";

export const auth = betterAuth({
  database: prismaAdapter(db, {
    provider: "postgresql",
  }),
  emailAndPassword: {
    enabled: true,
  },
  trustedOrigins: [process.env.BETTER_AUTH_URL ?? "http://localhost:3000"],
});
```

- [ ] **Step 3: Generate BetterAuth schema additions**

```bash
bunx better-auth generate
```

This outputs SQL or Prisma additions for BetterAuth's session/account/verification tables. Apply them:

```bash
bunx prisma migrate dev --name better-auth
```

- [ ] **Step 4: Create `app/api/auth/[...all]/route.ts`**

```typescript
import { auth } from "@/lib/auth";
import { toNextJsHandler } from "better-auth/next-js";

export const { GET, POST } = toNextJsHandler(auth);
```

- [ ] **Step 5: Create `lib/auth-client.ts`**

```typescript
import { createAuthClient } from "better-auth/react";

export const authClient = createAuthClient({
  baseURL: process.env.NEXT_PUBLIC_APP_URL ?? "http://localhost:3000",
});
```

- [ ] **Step 6: Verify auth route responds**

Start dev server: `bun dev`

Visit `http://localhost:3000/api/auth/get-session` in browser.
Expected: `{"session":null,"user":null}` (no error).

- [ ] **Step 7: Commit**

```bash
git add lib/auth.ts lib/auth-client.ts app/api/auth/
git commit -m "feat: wire BetterAuth with email+password"
```

---

## Task 4: Health Check Endpoint

**Files:**
- Create: `app/api/health/route.ts`

- [ ] **Step 1: Create `app/api/health/route.ts`**

```typescript
import { NextResponse } from "next/server";

export async function GET() {
  return NextResponse.json({
    status: "ok",
    env: process.env.NEXT_PUBLIC_APP_ENV ?? "unknown",
    timestamp: new Date().toISOString(),
  });
}
```

- [ ] **Step 2: Verify**

With `bun dev` running, visit `http://localhost:3000/api/health`.
Expected:
```json
{ "status": "ok", "env": "development", "timestamp": "2026-..." }
```

- [ ] **Step 3: Commit**

```bash
git add app/api/health/
git commit -m "feat: add health check endpoint"
```

---

## Task 5: Sentry Setup

**Files:** wizard handles `instrumentation.ts`, `sentry.client.config.ts`, `sentry.server.config.ts`, `next.config.ts`

- [ ] **Step 1: Run Sentry wizard**

```bash
npx @sentry/wizard@latest -i nextjs --saas --org forward-bz --project squad-saas
```

Follow prompts. When asked for DSN, it auto-detects from your Sentry project. Accept all defaults.

- [ ] **Step 2: Add `SENTRY_AUTH_TOKEN` to `.env.local`**

The wizard will print a token. Copy it into `.env.local`:
```bash
SENTRY_AUTH_TOKEN="<token from wizard>"
```

Also add to `.env.example`:
```bash
SENTRY_AUTH_TOKEN=""
```

- [ ] **Step 3: Verify Sentry captures errors**

Start dev server: `bun dev`

Visit `http://localhost:3000/sentry-example-page` (wizard creates this page).
Click the error button.
Check `https://forward-bz.sentry.io` — you should see the issue appear within 30 seconds.

- [ ] **Step 4: Commit**

```bash
git add instrumentation.ts sentry.client.config.ts sentry.server.config.ts next.config.ts .env.example
git commit -m "feat: wire Sentry SDK via wizard"
```

---

## Task 6: Design System + Fonts

**Files:**
- Modify: `app/layout.tsx`
- Modify: `app/globals.css`

- [ ] **Step 1: Update `app/layout.tsx`**

```typescript
import type { Metadata } from "next";
import {
  Manrope,
  Cabin,
  Instrument_Serif,
  Inter,
} from "next/font/google";
import "./globals.css";

const manrope = Manrope({
  variable: "--font-manrope",
  subsets: ["latin"],
  display: "swap",
});

const cabin = Cabin({
  variable: "--font-cabin",
  subsets: ["latin"],
  display: "swap",
});

const instrumentSerif = Instrument_Serif({
  variable: "--font-instrument-serif",
  subsets: ["latin"],
  weight: "400",
  display: "swap",
});

const inter = Inter({
  variable: "--font-inter",
  subsets: ["latin"],
  display: "swap",
});

export const metadata: Metadata = {
  title: "Squad — Team Task Manager",
  description:
    "Lightweight multi-tenant team task manager. Invite your team, assign tasks, ship faster.",
};

export default function RootLayout({
  children,
}: Readonly<{
  children: React.ReactNode;
}>) {
  return (
    <html
      lang="en"
      className={`${manrope.variable} ${cabin.variable} ${instrumentSerif.variable} ${inter.variable} h-full antialiased`}
    >
      <body className="min-h-full flex flex-col">{children}</body>
    </html>
  );
}
```

- [ ] **Step 2: Update `app/globals.css`**

```css
@import "tailwindcss";

:root {
  --color-primary: #7b39fc;
  --color-dark: #2b2344;
  --color-background: #ffffff;
  --color-foreground: #171717;
}

@theme inline {
  --color-primary: var(--color-primary);
  --color-dark: var(--color-dark);
  --color-background: var(--color-background);
  --color-foreground: var(--color-foreground);
  --font-manrope: var(--font-manrope);
  --font-cabin: var(--font-cabin);
  --font-instrument-serif: var(--font-instrument-serif);
  --font-inter: var(--font-inter);
}

body {
  background: var(--color-background);
  color: var(--color-foreground);
  font-family: var(--font-inter), Arial, sans-serif;
}
```

- [ ] **Step 3: Verify fonts load**

```bash
bun dev
```

Open `http://localhost:3000`. Open DevTools → Network → filter by "font". You should see Manrope, Cabin, Instrument_Serif, Inter requests.

- [ ] **Step 4: Commit**

```bash
git add app/layout.tsx app/globals.css
git commit -m "feat: wire design system fonts and color tokens"
```

---

## Task 7: Env Badge Component

**Files:**
- Create: `components/env-badge.tsx`

- [ ] **Step 1: Create `components/env-badge.tsx`**

```typescript
const ENV_STYLES: Record<string, string> = {
  development: "bg-blue-500 text-white",
  preview: "bg-yellow-400 text-black",
  staging: "bg-orange-500 text-white",
  production: "bg-red-500 text-white",
};

export function EnvBadge() {
  const env = process.env.NEXT_PUBLIC_APP_ENV ?? "development";
  const styles = ENV_STYLES[env] ?? "bg-gray-500 text-white";

  if (env === "production") return null;

  return (
    <span
      className={`font-[family-name:var(--font-cabin)] text-xs font-medium px-2 py-0.5 rounded-full ${styles}`}
    >
      {env.toUpperCase()}
    </span>
  );
}
```

- [ ] **Step 2: Commit**

```bash
git add components/env-badge.tsx
git commit -m "feat: add env badge component"
```

---

## Task 8: Navbar Component

**Files:**
- Create: `components/navbar.tsx`

- [ ] **Step 1: Create `components/navbar.tsx`**

```typescript
"use client";

import { useState } from "react";
import { ChevronDown, Menu, X } from "lucide-react";
import { EnvBadge } from "./env-badge";

const NAV_LINKS = [
  { label: "Home", href: "/" },
  { label: "Services", href: "#services", hasDropdown: true },
  { label: "Reviews", href: "#reviews" },
  { label: "Contact us", href: "#contact" },
];

export function Navbar() {
  const [mobileOpen, setMobileOpen] = useState(false);

  return (
    <nav className="absolute top-0 left-0 right-0 z-20 flex items-center justify-between px-6 lg:px-[120px] py-4">
      {/* Logo */}
      <a href="/" className="flex items-center gap-2">
        <svg
          width="28"
          height="20"
          viewBox="0 0 28 20"
          fill="none"
          xmlns="http://www.w3.org/2000/svg"
        >
          <path
            d="M1.04356 6.35771L13.6437 0.666504L26.2438 6.35771V13.6423L13.6437 19.3335L1.04356 13.6423V6.35771Z"
            fill="white"
          />
        </svg>
        <span className="font-[family-name:var(--font-manrope)] font-semibold text-white text-lg">
          Squad
        </span>
        <EnvBadge />
      </a>

      {/* Desktop Nav Links */}
      <ul className="hidden lg:flex items-center gap-8">
        {NAV_LINKS.map((link) => (
          <li key={link.label}>
            <a
              href={link.href}
              className="flex items-center gap-1 font-[family-name:var(--font-manrope)] font-medium text-sm text-white hover:opacity-80 transition-opacity"
            >
              {link.label}
              {link.hasDropdown && <ChevronDown className="w-4 h-4" />}
            </a>
          </li>
        ))}
      </ul>

      {/* Desktop Action Buttons */}
      <div className="hidden lg:flex items-center gap-3">
        <a
          href="/sign-in"
          className="font-[family-name:var(--font-manrope)] font-semibold text-sm text-[#171717] bg-white border border-[#d4d4d4] rounded-lg px-4 py-2 hover:bg-gray-50 transition-colors"
        >
          Sign In
        </a>
        <a
          href="/sign-up"
          className="font-[family-name:var(--font-manrope)] font-semibold text-sm text-[#fafafa] bg-[#7b39fc] rounded-lg px-4 py-2 shadow-md hover:bg-[#6a2de0] transition-colors"
        >
          Get Started
        </a>
      </div>

      {/* Mobile Hamburger */}
      <button
        className="lg:hidden text-white"
        onClick={() => setMobileOpen(true)}
        aria-label="Open menu"
      >
        <Menu className="w-6 h-6" />
      </button>

      {/* Mobile Overlay Menu */}
      {mobileOpen && (
        <div className="fixed inset-0 z-50 bg-black flex flex-col items-center justify-center gap-8">
          <button
            className="absolute top-4 right-6 text-white"
            onClick={() => setMobileOpen(false)}
            aria-label="Close menu"
          >
            <X className="w-6 h-6" />
          </button>
          {NAV_LINKS.map((link) => (
            <a
              key={link.label}
              href={link.href}
              className="font-[family-name:var(--font-manrope)] font-medium text-xl text-white hover:opacity-80"
              onClick={() => setMobileOpen(false)}
            >
              {link.label}
            </a>
          ))}
          <a
            href="/sign-in"
            className="font-[family-name:var(--font-manrope)] font-semibold text-base text-[#171717] bg-white rounded-lg px-6 py-3"
          >
            Sign In
          </a>
          <a
            href="/sign-up"
            className="font-[family-name:var(--font-manrope)] font-semibold text-base text-white bg-[#7b39fc] rounded-lg px-6 py-3"
          >
            Get Started
          </a>
        </div>
      )}
    </nav>
  );
}
```

- [ ] **Step 2: Install lucide-react**

```bash
bun add lucide-react
```

- [ ] **Step 3: Commit**

```bash
git add components/navbar.tsx
git commit -m "feat: add navbar with env badge and mobile menu"
```

---

## Task 9: Hero Section

**Files:**
- Create: `components/hero.tsx`

- [ ] **Step 1: Create `components/hero.tsx`**

```typescript
export function Hero() {
  return (
    <section className="relative min-h-screen flex flex-col overflow-hidden">
      {/* Video Background */}
      <video
        autoPlay
        loop
        muted
        playsInline
        className="absolute inset-0 w-full h-full object-cover"
      >
        <source
          src="https://d8j0ntlcm91z4.cloudfront.net/user_38xzZboKViGWJOttwIXH07lWA1P/hf_20260210_031346_d87182fb-b0af-4273-84d1-c6fd17d6bf0f.mp4"
          type="video/mp4"
        />
      </video>

      {/* Hero Content */}
      <div className="relative z-10 flex flex-col items-center text-center mt-32 px-6">
        {/* Tagline Pill */}
        <div
          className="flex items-center gap-2 px-3 h-[38px] rounded-[10px] border mb-8"
          style={{
            background: "rgba(85,80,110,0.4)",
            backdropFilter: "blur(8px)",
            borderColor: "rgba(164,132,215,0.5)",
          }}
        >
          <span className="font-[family-name:var(--font-cabin)] font-medium text-sm text-white bg-[#7b39fc] px-2 py-0.5 rounded-[6px]">
            New
          </span>
          <span className="font-[family-name:var(--font-cabin)] font-medium text-sm text-white">
            Introducing Squad v1.0
          </span>
        </div>

        {/* Headline */}
        <h1
          className="font-[family-name:var(--font-instrument-serif)] text-white text-5xl lg:text-[96px] max-w-4xl mb-6"
          style={{ lineHeight: 1.1 }}
        >
          Manage your team tasks instantly{" "}
          <em className="not-italic italic tracking-tight">and</em>{" "}
          effortlessly
        </h1>

        {/* Subtext */}
        <p className="font-[family-name:var(--font-inter)] text-white/70 text-lg max-w-[662px] mb-10">
          Invite your team, assign tasks with priorities and due dates, and
          track progress together. One workspace, every role, zero friction.
        </p>

        {/* CTA Buttons */}
        <div className="flex items-center gap-4 flex-wrap justify-center">
          <a
            href="/sign-up"
            className="font-[family-name:var(--font-cabin)] font-medium text-base text-white bg-[#7b39fc] rounded-[10px] px-6 py-3 hover:bg-[#6a2de0] transition-colors"
          >
            Start for Free
          </a>
          <a
            href="#features"
            className="font-[family-name:var(--font-cabin)] font-medium text-base text-[#f6f7f9] bg-[#2b2344] rounded-[10px] px-6 py-3 hover:bg-[#3d3160] transition-colors"
          >
            See How It Works
          </a>
        </div>
      </div>
    </section>
  );
}
```

- [ ] **Step 2: Commit**

```bash
git add components/hero.tsx
git commit -m "feat: add hero section with video background"
```

---

## Task 10: Features Section

**Files:**
- Create: `components/features.tsx`

- [ ] **Step 1: Create `components/features.tsx`**

```typescript
import { CheckSquare, Users, BarChart3, Shield } from "lucide-react";

const FEATURES = [
  {
    icon: CheckSquare,
    title: "Task Management",
    description:
      "Create, assign, and track tasks with priorities, due dates, and statuses. Full-text search included.",
  },
  {
    icon: Users,
    title: "Team Workspaces",
    description:
      "Invite teammates via email. Roles: Owner, Admin, Member, Viewer. Switch between multiple teams instantly.",
  },
  {
    icon: BarChart3,
    title: "Progress Tracking",
    description:
      "Filter by assignee, status, priority. State lives in the URL — share any view with a link.",
  },
  {
    icon: Shield,
    title: "Secure by Default",
    description:
      "Email verification, bot protection on every form, signed file URLs. Your data never leaks.",
  },
];

export function Features() {
  return (
    <section id="features" className="bg-white py-24 px-6 lg:px-[120px]">
      <div className="max-w-6xl mx-auto">
        {/* Section Label */}
        <div className="flex justify-center mb-4">
          <span className="font-[family-name:var(--font-cabin)] font-medium text-sm text-[#7b39fc] bg-purple-50 border border-purple-200 px-3 py-1 rounded-full">
            Features
          </span>
        </div>

        <h2 className="font-[family-name:var(--font-instrument-serif)] text-[#171717] text-4xl lg:text-5xl text-center mb-4">
          Everything your team needs
        </h2>
        <p className="font-[family-name:var(--font-inter)] text-gray-500 text-lg text-center max-w-xl mx-auto mb-16">
          Built for small teams who want to move fast without the overhead of
          enterprise tools.
        </p>

        <div className="grid grid-cols-1 md:grid-cols-2 lg:grid-cols-4 gap-8">
          {FEATURES.map((f) => (
            <div
              key={f.title}
              className="flex flex-col gap-4 p-6 rounded-2xl border border-gray-100 hover:border-purple-200 hover:shadow-md transition-all"
            >
              <div className="w-10 h-10 rounded-xl bg-purple-50 flex items-center justify-center">
                <f.icon className="w-5 h-5 text-[#7b39fc]" />
              </div>
              <h3 className="font-[family-name:var(--font-manrope)] font-semibold text-[#171717] text-lg">
                {f.title}
              </h3>
              <p className="font-[family-name:var(--font-inter)] text-gray-500 text-sm leading-relaxed">
                {f.description}
              </p>
            </div>
          ))}
        </div>
      </div>
    </section>
  );
}
```

- [ ] **Step 2: Commit**

```bash
git add components/features.tsx
git commit -m "feat: add features section"
```

---

## Task 11: How It Works Section

**Files:**
- Create: `components/how-it-works.tsx`

- [ ] **Step 1: Create `components/how-it-works.tsx`**

```typescript
const STEPS = [
  {
    number: "01",
    title: "Create your workspace",
    description:
      "Sign up, verify your email, and create a team in under 2 minutes. No credit card required.",
  },
  {
    number: "02",
    title: "Invite your team",
    description:
      "Send email invites. Teammates accept, get a role, and land straight in the shared task board.",
  },
  {
    number: "03",
    title: "Ship together",
    description:
      "Create tasks, set priorities, assign owners, filter by anything. Watch progress in real time.",
  },
];

export function HowItWorks() {
  return (
    <section id="how-it-works" className="bg-[#2b2344] py-24 px-6 lg:px-[120px]">
      <div className="max-w-6xl mx-auto">
        <div className="flex justify-center mb-4">
          <span className="font-[family-name:var(--font-cabin)] font-medium text-sm text-purple-300 bg-white/10 border border-white/20 px-3 py-1 rounded-full">
            How it works
          </span>
        </div>

        <h2 className="font-[family-name:var(--font-instrument-serif)] text-white text-4xl lg:text-5xl text-center mb-4">
          Up and running in minutes
        </h2>
        <p className="font-[family-name:var(--font-inter)] text-white/60 text-lg text-center max-w-xl mx-auto mb-16">
          No complex onboarding. No sales call. Just sign up and start shipping.
        </p>

        <div className="grid grid-cols-1 md:grid-cols-3 gap-8">
          {STEPS.map((step) => (
            <div key={step.number} className="flex flex-col gap-4">
              <span className="font-[family-name:var(--font-instrument-serif)] text-[#7b39fc] text-5xl">
                {step.number}
              </span>
              <h3 className="font-[family-name:var(--font-manrope)] font-semibold text-white text-xl">
                {step.title}
              </h3>
              <p className="font-[family-name:var(--font-inter)] text-white/60 text-base leading-relaxed">
                {step.description}
              </p>
            </div>
          ))}
        </div>
      </div>
    </section>
  );
}
```

- [ ] **Step 2: Commit**

```bash
git add components/how-it-works.tsx
git commit -m "feat: add how it works section"
```

---

## Task 12: Pricing Section

**Files:**
- Create: `components/pricing.tsx`

- [ ] **Step 1: Create `components/pricing.tsx`**

```typescript
import { Check } from "lucide-react";

const PLANS = [
  {
    name: "Free",
    price: "$0",
    description: "For small teams getting started.",
    features: ["3 members", "100 tasks", "Email support", "1 workspace"],
    cta: "Get started",
    href: "/sign-up",
    highlighted: false,
  },
  {
    name: "Pro",
    price: "$12",
    per: "/ user / mo",
    description: "For growing teams that need more.",
    features: [
      "Unlimited members",
      "Unlimited tasks",
      "Priority support",
      "Multiple workspaces",
      "File attachments",
      "Advanced filters",
    ],
    cta: "Start free trial",
    href: "/sign-up?plan=pro",
    highlighted: true,
  },
  {
    name: "Team",
    price: "$20",
    per: "/ user / mo",
    description: "For teams that need admin controls.",
    features: [
      "Everything in Pro",
      "Audit log",
      "SSO (coming soon)",
      "Custom roles",
      "Priority SLA",
    ],
    cta: "Contact us",
    href: "#contact",
    highlighted: false,
  },
];

export function Pricing() {
  return (
    <section id="pricing" className="bg-white py-24 px-6 lg:px-[120px]">
      <div className="max-w-6xl mx-auto">
        <div className="flex justify-center mb-4">
          <span className="font-[family-name:var(--font-cabin)] font-medium text-sm text-[#7b39fc] bg-purple-50 border border-purple-200 px-3 py-1 rounded-full">
            Pricing
          </span>
        </div>

        <h2 className="font-[family-name:var(--font-instrument-serif)] text-[#171717] text-4xl lg:text-5xl text-center mb-4">
          Simple, transparent pricing
        </h2>
        <p className="font-[family-name:var(--font-inter)] text-gray-500 text-lg text-center max-w-xl mx-auto mb-16">
          Start free. Upgrade when your team grows.
        </p>

        <div className="grid grid-cols-1 md:grid-cols-3 gap-8 items-start">
          {PLANS.map((plan) => (
            <div
              key={plan.name}
              className={`rounded-2xl p-8 border flex flex-col gap-6 ${
                plan.highlighted
                  ? "bg-[#7b39fc] border-[#7b39fc] shadow-xl shadow-purple-200"
                  : "bg-white border-gray-100"
              }`}
            >
              <div>
                <p
                  className={`font-[family-name:var(--font-cabin)] font-medium text-sm mb-2 ${
                    plan.highlighted ? "text-purple-200" : "text-[#7b39fc]"
                  }`}
                >
                  {plan.name}
                </p>
                <div className="flex items-baseline gap-1">
                  <span
                    className={`font-[family-name:var(--font-instrument-serif)] text-4xl ${
                      plan.highlighted ? "text-white" : "text-[#171717]"
                    }`}
                  >
                    {plan.price}
                  </span>
                  {plan.per && (
                    <span
                      className={`font-[family-name:var(--font-inter)] text-sm ${
                        plan.highlighted ? "text-purple-200" : "text-gray-400"
                      }`}
                    >
                      {plan.per}
                    </span>
                  )}
                </div>
                <p
                  className={`font-[family-name:var(--font-inter)] text-sm mt-2 ${
                    plan.highlighted ? "text-purple-100" : "text-gray-500"
                  }`}
                >
                  {plan.description}
                </p>
              </div>

              <ul className="flex flex-col gap-3">
                {plan.features.map((f) => (
                  <li key={f} className="flex items-center gap-3">
                    <Check
                      className={`w-4 h-4 flex-shrink-0 ${
                        plan.highlighted ? "text-white" : "text-[#7b39fc]"
                      }`}
                    />
                    <span
                      className={`font-[family-name:var(--font-inter)] text-sm ${
                        plan.highlighted ? "text-white" : "text-gray-600"
                      }`}
                    >
                      {f}
                    </span>
                  </li>
                ))}
              </ul>

              <a
                href={plan.href}
                className={`font-[family-name:var(--font-cabin)] font-medium text-base text-center rounded-[10px] px-6 py-3 transition-colors mt-auto ${
                  plan.highlighted
                    ? "bg-white text-[#7b39fc] hover:bg-purple-50"
                    : "bg-[#7b39fc] text-white hover:bg-[#6a2de0]"
                }`}
              >
                {plan.cta}
              </a>
            </div>
          ))}
        </div>
      </div>
    </section>
  );
}
```

- [ ] **Step 2: Commit**

```bash
git add components/pricing.tsx
git commit -m "feat: add pricing section"
```

---

## Task 13: Testimonials Section

**Files:**
- Create: `components/testimonials.tsx`

- [ ] **Step 1: Create `components/testimonials.tsx`**

```typescript
const TESTIMONIALS = [
  {
    quote:
      "We replaced three different tools with Squad. The invite flow is so clean our non-technical founder set it up herself.",
    name: "Sara K.",
    role: "CTO, Fintech Startup",
    initials: "SK",
  },
  {
    quote:
      "The URL-based filters are a game changer. I send my PM a link and she sees exactly what I see. No more 'which filter are you on?'",
    name: "Ahmed M.",
    role: "Lead Engineer",
    initials: "AM",
  },
  {
    quote:
      "Migrated from Jira in one afternoon. The whole team was onboarded by end of day. We haven't looked back.",
    name: "Lena T.",
    role: "Product Manager",
    initials: "LT",
  },
];

export function Testimonials() {
  return (
    <section id="reviews" className="bg-gray-50 py-24 px-6 lg:px-[120px]">
      <div className="max-w-6xl mx-auto">
        <div className="flex justify-center mb-4">
          <span className="font-[family-name:var(--font-cabin)] font-medium text-sm text-[#7b39fc] bg-purple-50 border border-purple-200 px-3 py-1 rounded-full">
            Reviews
          </span>
        </div>

        <h2 className="font-[family-name:var(--font-instrument-serif)] text-[#171717] text-4xl lg:text-5xl text-center mb-16">
          Teams love Squad
        </h2>

        <div className="grid grid-cols-1 md:grid-cols-3 gap-8">
          {TESTIMONIALS.map((t) => (
            <div
              key={t.name}
              className="bg-white rounded-2xl p-8 border border-gray-100 flex flex-col gap-6"
            >
              <p className="font-[family-name:var(--font-inter)] text-gray-700 text-base leading-relaxed">
                &ldquo;{t.quote}&rdquo;
              </p>
              <div className="flex items-center gap-3 mt-auto">
                <div className="w-10 h-10 rounded-full bg-[#7b39fc] flex items-center justify-center">
                  <span className="font-[family-name:var(--font-manrope)] font-semibold text-white text-sm">
                    {t.initials}
                  </span>
                </div>
                <div>
                  <p className="font-[family-name:var(--font-manrope)] font-semibold text-[#171717] text-sm">
                    {t.name}
                  </p>
                  <p className="font-[family-name:var(--font-inter)] text-gray-400 text-xs">
                    {t.role}
                  </p>
                </div>
              </div>
            </div>
          ))}
        </div>
      </div>
    </section>
  );
}
```

- [ ] **Step 2: Commit**

```bash
git add components/testimonials.tsx
git commit -m "feat: add testimonials section"
```

---

## Task 14: CTA Section

**Files:**
- Create: `components/cta.tsx`

- [ ] **Step 1: Create `components/cta.tsx`**

```typescript
export function CTA() {
  return (
    <section
      id="contact"
      className="bg-[#7b39fc] py-24 px-6 lg:px-[120px]"
    >
      <div className="max-w-3xl mx-auto flex flex-col items-center text-center gap-6">
        <h2 className="font-[family-name:var(--font-instrument-serif)] text-white text-4xl lg:text-5xl">
          Ready to ship faster?
        </h2>
        <p className="font-[family-name:var(--font-inter)] text-white/80 text-lg max-w-lg">
          Join teams that moved off spreadsheets and clunky project tools.
          Start free — no credit card needed.
        </p>
        <div className="flex items-center gap-4 flex-wrap justify-center">
          <a
            href="/sign-up"
            className="font-[family-name:var(--font-cabin)] font-medium text-base text-[#7b39fc] bg-white rounded-[10px] px-8 py-3 hover:bg-purple-50 transition-colors"
          >
            Get started free
          </a>
          <a
            href="#how-it-works"
            className="font-[family-name:var(--font-cabin)] font-medium text-base text-white bg-[#2b2344] rounded-[10px] px-8 py-3 hover:bg-[#3d3160] transition-colors"
          >
            Learn more
          </a>
        </div>
      </div>
    </section>
  );
}
```

- [ ] **Step 2: Commit**

```bash
git add components/cta.tsx
git commit -m "feat: add CTA section"
```

---

## Task 15: Footer

**Files:**
- Create: `components/footer.tsx`

- [ ] **Step 1: Create `components/footer.tsx`**

```typescript
const FOOTER_LINKS = {
  Product: [
    { label: "Features", href: "#features" },
    { label: "Pricing", href: "#pricing" },
    { label: "Changelog", href: "/changelog" },
  ],
  Company: [
    { label: "About", href: "/about" },
    { label: "Blog", href: "/blog" },
    { label: "Contact", href: "#contact" },
  ],
  Legal: [
    { label: "Privacy", href: "/privacy" },
    { label: "Terms", href: "/terms" },
  ],
};

export function Footer() {
  return (
    <footer className="bg-[#2b2344] py-16 px-6 lg:px-[120px]">
      <div className="max-w-6xl mx-auto">
        <div className="grid grid-cols-2 md:grid-cols-4 gap-8 mb-12">
          <div className="col-span-2 md:col-span-1">
            <span className="font-[family-name:var(--font-manrope)] font-semibold text-white text-xl">
              Squad
            </span>
            <p className="font-[family-name:var(--font-inter)] text-white/50 text-sm mt-3 max-w-xs">
              Lightweight team task manager for teams that ship.
            </p>
          </div>
          {Object.entries(FOOTER_LINKS).map(([group, links]) => (
            <div key={group}>
              <p className="font-[family-name:var(--font-manrope)] font-semibold text-white text-sm mb-4">
                {group}
              </p>
              <ul className="flex flex-col gap-3">
                {links.map((link) => (
                  <li key={link.label}>
                    <a
                      href={link.href}
                      className="font-[family-name:var(--font-inter)] text-white/50 text-sm hover:text-white/80 transition-colors"
                    >
                      {link.label}
                    </a>
                  </li>
                ))}
              </ul>
            </div>
          ))}
        </div>

        <div className="border-t border-white/10 pt-8">
          <p className="font-[family-name:var(--font-inter)] text-white/30 text-sm text-center">
            © {new Date().getFullYear()} Squad. Built for learning.
          </p>
        </div>
      </div>
    </footer>
  );
}
```

- [ ] **Step 2: Commit**

```bash
git add components/footer.tsx
git commit -m "feat: add footer"
```

---

## Task 16: Wire Landing Page

**Files:**
- Modify: `app/page.tsx`

- [ ] **Step 1: Update `app/page.tsx`**

```typescript
import { Navbar } from "@/components/navbar";
import { Hero } from "@/components/hero";
import { Features } from "@/components/features";
import { HowItWorks } from "@/components/how-it-works";
import { Pricing } from "@/components/pricing";
import { Testimonials } from "@/components/testimonials";
import { CTA } from "@/components/cta";
import { Footer } from "@/components/footer";

export default function Home() {
  return (
    <main>
      <Navbar />
      <Hero />
      <Features />
      <HowItWorks />
      <Pricing />
      <Testimonials />
      <CTA />
      <Footer />
    </main>
  );
}
```

- [ ] **Step 2: Verify full page renders**

```bash
bun dev
```

Open `http://localhost:3000`. Scroll through all sections:
- Hero: video background playing, headline, tagline pill, two CTA buttons
- Features: 4 cards
- How It Works: 3 steps on dark background
- Pricing: 3 plans, Pro card purple-highlighted
- Testimonials: 3 cards on gray background
- CTA: purple section
- Footer: dark with links

- [ ] **Step 3: Commit**

```bash
git add app/page.tsx
git commit -m "feat: wire landing page with all sections"
```

---

## Task 17: GitHub Actions CI

**Files:**
- Create: `.github/workflows/ci.yml`

- [ ] **Step 1: Create `.github/workflows/ci.yml`**

```yaml
name: CI

on:
  pull_request:
    branches: [main]
  push:
    branches: [main]

jobs:
  ci:
    runs-on: ubuntu-latest
    steps:
      - uses: actions/checkout@v4

      - uses: oven-sh/setup-bun@v2
        with:
          bun-version: latest

      - name: Install dependencies
        run: bun install --frozen-lockfile

      - name: Lint
        run: bun run lint

      - name: Typecheck
        run: bun run typecheck

      - name: Build
        run: bun run build
        env:
          DATABASE_URL: ${{ secrets.DATABASE_URL }}
          BETTER_AUTH_SECRET: ${{ secrets.BETTER_AUTH_SECRET }}
          BETTER_AUTH_URL: https://example.com
          NEXT_PUBLIC_APP_URL: https://example.com
          NEXT_PUBLIC_APP_ENV: preview
          NEXT_PUBLIC_SENTRY_DSN: ${{ secrets.NEXT_PUBLIC_SENTRY_DSN }}
          SENTRY_ORG: ${{ secrets.SENTRY_ORG }}
          SENTRY_PROJECT: ${{ secrets.SENTRY_PROJECT }}
          SENTRY_AUTH_TOKEN: ${{ secrets.SENTRY_AUTH_TOKEN }}
```

- [ ] **Step 2: Add `typecheck` script to `package.json`**

Open `package.json`. Add to `scripts`:
```json
"typecheck": "tsc --noEmit"
```

- [ ] **Step 3: Add GitHub Actions secrets**

Go to `https://github.com/MustafaAFarag/Squad-SaaS/settings/secrets/actions`.

Add these secrets:
- `DATABASE_URL` — your Neon connection string
- `BETTER_AUTH_SECRET` — same value as in `.env.local`
- `NEXT_PUBLIC_SENTRY_DSN` — the Sentry DSN
- `SENTRY_ORG` — `forward-bz`
- `SENTRY_PROJECT` — `squad-saas`
- `SENTRY_AUTH_TOKEN` — from wizard output

- [ ] **Step 4: Commit and push to trigger CI**

```bash
git add .github/workflows/ci.yml package.json
git commit -m "ci: add GitHub Actions lint + typecheck + build"
git push
```

- [ ] **Step 5: Verify CI passes**

Go to `https://github.com/MustafaAFarag/Squad-SaaS/actions`. Watch the run. Expected: all 3 steps green.

---

## Phase 1 Done Checklist

- [ ] `bun dev` runs, all landing sections visible at `localhost:3000`
- [ ] Env badge shows `DEV` in blue in navbar
- [ ] Prisma migrated against Neon, `prisma studio` shows 4 tables + BetterAuth tables
- [ ] `GET /api/health` returns `{ status: "ok", env: "development" }`
- [ ] `GET /api/auth/get-session` returns `{ session: null, user: null }`
- [ ] Sentry test error visible in Sentry dashboard
- [ ] CI green on GitHub Actions
- [ ] No `.env.local` committed (verify with `git log --oneline`)
