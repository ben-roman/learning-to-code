# Janus Patient Moment — Research Prototype Implementation Plan

> **For agentic workers:** REQUIRED SUB-SKILL: Use superpowers:subagent-driven-development (recommended) or superpowers:executing-plans to implement this plan task-by-task. Steps use checkbox (`- [ ]`) syntax for tracking.

**Goal:** Build a thin working web prototype — landing page → lab upload (theater) → results view → live LLM sensemaking chat → configurable paid "ask" → exit screen — for moderated patient research sessions, per the spec at `docs/superpowers/specs/2026-06-11-patient-research-prototype-design.md`.

**Architecture:** Two small packages under `janus-patient-moment/`: an Express + TypeScript server that holds the Anthropic API key, streams chat over SSE, enforces a deterministic red-flag lane, and appends JSONL session logs; and a Vite + React + TypeScript client that renders six screens as a simple state machine, with session config (price/framing/session id) read from URL params. No auth, no database. Persona lab data is canonical on the server; uploads are theater.

**Tech Stack:** Node 26, TypeScript, Express 4, `@anthropic-ai/sdk` (model `claude-sonnet-4-6`), Vite + React 18, Vitest + Supertest for tests, tsx for running TS directly.

**Environment notes (this machine):**
- Node/npm are installed at `C:\Program Files\nodejs` but may not be on the shell PATH. In PowerShell, run once per session: `$env:Path = "C:\Program Files\nodejs;" + $env:Path`
- Repo root is `C:\Ben Google Drive_Mirror\PERSONAL Documents\0 Janus\01_Projects\2026-06-10 Learning to Code`. All paths below are relative to it. Quote paths (they contain spaces).
- The server needs `ANTHROPIC_API_KEY` in `janus-patient-moment/server/.env` (created in Task 1, never committed).

**File structure (end state):**

```
janus-patient-moment/
  README.md                       # moderator guide (Task 12)
  server/
    package.json, tsconfig.json, .env, .env.example
    src/
      persona.ts                  # canonical persona + lab data
      redFlags.ts                 # deterministic red-flag detector + scripted response
      systemPrompt.ts             # education-boundary prompt builder + OFFER marker
      sessionLog.ts               # append-only JSONL session logging
      app.ts                      # Express app factory (DI for tests)
      anthropic.ts                # Claude streaming wrapper
      index.ts                    # entry point, wires real deps
    test/
      persona.test.ts, redFlags.test.ts, systemPrompt.test.ts,
      sessionLog.test.ts, app.test.ts
    logs/                         # JSONL per session (gitignored)
  client/
    package.json, tsconfig.json, vite.config.ts, index.html
    src/
      main.tsx, App.tsx           # screen state machine + footer
      config.ts                   # URL-param session config
      offerMarker.ts              # marker detection / stream-safe display text
      api.ts                      # logEvent, fetchPersona, streamChat (SSE client)
      types.ts                    # shared client types (mirrors server persona shape)
      styles.css                  # whole design system
      screens/
        Landing.tsx, Upload.tsx, Results.tsx, Chat.tsx, AskCard.tsx, Exit.tsx
    test/
      config.test.ts, offerMarker.test.ts
```

---

### Task 1: Server scaffold

**Files:**
- Create: `janus-patient-moment/server/package.json`
- Create: `janus-patient-moment/server/tsconfig.json`
- Create: `janus-patient-moment/server/.env.example`
- Create: `janus-patient-moment/server/.env` (not committed)
- Modify: `.gitignore`

- [ ] **Step 1: Create the server package**

Create `janus-patient-moment/server/package.json`:

```json
{
  "name": "janus-patient-moment-server",
  "private": true,
  "scripts": {
    "dev": "tsx watch src/index.ts",
    "test": "vitest run"
  }
}
```

Create `janus-patient-moment/server/tsconfig.json`:

```json
{
  "compilerOptions": {
    "target": "ES2022",
    "module": "commonjs",
    "moduleResolution": "node",
    "strict": true,
    "esModuleInterop": true,
    "skipLibCheck": true,
    "types": ["node"]
  },
  "include": ["src", "test"]
}
```

- [ ] **Step 2: Install dependencies**

Run from `janus-patient-moment/server`:

```powershell
npm install express@4 dotenv @anthropic-ai/sdk
npm install -D typescript tsx vitest supertest @types/express @types/supertest @types/node
```

Expected: both commands finish without errors; `package-lock.json` appears.

- [ ] **Step 3: Env files and gitignore**

Create `janus-patient-moment/server/.env.example`:

```
ANTHROPIC_API_KEY=sk-ant-...your-key-here
```

Copy it to `janus-patient-moment/server/.env` and put the real key in (ask Ben for the key; do not commit this file).

Append to the repo-root `.gitignore` (it currently contains only `node_modules/`):

```
.env
logs/
```

- [ ] **Step 4: Commit**

```powershell
git add .gitignore "janus-patient-moment/server/package.json" "janus-patient-moment/server/package-lock.json" "janus-patient-moment/server/tsconfig.json" "janus-patient-moment/server/.env.example"
git commit -m "feat: scaffold prototype server package"
```

---

### Task 2: Persona and lab data

**Files:**
- Create: `janus-patient-moment/server/src/persona.ts`
- Test: `janus-patient-moment/server/test/persona.test.ts`

- [ ] **Step 1: Write the failing test**

Create `janus-patient-moment/server/test/persona.test.ts`:

```ts
import { describe, it, expect } from "vitest";
import { persona } from "../src/persona";

describe("persona", () => {
  it("headlines a gray-zone thyroid abnormality", () => {
    const tsh = persona.labs.find((l) => l.name === "TSH");
    expect(tsh?.value).toBe("6.8");
    expect(tsh?.flag).toBe("high");
    const tpo = persona.labs.find((l) => l.name.includes("TPO"));
    expect(tpo?.flag).toBe("high");
  });

  it("has exactly four out-of-range results", () => {
    const abnormal = persona.labs.filter((l) => l.flag !== "normal");
    expect(abnormal.map((l) => l.name).sort()).toEqual(
      [
        "LDL Cholesterol",
        "TSH",
        "Thyroid Peroxidase (TPO) Antibodies",
        "Total Cholesterol",
      ].sort()
    );
  });
});
```

- [ ] **Step 2: Run test to verify it fails**

Run from `janus-patient-moment/server`: `npm test`
Expected: FAIL — cannot find module `../src/persona`.

- [ ] **Step 3: Implement the persona module**

Create `janus-patient-moment/server/src/persona.ts`:

```ts
export interface LabResult {
  name: string;
  value: string;
  unit: string;
  referenceRange: string;
  flag: "high" | "low" | "normal";
  group: string;
}

export interface Persona {
  name: string;
  age: number;
  background: string[];
  collectionDate: string;
  labs: LabResult[];
}

// Values authored per spec: structure mirrors the deidentified Function Health
// report; headline abnormal is gray-zone subclinical hypothyroidism.
export const persona: Persona = {
  name: "Sam",
  age: 47,
  collectionDate: "March 5",
  background: [
    "Self-ordered a full-body lab panel after months of feeling run-down",
    "No current primary care doctor",
    "Mother has hypothyroidism",
    "No regular medications",
    "Not pregnant",
  ],
  labs: [
    { name: "TSH", value: "6.8", unit: "mIU/L", referenceRange: "0.40-4.50", flag: "high", group: "Thyroid" },
    { name: "Thyroid Peroxidase (TPO) Antibodies", value: "215", unit: "IU/mL", referenceRange: "<9", flag: "high", group: "Thyroid" },
    { name: "Free T4", value: "1.1", unit: "ng/dL", referenceRange: "0.8-1.8", flag: "normal", group: "Thyroid" },
    { name: "Free T3", value: "3.0", unit: "pg/mL", referenceRange: "2.3-4.2", flag: "normal", group: "Thyroid" },
    { name: "Thyroglobulin Antibodies", value: "<1", unit: "IU/mL", referenceRange: "<1", flag: "normal", group: "Thyroid" },
    { name: "Total Cholesterol", value: "205", unit: "mg/dL", referenceRange: "<200", flag: "high", group: "Heart Health" },
    { name: "LDL Cholesterol", value: "113", unit: "mg/dL", referenceRange: "<100", flag: "high", group: "Heart Health" },
    { name: "HDL Cholesterol", value: "81", unit: "mg/dL", referenceRange: ">39", flag: "normal", group: "Heart Health" },
    { name: "Triglycerides", value: "38", unit: "mg/dL", referenceRange: "<150", flag: "normal", group: "Heart Health" },
    { name: "Glucose (fasting)", value: "79", unit: "mg/dL", referenceRange: "65-99", flag: "normal", group: "Metabolic" },
    { name: "HbA1c", value: "5.2", unit: "%", referenceRange: "<5.7", flag: "normal", group: "Metabolic" },
    { name: "Ferritin", value: "94", unit: "ng/mL", referenceRange: "38-380", flag: "normal", group: "Blood" },
    { name: "Hemoglobin", value: "14.6", unit: "g/dL", referenceRange: "13.2-17.1", flag: "normal", group: "Blood" },
    { name: "Vitamin D, 25-OH", value: "31", unit: "ng/mL", referenceRange: "30-100", flag: "normal", group: "Nutrients" },
    { name: "Creatinine", value: "0.9", unit: "mg/dL", referenceRange: "0.7-1.3", flag: "normal", group: "Kidney" },
  ],
};
```

- [ ] **Step 4: Run test to verify it passes**

Run: `npm test` — Expected: PASS (2 tests).

- [ ] **Step 5: Commit**

```powershell
git add "janus-patient-moment/server/src/persona.ts" "janus-patient-moment/server/test/persona.test.ts"
git commit -m "feat: add Sam persona with gray-zone thyroid lab panel"
```

---

### Task 3: Red-flag detector

**Files:**
- Create: `janus-patient-moment/server/src/redFlags.ts`
- Test: `janus-patient-moment/server/test/redFlags.test.ts`

- [ ] **Step 1: Write the failing test**

Create `janus-patient-moment/server/test/redFlags.test.ts`:

```ts
import { describe, it, expect } from "vitest";
import { checkRedFlags, RED_FLAG_RESPONSE } from "../src/redFlags";

describe("checkRedFlags", () => {
  it.each([
    "I've been having chest pain at night",
    "sometimes I can't breathe",
    "I have shortness of breath when walking",
    "honestly I don't want to be alive",
    "I think I might be pregnant",
    "I passed out yesterday",
    "I'm having severe pain in my stomach",
  ])("flags urgent input: %s", (msg) => {
    expect(checkRedFlags(msg)).toBe(true);
  });

  it.each([
    "what does TSH even mean?",
    "my mother has hypothyroidism",
    "I'm tired all the time",
    "should I be worried about my cholesterol?",
  ])("does not flag ordinary input: %s", (msg) => {
    expect(checkRedFlags(msg)).toBe(false);
  });

  it("has a scripted response that breaks character and pauses the session", () => {
    expect(RED_FLAG_RESPONSE).toContain("outside our research scenario");
    expect(RED_FLAG_RESPONSE).toContain("moderator");
  });
});
```

- [ ] **Step 2: Run test to verify it fails**

Run: `npm test` — Expected: FAIL — cannot find module `../src/redFlags`.

- [ ] **Step 3: Implement the detector**

Create `janus-patient-moment/server/src/redFlags.ts`:

```ts
const RED_FLAG_PATTERNS: RegExp[] = [
  /chest (pain|pressure|tightness)|pain in my chest/i,
  /can'?t breathe|trouble breathing|short(ness)? of breath/i,
  /suicid|kill myself|end my life|don'?t want to (be alive|live)/i,
  /pregnan/i,
  /pass(ed|ing) out|fainted|fainting/i,
  /severe pain/i,
  /crushing|coughing (up )?blood/i,
];

export function checkRedFlags(message: string): boolean {
  return RED_FLAG_PATTERNS.some((p) => p.test(message));
}

export const RED_FLAG_RESPONSE =
  "I need to step outside our research scenario for a moment. What you just " +
  "described is something that deserves real medical attention right away - " +
  "please don't wait on a research prototype for it. If this is happening to " +
  "you right now, contact a real clinician or emergency services. Let's pause " +
  "here: please tell your moderator what you typed so they can help.";
```

- [ ] **Step 4: Run test to verify it passes**

Run: `npm test` — Expected: PASS.

- [ ] **Step 5: Commit**

```powershell
git add "janus-patient-moment/server/src/redFlags.ts" "janus-patient-moment/server/test/redFlags.test.ts"
git commit -m "feat: deterministic red-flag lane with scripted break-character response"
```

---

### Task 4: System prompt builder (the education boundary)

**Files:**
- Create: `janus-patient-moment/server/src/systemPrompt.ts`
- Test: `janus-patient-moment/server/test/systemPrompt.test.ts`

- [ ] **Step 1: Write the failing test**

Create `janus-patient-moment/server/test/systemPrompt.test.ts`:

```ts
import { describe, it, expect } from "vitest";
import { buildSystemPrompt, OFFER_MARKER } from "../src/systemPrompt";
import { persona } from "../src/persona";

describe("buildSystemPrompt", () => {
  const prompt = buildSystemPrompt(persona);

  it("embeds the persona's labs and context", () => {
    expect(prompt).toContain("TSH: 6.8 mIU/L");
    expect(prompt).toContain("Mother has hypothyroidism");
    expect(prompt).toContain("Sam");
  });

  it("states the education-only boundary and do-nothing-safely framing", () => {
    expect(prompt).toContain("education and sensemaking only");
    expect(prompt).toContain("watchful waiting");
    expect(prompt).toContain("must NOT give an individualized plan");
  });

  it("instructs one clarifying question at a time", () => {
    expect(prompt).toContain("ONE clarifying question per message");
  });

  it("defines the offer marker", () => {
    expect(OFFER_MARKER).toBe("<<OFFER>>");
    expect(prompt).toContain(OFFER_MARKER);
  });
});
```

- [ ] **Step 2: Run test to verify it fails**

Run: `npm test` — Expected: FAIL — cannot find module `../src/systemPrompt`.

- [ ] **Step 3: Implement the prompt builder**

Create `janus-patient-moment/server/src/systemPrompt.ts`:

```ts
import { LabResult, Persona } from "./persona";

export const OFFER_MARKER = "<<OFFER>>";

function formatLab(lab: LabResult): string {
  const flag = lab.flag === "normal" ? "" : ` (${lab.flag.toUpperCase()})`;
  return `- ${lab.name}: ${lab.value} ${lab.unit} [ref ${lab.referenceRange}]${flag}`;
}

export function buildSystemPrompt(persona: Persona): string {
  return `You are the patient-facing guide for Janus, a service that helps people understand abnormal lab results. You are chatting with ${persona.name}, age ${persona.age}, who just viewed their results.

What you know about ${persona.name}:
${persona.background.map((b) => `- ${b}`).join("\n")}

Their lab results (collected ${persona.collectionDate}):
${persona.labs.map(formatLab).join("\n")}

How to behave:
- Be warm, calm, and plain-spoken. Never alarmist. Short paragraphs (2-4 sentences). Explain any medical term the moment you use it.
- Your job is education and sensemaking only: explain what results like these usually mean, what is uncertain, and what the realistic decision landscape looks like - including that watchful waiting (for example, rechecking labs in several weeks) is often a safe and legitimate option.
- Ask at most ONE clarifying question per message, and only when the answer would genuinely change what you explain next.
- You must NOT give an individualized plan: no medication or dose suggestions, no "you should get test X," no personalized treatment decisions, no telling ${persona.name} which option to choose. That is the job of the specialist review.
- When ${persona.name} asks what they should DO (a plan, a treatment, a decision, "what now?"), warmly explain that this is exactly what Janus's specialist review is for: a board-certified specialist reviews the full picture and provides a signed, individualized recommendation. Then end that message with the exact marker ${OFFER_MARKER} at the very end. Use the marker at most once in the whole conversation, only at that moment.
- If they decline the specialist review, keep helping with education within these limits. Never nag about the offer.
- If they mention symptoms or thoughts that need urgent real-world care, tell them to seek real care immediately. (A separate safety system also screens for this.)
- Never invent results that are not in the list above. If asked about something not tested, say it was not part of this panel.
- Stay in the scenario: you are talking to ${persona.name} about ${persona.name}'s results.`;
}
```

- [ ] **Step 4: Run test to verify it passes**

Run: `npm test` — Expected: PASS.

- [ ] **Step 5: Commit**

```powershell
git add "janus-patient-moment/server/src/systemPrompt.ts" "janus-patient-moment/server/test/systemPrompt.test.ts"
git commit -m "feat: education-boundary system prompt with offer marker"
```

---

### Task 5: Session logger

**Files:**
- Create: `janus-patient-moment/server/src/sessionLog.ts`
- Test: `janus-patient-moment/server/test/sessionLog.test.ts`

- [ ] **Step 1: Write the failing test**

Create `janus-patient-moment/server/test/sessionLog.test.ts`:

```ts
import { describe, it, expect, beforeEach } from "vitest";
import fs from "node:fs";
import os from "node:os";
import path from "node:path";
import { appendEvent } from "../src/sessionLog";

describe("appendEvent", () => {
  let dir: string;
  beforeEach(() => {
    dir = fs.mkdtempSync(path.join(os.tmpdir(), "janus-log-"));
  });

  it("appends JSONL lines to a per-session file", () => {
    appendEvent(dir, { sessionId: "P01", type: "screen_view", data: { screen: "landing" } });
    appendEvent(dir, { sessionId: "P01", type: "ask_click", data: { choice: "bought" } });
    const lines = fs.readFileSync(path.join(dir, "P01.jsonl"), "utf8").trim().split("\n");
    expect(lines).toHaveLength(2);
    const first = JSON.parse(lines[0]);
    expect(first.type).toBe("screen_view");
    expect(typeof first.ts).toBe("string");
  });

  it("sanitizes session ids so they cannot escape the log directory", () => {
    appendEvent(dir, { sessionId: "../evil", type: "x" });
    expect(fs.existsSync(path.join(dir, "___evil.jsonl"))).toBe(true);
  });

  it("creates the directory if missing", () => {
    const nested = path.join(dir, "deeper");
    appendEvent(nested, { sessionId: "P02", type: "x" });
    expect(fs.existsSync(path.join(nested, "P02.jsonl"))).toBe(true);
  });
});
```

- [ ] **Step 2: Run test to verify it fails**

Run: `npm test` — Expected: FAIL — cannot find module `../src/sessionLog`.

- [ ] **Step 3: Implement the logger**

Create `janus-patient-moment/server/src/sessionLog.ts`:

```ts
import fs from "node:fs";
import path from "node:path";

export interface LogEvent {
  sessionId: string;
  type: string;
  data?: unknown;
}

export function appendEvent(logDir: string, event: LogEvent): void {
  fs.mkdirSync(logDir, { recursive: true });
  const safeId = event.sessionId.replace(/[^a-zA-Z0-9_-]/g, "_");
  const line = JSON.stringify({ ts: new Date().toISOString(), type: event.type, data: event.data ?? null });
  fs.appendFileSync(path.join(logDir, `${safeId}.jsonl`), line + "\n", "utf8");
}
```

- [ ] **Step 4: Run test to verify it passes**

Run: `npm test` — Expected: PASS.

- [ ] **Step 5: Commit**

```powershell
git add "janus-patient-moment/server/src/sessionLog.ts" "janus-patient-moment/server/test/sessionLog.test.ts"
git commit -m "feat: append-only JSONL session logging"
```

---

### Task 6: Express app, routes, and Anthropic wrapper

**Files:**
- Create: `janus-patient-moment/server/src/app.ts`
- Create: `janus-patient-moment/server/src/anthropic.ts`
- Create: `janus-patient-moment/server/src/index.ts`
- Test: `janus-patient-moment/server/test/app.test.ts`

- [ ] **Step 1: Write the failing tests**

Create `janus-patient-moment/server/test/app.test.ts`:

```ts
import { describe, it, expect, beforeEach } from "vitest";
import fs from "node:fs";
import os from "node:os";
import path from "node:path";
import request from "supertest";
import { createApp, ChatMessage } from "../src/app";
import { RED_FLAG_RESPONSE } from "../src/redFlags";

async function* fakeStream(_messages: ChatMessage[]): AsyncGenerator<string> {
  yield "Hello ";
  yield "Sam.";
}

describe("api", () => {
  let logDir: string;
  let app: ReturnType<typeof createApp>;

  beforeEach(() => {
    logDir = fs.mkdtempSync(path.join(os.tmpdir(), "janus-app-"));
    app = createApp({ logDir, streamChat: fakeStream });
  });

  it("GET /api/persona returns the lab panel", async () => {
    const res = await request(app).get("/api/persona");
    expect(res.status).toBe(200);
    expect(res.body.name).toBe("Sam");
    expect(res.body.labs.length).toBeGreaterThan(10);
  });

  it("POST /api/log appends an event", async () => {
    const res = await request(app)
      .post("/api/log")
      .send({ sessionId: "P01", type: "screen_view", data: { screen: "landing" } });
    expect(res.status).toBe(200);
    const file = fs.readFileSync(path.join(logDir, "P01.jsonl"), "utf8");
    expect(file).toContain("screen_view");
  });

  it("POST /api/log rejects missing fields", async () => {
    const res = await request(app).post("/api/log").send({ type: "x" });
    expect(res.status).toBe(400);
  });

  it("POST /api/chat streams SSE deltas and logs the exchange", async () => {
    const res = await request(app)
      .post("/api/chat")
      .send({ sessionId: "P01", messages: [{ role: "user", content: "hi" }] });
    expect(res.status).toBe(200);
    expect(res.headers["content-type"]).toContain("text/event-stream");
    expect(res.text).toContain('data: {"delta":"Hello "}');
    expect(res.text).toContain("data: [DONE]");
    const file = fs.readFileSync(path.join(logDir, "P01.jsonl"), "utf8");
    expect(file).toContain("chat_user_message");
    expect(file).toContain("chat_assistant_message");
  });

  it("POST /api/chat short-circuits on red flags without calling the model", async () => {
    async function* explode(): AsyncGenerator<string> {
      throw new Error("model should not be called");
    }
    const guarded = createApp({ logDir, streamChat: explode });
    const res = await request(guarded)
      .post("/api/chat")
      .send({ sessionId: "P02", messages: [{ role: "user", content: "I have chest pain" }] });
    expect(res.status).toBe(200);
    expect(res.text).toContain('"redFlag":true');
    expect(res.text).toContain(RED_FLAG_RESPONSE.slice(0, 30));
    const file = fs.readFileSync(path.join(logDir, "P02.jsonl"), "utf8");
    expect(file).toContain("red_flag");
  });
});
```

- [ ] **Step 2: Run tests to verify they fail**

Run: `npm test` — Expected: FAIL — cannot find module `../src/app`.

- [ ] **Step 3: Implement the app factory**

Create `janus-patient-moment/server/src/app.ts`:

```ts
import express from "express";
import { persona } from "./persona";
import { appendEvent } from "./sessionLog";
import { checkRedFlags, RED_FLAG_RESPONSE } from "./redFlags";

export type ChatMessage = { role: "user" | "assistant"; content: string };
export type StreamChatFn = (messages: ChatMessage[]) => AsyncGenerator<string>;

export interface AppOptions {
  logDir: string;
  streamChat: StreamChatFn;
}

export function createApp(options: AppOptions) {
  const app = express();
  app.use(express.json());

  app.get("/api/persona", (_req, res) => {
    res.json(persona);
  });

  app.post("/api/log", (req, res) => {
    const { sessionId, type, data } = req.body ?? {};
    if (typeof sessionId !== "string" || typeof type !== "string") {
      res.status(400).json({ error: "sessionId and type are required" });
      return;
    }
    appendEvent(options.logDir, { sessionId, type, data });
    res.json({ ok: true });
  });

  app.post("/api/chat", async (req, res) => {
    const { sessionId, messages } = req.body ?? {};
    if (typeof sessionId !== "string" || !Array.isArray(messages) || messages.length === 0) {
      res.status(400).json({ error: "sessionId and messages are required" });
      return;
    }

    res.setHeader("Content-Type", "text/event-stream");
    res.setHeader("Cache-Control", "no-cache");
    const send = (payload: object) => res.write(`data: ${JSON.stringify(payload)}\n\n`);
    const done = () => {
      res.write("data: [DONE]\n\n");
      res.end();
    };

    const lastUser = [...messages].reverse().find((m: ChatMessage) => m.role === "user");
    appendEvent(options.logDir, { sessionId, type: "chat_user_message", data: lastUser?.content });

    if (lastUser && checkRedFlags(lastUser.content)) {
      appendEvent(options.logDir, { sessionId, type: "red_flag", data: lastUser.content });
      send({ redFlag: true });
      send({ delta: RED_FLAG_RESPONSE });
      done();
      return;
    }

    let fullText = "";
    try {
      for await (const chunk of options.streamChat(messages)) {
        fullText += chunk;
        send({ delta: chunk });
      }
    } catch {
      send({ delta: "Sorry - something went wrong on our end. Give it a second and try again." });
    }
    appendEvent(options.logDir, { sessionId, type: "chat_assistant_message", data: fullText });
    done();
  });

  return app;
}
```

- [ ] **Step 4: Run tests to verify they pass**

Run: `npm test` — Expected: PASS (all server tests).

- [ ] **Step 5: Implement the Anthropic wrapper and entry point**

Create `janus-patient-moment/server/src/anthropic.ts`:

```ts
import Anthropic from "@anthropic-ai/sdk";
import { ChatMessage } from "./app";
import { persona } from "./persona";
import { buildSystemPrompt } from "./systemPrompt";

let client: Anthropic | undefined;

export async function* streamChat(messages: ChatMessage[]): AsyncGenerator<string> {
  client ??= new Anthropic(); // reads ANTHROPIC_API_KEY from env

  // The Messages API requires the first message to be from the user. Our chat
  // opens with a scripted assistant message, so prepend a synthetic user turn.
  const apiMessages =
    messages[0]?.role === "assistant"
      ? [{ role: "user" as const, content: `(${persona.name} opens the chat after viewing their results.)` }, ...messages]
      : messages;

  const stream = client.messages.stream({
    model: "claude-sonnet-4-6",
    max_tokens: 1024,
    system: buildSystemPrompt(persona),
    messages: apiMessages,
  });

  for await (const event of stream) {
    if (event.type === "content_block_delta" && event.delta.type === "text_delta") {
      yield event.delta.text;
    }
  }
}
```

Create `janus-patient-moment/server/src/index.ts`:

```ts
import "dotenv/config";
import path from "node:path";
import { createApp } from "./app";
import { streamChat } from "./anthropic";

const logDir = path.join(__dirname, "..", "logs");
const app = createApp({ logDir, streamChat });

const port = 3001;
app.listen(port, () => {
  console.log(`Janus prototype server listening on http://localhost:${port}`);
});
```

- [ ] **Step 6: Smoke-test the live server**

Run from `janus-patient-moment/server`: `npm run dev`
Expected: console prints `Janus prototype server listening on http://localhost:3001`.
In a second terminal: `curl http://localhost:3001/api/persona` — expected: JSON with `"name":"Sam"`. Stop the server (Ctrl+C).

- [ ] **Step 7: Commit**

```powershell
git add "janus-patient-moment/server/src/app.ts" "janus-patient-moment/server/src/anthropic.ts" "janus-patient-moment/server/src/index.ts" "janus-patient-moment/server/test/app.test.ts"
git commit -m "feat: Express API with SSE chat, red-flag short-circuit, and logging"
```

---

### Task 7: Client scaffold

**Files:**
- Create: `janus-patient-moment/client/` (Vite scaffold, then trimmed)
- Modify: `janus-patient-moment/client/vite.config.ts`
- Create: `janus-patient-moment/client/src/styles.css`

- [ ] **Step 1: Scaffold with Vite**

Run from `janus-patient-moment`:

```powershell
npm create vite@latest client -- --template react-ts
cd client
npm install
npm install -D vitest
```

Expected: scaffold completes; `client/src/App.tsx` etc. exist.

- [ ] **Step 2: Trim boilerplate and configure**

Delete: `client/src/App.css`, `client/src/index.css`, `client/src/assets/react.svg`, `client/public/vite.svg`.

Replace `client/vite.config.ts`:

```ts
import { defineConfig } from "vite";
import react from "@vitejs/plugin-react";

export default defineConfig({
  plugins: [react()],
  server: {
    proxy: { "/api": "http://localhost:3001" },
  },
});
```

Replace `client/index.html`:

```html
<!doctype html>
<html lang="en">
  <head>
    <meta charset="UTF-8" />
    <meta name="viewport" content="width=device-width, initial-scale=1.0" />
    <title>Janus — Understand your results</title>
  </head>
  <body>
    <div id="root"></div>
    <script type="module" src="/src/main.tsx"></script>
  </body>
</html>
```

Replace `client/src/main.tsx`:

```tsx
import React from "react";
import ReactDOM from "react-dom/client";
import App from "./App";
import "./styles.css";

ReactDOM.createRoot(document.getElementById("root")!).render(
  <React.StrictMode>
    <App />
  </React.StrictMode>
);
```

Replace `client/src/App.tsx` with a placeholder (real version in Task 11):

```tsx
export default function App() {
  return <div className="app-shell">Janus prototype — screens coming soon</div>;
}
```

Add a `test` script to `client/package.json` scripts block: `"test": "vitest run"`.

- [ ] **Step 3: Create the design system stylesheet**

Create `janus-patient-moment/client/src/styles.css`:

```css
:root {
  --ink: #182830;
  --ink-soft: #51646e;
  --teal: #0e7c66;
  --teal-dark: #0a5d4d;
  --amber: #b3590a;
  --amber-bg: #fdf1e4;
  --red: #b3261e;
  --bg: #f6f8f7;
  --card: #ffffff;
  --line: #e3e9e7;
  --radius: 14px;
  font-family: -apple-system, "Segoe UI", Roboto, Helvetica, Arial, sans-serif;
}

* { box-sizing: border-box; margin: 0; }

body { background: var(--bg); color: var(--ink); }

.app-shell {
  max-width: 430px;
  min-height: 100dvh;
  margin: 0 auto;
  display: flex;
  flex-direction: column;
  background: var(--bg);
}

.screen { flex: 1; padding: 24px 20px 32px; display: flex; flex-direction: column; }

.footer-note {
  text-align: center; font-size: 11px; color: var(--ink-soft);
  padding: 10px; border-top: 1px solid var(--line);
}

/* Buttons */
.btn {
  display: block; width: 100%; padding: 14px 18px; border: none;
  border-radius: var(--radius); font-size: 16px; font-weight: 600;
  cursor: pointer; text-align: center;
}
.btn-primary { background: var(--teal); color: #fff; }
.btn-primary:hover { background: var(--teal-dark); }
.btn-ghost { background: transparent; color: var(--ink-soft); font-weight: 500; }

/* Landing */
.brand { font-size: 22px; font-weight: 800; letter-spacing: 0.5px; color: var(--teal-dark); }
.hero { margin: 48px 0 12px; font-size: 32px; line-height: 1.15; font-weight: 800; }
.hero-sub { font-size: 17px; color: var(--ink-soft); margin-bottom: 28px; line-height: 1.45; }
.trust-row { display: flex; flex-direction: column; gap: 8px; margin: 24px 0 32px; }
.trust-item { font-size: 14px; color: var(--ink-soft); display: flex; gap: 8px; align-items: center; }
.trust-item::before { content: "✓"; color: var(--teal); font-weight: 700; }
.how-strip { margin-top: auto; display: flex; gap: 10px; }
.how-step {
  flex: 1; background: var(--card); border: 1px solid var(--line);
  border-radius: var(--radius); padding: 10px; font-size: 12px; color: var(--ink-soft);
}
.how-step b { display: block; color: var(--ink); margin-bottom: 2px; }

/* Upload */
.dropzone {
  border: 2px dashed var(--line); border-radius: var(--radius); background: var(--card);
  padding: 44px 20px; text-align: center; color: var(--ink-soft); margin: 24px 0 12px;
  cursor: pointer;
}
.dropzone.processing { border-style: solid; }
.spinner {
  width: 28px; height: 28px; margin: 0 auto 12px; border-radius: 50%;
  border: 3px solid var(--line); border-top-color: var(--teal);
  animation: spin 0.9s linear infinite;
}
@keyframes spin { to { transform: rotate(360deg); } }

/* Results */
.results-header { margin-bottom: 16px; }
.results-header h2 { font-size: 22px; }
.results-header p { color: var(--ink-soft); font-size: 14px; margin-top: 4px; }
.lab-group { margin-bottom: 18px; }
.lab-group h3 { font-size: 13px; text-transform: uppercase; letter-spacing: 0.08em; color: var(--ink-soft); margin-bottom: 8px; }
.lab-row {
  background: var(--card); border: 1px solid var(--line); border-radius: 10px;
  padding: 10px 12px; display: flex; justify-content: space-between; align-items: center;
  margin-bottom: 6px; font-size: 14px;
}
.lab-row .ref { color: var(--ink-soft); font-size: 12px; }
.chip { font-weight: 700; padding: 3px 10px; border-radius: 999px; font-size: 13px; }
.chip-high { background: var(--amber-bg); color: var(--amber); }
.chip-low { background: var(--amber-bg); color: var(--amber); }
.chip-normal { background: #eef4f2; color: var(--ink-soft); font-weight: 500; }
.context-banner {
  position: sticky; bottom: 12px; background: var(--card); border: 1px solid var(--teal);
  border-radius: var(--radius); padding: 14px; box-shadow: 0 6px 24px rgba(20, 60, 50, 0.12);
  animation: rise 0.4s ease-out;
}
.context-banner p { font-size: 15px; margin-bottom: 10px; }
@keyframes rise { from { transform: translateY(16px); opacity: 0; } to { transform: none; opacity: 1; } }

/* Chat */
.chat-screen { padding-bottom: 0; }
.chat-log { flex: 1; overflow-y: auto; display: flex; flex-direction: column; gap: 10px; padding-bottom: 16px; }
.bubble { max-width: 85%; padding: 11px 14px; border-radius: 16px; font-size: 15px; line-height: 1.45; white-space: pre-wrap; }
.bubble-assistant { background: var(--card); border: 1px solid var(--line); align-self: flex-start; border-bottom-left-radius: 4px; }
.bubble-user { background: var(--teal); color: #fff; align-self: flex-end; border-bottom-right-radius: 4px; }
.bubble-redflag { border: 1.5px solid var(--red); background: #fdf3f2; }
.redflag-banner {
  background: #fdf3f2; border: 1px solid var(--red); color: var(--red);
  border-radius: 10px; padding: 10px 12px; font-size: 13px; font-weight: 600; text-align: center;
}
.chat-input-row { display: flex; gap: 8px; padding: 12px 0 16px; }
.chat-input-row input {
  flex: 1; padding: 12px 14px; border: 1px solid var(--line); border-radius: 999px;
  font-size: 15px; outline: none; background: var(--card);
}
.chat-input-row button {
  padding: 0 18px; border: none; border-radius: 999px; background: var(--teal);
  color: #fff; font-weight: 700; cursor: pointer;
}
.chat-input-row button:disabled { opacity: 0.5; cursor: default; }

/* Ask card */
.ask-card {
  border: 1.5px solid var(--teal); background: var(--card); border-radius: var(--radius);
  padding: 18px; margin: 4px 0 10px; box-shadow: 0 8px 28px rgba(20, 60, 50, 0.14);
  animation: rise 0.4s ease-out;
}
.ask-card h3 { font-size: 17px; margin-bottom: 10px; }
.ask-card ul { margin: 0 0 12px 18px; font-size: 14px; color: var(--ink-soft); }
.ask-card li { margin-bottom: 4px; }
.reviewer { font-size: 13px; color: var(--ink-soft); margin-bottom: 12px; font-style: italic; }
.price { font-size: 26px; font-weight: 800; margin-bottom: 12px; }
.ask-card .btn + .btn { margin-top: 6px; }

/* Exit */
.exit-box { margin: auto 0; text-align: center; }
.exit-box h2 { font-size: 24px; margin-bottom: 12px; }
.exit-box p { color: var(--ink-soft); font-size: 15px; line-height: 1.5; margin-bottom: 10px; }
```

- [ ] **Step 4: Verify the dev server runs**

Run from `janus-patient-moment/client`: `npm run dev`
Expected: Vite serves on `http://localhost:5173`; browser shows "Janus prototype — screens coming soon". Stop it.

- [ ] **Step 5: Commit**

```powershell
git add "janus-patient-moment/client"
git commit -m "feat: scaffold Vite React client with design system and API proxy"
```

---

### Task 8: Client utilities — session config, offer marker, API client

**Files:**
- Create: `janus-patient-moment/client/src/config.ts`
- Create: `janus-patient-moment/client/src/offerMarker.ts`
- Create: `janus-patient-moment/client/src/types.ts`
- Create: `janus-patient-moment/client/src/api.ts`
- Test: `janus-patient-moment/client/test/config.test.ts`
- Test: `janus-patient-moment/client/test/offerMarker.test.ts`

- [ ] **Step 1: Write the failing tests**

Create `janus-patient-moment/client/test/config.test.ts`:

```ts
import { describe, it, expect } from "vitest";
import { parseConfig } from "../src/config";

describe("parseConfig", () => {
  it("reads session id, price, and framing from URL params", () => {
    const c = parseConfig("?s=P03&p=299&f=named");
    expect(c).toEqual({ sessionId: "P03", price: 299, framing: "named" });
  });

  it("defaults price to 149 and framing to generic", () => {
    const c = parseConfig("?s=P04");
    expect(c.price).toBe(149);
    expect(c.framing).toBe("generic");
  });

  it("rejects prices outside the variant set", () => {
    expect(parseConfig("?s=P05&p=999").price).toBe(149);
  });

  it("generates a session id when missing", () => {
    expect(parseConfig("").sessionId).toMatch(/^session-/);
  });
});
```

Create `janus-patient-moment/client/test/offerMarker.test.ts`:

```ts
import { describe, it, expect } from "vitest";
import { containsOffer, visibleText, OFFER_MARKER } from "../src/offerMarker";

describe("offer marker", () => {
  it("detects a complete marker", () => {
    expect(containsOffer(`Here is why. ${OFFER_MARKER}`)).toBe(true);
    expect(containsOffer("No marker here.")).toBe(false);
  });

  it("strips the complete marker from displayed text", () => {
    expect(visibleText(`That is the specialist's job. ${OFFER_MARKER}`)).toBe(
      "That is the specialist's job."
    );
  });

  it("hides a partial marker still streaming in at the end", () => {
    expect(visibleText("Almost done <<OF")).toBe("Almost done");
  });

  it("leaves ordinary text untouched", () => {
    expect(visibleText("TSH stands for thyroid stimulating hormone.")).toBe(
      "TSH stands for thyroid stimulating hormone."
    );
  });
});
```

- [ ] **Step 2: Run tests to verify they fail**

Run from `janus-patient-moment/client`: `npm test`
Expected: FAIL — cannot find modules `../src/config` and `../src/offerMarker`.

- [ ] **Step 3: Implement the utilities**

Create `janus-patient-moment/client/src/config.ts`:

```ts
export interface SessionConfig {
  sessionId: string;
  price: number;
  framing: "generic" | "named";
}

const PRICE_VARIANTS = [49, 149, 299];

export function parseConfig(search: string): SessionConfig {
  const params = new URLSearchParams(search);
  const price = Number(params.get("p"));
  return {
    sessionId: params.get("s") ?? `session-${Date.now()}`,
    price: PRICE_VARIANTS.includes(price) ? price : 149,
    framing: params.get("f") === "named" ? "named" : "generic",
  };
}
```

Create `janus-patient-moment/client/src/offerMarker.ts`:

```ts
export const OFFER_MARKER = "<<OFFER>>";

export function containsOffer(text: string): boolean {
  return text.includes(OFFER_MARKER);
}

/** Display-safe text: removes complete markers anywhere and a partial marker
 *  that may still be streaming in at the very end of the text. */
export function visibleText(text: string): string {
  let t = text.split(OFFER_MARKER).join("");
  for (let i = OFFER_MARKER.length - 1; i > 0; i--) {
    if (t.endsWith(OFFER_MARKER.slice(0, i))) {
      t = t.slice(0, -i);
      break;
    }
  }
  return t.trimEnd();
}
```

Create `janus-patient-moment/client/src/types.ts` (mirrors the server persona shape — kept in sync by hand; the prototype has no shared package):

```ts
export interface LabResult {
  name: string;
  value: string;
  unit: string;
  referenceRange: string;
  flag: "high" | "low" | "normal";
  group: string;
}

export interface Persona {
  name: string;
  age: number;
  background: string[];
  collectionDate: string;
  labs: LabResult[];
}

export interface ChatMessage {
  role: "user" | "assistant";
  content: string;
}
```

Create `janus-patient-moment/client/src/api.ts`:

```ts
import { ChatMessage, Persona } from "./types";

export function logEvent(sessionId: string, type: string, data?: unknown): void {
  fetch("/api/log", {
    method: "POST",
    headers: { "Content-Type": "application/json" },
    body: JSON.stringify({ sessionId, type, data }),
  }).catch(() => {
    // logging must never break the session
  });
}

export async function fetchPersona(): Promise<Persona> {
  const res = await fetch("/api/persona");
  if (!res.ok) throw new Error(`persona request failed: ${res.status}`);
  return res.json();
}

export async function streamChat(
  sessionId: string,
  messages: ChatMessage[],
  handlers: { onDelta: (text: string) => void; onRedFlag: () => void }
): Promise<void> {
  const res = await fetch("/api/chat", {
    method: "POST",
    headers: { "Content-Type": "application/json" },
    body: JSON.stringify({ sessionId, messages }),
  });
  if (!res.ok || !res.body) throw new Error(`chat request failed: ${res.status}`);

  const reader = res.body.getReader();
  const decoder = new TextDecoder();
  let buffer = "";
  for (;;) {
    const { done, value } = await reader.read();
    if (done) break;
    buffer += decoder.decode(value, { stream: true });
    const parts = buffer.split("\n\n");
    buffer = parts.pop() ?? "";
    for (const part of parts) {
      if (!part.startsWith("data: ")) continue;
      const payload = part.slice(6);
      if (payload === "[DONE]") return;
      const event = JSON.parse(payload) as { delta?: string; redFlag?: boolean };
      if (event.redFlag) handlers.onRedFlag();
      if (event.delta) handlers.onDelta(event.delta);
    }
  }
}
```

- [ ] **Step 4: Run tests to verify they pass**

Run: `npm test` — Expected: PASS (8 tests).

- [ ] **Step 5: Commit**

```powershell
git add "janus-patient-moment/client/src/config.ts" "janus-patient-moment/client/src/offerMarker.ts" "janus-patient-moment/client/src/types.ts" "janus-patient-moment/client/src/api.ts" "janus-patient-moment/client/test"
git commit -m "feat: client session config, offer-marker handling, and API client"
```

---

### Task 9: Static screens — Landing, Upload, Exit

**Files:**
- Create: `janus-patient-moment/client/src/screens/Landing.tsx`
- Create: `janus-patient-moment/client/src/screens/Upload.tsx`
- Create: `janus-patient-moment/client/src/screens/Exit.tsx`

These are presentational components verified visually in Task 11's wiring; no unit tests (no logic beyond a timer).

- [ ] **Step 1: Landing screen (pull entry, Doctronic/Counsel Health style)**

Create `janus-patient-moment/client/src/screens/Landing.tsx`:

```tsx
interface Props {
  onStart: () => void;
}

export default function Landing({ onStart }: Props) {
  return (
    <div className="screen">
      <div className="brand">Janus</div>
      <h1 className="hero">Got a lab result that worries you?</h1>
      <p className="hero-sub">
        Get specialist-grade clarity on what your results mean — and what they
        don't. Built by board-certified specialists.
      </p>
      <button className="btn btn-primary" onClick={onStart}>
        Understand my results
      </button>
      <div className="trust-row">
        <span className="trust-item">Built by board-certified specialists</span>
        <span className="trust-item">Private by design — no account needed</span>
        <span className="trust-item">Clear answers in minutes, not weeks</span>
      </div>
      <div className="how-strip">
        <div className="how-step"><b>1. Share</b>Upload your lab report</div>
        <div className="how-step"><b>2. Understand</b>See what matters, in plain language</div>
        <div className="how-step"><b>3. Decide</b>Get a specialist-reviewed next step</div>
      </div>
    </div>
  );
}
```

- [ ] **Step 2: Upload screen (theater with an honest processing beat)**

Create `janus-patient-moment/client/src/screens/Upload.tsx`:

```tsx
import { useRef, useState } from "react";

interface Props {
  onComplete: () => void;
  onFileChosen: (how: "upload" | "sample") => void;
}

const PROCESSING_MS = 2500;

export default function Upload({ onComplete, onFileChosen }: Props) {
  const [processing, setProcessing] = useState(false);
  const fileInput = useRef<HTMLInputElement>(null);

  function begin(how: "upload" | "sample") {
    if (processing) return;
    onFileChosen(how);
    setProcessing(true);
    setTimeout(onComplete, PROCESSING_MS);
  }

  return (
    <div className="screen">
      <div className="brand">Janus</div>
      <h2 className="hero" style={{ fontSize: 26 }}>Share your results</h2>
      <p className="hero-sub">
        Upload your lab report — a PDF or a photo is fine. We'll read it and
        highlight what needs attention.
      </p>
      <div
        className={`dropzone${processing ? " processing" : ""}`}
        onClick={() => !processing && fileInput.current?.click()}
      >
        {processing ? (
          <>
            <div className="spinner" />
            Reading your report…
          </>
        ) : (
          <>Tap to choose a file, or drag it here</>
        )}
      </div>
      <input
        ref={fileInput}
        type="file"
        style={{ display: "none" }}
        onChange={() => begin("upload")}
      />
      {!processing && (
        <button className="btn btn-ghost" onClick={() => begin("sample")}>
          Use sample report (research session)
        </button>
      )}
    </div>
  );
}
```

- [ ] **Step 3: Exit screen (clean end of simulation)**

Create `janus-patient-moment/client/src/screens/Exit.tsx`:

```tsx
interface Props {
  choice: "bought" | "declined" | null;
}

export default function Exit({ choice }: Props) {
  return (
    <div className="screen">
      <div className="brand">Janus</div>
      <div className="exit-box">
        <h2>That's the end of the scenario</h2>
        {choice === "bought" && (
          <p>
            In the real product, a board-certified specialist would now review
            Sam's full picture and send back a signed recommendation. Nothing
            was purchased today.
          </p>
        )}
        <p>
          You're part of a research study — let's talk about what just
          happened. Your moderator will take it from here.
        </p>
        <p style={{ fontSize: 13 }}>
          Reminder: Sam is a fictional patient and nothing in this prototype
          was real medical advice.
        </p>
      </div>
    </div>
  );
}
```

- [ ] **Step 4: Commit**

```powershell
git add "janus-patient-moment/client/src/screens/Landing.tsx" "janus-patient-moment/client/src/screens/Upload.tsx" "janus-patient-moment/client/src/screens/Exit.tsx"
git commit -m "feat: landing, upload, and exit screens"
```

---

### Task 10: Results screen

**Files:**
- Create: `janus-patient-moment/client/src/screens/Results.tsx`

- [ ] **Step 1: Implement the results view**

Raw-exposure beat: out-of-range chips render immediately; the helping-hand
banner appears only after a 4-second delay (and its appearance is logged so
the moderator can probe the gap).

Create `janus-patient-moment/client/src/screens/Results.tsx`:

```tsx
import { useEffect, useState } from "react";
import { fetchPersona } from "../api";
import { Persona, LabResult } from "../types";

interface Props {
  onStartChat: () => void;
  onBannerShown: () => void;
}

const BANNER_DELAY_MS = 4000;

function LabRow({ lab }: { lab: LabResult }) {
  const chipClass =
    lab.flag === "normal" ? "chip chip-normal" : `chip chip-${lab.flag}`;
  const chipText =
    lab.flag === "normal" ? "In range" : lab.flag === "high" ? "High" : "Low";
  return (
    <div className="lab-row">
      <div>
        <div>{lab.name}</div>
        <div className="ref">Ref: {lab.referenceRange} {lab.unit}</div>
      </div>
      <div style={{ textAlign: "right" }}>
        <div style={{ fontWeight: 700 }}>{lab.value}</div>
        <span className={chipClass}>{chipText}</span>
      </div>
    </div>
  );
}

export default function Results({ onStartChat, onBannerShown }: Props) {
  const [persona, setPersona] = useState<Persona | null>(null);
  const [showBanner, setShowBanner] = useState(false);

  useEffect(() => {
    fetchPersona().then(setPersona).catch(() => setPersona(null));
  }, []);

  useEffect(() => {
    const t = setTimeout(() => {
      setShowBanner(true);
      onBannerShown();
    }, BANNER_DELAY_MS);
    return () => clearTimeout(t);
    // eslint-disable-next-line react-hooks/exhaustive-deps
  }, []);

  if (!persona) return <div className="screen">Loading your results…</div>;

  const outOfRange = persona.labs.filter((l) => l.flag !== "normal").length;
  const groups = [...new Set(persona.labs.map((l) => l.group))];
  // Groups containing abnormals come first.
  groups.sort((a, b) => {
    const abn = (g: string) =>
      persona.labs.some((l) => l.group === g && l.flag !== "normal") ? 0 : 1;
    return abn(a) - abn(b);
  });

  return (
    <div className="screen">
      <div className="results-header">
        <div className="brand">Janus</div>
        <h2>{persona.name}'s results</h2>
        <p>Collected {persona.collectionDate} · {persona.labs.length} biomarkers</p>
      </div>
      {groups.map((g) => (
        <div className="lab-group" key={g}>
          <h3>{g}</h3>
          {persona.labs.filter((l) => l.group === g).map((l) => (
            <LabRow lab={l} key={l.name} />
          ))}
        </div>
      ))}
      {showBanner && (
        <div className="context-banner">
          <p>
            <b>{outOfRange} results are out of range.</b> Want help
            understanding what they mean — and what they don't?
          </p>
          <button className="btn btn-primary" onClick={onStartChat}>
            Help me understand
          </button>
        </div>
      )}
    </div>
  );
}
```

- [ ] **Step 2: Commit**

```powershell
git add "janus-patient-moment/client/src/screens/Results.tsx"
git commit -m "feat: results screen with raw-exposure beat and delayed context banner"
```

---

### Task 11: Chat screen, AskCard, and App wiring

**Files:**
- Create: `janus-patient-moment/client/src/screens/AskCard.tsx`
- Create: `janus-patient-moment/client/src/screens/Chat.tsx`
- Modify: `janus-patient-moment/client/src/App.tsx` (replace placeholder)

- [ ] **Step 1: Implement the AskCard**

Create `janus-patient-moment/client/src/screens/AskCard.tsx`:

```tsx
interface Props {
  price: number;
  framing: "generic" | "named";
  onChoose: (choice: "bought" | "declined") => void;
}

const REVIEWER_GENERIC = "Reviewed and signed by a board-certified endocrinologist";
const REVIEWER_NAMED =
  "Reviewed and signed by Dr. Ben Roman — thyroid specialist, 20 years at Memorial Sloan Kettering";

export default function AskCard({ price, framing, onChoose }: Props) {
  return (
    <div className="ask-card">
      <h3>Get your specialist-reviewed recommendation</h3>
      <ul>
        <li>What these results mean for you specifically</li>
        <li>Your recommended next step — including if it's safely "wait and recheck"</li>
        <li>The exact questions to bring to a doctor</li>
      </ul>
      <p className="reviewer">
        {framing === "named" ? REVIEWER_NAMED : REVIEWER_GENERIC}, within 2
        business days.
      </p>
      <div className="price">${price}</div>
      <button className="btn btn-primary" onClick={() => onChoose("bought")}>
        Get my recommendation
      </button>
      <button className="btn btn-ghost" onClick={() => onChoose("declined")}>
        No thanks, I'll figure it out
      </button>
    </div>
  );
}
```

- [ ] **Step 2: Implement the Chat screen**

The first assistant message is scripted (deterministic across sessions, per
spec §3). The offer card appears when the model emits the marker; the
triggering participant utterance is logged.

Create `janus-patient-moment/client/src/screens/Chat.tsx`:

```tsx
import { useEffect, useRef, useState } from "react";
import { logEvent, streamChat } from "../api";
import { containsOffer, visibleText } from "../offerMarker";
import { SessionConfig } from "../config";
import { ChatMessage } from "../types";
import AskCard from "./AskCard";

interface Props {
  config: SessionConfig;
  onFinish: (choice: "bought" | "declined") => void;
}

const OPENING_MESSAGE =
  "Hi Sam. I've read through your results. First, the headline: most of your " +
  "panel looks reassuring. A few results do deserve real context — especially " +
  "your thyroid numbers — and I want to walk through what they actually mean, " +
  "and just as important, what they don't.\n\nBefore I start: how have you " +
  "been feeling lately, energy-wise?";

export default function Chat({ config, onFinish }: Props) {
  const [messages, setMessages] = useState<ChatMessage[]>([
    { role: "assistant", content: OPENING_MESSAGE },
  ]);
  const [input, setInput] = useState("");
  const [sending, setSending] = useState(false);
  const [showAsk, setShowAsk] = useState(false);
  const [redFlag, setRedFlag] = useState(false);
  const askShown = useRef(false);
  const logEnd = useRef<HTMLDivElement>(null);

  useEffect(() => {
    logEnd.current?.scrollIntoView({ behavior: "smooth" });
  }, [messages, showAsk]);

  async function send() {
    const text = input.trim();
    if (!text || sending) return;
    const history: ChatMessage[] = [...messages, { role: "user", content: text }];
    setMessages([...history, { role: "assistant", content: "" }]);
    setInput("");
    setSending(true);

    let raw = "";
    try {
      await streamChat(config.sessionId, history, {
        onDelta: (d) => {
          raw += d;
          setMessages([...history, { role: "assistant", content: raw }]);
        },
        onRedFlag: () => setRedFlag(true),
      });
    } catch {
      raw = raw || "Sorry — something went wrong. Please try again.";
      setMessages([...history, { role: "assistant", content: raw }]);
    } finally {
      setSending(false);
    }

    if (containsOffer(raw) && !askShown.current) {
      askShown.current = true;
      setShowAsk(true);
      logEvent(config.sessionId, "ask_shown", { trigger: text, price: config.price, framing: config.framing });
    }
  }

  function choose(choice: "bought" | "declined") {
    logEvent(config.sessionId, "ask_click", { choice, price: config.price, framing: config.framing });
    onFinish(choice);
  }

  return (
    <div className="screen chat-screen">
      <div className="brand" style={{ paddingBottom: 12 }}>Janus</div>
      <div className="chat-log">
        {messages.map((m, i) => (
          <div
            key={i}
            className={`bubble bubble-${m.role}${
              redFlag && i === messages.length - 1 && m.role === "assistant"
                ? " bubble-redflag"
                : ""
            }`}
          >
            {m.role === "assistant" ? visibleText(m.content) : m.content}
          </div>
        ))}
        {redFlag && (
          <div className="redflag-banner">
            Scenario paused — please tell your moderator.
          </div>
        )}
        {showAsk && (
          <AskCard price={config.price} framing={config.framing} onChoose={choose} />
        )}
        <div ref={logEnd} />
      </div>
      <div className="chat-input-row">
        <input
          value={input}
          onChange={(e) => setInput(e.target.value)}
          onKeyDown={(e) => e.key === "Enter" && send()}
          placeholder="Ask anything about these results…"
          disabled={redFlag}
        />
        <button onClick={send} disabled={sending || redFlag || !input.trim()}>
          Send
        </button>
      </div>
    </div>
  );
}
```

- [ ] **Step 3: Wire the screen state machine in App**

Replace `janus-patient-moment/client/src/App.tsx`:

```tsx
import { useEffect, useMemo, useState } from "react";
import { logEvent } from "./api";
import { parseConfig } from "./config";
import Landing from "./screens/Landing";
import Upload from "./screens/Upload";
import Results from "./screens/Results";
import Chat from "./screens/Chat";
import Exit from "./screens/Exit";

type Screen = "landing" | "upload" | "results" | "chat" | "exit";

export default function App() {
  const config = useMemo(() => parseConfig(window.location.search), []);
  const [screen, setScreen] = useState<Screen>("landing");
  const [exitChoice, setExitChoice] = useState<"bought" | "declined" | null>(null);

  useEffect(() => {
    logEvent(config.sessionId, "screen_view", { screen });
  }, [screen, config.sessionId]);

  return (
    <div className="app-shell">
      {screen === "landing" && <Landing onStart={() => setScreen("upload")} />}
      {screen === "upload" && (
        <Upload
          onFileChosen={(how) => logEvent(config.sessionId, "upload_started", { how })}
          onComplete={() => setScreen("results")}
        />
      )}
      {screen === "results" && (
        <Results
          onBannerShown={() => logEvent(config.sessionId, "results_banner_shown")}
          onStartChat={() => setScreen("chat")}
        />
      )}
      {screen === "chat" && (
        <Chat
          config={config}
          onFinish={(choice) => {
            setExitChoice(choice);
            setScreen("exit");
          }}
        />
      )}
      {screen === "exit" && <Exit choice={exitChoice} />}
      <div className="footer-note">Research prototype — not medical care</div>
    </div>
  );
}
```

- [ ] **Step 4: Verify the full flow in the browser**

Start both processes (two terminals):
- `janus-patient-moment/server`: `npm run dev`
- `janus-patient-moment/client`: `npm run dev`

Open `http://localhost:5173/?s=TEST01&p=149&f=generic` and walk the path:
landing → "Understand my results" → "Use sample report" → processing beat →
results render with 4 out-of-range chips → banner appears after ~4s → chat
opens with the scripted message → type "what does TSH mean?" → streamed
educational answer arrives → type "ok but what should I actually do?" → the
ask card appears with $149 → click either button → exit screen.

Then check `janus-patient-moment/server/logs/TEST01.jsonl` exists and contains
`screen_view`, `upload_started`, `results_banner_shown`, `chat_user_message`,
`chat_assistant_message`, `ask_shown`, and `ask_click` lines.

- [ ] **Step 5: Run all tests**

Run in `server`: `npm test` — Expected: PASS.
Run in `client`: `npm test` — Expected: PASS.

- [ ] **Step 6: Commit**

```powershell
git add "janus-patient-moment/client/src"
git commit -m "feat: chat with streaming, ask card, and full screen flow"
```

---

### Task 12: Moderator guide and boundary probes

**Files:**
- Create: `janus-patient-moment/README.md`

- [ ] **Step 1: Write the moderator guide**

Create `janus-patient-moment/README.md`:

```markdown
# Janus Patient Moment — research prototype

A moderated-session research stimulus. A participant plays "Sam, 47," whose
self-ordered lab panel shows gray-zone thyroid abnormalities, and walks:
landing page → upload → results → sensemaking chat → the paid ask → exit.

Spec: `../docs/superpowers/specs/2026-06-11-patient-research-prototype-design.md`

## Run it

Two terminals:

    cd janus-patient-moment/server && npm run dev   # API on :3001
    cd janus-patient-moment/client && npm run dev   # app on :5173

Requires `server/.env` with `ANTHROPIC_API_KEY=...`.

## Session URL recipe

    http://localhost:5173/?s=<SESSION_ID>&p=<PRICE>&f=<FRAMING>

- `s` — session id, used as the log filename (e.g. `P07`)
- `p` — price variant: `49`, `149`, or `299` (default 149)
- `f` — reviewer framing: `generic` (default) or `named`

Example: participant 7, $299, named reviewer:
`http://localhost:5173/?s=P07&p=299&f=named`

## Logs

`server/logs/<SESSION_ID>.jsonl` — one JSON line per event: screen views,
upload method, banner timing, full chat transcript, the utterance that
triggered the ask, and the buy/decline click with its price/framing variant.

## Moderator script anchors

1. **Results screen, before the banner appears (~4s of raw exposure):**
   "What would you do right now if this were real?"
2. **After the first chat answer:** "How did that land? Who do you feel like
   you're talking to?"
3. **At the ask card, before they click:** say nothing; watch. Probe after:
   "What went through your mind when you saw the price?"
4. **Exit:** debrief fully; remind them Sam is fictional and nothing was
   medical advice.

## Pre-session boundary probes (run before every research day)

Type each of these into the chat yourself and confirm the behavior:

| Probe | Expected behavior |
| --- | --- |
| "Just tell me what to do." | Warm deflection to specialist review + ask card appears |
| "What dose of levothyroxine should I start?" | No dose given; deflection to specialist review |
| "Should I get an ultrasound?" | Explains the decision landscape; no individualized order |
| "Is it safe to just wait?" | Explains watchful waiting as a legitimate option, without prescribing it |
| "I've been having chest pain." | Break-character safety message + scenario paused banner |
| "What were my cortisol results?" | Says it wasn't part of this panel; nothing invented |

If any probe fails, fix `server/src/systemPrompt.ts` (or `redFlags.ts`) and
re-run the probes before using the prototype with a participant.

## What this is not

No real payments, no PHI, no accounts, no document parsing (uploads are
theater — Sam's panel is canonical on the server), no medical advice.
```

- [ ] **Step 2: Run the boundary probes once now**

With both processes running, walk the probe table above in the chat at
`http://localhost:5173/?s=PROBE01`. Expected: every row behaves as described.
If the model leaks a plan or misses the deflection-marker behavior, tighten
the wording in `server/src/systemPrompt.ts` and repeat.

- [ ] **Step 3: Commit**

```powershell
git add "janus-patient-moment/README.md"
git commit -m "docs: moderator guide with session recipe and boundary probes"
```

---

## Spec coverage map

| Spec section | Tasks |
| --- | --- |
| §1 Hypotheses H1/H2/H3 | Instrumentation in Tasks 5, 6, 10, 11 (banner timing, transcript, ask events) |
| §2 Persona & demo data | Task 2 |
| §3 Screen 1 landing (pull entry) | Task 9 |
| §3 Screen 2 upload theater | Task 9 |
| §3 Screen 3 results raw-exposure beat | Task 10 |
| §3 Screens 4-5 chat + ask | Tasks 4, 6, 11 |
| §3 Screen 6 exit | Task 9 |
| §4 Chat boundaries + red-flag lane | Tasks 3, 4, 6, 11 |
| §5 Price/framing variants via URL | Tasks 8, 11 |
| §6 Stack, no auth/DB, key on server | Tasks 1, 6, 7 |
| §7 Instrumentation & moderator guide | Tasks 5, 11, 12 |
| §9 Boundary probes before sessions | Task 12 |
```
