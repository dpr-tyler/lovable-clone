# Vibe

A **Lovable-style** web app: users sign in, create **projects**, and chat with an AI that runs in an **E2B code sandbox**. Each assistant turn can produce a **fragment**—a live preview URL plus the generated file tree—shown beside the conversation.

## What it does

- **Home (`/`)** — Create projects and open recent ones (public landing; auth required for the rest of the app via middleware).
- **Project workspace (`/projects/[projectId]`)** — Chat on the left; preview and code tabs on the right (file explorer, embedded sandbox host).
- **Credits** — Message sends consume credits stored in Postgres (`Usage`); free vs **Clerk Pro** plans get different monthly point pools (`src/lib/usage.ts`).
- **Background agent** — Sending a user message enqueues an **Inngest** function that runs **Inngest Agent Kit** + **OpenAI** inside an **E2B** Next.js template, then persists an assistant message and optional `Fragment` (sandbox URL, title, files JSON).

## Stack

| Layer | Choice |
|--------|--------|
| Framework | [Next.js](https://nextjs.org) 15 (App Router), React 19 |
| UI | Tailwind CSS 4, [shadcn-style](https://ui.shadcn.com) components (`src/components/ui`), Geist fonts |
| Auth | [Clerk](https://clerk.com) (`src/middleware.ts`) |
| API | [tRPC](https://trpc.io) + TanStack Query + [superjson](https://github.com/blitz-js/superjson) |
| Database | PostgreSQL + [Prisma](https://www.prisma.io) 7 (`@prisma/adapter-pg`) |
| Jobs | [Inngest](https://www.inngest.com) (`src/inngest/functions.ts`, `src/app/api/inngest/route.ts`) |
| Agent & sandbox | [@inngest/agent-kit](https://github.com/inngest/agent-kit), OpenAI models, [@e2b/code-interpreter](https://e2b.dev) |

## Project layout (high level)

- `src/app` — Routes: home, auth, pricing, project page, tRPC and Inngest API routes.
- `src/modules` — Feature code: `home`, `projects` (UI + server procedures), `messages`, `usage`.
- `src/inngest` — Inngest client and `code-agent` function (sandbox lifecycle, tools, DB writes).
- `src/trpc` — Router composition (`messages`, `projects`, `usage`).
- `prisma` — Schema: `Project`, `Message`, `Fragment`, `Usage`.

## Getting started

### Prerequisites

- Node.js (match your team’s LTS; project uses TypeScript and modern Next.js).
- PostgreSQL database URL.
- [Clerk](https://dashboard.clerk.com) application (publishable + secret keys).
- [Inngest](https://app.inngest.com) app (for dev, run the Inngest dev server; for production, configure signing keys as Inngest documents).
- [OpenAI](https://platform.openai.com) API access (used by Agent Kit).
- [E2B](https://e2b.dev) API key and a compatible sandbox template (the code references template `vibe-nextjs-test-2`; align this with your E2B setup).

### Environment variables

Set at least:

- `DATABASE_URL` — PostgreSQL connection string (used by Prisma and the usage limiter).
- Clerk: `NEXT_PUBLIC_CLERK_PUBLISHABLE_KEY`, `CLERK_SECRET_KEY` (and any other keys your Clerk Next.js setup requires).
- `NEXT_PUBLIC_APP_URL` — Absolute origin for server-side tRPC calls (e.g. `http://localhost:3000` in dev).

Also configure secrets for **Inngest**, **OpenAI**, and **E2B** per their SDKs and your deployment target (local Inngest dev vs cloud).

### Install and run

```bash
npm install
npx prisma migrate dev   # apply schema / create DB
npm run dev              # Next.js with Turbopack → http://localhost:3000
```

In another terminal, run the **Inngest dev server** so `code-agent/run` jobs execute locally (see [Inngest Next.js quick start](https://www.inngest.com/docs/sdk/overview)).

```bash
npm run build   # production build
npm run start   # production server
npm run lint    # ESLint
```

## API surface (tRPC)

Routers are defined in `src/trpc/routers/_app.ts`:

- **`projects`** — List/create projects scoped to the signed-in user.
- **`messages`** — List messages for a project; **create** inserts the user message, consumes a credit, and sends `code-agent/run` to Inngest.
- **`usage`** — Read remaining credits for the current user.

## Deploy notes

- **Vercel** (or similar): set all env vars; ensure Inngest can reach `/api/inngest` and that your E2B template and OpenAI usage match production limits.
- **Database**: run Prisma migrations against the production `DATABASE_URL`.
- **Clerk**: configure production URLs and allowed origins.

---

This repo started from `create-next-app`; the product and integrations above reflect the current codebase rather than the default Next.js template readme.
