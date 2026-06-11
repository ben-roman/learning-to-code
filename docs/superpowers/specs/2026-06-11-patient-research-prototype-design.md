# Janus Patient Moment — Research Prototype Design

**Date:** 2026-06-11
**Status:** Approved by Ben (pending final spec review)
**Type:** Patient research stimulus (moderated sessions, hands-on with sample persona)

## 1. Purpose & hypotheses

A thin working web prototype used as a stimulus in moderated patient research
sessions. The participant plays a fictional patient whose self-ordered D2C lab
panel came back with abnormal results. Three hypotheses, tested at three
moments:

- **H1 — The result lands:** A calm, contextual first presentation of an
  abnormal lab result beats the raw report + Google/ChatGPT on anxiety and
  perceived clarity.
- **H2 — Sensemaking chat:** Patients tolerate — and trust more after —
  clarifying questions before answers, when the chat is warm, instant, and
  clearly bounded.
- **H3 — The ask:** A meaningful fraction will click "buy" on a human-reviewed
  specialist recommendation in-flow; price sensitivity is measurable across
  $49 / $149 / $299 variants.

This is research apparatus, not a product MVP. Everything behind the tested
moments may be smoke and mirrors.

## 2. Persona & demo data

- Fictional patient **"Sam, 47"** — self-ordered a Function Health–style full
  panel; vague fatigue; no established PCP relationship (mirrors the Hannah
  interview); not pregnant; no red-flag symptoms.
- Demo data uses the **authentic structure of the deidentified Function Health
  report** (`02_Areas/04_Product/2026-03-05_FunctionHealth_Full_Lab_Results_DEIDENTIFIED.pdf`)
  but with **authored values**: headline abnormal is **TSH ~6.8 with positive
  TPO antibodies** (Wave 1 thyroid-hormone pathway; textbook gray-zone result
  where "watch and recheck" is a legitimate option). The panel's real
  borderline lipids (total cholesterol 205, LDL 113) remain as secondary
  flags, mirroring the multi-flag reality of D2C panels.
- A one-page persona card is shown/read to the participant before they start.

## 3. Flow & screens (6 screens, one path)

1. **Landing page (referral entry).** The participant arrives the way a real
   Janus patient would — someone told them to come here. Styled like
   Doctronic AI / Counsel Health: consumer-health front door with a hero
   promise ("Got a lab result that worries you? Get specialist-grade
   clarity."), trust signals (built by specialists, private, no account
   needed), and a single CTA. No signup.
2. **Bring your results.** Upload step: drag-and-drop / file picker. In
   session, the participant "uploads" the persona's sample report (a visible
   "use sample report" affordance keeps sessions smooth; any uploaded file is
   accepted but the persona data is what gets used). A brief, honest
   processing beat ("reading your report…").
3. **Results view.** Clean lab summary, out-of-range chips highlighted, full
   report viewable. Deliberately a beat of raw exposure — let the anxiety
   spike happen — before a quiet banner offers the hand: *"3 results need
   context. Want help understanding them?"*
4. **Sensemaking chat.** Live LLM (see §4). Opens with calm framing of the
   TSH result, then asks its first clarifying question.
5. **The ask.** When the participant pushes for "so what should I do?", the
   chat reaches its boundary honestly and a card slides in:
   *Specialist-reviewed recommendation — what this means for you, your next
   step, and questions for your doctor, reviewed and signed by a
   board-certified specialist within 2 business days — $[X]*.
   Buttons: **Get my recommendation** / **No thanks, I'll figure it out**.
6. **Exit screen.** Either path lands on "You're part of a research study —
   let's talk about what just happened," ending the simulation cleanly.

## 4. Chat engine & boundaries

One Claude API call chain. The system prompt encodes — and becomes a working
draft of — the patient-education boundary from the Technical Operating Plan:

- Knows the full persona panel and history. Warm, plain-language, never
  alarmist. **One clarifying question at a time.**
- **Education only, no individualized plan.** Explains what a TSH of 6.8
  usually means, lays out the decision landscape including "watch and recheck
  is often safe" (do-nothing-safely as a first-class option, per the
  orchestrator-rules report), but deflects plan requests to the specialist
  review. The deflection is the paywall trigger.
- **Deterministic red-flag lane:** mentions of chest pain, suicidality,
  pregnancy, severe symptoms break character, advise real-world care, and
  visibly signal the moderator to pause the session.
- Tone target: instant, on the mark, warm — the bar set by the Carrie
  transcript ("even if it's just AI, I don't care").

## 5. The ask — configurable variants

Per-session config via URL parameters (Thobith-style link + code, no auth):

- **Price:** $49 / $149 / $299.
- **Reviewer framing (optional):** generic ("reviewed by an endocrinologist")
  vs. named-trust ("reviewed by Dr. [Name], thyroid specialist, 20 years at
  MSK") — a direct test of the demeanor/credential trust dynamic from the
  Hannah interview.

Click-through is logged. No real payment.

## 6. Architecture & stack

- **Front end:** Vite + React + TypeScript. Mobile-first layout (real patients
  do this on phones).
- **Back end:** minimal Node/Express server — holds the Anthropic API key
  (never in the browser), streams chat responses, writes session logs.
- **Hosting:** local for in-person sessions; cheap host (Render/Railway) for
  remote share-a-link sessions.
- **No auth, no database.** Session = one URL with config params. Logs =
  append-only JSON files per session on the server.
- Doubles as Ben's learning-to-code project; code should stay small and
  readable.

## 7. Instrumentation & debrief

Each session log captures: timestamps per screen, full chat transcript, the
exact utterance that triggered the ask, the button clicked, price/framing
variant. Paired with a one-page moderator guide probing each moment (e.g., at
the results view, before chat is offered: "What would you do right now if this
were real?").

## 8. Out of scope

No real payments, no PHI, no account system, no final decision object, no
clinician console, no Ongoing Note persistence, no image interpretation, no
real document parsing (uploads are theater; persona data is canonical). One
pathway, one persona — config keeps a second persona cheap later.

## 9. Risks & mitigations

- **Participant types something clinically urgent →** deterministic red-flag
  lane + moderator pause (per §4).
- **Chat output drifts beyond the education boundary →** boundary lives in
  one versioned system prompt file; test with adversarial "just tell me what
  to do" probes before first session.
- **Simulation feels fake at the upload step →** processing beat + results
  view rendered from persona data in the report's authentic visual structure.
- **Confusion that this is real medical care →** persona framing in the
  briefing, exit screen debrief, and a persistent subtle "research prototype"
  marker in the footer.
