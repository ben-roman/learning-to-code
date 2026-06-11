# Learning to Code — Janus Projects

Personal workspace for Ben's coding learning journey, started 2026-06-10.

## Structure

```
.
├── hello.ts                    # Initial hello world TypeScript script
├── package.json                # Root workspace (tsx + typescript)
├── docs/
│   └── superpowers/
│       ├── specs/              # Design specs
│       └── plans/              # Implementation plans
└── janus-patient-moment/
    └── server/                 # Express + TypeScript backend
```

## Projects

### Janus Patient Moment (`janus-patient-moment/`)

A research prototype used as a stimulus in moderated patient research sessions.
Fictional patient "Sam, 47" receives abnormal D2C lab results (elevated TSH with
positive TPO antibodies) and navigates a calming, AI-assisted result experience.

Tests three hypotheses:
- **H1** — Calm, contextual result presentation reduces anxiety vs. raw report + Google
- **H2** — Patients trust more after clarifying questions before AI answers
- **H3** — Meaningful conversion on in-flow specialist recommendation ($49/$149/$299)

**Stack:** Node.js, Express, TypeScript, Anthropic Claude API

#### Server setup

```bash
cd janus-patient-moment/server
cp .env.example .env        # add your ANTHROPIC_API_KEY
npm install
npm run dev                 # starts with tsx watch
npm test                    # runs vitest
```

## Root scripts

```bash
npm start    # runs hello.ts via tsx
```
