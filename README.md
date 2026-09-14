# Al-Noor Institute — Website & Chatbot

A Next.js marketing website for **Al-Noor Institute** (a UK-based online Islamic
education provider) with a built-in streaming AI chat assistant, a course
enrolment form, and a password-protected admin dashboard for reviewing chat
transcripts and enrolment requests. Everything — marketing pages, API routes,
and admin panel — lives in a single Next.js App Router project; there is no
separate backend service.

## Overview

The site presents Al-Noor Institute's courses (Quran & Tajweed, Arabic,
Islamic Studies, Seerah, Hifz) with a full marketing page (hero, stats,
courses, features, pricing, FAQ, etc.), a 3-step enrolment form, and a
floating chat widget answering visitor questions using Google Gemini via the
Vercel AI SDK. Chat replies stream token-by-token, and if a database is
configured, conversations and enrolment requests are persisted and reviewable
from an internal `/admin` dashboard behind a simple JWT-based login.

The assistant is grounded in a fixed knowledge base (`prompts/knowledge-base.md`,
mirrored into `lib/server/prompt.ts`) describing the institute's courses,
fees, policies, and teachers — it is instructed to answer only from that
knowledge base rather than inventing information.

## Features

- **Marketing site** — Hero, stats, about, courses, features, how-it-works,
  teachers, pricing, and FAQ sections (`components/site/`), each lazy-loaded
  below the fold via `next/dynamic` with skeleton placeholders for a fast
  first paint.
- **AI chat widget** — Floating launcher (`components/chat/ChatWidget.tsx`),
  lazy-loaded client-side (`ChatWidgetLazy.tsx`) with a skeleton fallback so
  it never blocks the initial page render. Streams responses from Gemini
  through `POST /api/chat` and threads follow-up messages using a returned
  conversation ID.
- **Course enrolment form** — Multi-step form at `/enrol` that validates and
  submits to `POST /api/enrol`, saved to MongoDB when configured.
- **Admin dashboard** — `/admin` (stats overview), `/admin/conversations/[id]`
  (full chat transcript), and `/admin/enrolments` (enrolment requests), all
  behind a JWT-protected login at `/admin/login` with basic in-memory
  rate-limiting (5 attempts / 15 minutes per IP).
- **Legal pages** — `/privacy` and `/terms`, built on a shared
  `components/site/LegalPage.tsx` layout.
- **Resilient persistence** — MongoDB is optional. If `MONGODB_URI` is unset
  or the database is briefly unreachable, chat and the enrolment form still
  work (responses just aren't saved); connection attempts fail fast (~2.5s)
  instead of hanging requests.
- **Health check** — `GET /api/health` reports whether the database and AI
  provider are configured/reachable.

## Tech Stack

- **Framework:** [Next.js 16](https://nextjs.org/) (App Router, TypeScript)
- **UI:** React 19, Tailwind CSS 4
- **AI:** [Vercel AI SDK](https://sdk.vercel.ai/) (`ai`) + [`@ai-sdk/google`](https://sdk.vercel.ai/providers/ai-sdk-providers/google) (Google Gemini, model `gemini-2.5-flash`)
- **Database:** MongoDB via [Mongoose](https://mongoosejs.com/) (optional — used for chat transcripts and enrolment records)
- **Auth:** [`jsonwebtoken`](https://github.com/auth0/node-jsonwebtoken) for signing/verifying admin session tokens
- **Markdown rendering:** `react-markdown` (chat message formatting)
- **Linting:** ESLint 9 (`eslint-config-next`)
- **Package manager:** [pnpm](https://pnpm.io/)

## Project Structure

```
app/
  page.tsx                    Marketing homepage (composes components/site/*)
  layout.tsx                  Root layout, fonts, metadata, mounts ChatWidgetLazy
  enrol/page.tsx               Enrolment form
  privacy/page.tsx, terms/page.tsx   Legal pages
  admin/                       Admin dashboard pages (login, overview, conversations, enrolments)
  api/
    chat/route.ts               POST — streaming Gemini chat reply
    conversations/route.ts      GET/POST — list / create conversations
    conversations/[id]/route.ts GET/DELETE — read / delete a conversation
    enrol/route.ts               POST — submit an enrolment request
    health/route.ts              GET — DB + AI configuration/health status
    admin/login/route.ts         POST — admin login, returns a JWT
    admin/stats/route.ts         GET — dashboard totals (admin only)
    admin/conversations/route.ts GET — paginated conversation list (admin only)
    admin/conversations/[id]/route.ts GET — conversation detail (admin only)
    admin/enrolments/route.ts    GET — enrolment requests (admin only)
components/
  site/            Marketing page sections (Hero, Courses, Pricing, Faq, Footer, ...)
  chat/            ChatWidget + its lazy-loaded wrapper
  admin/           Shared admin dashboard header/nav
lib/
  server/
    db.ts            Cached Mongoose connection with short, fail-fast timeouts
    models.ts         Mongoose schemas: Conversation, Enrolment
    auth.ts            Admin credential check + JWT sign/verify
    prompt.ts           System prompt + knowledge base fed to Gemini
    http.ts             JSON response helpers
  admin.ts           Client-side fetch helpers for the admin dashboard (token storage, API calls)
  chat.ts            Shared chat types
  site-data.ts       Centralised marketing copy/content used by components/site/*
prompts/
  system-prompt.md        Source system prompt (mirrored in lib/server/prompt.ts)
  knowledge-base.md       Source knowledge base the assistant answers from
public/                Static assets
```

> Note: `components/ChatWindow.tsx`, `ConversationSidebar.tsx`,
> `MessageBubble.tsx`, `MessageInput.tsx`, and `TypingIndicator.tsx` are
> leftover UI pieces from an earlier iteration and are not used by the
> current chat widget (`components/chat/ChatWidget.tsx`).

## Getting Started

### Prerequisites

- Node.js 20+
- [pnpm](https://pnpm.io/) (this repo uses pnpm — see `pnpm-lock.yaml`)

### Setup

1. Install dependencies:

   ```bash
   pnpm install
   ```

2. Copy the example environment file and fill in your values (see table below):

   ```bash
   cp .env.example .env.local
   ```

3. Start the development server:

   ```bash
   pnpm dev
   ```

   Open [http://localhost:3000](http://localhost:3000).

The chat widget requires a Gemini API key to respond. The admin dashboard
(`/admin`) requires `ADMIN_EMAIL`, `ADMIN_PASSWORD`, and
`ADMIN_SESSION_SECRET`. MongoDB is optional; without it, chat and enrolment
still work but nothing is persisted or visible in `/admin`.

## Environment Variables

| Variable | Required for | Purpose |
|---|---|---|
| `GOOGLE_GENERATIVE_AI_API_KEY` | Chat | Google Gemini API key used by `POST /api/chat` via the Vercel AI SDK. Get one at [aistudio.google.com/app/apikey](https://aistudio.google.com/app/apikey). |
| `GEMINI_API_KEY` | Chat (fallback) | Alternate name accepted as a fallback if `GOOGLE_GENERATIVE_AI_API_KEY` is unset. |
| `MONGODB_URI` | Persistence | MongoDB connection string. Enables saved chat transcripts and enrolment records, and powers `/admin`. If unset, chat and enrolment still function statelessly. |
| `ADMIN_EMAIL` | Admin login | Email required to sign in at `/admin/login`. |
| `ADMIN_PASSWORD` | Admin login | Password required to sign in at `/admin/login`. |
| `ADMIN_SESSION_SECRET` | Admin login | Secret used to sign/verify admin JWT session tokens. Use a long random string. |
| `NEXT_PUBLIC_API_URL` | Optional | Overrides the API base URL used by the admin client (`lib/admin.ts`). Defaults to same-origin `/api`; only needed if the API is hosted on a different origin. |

If deploying to Vercel with MongoDB Atlas, allow network access from
`0.0.0.0/0` in Atlas (or add Vercel's IP ranges) so the deployed app can
connect.

## Available Scripts

| Command | Description |
|---|---|
| `pnpm dev` | Start the Next.js development server |
| `pnpm build` | Create a production build |
| `pnpm start` | Run the production build locally |
| `pnpm lint` | Run ESLint |

## Routes

| Route | Description |
|---|---|
| `/` | Marketing homepage with the chat widget |
| `/enrol` | Course enrolment form |
| `/privacy` | Privacy Policy |
| `/terms` | Terms & Conditions |
| `/admin/login` | Admin sign-in |
| `/admin` | Admin dashboard overview (stats) |
| `/admin/conversations/[id]` | Full chat transcript for a conversation |
| `/admin/enrolments` | List of submitted enrolment requests |

## Deployment

The project is a single Next.js app and deploys as one Vercel project (no
separate backend to configure):

1. Import this repository into Vercel.
2. Leave the root directory as the repo root — the Next.js framework preset
   is auto-detected.
3. Add the environment variables listed above under Project Settings →
   Environment Variables.
4. Deploy. If using MongoDB Atlas, make sure network access allows
   connections from Vercel (see note above).
