# AI Harness Weekend Project — Context

## About this file
This document is the working context for a weekend project to build an AI harness. It is intended to be opened in VS Code alongside Claude so that Claude understands the project goals, constraints, architecture, and what *not* to build. Read this file first before generating any code.

## About me (the builder)
- UX designer with enterprise CRM workflow experience.
- Goal is hands-on learning of AI harness architecture for application in day job.
- Not learning the code language itself — Claude writes the code, I review and learn the architecture and the reasoning behind each piece.
- Comfortable reading JavaScript/TypeScript and some Node.js; have not authored production code.
- Have used an Anthropic API key before in a simple agent tutorial.
- Have never set up OAuth before.
- Do not have a Google Cloud Console account yet.
- Dev environment is ready: Node installed, VS Code with Claude extension working.

## Goals for the weekend
1. Build a working AI harness end-to-end.
2. Be able to explain on Sunday night: *"I built a harness where deterministic rules, AI judgment, validation, and human feedback work together"* — not just *"I built an email classifier."*
3. Leave with a working triage tool I can use on my real inbox.
4. Take a sharper mental model back to my day job for reviewing AI feature proposals.

## Core principles (the meta-lessons — keep these visible while building)
1. **The shape is the contract.** Fake data and real data feed the same pipeline. Source is swappable.
2. **Use AI for judgment under ambiguity. Use code for everything else.** If an `if` statement can do it, do not call the model.
3. **The reasoning field is the audit trail.** It is the difference between a black-box agent and a harness.
4. **The validation layer separates two questions:** *what is this?* (AI's job) and *should we trust the answer?* (deterministic).
5. **Users trust systems where they can see what changed, when, and why.** Preference ledgers over invisible retraining.
6. **Discipline about what NOT to build is the hard part.** Every cut item is a Saturday-afternoon detour waiting to happen.

---

## Architecture: six layers

### 1. Input
Each email is normalized to this shape before anything else touches it:
```
{
  id: string,
  from: string,
  subject: string,
  snippet: string,
  date: string
}
```
- No full email bodies. Snippet only (~100 chars). This is deliberate — bodies are expensive in tokens, often noisy, and rarely change classification.
- Saturday morning: synthetic generator produces this shape.
- Saturday afternoon: Gmail fetcher produces this shape.
- Nothing downstream knows or cares which source it came from.

### 2. Rules layer (deterministic, runs BEFORE the AI)
Plain `if` statements. If a rule matches, the email is classified without an API call.

Starter rules:
```
from contains "nytimes.com" OR "wsj.com"        → News
from contains "morningbrew" OR "axios"           → Newsletter-Money
subject contains "receipt" OR "order #"          → Receipts
from contains "mychart" OR "kaiser"              → Medical
from contains "noreply"                          → Notification (catch-all)
```
- Rules run in order; order matters and encodes my priorities.
- Categories above are the *topic-based* taxonomy I chose (vs. urgency-based).
- Only emails that survive ALL rules get passed to the AI.

### 3. AI classification layer
Forced output schema (Zod or equivalent):
```
{
  category: enum["News", "Newsletter-Money", "Receipts", "Medical",
                 "Personal", "Work", "Promotional", "Unknown"],
  confidence_score: number,         // 0.0 - 1.0
  summary: string,                  // max 10 words
  reasoning: string                 // max 50 chars
}
```
- `category` enum is the leash that prevents Claude from inventing new categories.
- `"Unknown"` is the honest escape hatch — better than a confident wrong guess.
- `confidence_score` is a vibe, not a calibrated measurement — still useful as a signal.
- `summary` is what the UI shows instead of the subject line.
- `reasoning` is the audit trail. Without it, "Wrong" is data without diagnosis.

### 4. Validation layer (deterministic, runs AFTER the AI)
```
if (zodParse failed)                       → needs_review
if (confidence_score < 0.70)               → needs_review
if (category === "Unknown")                → needs_review
if (high-stakes category && score < 0.9)   → flag with warning
```
- This is the layer that earns the word "harness."
- AI layer asks: *what is this?* Validation layer asks: *should we trust the answer?*
- Thresholds (0.70, 0.9) are my design decisions and encode a risk posture.

### 5. Human layer (UI)
Per-email card shows:
```
[Newsletter-Money  82%]
Morning Brew — markets digest, 4 min read
Because: financial newsletter keywords detected

[✓ Correct]  [✗ Wrong]  [Skip]
```
- Buttons do NOT retrain anything. They append to `corrections.json`.
- The corrections file is a **preference ledger**, not a model update.
- This is plain code: a button click triggers a file write. No model call at runtime.
- Future work (NOT this weekend): a separate process reads the ledger and proposes new rules.

### 6. Action layer
Only allowed actions:
- Archive
- Mark Important
- Review Later

NOT allowed:
- Reply / compose
- Delete
- Any autonomous action

The `gmail.metadata` OAuth scope makes send/delete physically impossible at the token level — belt and suspenders.

---

## Roadmap

### Saturday morning (3–4 hrs) — harness on FAKE data
1. Vite + React scaffold (no Next.js, no router, no auth pages).
2. Synthetic email generator: ~30 emails across the category mix.
3. Rules layer with starter rules above.
4. AI classifier calling Anthropic API with the Zod schema.
5. Validation layer with the threshold checks.
6. Minimal UI: card per email with badge, summary, reasoning, three buttons.
7. `corrections.json` append handler.

**If I stop here, I've built the harness. That is the win condition.**

### Saturday afternoon (2–3 hrs) — swap data source to real Gmail
1. Create Google Cloud Console account + project.
2. Enable Gmail API.
3. Configure OAuth consent screen, add myself as test user.
4. Request `gmail.metadata` scope (read-only by token, not by trust).
5. Handle the OAuth flow, get a token.
6. Fetch 20 real emails, transform into the Input shape from Layer 1.
7. Point the harness at real data. Nothing downstream changes.

### Sunday — buffer
- Whatever ate Saturday.
- Or polish: better styling, more rules based on what I learned.
- Or extend: batch archive, more categories, a corrections-file viewer.
- Or write up what I learned for my day job.

---

## What I am explicitly NOT building this weekend
Cut from the original plan for good reasons. Do not let me drift into these:
- Next.js. Vite + React is plenty.
- Persistent memory beyond `corrections.json`.
- Any kind of retraining loop.
- Batch archive execution.
- Reply generation.
- Delete actions.
- A polished "dashboard" — the UI should look like a triage tool, not a dashboard.
- Authentication beyond Gmail OAuth.
- Routing, multiple pages.

---

## Constraints and gotchas to remember
- I have ~12–16 hours total, not 24+. Saturday morning + Sunday, not Friday evening through Sunday.
- OAuth is the single most likely thing to eat a day. That is why it comes AFTER the harness works on fake data.
- Google Cloud Console setup is manual UI clicking — Claude cannot do this part for me. Budget 30–60 minutes.
- I am using my real personal Gmail. Read-only via `gmail.metadata` scope is a hard guarantee, not a soft one.
- Two different Claudes are in play: the one inside VS Code helping me build, and the one inside my app doing classification. They have no memory of each other.

---

## How to use Claude in VS Code on this project
- Claude writes the code; I review and ask why.
- When I ask "why is this here?" — explain the architectural reason, not just the syntax.
- When I propose adding something, check it against the "explicitly NOT building" list above and push back if it fits.
- When something breaks, explain what layer it broke in (Input / Rules / AI / Validation / Human / Action) so I learn to diagnose by layer.
- Prefer the smallest change that teaches the lesson over the most elegant change.

---

## Success criteria for Sunday night
I can point at my running app and explain, for any email on screen:
1. Which layer classified it (rules vs AI).
2. If AI: what the confidence was and why validation either trusted or flagged it.
3. What clicking "Wrong" would do, and what it would NOT do.
4. Why each layer exists and what would break if I removed it.

If I can do that, the weekend worked — regardless of whether the Gmail OAuth piece landed.
