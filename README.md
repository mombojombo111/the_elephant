# The Elephant

A structured dialogue facilitation tool that pairs ideological opponents — people who genuinely disagree on contested topics like abortion, gun policy, religion, or geopolitics — and walks them through a twelve-week asynchronous conversation designed to produce durable shifts in good-faith attribution.

**The goal is not to change anyone's mind.** The goal is for each participant to end the process trusting that the other side is arguing in good faith from a coherent worldview, even when they continue to disagree.

Based on the methods in Alice Lombardo Maher's *Catalysis: A Recipe to Slow Down or Abort Humankind's Leap to War* (2018).

## Status

Full-stack prototype for closed-cohort testing. **Not** a public product, mobile app, or open-registration service.

## Core hypothesis

Most ideological conflicts persist not because parties understand each other poorly but because each side is fighting an *idealized or demonized version* of the other. Real resolution happens when at least one party relinquishes a performed position and allows the other to see them as limited, flawed, and accurately human. Understanding follows honesty; it does not precede it.

## The four acts

A paired dialogue runs asynchronously over twelve weeks across four acts:

| Act | Name | Duration | Purpose |
|-----|------|----------|---------|
| I | **Sandcastle** | 2 weeks | Low-stakes collaborative creative tasks to build rapport before any ideological content |
| II | **Letters** | 3 weeks | Autobiographical exchange — formative stories, not commentary — to build an empathic map |
| III | **Hot Stove** | 4 weeks | The contested topic engaged, supported by Translator and Temperature Gauge |
| IV | **Curtain** | 3 weeks | A single moment of facilitated honest self-exposure. The real event; everything else is scaffolding |

### The Curtain prompt (verbatim)

> Name one thing your partner has been needing you to be in this conversation that you are not. Name it directly, in the first person, without apology or self-flagellation.

## Non-negotiable principles

- **Slow is load-bearing.** Time gaps between messages are the mechanism, not UX friction. No "send now" overrides, typing indicators, read receipts, or reactions.
- **The AI paraphrases, it does not judge.** No scoring, winners, leaderboards, or "quality" metrics. The system is a container, not a judge.
- **Human facilitators are first-class.** Trained peer mediators gate progression into Act IV; AI assists at the margins only.
- **Every action is reversible, declinable, or exitable.** Withdrawal is free and unstigmatized at every stage.
- **Under-ship rather than over-ship.** A minimal working prototype with strong safety beats a polished product with weak safety.

## Architecture

### Tech stack

- **Backend:** Python 3.11+, FastAPI, Pydantic, SQLAlchemy
- **Database:** PostgreSQL 15+, column-level encryption for message content
- **Frontend:** Next.js 14+, React, TypeScript, Tailwind (server components where possible)
- **Auth:** Magic-link email only (no passwords, no social login)
- **AI:** Anthropic Claude API (`claude-opus-4-7`) for Translator and facilitator-facing readiness hints
- **Jobs:** Postgres-backed queue for scheduled unlocks, reminders, and cadence gaps

### Primary surfaces

- **Participant web app** — dialogue interface, act progression, private reflection
- **Facilitator dashboard** — pair monitoring, readiness flags, check-in tooling
- **Admin panel** — cohort management, pair matching, prompt curation, metrics

### Repository layout

```
backend/        FastAPI app, Alembic migrations, SQLAlchemy models
frontend/       Next.js 14 App Router app
prompts/        Claude prompt files (Markdown + YAML frontmatter, loaded at runtime)
seed/           JSON seed prompts per act
infra/          docker-compose for local Postgres + app
```

## Key mechanics

### Slow cadence (enforced)

Every message has an `unlocked_at` set from `sent_at + delay(act)`:

- Act 1 (Sandcastle): 2 hours
- Act 2 (Letters): 12 hours
- Act 3 (Hot Stove): 6 hours
- Act 4 (Curtain): 24 hours after sharing

The UI shows a countdown, not the message body, during the lock window.

### Translator (Act 3+)

A private, on-demand paraphrase of either side's last message in language stripped of trigger words. Returns `paraphrase`, `underlying_concern`, `suggested_question`. The author is never told a translation was requested. Refuses on slurs, threats, or doxxing — refusal surfaces a pause prompt to the requester only.

### Temperature Gauge

A persistent "I'm getting activated" button. Tapping pauses the dialogue for 24 hours, prompts the tapper for private reflection, and notifies the partner with a neutral pause notice (no reason given). Repeated taps within a week notify the facilitator.

### Facilitator-gated Act IV

Advancement to Act IV is **only** via a facilitator `ready_for_act_4` flag — no calendar fallback. If no flag is set by week 9, the pair ends in Act III. Ending in Act III is a successful outcome, not a failure.

### Readiness hint (facilitator-only)

Claude can read the full Act 3 transcript and return `signals_of_readiness`, `signals_of_risk`, and a `recommendation` (`ready` / `not_yet` / `should_end`). Facilitator-facing only; never shown to participants. Advisory — never decisive.

## Safety and privacy

- Column-level encryption on every message body, draft, translation, and reflection
- No logging of message content, client- or server-side; Anthropic API called with no-logging header where available
- Pre-post moderation on every message — self-harm content triggers crisis resources and immediate facilitator notification
- Participants can withdraw at any time; transcripts preserved, outgoing messages replaceable with a withdrawal placeholder
- No public feed, no "featured dialogue," no cross-pair content sharing, ever
- Crisis resources (988, Crisis Text Line, international equivalents) reachable from every screen
- Sessions: HttpOnly + Secure + SameSite=Strict cookies, 24h cap, magic-link re-auth

## Metrics

**Primary outcome:** trust-shift delta on *"How much do you trust that people who disagree with you are acting in good faith?"* (1–7), measured at intake and cohort completion.

Secondary: act completion rate, Curtain completion (none / one-sided / two-sided), understanding-shift delta, facilitator flag accuracy vs. post-Curtain trust shift.

**Do not track:** "minds changed," position-shift proxies, or anything usable to evaluate dialogue "winners."

## Build order

Each phase ships to internal testers before the next:

- **Phase 0** — Repo scaffold, tooling, local Postgres
- **Phase 1 (weeks 1–3)** — Data model, auth, intake, pair matching, admin panel
- **Phase 2 (weeks 4–6)** — Acts 1 and 2 with enforced cadence and pre-post argumentative-framing review
- **Phase 3 (weeks 7–9)** — Act 3 with Translator, Temperature Gauge, moderator, facilitator dashboard
- **Phase 4 (weeks 10–12)** — Act 4 private drafting, sharing, retraction, post-Curtain check-ins, readiness hint
- **Phase 5 (weeks 13–14)** — 4–6 internal pairs run the full program (compressed ~6 weeks) before any real volunteer cohort

## Before the first real cohort

Do not onboard real participants until every one of these is answered:

1. Source community for the first cohort
2. Initial facilitator roster and trainer
3. Clinician review of seed prompts complete
4. IRB / legal review path for transcript-derived research
5. Facilitator compensation model

A prototype with no participants is recoverable. A prototype that harms someone is not.

## What this is not

Not a therapy tool. Not a debate platform. Not a mind-changing tool. Not a scalable public product. It is a narrow, human-supervised experiment in whether structured asynchronous dialogue, Claude-assisted paraphrasing, and a single moment of facilitated honest exposure can produce durable shifts in good-faith attribution between ideological opponents who entered in genuine disagreement.

## Documents

- [`The_Elephant_Spec_Complete.md`](The_Elephant_Spec_Complete.md) — full design rationale (Part I) and build specification (Part II)
- [`BUILD_PLAN.md`](BUILD_PLAN.md) — phase-by-phase task breakdown with exit criteria
