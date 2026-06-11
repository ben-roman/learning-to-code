# How to Start Coding

This repo is Ben's learning-to-code workspace. It holds a tiny starter script, the **Janus Patient Moment** research prototype, and planning docs. Everything runs locally on your machine — no cloud account required to get going.

---

## What you need

| Tool | Purpose |
| --- | --- |
| **Node.js** (v20+) | Runs JavaScript/TypeScript on your computer |
| **npm** | Installs libraries (comes with Node) |
| **Cursor** | The editor where you write code and work with the AI agent |
| **Git** | Tracks changes (already set up for this repo) |

Node is installed at `C:\Program Files\nodejs`. On some PowerShell sessions it is not on your PATH — see [Fix Node on PATH](#fix-node-on-path) below.

---

## One-time setup

Open **Cursor**, open this folder as the workspace, then open a terminal (**Terminal → New Terminal**).

### 1. Fix Node on PATH

Run this once per terminal session (PowerShell):

```powershell
$env:Path = "C:\Program Files\nodejs;" + $env:Path
```

Check that it worked:

```powershell
node --version
npm --version
```

You should see version numbers, not "command not found."

### 2. Install root dependencies

From the repo root (the folder that contains `hello.ts` and `package.json`):

```powershell
npm install
```

This installs TypeScript and `tsx` so you can run `.ts` files directly.

### 3. Install server dependencies (for the Janus prototype)

```powershell
cd "janus-patient-moment\server"
npm install
cd ..\..
```

Quote paths — this repo lives in a folder with spaces.

### 4. Add your API key (server only)

The prototype's chat feature calls Claude. The key stays on the server, never in the browser.

```powershell
copy "janus-patient-moment\server\.env.example" "janus-patient-moment\server\.env"
```

Open `janus-patient-moment/server/.env` and replace the placeholder with a real `ANTHROPIC_API_KEY`. **Do not commit `.env`** — it is already in `.gitignore`.

---

## Your first run

The smallest program in the repo is `hello.ts`:

```ts
console.log("Hello, world");
console.log(new Date().toString());
```

From the repo root:

```powershell
npm start
```

Expected output:

```
Hello, world
Thu Jun 11 2026 ...
```

If that works, your toolchain is alive: Node runs code, TypeScript compiles via `tsx`, and npm scripts work.

---

## What's in this repo

```
Learning to Code/
  docs.md                          ← you are here
  hello.ts                         ← first script
  package.json                     ← root npm config

  janus-patient-moment/
    server/                        ← Express + TypeScript API (in progress)
      src/                         ← server source (added as you build)
      test/                        ← automated tests
      .env                         ← your API key (local only)
    client/                        ← Vite + React app (coming soon)

  docs/superpowers/
    specs/                         ← design spec for the prototype
    plans/                         ← step-by-step implementation plan
```

**Root** — minimal playground (`hello.ts`).

**`janus-patient-moment/server`** — back end for the research prototype: lab persona data, chat streaming, session logs.

**`janus-patient-moment/client`** — front end (landing page → upload → results → chat). Scaffolded in the implementation plan; build it task-by-task.

**`docs/superpowers/`** — the "why" and "what to build next." Start with the [design spec](docs/superpowers/specs/2026-06-11-patient-research-prototype-design.md), then follow the [implementation plan](docs/superpowers/plans/2026-06-11-patient-research-prototype.md).

---

## Running the Janus prototype

When the server has an entry point (`src/index.ts`), start it from `janus-patient-moment/server`:

```powershell
npm run dev
```

Expected: `Janus prototype server listening on http://localhost:3001`

When the client exists, open a **second** terminal:

```powershell
cd "janus-patient-moment\client"
npm run dev
```

Open the app at `http://localhost:5173`. Session URLs look like:

```
http://localhost:5173/?s=P01&p=149&f=generic
```

- `s` — session id (used in log filenames)
- `p` — price variant: `49`, `149`, or `299`
- `f` — reviewer framing: `generic` or `named`

---

## Running tests

Tests check that code behaves correctly before you run it in the browser.

From `janus-patient-moment/server`:

```powershell
npm test
```

Green output = passing. Red = something broke; read the error, fix the code, run again.

The implementation plan is written test-first: write a failing test → implement code → test passes → commit.

---

## How to work in Cursor

You do not need to memorize syntax to make progress. A practical loop:

1. **Pick a small task** — one step from the implementation plan, or a single bug.
2. **Ask the agent** — describe what you want in plain English. Point it at files or docs when helpful.
3. **Read the diff** — Cursor shows what changed. Skim it even if you do not understand every line.
4. **Run it** — `npm start`, `npm run dev`, or `npm test`. Broken code is normal; errors tell you what to fix.
5. **Commit when a step works** — the plan suggests commit messages; ask the agent to commit when you are ready.

Useful prompts:

- "Implement Task 2 from the patient research plan."
- "Explain what `persona.ts` does in plain language."
- "Tests are failing — fix them."
- "Walk me through what happens when someone hits `/api/chat`."

Keep tasks small. One file or one test at a time is easier to learn from than a giant "build the whole app" request.

---

## Key concepts (minimal glossary)

| Term | Meaning |
| --- | --- |
| **TypeScript (`.ts`)** | JavaScript with types; catches mistakes earlier |
| **`npm install`** | Downloads dependencies listed in `package.json` |
| **`npm run dev`** | Starts a dev server that reloads when you save |
| **`npm test`** | Runs automated tests |
| **Express** | A small web server library (handles HTTP requests) |
| **React** | A library for building interactive web pages |
| **API** | The server endpoints the client calls (e.g. `/api/persona`) |
| **`.env`** | Local secrets file — never commit it |

---

## Troubleshooting

### `node` or `npm` not recognized

Run the PATH fix at the top of this doc, then retry.

### Paths with spaces break commands

Always quote the repo path:

```powershell
cd "C:\Ben Google Drive_Mirror\PERSONAL Documents\0 Janus\01_Projects\2026-06-10 Learning to Code"
```

### Port already in use

Another process is using 3001 or 5173. Stop the old server (Ctrl+C in its terminal) or close the terminal that started it.

### Chat does nothing / API errors

Check that `janus-patient-moment/server/.env` exists and has a valid `ANTHROPIC_API_KEY`, and that the server terminal shows no startup errors.

### `npm install` fails

Confirm Node is on PATH, then delete `node_modules` in that folder and run `npm install` again.

---

## Where to go next

1. Run `npm start` at the repo root if you have not already.
2. Read the [design spec](docs/superpowers/specs/2026-06-11-patient-research-prototype-design.md) — what the prototype is for.
3. Open the [implementation plan](docs/superpowers/plans/2026-06-11-patient-research-prototype.md) and continue from the next unchecked task.
4. When the full app runs, use the moderator guide in `janus-patient-moment/README.md` (created in Task 12 of the plan).

Questions belong in Cursor chat — describe what you tried and paste any error text. That is faster than guessing.
