# The Elephant — Build Plan

Companion to `The_Elephant_Spec_Complete.md`. Expands §15 (Build order) into
concrete tasks. Phases gate on each other; ship each to internal testers before
the next.

## Cross-cutting principles (apply to every phase)

- **Slow is load-bearing.** No "send now", typing indicators, read receipts,
  reactions, or push-to-accelerate UX.
- **No evaluation of participants.** No scores, winners, quality metrics shown
  to users. Claude paraphrases; it never judges.
- **Reversible by default.** Every destructive or public action has an undo,
  decline, or exit path.
- **Under-ship.** Prefer a narrow, safe feature over a broad, unsafe one.

## Phase 0 — Project scaffolding (pre-week 1)

Goal: land the repo skeleton so every later phase drops into place.

1. Monorepo layout:
   - `backend/` — FastAPI app, Alembic migrations, SQLAlchemy models.
   - `frontend/` — Next.js 14 App Router, TypeScript, Tailwind.
   - `prompts/` — Markdown prompt files with YAML frontmatter (spec §11).
   - `seed/` — JSON seed prompts per act (spec §14).
   - `infra/` — docker-compose for local Postgres + app.
2. Tooling: ruff + pytest (backend), eslint + vitest + playwright (frontend),
   pre-commit hooks, GitHub Actions CI running lint + tests.
3. Secrets: `.env.example` enumerating `DATABASE_URL`, `ANTHROPIC_API_KEY`,
   `KMS_KEY_ID`, `MAGIC_LINK_SIGNING_KEY`, SMTP creds. Never commit real keys.
4. Observability baseline: structured JSON logs with request IDs; explicit
   allowlist of fields (spec §12.4 — never log message bodies).

Exit criteria: `make dev` brings up Postgres + backend + frontend locally.

## Phase 1 — Foundation (weeks 1–3)

Goal: cohorts exist, participants can be invited, intake runs, admins can
propose and commit pairs.

### 1.1 Data model + migrations
Implement every table in spec §7: `cohorts`, `participants`, `pairs`,
`messages`, `translations`, `temperature_events`, `curtain_responses`,
`facilitators`, `facilitator_notes`, `readiness_flags`, `prompts`, `checkins`.

- Enforce `pairs.CHECK (participant_a_id < participant_b_id)` canonical ordering.
- Add `UNIQUE (cohort_id, email)` on participants, `UNIQUE (pair_id, author_id)`
  on curtain_responses.
- Column-level encryption helpers (pgcrypto or app-side AES-GCM with KMS DEK)
  for: `messages.body`, `translations.paraphrase`, `translations.underlying_concern`,
  `translations.suggested_question`, `curtain_responses.draft_body`,
  `curtain_responses.shared_body`, `temperature_events.reflection`.
- Alembic migrations; seed migration for `prompts` from `/seed/*.json`.

### 1.2 Auth
Magic-link email only (spec §6). HttpOnly + Secure + SameSite=Strict cookies,
24h session cap, re-auth via fresh link. Three role scopes: participant,
facilitator, admin. Middleware enforces role on every route.

### 1.3 Intake
Render questionnaire from spec §8.1. Store raw in `intake_responses`, derive
`thinking_style_profile` JSONB on submit. Collect per-topic position score for
every `topic_willingness` entry.

### 1.4 Pair matching
Proposal algorithm (spec §8.2): topic filter → difference score on position →
similarity score on thinking-style → rank by `difference - similarity` → top 10.
Admin UI shows proposals with full profile context. Commit requires explicit
admin click — **no auto-commit** under any condition.

### 1.5 Admin panel
- Cohort CRUD, participant invites, facilitator assignment.
- Pair proposal review + commit.
- Prompt bank browser (read-only in Phase 1).

### 1.6 Prompt loader
Utility that loads markdown + YAML frontmatter from `/prompts/` at request
time (spec §11.6). Cache per-request, not per-process — facilitators iterate
without redeploy. Stubs for every prompt file so Phase 2+ can slot real content.

Exit criteria: admin can run a full flow from cohort creation through a
committed pair; pair sits in Act 1 day 0 waiting for Phase 2.

## Phase 2 — Acts 1 and 2 (weeks 4–6)

Goal: paired participants exchange messages with enforced slow cadence through
Sandcastle and Letters.

### 2.1 Message posting + cadence
- POST `/messages` writes with `sent_at = NOW()`, `unlocked_at = sent_at + delay(act, sequence)`.
  Delays: Act 1 = 2h, Act 2 = 12h, Act 3 = 6h, Act 4 = 24h (spec §9.2).
- Read endpoint hides body until `unlocked_at ≤ NOW()`; returns countdown only.
- Idempotency key on post to prevent double-sends.
- **No** typing indicator, read receipt, reaction, or send-now override.

### 2.2 Act progression engine
Background job (Postgres-backed queue — `pg-boss` or similar) that:
- Unlocks messages at `unlocked_at`.
- Sends reminder emails at configurable thresholds.
- Advances acts 1→2 and 2→3 on calendar + completion check + facilitator green-light
  (spec §9.1). Act 3→4 advance is **only** on facilitator `ready_for_act_4` flag —
  no calendar fallback.
- Handles withdrawal: immediate pair end, facilitator notify, check-in trigger.

### 2.3 Act 1 (Sandcastle)
- Six prompts, delivered every 2–3 days from `/seed/prompts_act1.json`.
- Translator disabled, Temperature Gauge available (stub handler OK).
- UI: current prompt, partner's previous message (if unlocked), composer.

### 2.4 Act 2 (Letters)
- Four prompts, long-form (500+ words).
- Pre-post review via `/prompts/act2_review.md` (spec §11.3). Returns
  `{is_argumentative, gentle_suggestion}`. Suggestion is advisory only — author
  can ignore and post anyway.
- Store `prompt_version` + `model_version` from the review call for audit.

### 2.5 Participant UI shell
- Dialogue view, act progress, private reflection area, withdraw button,
  crisis resources link (static, always visible — spec §12.3).
- Explicit copy that facilitators are not therapists (spec §10.2).

Exit criteria: internal test pair completes Act 1 + Act 2 end-to-end with real
cadence, no way to bypass delays.

## Phase 3 — Act 3, Translator, facilitator dashboard (weeks 7–9)

Goal: Hot Stove runs with full AI assist and facilitator oversight.

### 3.1 Translator (`/prompts/translator.md`, spec §11.1)
- Input: target message + last 10 messages of context.
- Strict JSON output: `paraphrase`, `underlying_concern`, `suggested_question`,
  or `{refusal: true, reason}`.
- Private to requester. Author is never told a translation occurred (spec §9.5).
- Claude call discipline (spec §11.6): prompts from disk, user-content as
  user-role messages only, never interpolated into system prompt.
- Persist `model_version`, `prompt_version` on every row.
- Refusal criteria per spec §11.2 → surface pause prompt to requester only.

### 3.2 Temperature Gauge
- Persistent "I'm getting activated" button in Act 3 UI.
- On tap: insert `temperature_events` row with `resumes_at = NOW() + 24h`,
  pause dialogue (both sides blocked from posting), show tapper private
  reflection prompt, notify partner with neutral "paused 24h" copy — no reason.
- Aggregate: ≥2 taps by same participant in 7 days → facilitator notification.

### 3.3 Content moderation (`/prompts/moderation.md`, spec §11.5)
Pre-post check on every message. Flow:
- Flagged (non-self-harm): author sees revise / send-anyway / pause dialog.
  Send-anyway is logged and facilitator-visible.
- Self-harm flag: hold message, show crisis resources, notify facilitator
  immediately. Author can still opt to send with explicit acknowledgment
  (spec §12.1).

### 3.4 Facilitator dashboard
- List of assigned pairs with: current act, days in act, last-message timestamps
  per participant, 7-day Temperature taps, Translator count, current readiness
  flags.
- Transcript view — all messages + translations (yes, translations visible to
  facilitator per spec §10.1), but **not** Act 4 private drafts.
- Private notes (`facilitator_notes`), readiness flag setting, check-in initiation.
- Role enforcement: facilitator sees only their assigned pairs.

### 3.5 Act 3 prompts
Load from `/seed/prompts_act3/<topic>.json`. 8–12 prompts per topic, ordered
least-to-most charged (spec §14). Topics: gun policy, abortion, immigration,
religious vs. secular ethics, Israel/Palestine. Do not ship any prompt that
has not been clinician-reviewed.

Exit criteria: internal pair runs a full Act 3 with Translator + Temperature
Gauge + moderator all active; facilitator can monitor and flag readiness.

## Phase 4 — Act 4 + check-ins (weeks 10–12)

Goal: the sensitive act works safely.

### 4.1 Private drafting
- 72h private drafting window on Act 4 start. Drafts stored in
  `curtain_responses.draft_body` — encrypted, visible only to author.
  **Facilitators and admins cannot read drafts** (enforce at query layer,
  not just UI).
- Auto-save, but no partner-visible activity indicator during drafting.

### 4.2 Sharing flow (spec §9.6)
- After 72h elapsed: "Share with partner" button unlocks. Declining is
  valid — pair advances to post-Act-4 check-in regardless.
- On share: set `shared_at`, `partner_unlocked_at = shared_at + 24h`. Partner
  cannot see shared content until unlock.
- Post-unlock, partner's options: (a) complete and share own Curtain,
  (b) final acknowledgment message, (c) end dialogue.
- **Retract window**: 1h after share; retraction replaces shared content with
  `[Content retracted by author.]` placeholder.

### 4.3 Curtain prompt copy
Render the full Curtain prompt (spec §9.6) **verbatim**. No softening, no
tooltips that reframe it. Copy lives in `/prompts/curtain.md` for auditability,
but must match spec text exactly.

### 4.4 Readiness hint (`/prompts/readiness_hint.md`, spec §11.4)
- Facilitator-triggered, facilitator-only.
- Input: full Act 3 transcript for pair.
- Output: `{signals_of_readiness[], signals_of_risk[], recommendation}`.
- Never shown to participants. Never stored in a participant-visible table.

### 4.5 Post-Curtain check-ins (spec §10.3)
- Auto-trigger 72h after Curtain unlock. Delivered separately per participant.
- Four Likert+free-text questions per spec.
- Flag routing: wellbeing score ≤3 or distressed free-text → facilitator
  follow-up within 24h, surfaced on their dashboard.

### 4.6 Withdrawal polish (spec §12.2)
- Single withdraw button on every participant screen.
- On withdraw: pair ends, transcript preserved for both sides, offer to replace
  own sent messages with `[Message withdrawn by author]` placeholder, trigger
  check-in for remaining participant.

Exit criteria: internal pair completes full Act 4 including retraction path,
decline-to-share path, and post-Curtain check-ins.

## Phase 5 — Internal test cohort (weeks 13–14)

Goal: 4–6 known-friendly pairs (staff, advisors) run the full program,
compressed to ~6 weeks (spec §14, §15 Phase 5).

1. Compress delays ~2× for test only, via config — not code change. Revert
   before any real cohort.
2. Run full intake → pairing → four acts → check-ins.
3. Daily triage: bugs, copy issues, facilitator UX gaps. Fix before real
   cohort.
4. Verify the §16 gating questions are all answered before proceeding:
   community, facilitator training, clinician review of seed prompts,
   IRB/legal status, facilitator compensation.

Exit criteria: every §16 question has a documented answer; all P0/P1 bugs
from test cohort closed.

## Safety/privacy checklist (verify before real cohort)

- [ ] Column-level encryption live on every field listed in §12.4.
- [ ] Anthropic API calls set the no-logging header where available.
- [ ] App logs contain zero message content — only metadata (spec §12.4).
- [ ] Access control tested: participant-cross-reads, facilitator-cross-reads,
      admin reading message content all blocked or audit-logged.
- [ ] Crisis resources page reachable from every screen (spec §12.3).
- [ ] No public feed, no "featured dialogue", no cross-pair content leakage
      (spec §12.5).
- [ ] Magic-link tokens single-use, short-lived, bound to email.
- [ ] Session cookie flags: HttpOnly, Secure, SameSite=Strict, 24h cap.

## Metrics to implement (spec §13)

Implement these; do not implement "minds changed" or any winner proxy.

- Trust-shift delta (intake vs. cohort completion).
- Act completion rate per pair; Curtain completion (none / one-sided / two-sided).
- Understanding-shift Likert delta.
- Facilitator flag accuracy vs. post-Curtain trust shift.
- Operational: Translator usage by pair/act, Temperature taps, withdrawal
  rate + timing, time-to-response per act.

## Open questions to resolve with product owner

Blocking for real cohort (spec §16):

1. Source community for first cohort.
2. Initial facilitator roster + trainer.
3. Clinician review of seed prompts complete?
4. IRB / legal review path for transcript-derived research.
5. Facilitator compensation model.

Do not onboard real participants until all five are answered.
