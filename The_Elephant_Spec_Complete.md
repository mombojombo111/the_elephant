# The Elephant — Design and Build Specification

**Version:** 2.0 (consolidated)
**Intended reader:** Claude Code (or a human engineer driving Claude Code)
**Scope:** Full-stack prototype of a dialogue facilitation tool for closed-cohort testing
**Not in scope:** Public launch, mobile apps, payment systems, open registration

---

## How to read this document

This document has two parts. **Part I** is the design rationale — what the tool is trying to do, why each mechanic exists, and what it's based on. **Part II** is the build specification — tech stack, data model, APIs, prompts, build order.

**Do not skip Part I.** The *why* behind the mechanics matters more than the mechanics themselves. Several decisions in Part II will look wrong or under-motivated without the context from Part I, and the implementing agent will make bad calls on edge cases if they only have the engineering spec.

---

# PART I — Design Rationale

## 1. What this tool is

The Elephant is a structured dialogue facilitation tool that pairs ideological opponents — people who genuinely disagree on contested topics like abortion, gun policy, religion, or geopolitics — and walks them through a twelve-week asynchronous conversation designed to produce durable shifts in good-faith attribution. The goal is not to change anyone's mind. The goal is for each participant to end the process trusting that the other side is arguing in good faith from a coherent worldview, even when they continue to disagree.

The tool is based on the methods described in Alice Lombardo Maher's *Catalysis: A Recipe to Slow Down or Abort Humankind's Leap to War* (2018). Maher is a psychiatrist and psychoanalyst who argues that humanity has developed extraordinary machinery for resolving scientific and technical disagreements but almost nothing comparable for resolving ideological and identity-based ones. Her book proposes that a new "language" for cross-ideological communication can be discovered by practice, modeled on psychoanalytic concepts like transference, binocular vision, and the distinction between individual and large-group identity.

## 2. The core hypothesis

Most ideological conflicts persist not because the parties understand each other poorly but because each side is fighting an *idealized or demonized version* of the other. Real resolution happens when at least one party relinquishes a performed position — righteous, certain, unimpeachable — and allows the other to see them as limited, flawed, and accurately human. Understanding follows honesty; it does not precede it.

This hypothesis comes from close reading of the autobiographical climax of Maher's book. She describes a decade-long collaboration with a senior analyst she calls "Dr. X," during which they developed an inverted psychoanalysis she named "catalysis." The collaboration eventually collapsed into a year-long "war" that nearly destroyed both of them. The war ended in the final minutes of what they had agreed would be their last meeting, when Dr. X said, weary and plainly, *"You're right, I am. See, I'm a disappointment to you."* Maher frames the breakthrough as an insight about thinking-style differences and language. On a closer reading, the active ingredient was Dr. X surrendering the idealized position Maher had needed him to hold for a decade and letting himself be seen as a flawed human being. That move — not the linguistic reframing — is what completed the work.

The design of this tool follows from that reading. Most mechanics that appear to be about "understanding" are actually preparation for a single, late-stage moment where one or both participants let themselves be seen accurately. That moment is the product. Everything else is scaffolding.

## 3. Why the tool exists in the form it does

### 3.1 Why asynchronous only

Real-time dialogue rewards quick reactions. Quick reactions are the enemy of what this tool is trying to produce. Maher is explicit throughout her book that instant responses are defensive, not reflective — that good-faith engagement across deep disagreement requires the slow metabolization of discomfort over hours and days, not seconds. Every design choice that accelerates response cadence works against the tool's purpose.

### 3.2 Why four acts, sequenced

The mechanics of Act IV (the Curtain) do not work without the conditions built in Acts I–III. Specifically:

- **Act I (Sandcastle)** builds rapport on low-stakes creative tasks before introducing any ideological content. Without this, the partner reads as a representative of an ideological tent rather than as a human being.
- **Act II (Letters)** exchanges formative autobiography before contested content. Without this, participants have no empathic map of each other, and the Curtain prompt in Act IV lands as either performance or attack.
- **Act III (Hot Stove)** is where the contested topic is actually engaged, with Translator and Temperature Gauge support. Without this, participants haven't yet encountered the specific ways their partner activates them, so they have nothing honest to name in Act IV.
- **Act IV (Curtain)** is the real event. It works only on the floor that Acts I–III build.

Pair matching is thinking-style-first, ideology-second. If Maher is right that most "ideological" fights are cognitive-style fights in costume, removing the cognitive-style mismatch should isolate the ideological content and make it tractable.

### 3.3 Why two AI mechanics: Translator and (optional) Readiness Hint

The Translator addresses the second-order problem: that people hear each other's words through activation. It paraphrases either side's last message, on demand and privately, in language stripped of trigger words. It does not judge, score, or summarize positions as equivalent. It is deliberately limited. It is useful but not transformative.

The Readiness Hint is a facilitator-facing tool that reads the Act III transcript and offers a recommendation on whether a pair is ready for Act IV. It is never shown to participants. Facilitators can disregard it. Its purpose is to help facilitators calibrate their own judgment, not replace it.

Neither AI mechanic evaluates participants. Any feature where Claude judges whose argument is better, scores dialogues, or declares winners is a feature that defeats the purpose.

### 3.4 Why human facilitators are not optional

Act IV requires human judgment that cannot be automated responsibly. A trained peer facilitator needs to monitor pair dynamics during Acts I–III, flag pairs that should not advance to Act IV, decide when a ready pair is actually ready, and hold the result if the Curtain lands badly. AI can help at the margins, but the core of the facilitator role is human.

Facilitators are not therapists. Their role is assessment and lightweight support. They are trained peer mediators who have completed the Curtain exercise themselves in a prior cohort. This creates a propagation model: practice spreads by practice, not by documentation.

### 3.5 Why the Curtain prompt is framed the way it is

The Curtain prompt in Act IV is:

> Name one thing your partner has been needing you to be in this conversation that you are not. Name it directly, in the first person, without apology or self-flagellation.

This framing is deliberate and should not be softened in implementation. The ask is not *confess your failings*. That framing invites either defensive self-flagellation or defensive refusal, both of which are defense mechanisms in Maher's taxonomy. The prompt is instead *describe what your partner has been needing from you, that you are not providing*. That framing keeps the focus on the relationship rather than on moral accounting. It asks for accuracy, not apology. It is much closer to the move Dr. X actually made — he did not apologize or promise change, he simply named, aloud, that he had fallen short of what Maher needed from him.

### 3.6 Why slow, why silent, why reversible

Three principles the implementation must honor:

- **Slow is load-bearing.** Time gaps between messages are not UX friction. They are the mechanism. Do not add "send now" overrides, typing indicators, read receipts, or reactions. Each small acceleration defeats the purpose.
- **The system does not evaluate participants.** No scoring, no winners, no leaderboards, no "quality" metrics shown to users. The system is a container, not a judge.
- **Every action is reversible, declinable, or exitable.** Withdrawal is free and unstigmatized at every stage. Declining the Curtain is valid. Ending a dialogue mid-act is a legitimate outcome.

## 4. Principles for the implementing agent

Hold onto these throughout the build. They matter more than the specific mechanics in Part II.

- **Slow is load-bearing.** Do not add features that accelerate dialogue.
- **The AI is a paraphraser, not a judge.** If you find yourself designing something where Claude evaluates dialogue quality, stop.
- **Human facilitators are first-class.** Do not design around them.
- **Failure modes are more dangerous than usual.** A botched dialogue here can produce real shame, retraumatization, or harm to relationships. Every user-facing action should be reversible, declinable, or exitable without penalty.
- **Under-ship rather than over-ship.** A minimal working prototype with strong safety is far more valuable than a polished product with weak safety.

---

# PART II — Build Specification

## 5. System overview

A web application hosting dialogues between paired participants over a structured twelve-week program. Four acts (Sandcastle, Letters, Hot Stove, Curtain), each with specific prompts, mechanics, and safeguards. Dialogues are asynchronous only — no real-time chat. AI (Claude) provides paraphrasing assistance via the Translator feature. Human facilitators monitor pair dynamics and gate progression between acts.

### 5.1 Primary users

- **Participants** — volunteers from closed communities (universities, religious institutions, workplaces) who have completed intake and been paired.
- **Facilitators** — trained peer mediators who have completed a prior cohort as participants. They monitor assigned pairs, flag readiness for Act IV, and check in after sensitive exchanges.
- **Administrators** — run the program, match pairs, manage the question bank, ingest facilitator feedback.

### 5.2 Primary surfaces

- Participant web app (dialogue interface, act progression, private reflection space)
- Facilitator dashboard (pair monitoring, readiness flags, check-in tooling)
- Admin panel (cohort management, pair matching, question bank curation, metrics)

## 6. Tech stack recommendation

The stack below is a recommendation, not a mandate. Use your judgment if something fits better given constraints you discover. Default to boring, well-understood tools over novel ones.

- **Backend:** Python 3.11+ with FastAPI. Pydantic for schemas. SQLAlchemy for ORM.
- **Database:** PostgreSQL 15+. All data at rest encrypted at the column level for message content.
- **Frontend:** Next.js 14+ with React, TypeScript, Tailwind. Server components where possible.
- **Auth:** Magic link email auth only (no passwords, no social login). Closed cohort means identity is verified out-of-band at intake.
- **AI integration:** Anthropic Claude API, `claude-opus-4-7` for Translator and readiness-hint features. Never route user content to logging/telemetry; see §12.
- **Queue/jobs:** A simple job queue for scheduled message unlocks, reminder emails, and 24-hour gaps. Postgres-backed is fine.
- **Deployment:** A single region, single environment for the prototype. Staging/prod split can wait.

## 7. Data model

Below are the core tables. Types are PostgreSQL. All tables include `created_at`, `updated_at`, and `id UUID PRIMARY KEY` unless noted.

### 7.1 `cohorts`

A time-bounded run of the program.

```
name                TEXT NOT NULL
starts_at           TIMESTAMPTZ NOT NULL
ends_at             TIMESTAMPTZ NOT NULL
source_community    TEXT NOT NULL   -- e.g., "Stanford-GSB-2026"
status              TEXT NOT NULL   -- 'planning' | 'active' | 'completed' | 'aborted'
```

### 7.2 `participants`

```
cohort_id               UUID REFERENCES cohorts(id)
email                   TEXT NOT NULL
display_name            TEXT NOT NULL
intake_completed_at     TIMESTAMPTZ
intake_responses        JSONB   -- see §8 for schema
thinking_style_profile  JSONB   -- derived from intake
topic_willingness       TEXT[]  -- topics they agreed to discuss
status                  TEXT    -- 'invited' | 'active' | 'withdrawn' | 'completed'
UNIQUE(cohort_id, email)
```

### 7.3 `pairs`

```
cohort_id               UUID REFERENCES cohorts(id)
participant_a_id        UUID REFERENCES participants(id)
participant_b_id        UUID REFERENCES participants(id)
topic                   TEXT NOT NULL
facilitator_id          UUID REFERENCES facilitators(id)
current_act             INTEGER NOT NULL DEFAULT 1   -- 1..4
act_started_at          TIMESTAMPTZ
status                  TEXT    -- 'active' | 'paused' | 'ended_complete' | 'ended_early'
ended_reason            TEXT    -- nullable; free-text when status='ended_early'
CHECK (participant_a_id < participant_b_id)   -- canonical ordering
```

### 7.4 `messages`

Every message in the dialogue. Immutable once sent.

```
pair_id             UUID REFERENCES pairs(id)
author_id           UUID REFERENCES participants(id)
act                 INTEGER NOT NULL
prompt_id           UUID REFERENCES prompts(id)   -- what prompt this responds to
body                TEXT NOT NULL                 -- encrypted at rest
sent_at             TIMESTAMPTZ NOT NULL
unlocked_at         TIMESTAMPTZ                   -- when partner can read it; see §9.2
```

### 7.5 `translations`

A Translator invocation. Private to the requester.

```
message_id          UUID REFERENCES messages(id)
requested_by        UUID REFERENCES participants(id)
paraphrase          TEXT NOT NULL     -- Claude's output, encrypted at rest
underlying_concern  TEXT NOT NULL
suggested_question  TEXT NOT NULL
model_version       TEXT NOT NULL     -- e.g., 'claude-opus-4-7'
prompt_version      TEXT NOT NULL
```

### 7.6 `temperature_events`

Records of the Temperature Gauge being tapped.

```
pair_id         UUID REFERENCES pairs(id)
tapped_by       UUID REFERENCES participants(id)
tapped_at       TIMESTAMPTZ NOT NULL
resumes_at      TIMESTAMPTZ NOT NULL
reflection      TEXT          -- private to tapper, nullable
```

### 7.7 `curtain_responses`

Private drafts and shared versions of Act IV responses.

```
pair_id             UUID REFERENCES pairs(id)
author_id           UUID REFERENCES participants(id)
draft_body          TEXT          -- current private draft, encrypted
draft_updated_at    TIMESTAMPTZ
shared_body         TEXT          -- published version; NULL until shared
shared_at           TIMESTAMPTZ
partner_unlocked_at TIMESTAMPTZ   -- shared_at + 24 hours
UNIQUE(pair_id, author_id)
```

### 7.8 `facilitators`, `facilitator_notes`, `readiness_flags`

```
facilitators
  email, display_name, completed_cohort_id, active BOOLEAN

facilitator_notes
  facilitator_id, pair_id, body, created_at   -- private to facilitator+admin

readiness_flags
  pair_id, facilitator_id, flag_type, reason, created_at
  -- flag_type: 'ready_for_act_4' | 'not_ready' | 'concern' | 'should_end'
```

### 7.9 `prompts`

Curated content. Seed data, not user-generated.

```
act             INTEGER
sequence        INTEGER    -- order within act
topic           TEXT       -- nullable; Act III prompts are topic-specific
body            TEXT NOT NULL
rationale       TEXT       -- facilitator-visible explanation of why this prompt
```

### 7.10 `checkins`

Post-Curtain and ad-hoc wellbeing checks.

```
pair_id         UUID REFERENCES pairs(id)
participant_id  UUID REFERENCES participants(id)
trigger         TEXT    -- 'post_curtain' | 'post_pause' | 'facilitator_initiated'
responses       JSONB
flagged         BOOLEAN  -- true if responses warrant facilitator attention
```

## 8. Intake and matching

### 8.1 Intake questionnaire

Intake is deliberately not a political survey. It measures thinking style and conflict tolerance. Store raw responses in `intake_responses`; derive `thinking_style_profile` as a JSONB object with these numeric fields (all 1–7 Likert unless noted):

- `narrative_vs_data` — "I understand ideas best through stories" vs "through data and structure"
- `structure_vs_improvisation` — preference for defined rules vs open-ended exploration
- `emotion_first_vs_content_first` — "I need to feel safe before I can engage with hard content" vs inverse
- `conflict_tolerance` — "I can stay in disagreement without needing resolution"
- `body_awareness` — "I notice physical sensations when I disagree with someone"
- `religious_orientation` — categorical: religious / spiritual / secular / declined
- `topic_willingness` — array; participants check topics they agree to discuss

Separately, for each topic the participant agrees to discuss, collect a position score (1–7) indicating how strongly they hold their current view. This drives the difference score in matching.

### 8.2 Pair matching algorithm

Matching is human-assisted, not automated. The system **proposes** candidate pairs to an admin, who approves or rejects each one.

Algorithm for proposals:

1. Filter to participants who share at least one topic in `topic_willingness`.
2. Within topic-compatible candidates, compute a **difference score** on the position axis (how strongly they differ on the chosen topic) and a **similarity score** on the thinking-style axes above.
3. Rank candidate pairings by `difference_score - similarity_score`. High difference on position, low difference on thinking style.
4. Present top 10 proposals to admin with all relevant profile data. Admin approves, rejects, or swaps.

**Do not auto-commit pair proposals.** Admin review is a hard gate.

## 9. The four acts — behavior spec

### 9.1 Act progression rules

- A pair enters Act 1 when the admin commits the pairing.
- Advancement between acts is **scheduled by calendar** for Acts 1→2 and 2→3, gated by completion percentage and facilitator approval.
- Advancement to Act 4 is **gated entirely by facilitator readiness flag**. No automatic advancement. If the facilitator has not flagged `ready_for_act_4` by the end of week 9, the pair ends in Act 3 — this is a successful outcome, not a failure.
- Either participant can withdraw at any time. Withdrawal ends the pair with `status='ended_early'`, triggers a facilitator notification, and initiates a check-in for the remaining participant.

### 9.2 Message posting and the slow cadence

Asynchronous only. When a participant posts a message:

1. The message is saved with `sent_at = NOW()` and `unlocked_at = sent_at + delay(act, sequence)`.
2. The partner cannot read the message until `unlocked_at`.
3. Default delays:
   - Act 1 (Sandcastle): 2 hours
   - Act 2 (Letters): 12 hours
   - Act 3 (Hot Stove): 6 hours
   - Act 4 (Curtain): 24 hours after sharing (see §9.6)
4. The UI shows the partner a countdown, not the message body, during the lock window.

The delays are intentional. Do not add a "send now" override. Do not add typing indicators. Do not add read receipts. Do not add reactions. Each of these small accelerations defeats the purpose.

### 9.3 Act 1 — Sandcastle

Two weeks. Six prompts, one every 2–3 days. All prompts are low-stakes collaborative creative tasks. Examples (use the seed prompts in `/seed/prompts_act1.json`):

- "Design a small fictional town together. One of you describes a feature; the other responds with what's next door."
- "Write alternating sentences of a short story about someone who loses something and finds something else instead."
- "Pick a room in a house neither of you has lived in. Describe it together, one detail at a time."

Translator is **disabled** in Act 1. Temperature Gauge is available but rarely used.

### 9.4 Act 2 — Letters

Three weeks. Four prompts. Each prompt is an autobiographical question. Long-form responses expected (500+ words). The partner reads and responds with their own autobiography, not with commentary.

Seed prompts include:
- "What were you taught about conflict as a child? Who taught you?"
- "Describe a time you felt misunderstood. Don't tell me why you were right."
- "What is a belief you hold now that would have surprised your 12-year-old self?"
- "What is something you believe that most people around you don't?"

Debate is disallowed in Act 2. The system enforces this soft-mechanically: before a message posts, Claude reviews it with the prompt at `/prompts/act2_review.md` (see §11.3) and, if it detects argumentative framing, returns a suggestion to the author: *This reads like a response to your partner rather than your own story. Would you like to revise?* The author can accept the suggestion or ignore it.

### 9.5 Act 3 — Hot Stove

Four weeks. Pair engages with their chosen topic via a sequence of facilitator-visible prompts. Translator is **enabled** starting here. Temperature Gauge is available throughout.

**Translator mechanic:**

Any message can be translated by either participant. Translation is private to the requester — the author is never told a translation was requested. When a translation is requested, the system calls Claude with the prompt at `/prompts/translator.md` (see §11.1). The response must parse into three fields: `paraphrase`, `underlying_concern`, `suggested_question`. Store in `translations`. Display to the requester only.

Claude must decline to translate messages containing direct personal attacks, slurs, or threats. Declined translations surface a pause prompt instead: *This message is activating. Would you like to pause the dialogue for 24 hours?* See §11.2 for the refusal criteria.

**Temperature Gauge mechanic:**

A button labeled "I'm getting activated" is persistently visible. Tapping:
1. Creates a `temperature_events` row with `resumes_at = NOW() + 24 hours`.
2. Pauses the dialogue — neither participant can post until `resumes_at`.
3. Shows the tapper a private reflection prompt: *What are you feeling in your body right now? What's the fear underneath the frustration?* Response is optional and stored in `temperature_events.reflection`.
4. Notifies the partner: *Your partner has requested a pause. The dialogue will resume in 24 hours.* No reason given.

Multiple tap-the-button events by the same participant in a week trigger a facilitator notification.

### 9.6 Act 4 — The Curtain

The sensitive act. Three weeks, but most of the work is private.

**Flow:**

1. On Act 4 start, each participant receives the Curtain prompt privately (see below). They have 72 hours of private drafting before they can share.
2. During drafting, the participant sees only their own draft. Partner activity in the same window is invisible.
3. After 72 hours of drafting have elapsed, a "Share with partner" button unlocks. Sharing is opt-in — either participant can decline to share and the pair advances to the post-Act-4 check-in.
4. When one participant shares, the partner sees the shared response but cannot respond for 24 hours.
5. After 24 hours, the partner can (a) complete and share their own Curtain response, (b) send a final message acknowledging what they read, or (c) end the dialogue.

**The Curtain prompt, verbatim:**

> Name one thing your partner has been needing you to be in this conversation that you are not. Name it directly, in the first person, without apology or self-flagellation.
>
> This is not a confession. You are not admitting wrongdoing. You are describing, accurately, a gap between what your partner has been hoping for from you and what you have been able to give. The goal is accuracy, not apology. You do not have to promise to change. You only have to let your partner see clearly what you are and what you are not.

**Guardrails:**

- Private drafting is genuinely private. Facilitators cannot see drafts. Admins cannot see drafts. The draft is only visible to its author until shared.
- Once shared, the content is visible to the partner and to the assigned facilitator (not other facilitators, not admins).
- A "retract" button is available for 1 hour after sharing. Retraction replaces the shared content with a placeholder: *[Content retracted by author.]*
- Immediately after the 24-hour unlock, both participants are offered a post-Curtain check-in (see §10.3).

### 9.7 Act 4 — Claude-assisted readiness hint (optional, facilitator-facing)

To help facilitators assess readiness, the system can offer a private readiness hint generated by Claude from the full Act 3 transcript. The facilitator sees the hint; participants do not. This is an assistive tool, not a decision-maker. See prompt at `/prompts/readiness_hint.md` (§11.4).

The hint returns three fields: `signals_of_readiness`, `signals_of_risk`, `recommendation` (one of `'ready'`, `'not_yet'`, `'should_end'`). The facilitator can disregard it entirely.

## 10. Facilitator layer

### 10.1 Facilitator dashboard

A facilitator sees a list of their assigned pairs with status indicators:

- Current act and days in act
- Last message timestamp per participant
- Temperature Gauge taps in the past 7 days
- Translator usage count
- Readiness flags already set

Clicking a pair opens the full transcript for that pair. Facilitators can:

- Read all messages (but not private drafts in Act 4)
- Read translations (even though translations are usually private to the requester, facilitators see them to assess dialogue health)
- Add private notes (`facilitator_notes` table)
- Set readiness flags (`readiness_flags`)
- Initiate a check-in with either participant

### 10.2 Facilitator boundaries

Facilitators are **not** therapists and the UI must make this explicit. Their role is assessment and lightweight support — they can flag concern, recommend ending a dialogue, and initiate check-ins, but they do not counsel participants through personal crises. If a participant discloses acute distress, the facilitator's only job is to surface the appropriate external resource (see §12.3) and notify an admin.

### 10.3 Post-Curtain check-in

Triggered automatically 72 hours after a Curtain exchange. Delivered separately to each participant. Structured questions (Likert 1–7):

- "How accurately do you feel your partner described themselves in their Curtain response?"
- "How accurately do you feel you described yourself?"
- "How are you feeling right now about the dialogue overall?"
- "Is there anything you want your facilitator to know?" (free-text)

If any response scores ≤3 on the wellbeing question, or if the free-text indicates distress, the check-in is flagged and routed to the facilitator for follow-up within 24 hours.

## 11. Claude prompts

All prompts live in `/prompts/` as Markdown files. Load them at runtime — never hardcode. Each prompt file has a YAML frontmatter block specifying model, max_tokens, and temperature.

### 11.1 `/prompts/translator.md`

Produces the Translator output. Input: the message to translate, plus the last 10 messages of context. Output: strict JSON with `paraphrase`, `underlying_concern`, `suggested_question`.

Key constraints in the prompt:

- Do not judge who is right.
- Do not summarize positions as equivalent. If one side is making a factual claim and the other a values claim, preserve that distinction.
- Strip tribal markers and charged terms. Render the same content in neutral vocabulary.
- The `underlying_concern` is a best guess at the fear, value, or experience the position is protecting. Frame it in the first person from the author's perspective.
- The `suggested_question` should open the topic further, not close it. Avoid leading questions.
- If the message contains slurs, threats, or direct attacks, refuse by returning `{"refusal": true, "reason": "..."}`.

Stub for the prompt file is in `/prompts/stubs/translator.md`.

### 11.2 Refusal criteria

The Translator refuses to translate when the input message contains:

- Slurs targeting a protected class
- Direct threats of violence against the partner or others
- Doxxing (addresses, workplaces, real names of third parties)
- Content that seems designed to provoke rather than communicate (judgment call; err on the side of translating unless it's obvious)

Refusal surfaces a pause prompt to the requester, not to the author. The author is never told a refusal occurred.

### 11.3 `/prompts/act2_review.md`

Reviews Act 2 messages pre-post for argumentative framing. Returns `{"is_argumentative": bool, "gentle_suggestion": string}`. The suggestion is shown to the author as advisory only.

### 11.4 `/prompts/readiness_hint.md`

Input: full Act 3 transcript for a pair. Output: `{"signals_of_readiness": string[], "signals_of_risk": string[], "recommendation": "ready" | "not_yet" | "should_end"}`. Facilitator-facing only.

Signals of readiness include: both participants have shown willingness to be wrong, messages have grown longer and more exploratory, Temperature Gauge taps have decreased over time, Translator usage has shifted from self-protection to curiosity.

Signals of risk include: escalating contempt, one-sided vulnerability, repeated misunderstandings of the same kind, one participant dominating the word count by >70%, disclosures of acute personal distress.

### 11.5 `/prompts/moderation.md`

Reviews every message pre-post for: direct personal attacks, slurs, doxxing, threats, or self-harm content. Returns `{"flags": string[], "severity": "low" | "medium" | "high", "suggestion": string}`. Severity high with self-harm flags routes to crisis resources immediately.

### 11.6 Prompt discipline

- Load prompts from disk at request time, not at import time. This lets facilitators and admins iterate on prompts without redeploying.
- Version prompts with a `version` field in frontmatter. Store the version used with each Claude call in the relevant row.
- Never interpolate user-controlled strings directly into system prompts. Pass them as separate user-role messages.

## 12. Safety, privacy, security

### 12.1 Content moderation

Claude reviews every message pre-post using the prompt from §11.5. Flagged messages are not blocked outright — instead the author sees: *This message may land harder than you intend. Would you like to revise, send anyway, or pause?* Author choice is respected; the flag is logged for facilitator review.

For self-harm content, the flow differs: the message is held, the author is shown crisis resources (see §12.3), and the facilitator is notified immediately. The author can still choose to send, but with explicit acknowledgment.

### 12.2 Withdrawal and data

Participants can withdraw at any time. Withdrawal:

- Ends the pair immediately with `status='ended_early'`.
- Preserves the transcript (the other participant retains access to what they wrote and what was shared with them).
- Offers the withdrawing participant the option to delete their own outgoing messages from the partner's view. This is honored even though it's partial — the partner sees a "[Message withdrawn by author]" placeholder.

### 12.3 Crisis resources

A static page accessible from every screen, listing:

- US: 988 Suicide & Crisis Lifeline
- Crisis Text Line: text HOME to 741741
- International equivalents where known
- A note that facilitators are not therapists and cannot provide clinical support

Surface crisis resources automatically any time the content moderator flags self-harm language.

### 12.4 Privacy and security

- All message bodies, draft bodies, translations, and reflections are encrypted at the column level using a KMS-managed key. Decryption happens in-process only for authorized reads.
- Email is the only PII in `participants`. Do not collect phone numbers, addresses, or government IDs.
- Claude API calls: set the appropriate header to disable logging of content on the Anthropic side where available. Do not log request/response bodies to your own logs either. Log only: endpoint, model, prompt_version, token counts, latency, and request ID.
- Access control:
  - Participants can read only their own messages and messages shared with them.
  - Facilitators can read transcripts for their assigned pairs only, with Act 4 private drafts excluded.
  - Admins can read metadata (pair status, timestamps, counts) but should not read message content without a specific justification. Route such access through an audit-logged admin action, not a general database query.
- Session cookies are HttpOnly, Secure, SameSite=Strict. Session lifetime capped at 24 hours; re-auth via magic link after that.

### 12.5 No public feed, ever

There is no public feed. There is no "featured dialogue." There is no sharing of other pairs' content with a current pair. Transcripts may be anonymized and used for facilitator training and prompt refinement — but only with explicit, post-completion consent from both participants, and never displayed in-product.

## 13. Metrics

### 13.1 Primary outcome

**Trust shift.** Delta on "How much do you trust that people who disagree with you are acting in good faith?" (1–7), measured at intake and at cohort completion.

### 13.2 Secondary outcomes

- Act completion rate by pair.
- Curtain completion: none / one-sided / two-sided.
- Self-reported understanding shift: "How well do you understand the best version of the opposing view?" (1–7), intake vs. completion.
- Facilitator readiness-flag accuracy: when a facilitator flagged `ready_for_act_4`, did the resulting Curtain exchange produce a positive trust shift?

### 13.3 Operational metrics

- Translator usage per pair, per act. Patterns over time.
- Temperature Gauge taps per pair.
- Withdrawal rate and timing.
- Time-to-response per act.

### 13.4 Do not track

- "Minds changed" or any proxy for position shift on the contested topic.
- Anything that could be used to evaluate "winners" of dialogues.
- Individual participant Translator usage displayed to their partner.

## 14. Seed data

Before first cohort, the following must exist:

- `/seed/prompts_act1.json` — 10+ Sandcastle prompts
- `/seed/prompts_act2.json` — 6+ Letters prompts
- `/seed/prompts_act3/*.json` — Topic-specific Hot Stove prompt sequences for: gun policy, abortion access, immigration, religious belief vs. secular ethics, Israel/Palestine. Each sequence has 8–12 prompts ordered from least to most charged.
- Curated facilitator training curriculum — separate from this spec. Facilitators must complete training before they are assigned pairs.
- A test cohort of 4–6 pairs with known-friendly participants (staff, advisors) before any real volunteer cohort runs.

Seed prompts must be reviewed by at least one trained clinician with experience in conflict dialogue before going live. Do not ship prompts that haven't been reviewed.

## 15. Build order

Build in this order. Ship each phase to internal testers before moving on.

**Phase 1 — Foundation (weeks 1–3)**
- Data model, auth, admin panel.
- Intake questionnaire and pair matching proposal algorithm.
- Seed prompt loading.

**Phase 2 — Acts 1 and 2 (weeks 4–6)**
- Message posting with slow cadence.
- Act 1 Sandcastle prompts and advancement logic.
- Act 2 Letters with Claude-assisted argumentative-framing review.

**Phase 3 — Act 3 and Translator (weeks 7–9)**
- Translator integration.
- Temperature Gauge.
- Content moderator.
- Facilitator dashboard (read-only transcripts, notes, flags).

**Phase 4 — Act 4 and check-ins (weeks 10–12)**
- Private drafting interface.
- Sharing flow with 24-hour unlock.
- Retraction window.
- Post-Curtain check-ins and facilitator routing.
- Readiness hint for facilitators.

**Phase 5 — Test cohort (weeks 13–14)**
- Run 4–6 internal pairs through the full program, compressed to ~6 weeks for testing.
- Fix anything that breaks.
- Only then open to a real volunteer cohort.

## 16. Things to check with the product owner before onboarding real participants

Do not assume defaults on any of these. Ask before the first real cohort.

1. What community is the first cohort recruited from? This affects intake wording, topic selection, and crisis resource regionalization.
2. Who are the initial facilitators and who is training them? Facilitator training is not in this spec.
3. Has a clinician reviewed the seed prompts? If not, the prototype is not ready for real participants regardless of engineering completeness.
4. What is the legal/ethical review process for research use of transcripts (metrics, prompt refinement)? Likely needs IRB-equivalent review if affiliated with a university.
5. How are facilitators compensated? This affects the sustainability of the model before any scaling conversation.

If any of these are unanswered, build the technical prototype but do not onboard real participants. A prototype with no participants is recoverable; a prototype that harms someone is not.

## 17. What this prototype is not

It is not a therapy tool. It is not a debate platform. It is not a mind-changing tool. It is not a scalable public product at this stage. It is a narrow, human-supervised experiment in whether structured asynchronous dialogue, Claude-assisted paraphrasing, and a single moment of facilitated honest exposure can produce durable shifts in good-faith attribution between ideological opponents who entered in genuine disagreement.

Build it accordingly. The goal is clarity of signal from a small, careful cohort — not polish for a product launch that isn't happening.

---

*End of specification.*
