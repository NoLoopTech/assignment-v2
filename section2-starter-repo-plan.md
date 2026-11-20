# Plan: Section 2 Starter Repo (Kanban Board, No-AI Frontend Assignment)

## Context

We are hiring a Junior Next.js / NestJS full stack developer. The take-home assignment has two sections. **Section 2** is a frontend-only task that candidates must complete **without AI tools**, using plain React and CSS. They will later be asked to modify their own code live in an interview.

This repo is the **starter** that candidates clone. It must give them a working, minimal foundation (tooling, types, and a mock API) and **nothing that implements the actual requirements**. The point is to see how they handle state, effects, async work, forms, and CSS layout on their own.

Build two things:

1. **`kanban-starter`**: the candidate-facing starter repo (public, GitHub template).
2. **`kanban-reference`**: a private reference solution used by interviewers to calibrate grading and rehearse the live-change tasks. Never shared with candidates.

---

## Part 1: `kanban-starter` (candidate-facing)

### Tech stack

- Next.js (latest stable) with the **App Router**, React, and TypeScript (strict mode). Use the `src/` directory layout.
- When scaffolding with `create-next-app`, choose: TypeScript yes, ESLint yes, **Tailwind no**, `src/` directory yes, App Router yes, import alias `@/*`.
- Plain CSS only: `src/app/globals.css` with a minimal reset, and CSS Modules allowed. No Tailwind, CSS-in-JS, or UI libraries.
- Remove all `create-next-app` boilerplate (demo page content, demo styles, logos/SVGs in `public/`, and Google font setup in the layout).
- Node version pinned in `.nvmrc` (current LTS) and `"engines"` in `package.json`.
- npm as the package manager; commit `package-lock.json`.
- ESLint with `eslint-config-next`. No Prettier enforcement (we want to see their own formatting habits).

### Runtime dependencies (exact allowlist)

`next`, `react`, `react-dom`. Nothing else.

Dev dependencies: TypeScript, `@types/*` packages added by `create-next-app`, ESLint and `eslint-config-next`, Vitest (for the mock API tests only).

### Folder structure

```
kanban-starter/
├── .nvmrc
├── README.md
├── next.config.ts
├── package.json
├── scripts/
│   └── check-deps.mjs
├── src/
│   ├── app/
│   │   ├── layout.tsx     # minimal root layout, no fonts
│   │   ├── page.tsx       # placeholder only
│   │   └── globals.css    # minimal reset only
│   ├── types.ts
│   └── api/
│       ├── tasks.ts       # mock API (candidates must NOT edit)
│       ├── seed.ts        # seed data
│       └── tasks.test.ts  # tests for the mock API
└── tsconfig.json
```

### `src/types.ts`

Mirror the Section 1 Task resource:

```ts
export type TaskStatus = 'TODO' | 'IN_PROGRESS' | 'DONE';

export interface Task {
  id: string;
  title: string;
  description?: string;
  status: TaskStatus;
  dueDate?: string;   // ISO date string (YYYY-MM-DD)
  createdAt: string;  // ISO datetime string
}

export interface CreateTaskInput {
  title: string;
  description?: string;
  status?: TaskStatus; // defaults to TODO
  dueDate?: string;
}
```

### `src/api/tasks.ts`: mock API

An in-memory, promise-based mock that behaves like a flaky network API. Data resets on page reload.

**Browser-only module.** The mock's state lives in the browser, so it must only be called from Client Components (`'use client'`). Every exported function should throw a clear error if called on the server (`typeof window === 'undefined'`): "The mock API runs in the browser only. Call it from a Client Component (add 'use client')." This keeps the server/client boundary explicit and is something we can ask about in the interview.

**Exports:**

| Function | Behaviour |
|---|---|
| `getTasks(): Promise<Task[]>` | Returns a copy of all tasks. Never fails. Latency 300–800 ms. |
| `createTask(input: CreateTaskInput): Promise<Task>` | Validates input (see below). Generates `id` (`crypto.randomUUID()`) and `createdAt`. Latency 300–800 ms. Does **not** fail randomly. |
| `updateTaskStatus(id: string, status: TaskStatus): Promise<Task>` | Moves a task. **Fails randomly at the configured failure rate** (default 20%) with `ApiError` "Network error: could not move task". Rejects with a 404-style `ApiError` if the id does not exist. Latency 300–800 ms. |
| `class ApiError extends Error` | Has `status: number` (400, 404 or 503) and `message`. |

**Validation in `createTask`** (rejects with `ApiError` status 400):
- `title` is required after trimming.
- `title` max 80 characters.
- `status`, if provided, must be a valid `TaskStatus`.
- `dueDate`, if provided, must be a valid `YYYY-MM-DD` date.

**Implementation rules:**
- Always return **copies** of tasks (structured clone), never internal references, so candidates can't accidentally mutate the "server" state and hide bugs.
- Latency: a random delay between 300 and 800 ms on every call.
- Failure rate is configurable for graders/testing, in this priority order:
  1. URL query param `?failRate=0.5` (0–1), read from `window.location.search` at call time
  2. Env var `NEXT_PUBLIC_FAIL_RATE`
  3. Default `0.2`
- Expose `__setFailRate(rate: number)` and `__resetTasks()` **for tests only** (prefix with `__`, and add a JSDoc note "test helper, do not use in the app").
- Put a clear header comment at the top: "Do not modify this file. Treat it as an external API you cannot change."
- Keep the code readable: candidates will read it to understand the contract.

### `src/api/seed.ts`

12 seed tasks, mixed across statuses (roughly 5 TODO, 4 IN_PROGRESS, 3 DONE), realistic titles (e.g. "Set up CI pipeline", "Write onboarding docs"), some with descriptions, some with due dates (a few in the past, a few in the future), varied title lengths (include one near 80 characters to test layout wrapping).

### `src/app/page.tsx`

A placeholder only (a Server Component, no `'use client'`; deciding where the client boundary goes is part of the task):

```tsx
export default function Home() {
  return (
    <main>
      <h1>Task Board</h1>
      <p>Start building here. See README.md for the requirements.</p>
    </main>
  );
}
```

`src/app/layout.tsx` keeps only the `<html>`/`<body>` wrapper, the `globals.css` import, and basic metadata (title "Task Board"). Don't add any components, hooks, state, or styling beyond this.

### `src/api/tasks.test.ts`

Vitest tests covering:
- `getTasks` returns 12 seeded tasks.
- Returned tasks are copies (mutating the result doesn't change the next `getTasks` result).
- `createTask` rejects empty/whitespace titles, titles over 80 characters, and invalid statuses/dates with status 400.
- `createTask` defaults `status` to `TODO`.
- `updateTaskStatus` succeeds with fail rate 0, always fails with fail rate 1 (status 503), and rejects unknown ids with status 404.

Use fake timers so the tests run fast. Run the tests in Vitest's `jsdom` (or `happy-dom`) environment so the browser-only guard passes, and add one test that the guard throws when `window` is undefined.

### `scripts/check-deps.mjs`

A script run as `npm run check-deps` that:
- Reads `package.json`.
- Fails (exit code 1) if `dependencies` contains anything other than `next`, `react` and `react-dom`.
- Fails if `devDependencies` contains any package outside the original allowlist.
- Prints which packages broke the rule.

Graders run this on submissions to spot added UI/utility libraries.

### `package.json` scripts

```json
{
  "dev": "next dev",
  "build": "next build",
  "start": "next start",
  "lint": "next lint",
  "test": "vitest run",
  "check-deps": "node scripts/check-deps.mjs"
}
```

(If the installed Next.js version has removed `next lint`, use `eslint .` instead.)

### `README.md` (candidate-facing)

Write it for a junior developer, in plain, friendly English. Sections:

1. **Getting started**: prerequisites (Node version from `.nvmrc`), `npm install`, `npm run dev`.
2. **What's provided**: the types, the mock API (functions, latency, random failure on moves, validation rules), that the mock API must not be modified, and that it runs in the browser only, so it must be called from Client Components.
3. **The task**: the Kanban board requirements, exactly as below:
   - Three columns (TODO / IN_PROGRESS / DONE) using a responsive layout that stacks into one column on screens under 768 px.
   - Move a task between columns using buttons. Drag-and-drop is optional.
   - A search box that filters tasks by title, with a debounce of 300 ms that you implement yourself.
   - Optimistic moves: the card moves immediately; if the API call fails, it moves back and an error message is shown.
   - An inline "Add task" form: title required, max 80 characters, error shown under the field.
   - Each column header shows the number of tasks in that column, and it must always be correct.
   - Basic accessibility: real `<button>` elements, labelled inputs, visible focus styles.
   - Loading state while tasks are first fetched.
4. **Rules**:
   - No AI tools of any kind, including Copilot/Cursor autocomplete and chat assistants.
   - Official docs (MDN, react.dev) and normal web search are allowed.
   - No new runtime dependencies: no UI libraries, no utility libraries (e.g. lodash), no drag-and-drop libraries. `npm run check-deps` must pass.
   - Plain CSS or CSS Modules only.
   - Commit as you work (at least 5 commits). Don't squash.
   - Expected time: 90–120 minutes.
5. **In the interview**: you'll be asked to make changes to this code live, without AI, while sharing your screen.
6. **Declaration**: a line to fill in: `I completed this section without using AI tools. — <Your Name>`
7. **Tips**: open with `?failRate=0` to turn off random failures while building, and `?failRate=1` to test your error handling.

### Repo setup

- Initialise git with a clean history (a single "Initial starter" commit).
- Mark it as a **GitHub template repository** so candidates use "Use this template" (this gives them a fresh history).
- Add a `.gitignore` for Node/Vite.

### Acceptance criteria

- [ ] `npm install && npm run dev` starts with no errors or warnings and shows the placeholder page.
- [ ] No `create-next-app` boilerplate is left (demo content, logos, Tailwind, font setup).
- [ ] Calling a mock API function from a Server Component shows the browser-only error message.
- [ ] `npm run build`, `npm run lint`, and `npm test` all pass.
- [ ] `npm run check-deps` passes on the clean repo and fails when e.g. `lodash` is added.
- [ ] `?failRate=0` and `?failRate=1` behave as described.
- [ ] Nothing in `src/` implements any assignment requirement (no columns, cards, search, forms, hooks, or layout CSS).
- [ ] The README reads clearly to someone who has never seen the assignment.

---

## Part 2: `kanban-reference` (private, interviewers only)

A separate private repo, built **on top of** the starter, that fully implements the Section 2 requirements. Used to:
- Calibrate how long the task really takes and what "good" looks like for a junior.
- Rehearse the live-change tasks so interviewers know what a clean change looks like.

### Implementation guidelines

Write it the way we'd want a strong junior to write it: simple and readable, not clever.

- Keep the column list data-driven (a `COLUMNS` array), so "add a BLOCKED column" is a small change.
- Store only `tasks` and `query`; **derive** filtered tasks and column counts during render (no duplicated count state).
- Debounce with `useEffect` + `setTimeout` + cleanup in a small `useDebouncedValue` hook.
- Optimistic move: keep the previous status, update local state, call the API, and roll back that specific task on failure. Handle two overlapping moves correctly (roll back by task id, not by restoring a whole-array snapshot).
- Controlled inputs for the add form, with validation on submit and errors under the field.
- CSS Grid for columns, a media query at 768 px, and `:focus-visible` styles.
- `src/app/page.tsx` stays a Server Component that renders a `'use client'` `Board`. Keep the client boundary as low as is reasonable and comment on why it sits there.
- Components: `Board`, `Column`, `TaskCard`, `AddTaskForm`, `SearchBox`. No context or state library.

### Live-change rehearsal

Add a `LIVE_CHANGES.md` with each live-change task below, the files a candidate would most likely touch, and a short note on what a good vs weak approach looks like:

1. Add a 4th column "BLOCKED". (Requires adding `BLOCKED` to `TaskStatus`. Note that the mock API validation must accept it, so the grader may need to allow this edit to the API file.)
2. Change the debounce to 500 ms and make the search also trim whitespace.
3. Make column counts reflect only tasks matching the current search.
4. Make a failed move show which task failed in the error message.
5. Add a "max 5 tasks in IN_PROGRESS" rule with an inline warning.
6. On mobile, show columns as tabs instead of stacked.

Implement each one on its own branch (`live/01-blocked-column`, etc.) so interviewers can see the diff size.

### Acceptance criteria

- [ ] All Section 2 requirements work, including rollback under `?failRate=1`.
- [ ] `npm run check-deps` passes (no extra dependencies).
- [ ] The whole thing is achievable in about 2 hours by a competent junior. If the reference took notably longer to write cleanly, flag which requirements to cut.
- [ ] Each live-change branch is a small, focused diff.

---

## Out of scope

- Section 1 (NestJS + Next.js full stack) and its repo.
- The candidate brief and grader docs (these are written separately in Google Docs).
- Any backend or real network calls.
