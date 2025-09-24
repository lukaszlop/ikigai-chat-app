# IkigAI Chat App (MVP)

![Node](https://img.shields.io/badge/node-22.14.0-339933?logo=node.js)
![Next.js](https://img.shields.io/badge/Next.js-15-black?logo=nextdotjs)
![React](https://img.shields.io/badge/React-19-61DAFB?logo=react&logoColor=black)
![TypeScript](https://img.shields.io/badge/TypeScript-5-3178C6?logo=typescript&logoColor=white)
![Version](https://img.shields.io/badge/version-0.1.0-blue)
![License](https://img.shields.io/badge/license-Unlicensed-lightgrey)

A lightweight chat application that demonstrates real‑time AI response streaming, attachment handling, simple profile personalization, and mocked authentication. The app targets a desktop-first developer environment without a production backend or persistent database.

## Table of Contents

- [1. Project name](#1-project-name)
- [2. Project description](#2-project-description)
- [3. Tech stack](#3-tech-stack)
- [4. Getting started locally](#4-getting-started-locally)
- [5. Available scripts](#5-available-scripts)
- [6. Project scope](#6-project-scope)
- [7. Project status](#7-project-status)
- [8. License](#8-license)

## 1. Project name

IkigAI Chat App (MVP)

## 2. Project description

IkigAI Chat App is an MVP that enables conversations with a multimodal language model and supports attaching images and text files within a single conversation. It showcases:

- Real-time streaming via Vercel AI SDK `streamText()` with a skeleton placeholder until TTFT.
- Attachments: `jpg`, `png`, `txt`, `md`. Limits: up to 3 files, ≤5 MB each, ≤15 MB total. Drag & drop, paste, and file picker supported; previews for images and concise info for `txt/md`.
- Message and content limits: input field up to 10,000 characters; combined limit up to 30,000 characters for text + `txt/md` content. Hard blocking and clear messaging on overflow.
- Server-side image normalization with `sharp`: auto-orient, max longer side 2048 px, EXIF stripped; WEBP q≈80 or PNG for text/transparency.
- Controls: Stop (keep partial), Retry (on error), Regenerate (new answer to the same message).
- Context window: approximately last 6 messages + older summaries.
- Markdown rendering, URL auto-detection opening in a new tab with safe attributes; code blocks with syntax highlighting and “Copy”.
- Offline handling: sending blocked with a clear message; retry after reconnect.
- Profile: Name (editable), Email (read-only, equals login), Avatar upload (`jpg/png/webp`) with 1:1 crop, min 256×256, ≤2 MB; saved in `localStorage`.
- Session and state: chat history and UI state in `sessionStorage` (no persistent DB; no multi-thread history).
- Daily cost budget: enforced 0.5 USD/day with a hard send block once exceeded.

Mocked authentication:

- Test credentials: `test@example.com` / `password123`.
- Access to Chat and Profile gated by an in-session login stored in `sessionStorage`.

## 3. Tech stack

- Frameworks: Next.js 15, React 19, TypeScript 5, Tailwind CSS 4.
- State management: Zustand 5.
- UI components: Shadcn/ui, `lucide-react`, `tailwind-merge`.
- AI: Vercel AI SDK (`ai`), `@ai-sdk/openai` targeting OpenAI `gpt-4o-mini` (vision + text).
- Files & media: `react-dropzone` (DnD/paste), `react-easy-crop` (avatar crop).
- Markdown & code: `react-markdown`, `remark-gfm`, `rehype-highlight`, `rehype-sanitize`, `highlight.js`.
- Server image processing: `sharp`.
- Validation & utils: `zod`, `clsx`, `class-variance-authority`.
- Tooling: ESLint 9, TypeScript 5, Tailwind 4, Next.js Turbopack.
- Node.js: 22.14.0 (see `.nvmrc`).

Project structure:

```text
./src                - source code
./src/app            - Next.js App Router pages and layouts
./src/app/api        - API routes (server-side endpoints)
./src/app/globals.css- Global CSS styles
./src/components     - React components
./src/components/ui  - Shadcn/ui components
./src/lib            - Utility functions, services, helpers
./src/types.ts       - Shared TypeScript types and interfaces
./src/stores         - Zustand stores
./src/hooks          - Custom hooks
./src/assets         - Static internal assets
./public             - Public assets via URL
./middleware.ts      - Next.js middleware (root)
```

## 4. Getting started locally

Prerequisites:

- Node.js 22.14.0 (recommended via `nvm`)
- npm 10+ (uses `package-lock.json`)

Setup:

```bash
# Use the exact Node version
nvm use

# Install dependencies
npm install

# Run the dev server (Turbopack)
npm run dev
# → http://localhost:3000
```

Optional environment:

```bash
# If you plan to call OpenAI via Vercel AI SDK, set:
# Create .env.local and add:
OPENAI_API_KEY=your_openai_api_key
```

Mock login (dev):

- Email: `test@example.com`
- Password: `password123`

Production-like build:

```bash
npm run build
npm run start
# → http://localhost:3000
```

Lint:

```bash
npm run lint
```

## 5. Available scripts

- `npm run dev`: Start Next.js dev server with Turbopack.
- `npm run build`: Build the app with Turbopack.
- `npm run start`: Start the production server.
- `npm run lint`: Run ESLint.

## 6. Project scope

**In scope (MVP):**

- Mocked authentication; gated access to Chat/Profile.
- Single ongoing conversation with AI; attachments and streaming.
- Basic Profile (Name editable, Email read-only), Avatar upload with 1:1 crop.
- Local/session storage for profile/chat per session; no persistent DB.
- Attachment validation (types/sizes/count), content limits (10k input, 30k combined).
- Server-side image normalization and EXIF removal.
- Markdown rendering, link safety, code highlighting with Copy.
- Offline handling, daily budget cap (0.5 USD/day).

**Out of scope (MVP):**

- Real user accounts (signup, password reset, persistent users).
- Persistent multi-thread chat history or media library.
- Advanced file management and rare formats.
- Speech-to-text.
- Model parameter controls from UI.

**Assumptions:**

- User accepts file type/size and content limits.
- Single-user daily use aligned to the 0.5 USD/day budget.
- Desktop-first; basic mobile support.

## 7. Project status

- **Status:** MVP in active development, dev environment only (no production hosting).
- **Success criteria (MVP):**
  - E2E flow: login → send a message with a file → streaming answer referencing the attachment.
  - Smooth streaming during dev; ≥95% upload success for `jpg/png/txt/md`.
  - Proper daily budget enforcement (0.5 USD/day).
- **Proposed SLOs (to finalize):**
  - TTFT ≤ 1.5 s (text) / ≤ 2.0 s (with file).
  - Avg response time ≤ 10 s (text) / ≤ 20 s (with file).
- **Open questions:**
  - Final SLOs (TTFT/latency) and exceptions for complex files.
  - Local `/metrics` page feasibility for dev insights.
  - Token/cost calculation method for `gpt-4o-mini` and daily reset policy.
  - Minimum browser/OS versions for desktop support.
  - Exact copy for budget/offline/errors; character counting precision for the 30,000 limit.

**Additional documentation:**

- Product Requirements (PRD): `.ai/prd.md`
- Tech stack overview: `.ai/tech-stack.md`

## 8. License

This repository currently has no LICENSE file and is treated as Unlicensed. If you intend to open-source it, consider adding a LICENSE (e.g., MIT). Until then, all rights reserved by the repository owner.
