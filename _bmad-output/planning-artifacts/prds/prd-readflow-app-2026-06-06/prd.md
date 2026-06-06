---
title: ReadFlow — Product Requirements Document
status: final
created: 2026-06-06
updated: 2026-06-06
---

# ReadFlow — Product Requirements Document

## Executive Summary

ReadFlow is an AI-powered PDF reader that closes the broken read → note → forget loop for Vietnamese tech learners processing English-language academic and technical papers. It delivers bilingual in-context explanations and structured smart notes — without the tab-switching, tool-switching, and knowledge decay that currently plague this workflow. Unlike SciSpace or RemNote, ReadFlow is built specifically for InfoSec and DevOps learners, speaks Vietnamese, and runs on infrastructure the operator controls.

---

## Problem

Vietnamese master's students in Information Security and DevOps read a high volume of English technical papers with no effective workflow:

1. **Slow reading** — unfamiliar English jargon forces constant tab-switching to search terms
2. **No structured notes** — only ad-hoc highlights survive the reading session
3. **Knowledge decay** — without a review system, retention drops to near zero; re-reading from scratch is the only option

Existing tools address parts of this problem but none address this user:

- **SciSpace / Scholarcy** — inline AI explain and structured summaries, but US-hosted, no Vietnamese language support
- **RemNote** — PDF + spaced repetition, but general-purpose, no bilingual support, no domain-specific prompts
- **Mochi / Space / Retain** — strong spaced repetition, no integrated PDF reader

No existing tool combines domain-specific AI understanding, bilingual output (Vietnamese + English), and data hosting the user can trust.

---

## Target Users

**Primary segment:** Vietnamese tech learners — specifically master's students in Information Security and DevOps — who regularly read English-language academic and research papers as part of their coursework or self-study.

**Protagonist — Minh, InfoSec Master's Student:**
Minh reads 3–8 technical English papers per week. When he encounters an unfamiliar term — say, "certificate transparency log" — he opens a new tab, searches, reads a Wikipedia entry, loses his reading context, and often forgets the explanation by the time he returns to the paper. He doesn't take structured notes (just highlights). Two weeks later, when he needs the concept again, he re-reads from scratch.

Minh's jobs to be done:

| Job | Current failure |
|---|---|
| **Read** — understand a paper without losing flow | Tab-switching breaks concentration; jargon halts progress |
| **Retain** — not forget after finishing | No review system; knowledge evaporates |
| **Organize** — find knowledge when needed | Only highlights; no searchable structure |

---

## User Journey — UJ-1: Minh's First Reading Session

*Minh, InfoSec master's student, opens ReadFlow for the first time with a paper he's been putting off.*

1. **Sign up** — Minh lands on ReadFlow, registers with email. (FR-001)
2. **Upload** — Empty library prompts him to add a paper. He uploads a 3 MB PDF — a TLS 1.3 paper from USENIX Security. He reads the one-line Bedrock disclosure and uploads. (FR-010, FR-037)
3. **Read** — First page renders in under 3 seconds. He navigates through the paper. (FR-011, FR-012)
4. **Explain** — On page 4 he hits "certificate transparency log" — a term he's vague on. He selects it; the popup appears with a Vietnamese explanation ("nhật ký minh bạch chứng chỉ...") and the English technical context. He reads it, understands it, and keeps reading without opening a new tab. (FR-020–FR-023)
5. **Save an explanation** — On page 7 he encounters "OCSP stapling" — a term he knows he'll see again. He clicks "Save explanation." The text gains a subtle underline. (FR-025a)
6. **Reach last page** — A banner appears: *"You've finished this paper. Generate smart note?"* He clicks Confirm. (FR-030)
7. **Smart note generates** — 20 seconds later the note appears: What / Why / How and 8 Key Terms with bilingual definitions. He reads it. He types two sentences of his own analysis in the "My Notes" block — a comparison to TLS 1.2 behavior he just remembered. (FR-033, FR-035a)
8. **Library updated** — He returns to the library. The paper shows "Finished" with a smart note indicator. (FR-041)

*Outcome: Minh read a difficult paper, understood every blocked term without leaving the app, and has a structured note he can find in two weeks — without re-reading the paper.*

---

## Product Vision & Differentiators

ReadFlow's value proposition rests on three pillars that incumbents cannot easily replicate for this market:

1. **Bilingual, domain-specific AI** — explanations are in Vietnamese with English technical context, tuned for security and DevOps vocabulary. Not a dictionary lookup; a contextual explanation a Vietnamese reader can use without leaving the app.
2. **Controlled data hosting** — all reading data, notes, and uploads live in the operator's AWS account. No foreign SaaS intermediary touches user data.
3. **Tight scope** — ReadFlow is not a general research tool. It is purpose-built for one workflow: read a technical paper, understand it, capture it, keep it. Features that serve different workflows or different users are out of scope — the roadmap lists what *could* exist, not what ReadFlow is trying to become. The post-v1 roadmap is driven by validated user demand, not by competitive parity.

---

## Goals & Success Metrics

### v1 Goal
Validate that the core read → note loop reduces time-to-comprehension and creates durable knowledge capture for Vietnamese readers of English technical papers.

**v1 primary user: the builder.** The initial usage is dogfooding — the solo developer using ReadFlow to read their own papers. Success metrics are validated against this real usage before the product opens to additional users. This is intentional: if ReadFlow doesn't solve the builder's own reading problem, it won't solve it for anyone else.

### Success Metrics

| Metric | Target | Rationale |
|---|---|---|
| Core loop completion rate | ≥ 70% | Users who upload a paper also generate a smart note — the loop is actually used |
| AI explain retention (popup read rate) | ≥ 80% | User reads the explanation rather than immediately dismissing it |
| 7-day return rate | ≥ 40% | Users come back — product has enough value to re-open |
| Smart note access rate | ≥ 50% | Generated notes are referenced, not ignored |
| Reading flow proxy: explain-to-finish rate | ≥ 60% | % of sessions with ≥1 explain call that reach the last page (smart note offer triggered) — proxy for bilingual explain keeping users in reading flow rather than abandoning |

### Counter-Metrics (Health Guardrails)

| Metric | Threshold | Rationale |
|---|---|---|
| Bedrock cost per active user/month | < $0.50 | Financial sustainability |
| AI explain calls per paper session | < 20 | Proxy for comprehension: user is learning, not offloading all reading to AI |

---

## Scope

### v1 — Core Read → Note Loop

The minimum scope to validate the core hypothesis: can ReadFlow meaningfully improve how a Vietnamese learner reads and captures knowledge from a technical English paper?

| Capability | Included |
|---|---|
| Authentication (email + Google OAuth) | Yes |
| PDF upload & in-browser render | Yes |
| Highlight + AI bilingual explain popup | Yes |
| Auto smart note generation (What / Why / How / Key Terms) | Yes |
| Paper library | Yes |
| AI quota enforcement (free tier) | Yes |

### What v1 Is NOT
v1 does not include flashcard generation, spaced repetition, reading progress tracking, manual notes, persistent highlights, search, or export. These are v1.5 or later — deliberately excluded because the reading hypothesis must be validated before the retention hypothesis. Building both simultaneously risks building neither well.

### v1.5 — Retain Loop
*(Unlocked after core loop validated with real users)*

- Auto flashcards from Key Terms (SM-2 spaced repetition)
- Reading progress tracking (last page, % completion)
- Manual sidebar notes while reading

### Should-Have (Post-v1.5, Demand Validation Required)

- Persistent color-coded highlights
- Inline translation
- Ask AI (chat interface over paper content)
- Daily flashcard review queue
- Export notes to Markdown / Notion
- Tags & folders by course or project
- Full-text search across notes

### v2+ (Future)

- URL / ArXiv import
- Feynman test (AI quiz)
- FSRS algorithm (upgrade from SM-2)
- Retention analytics
- Knowledge graph
- Related paper suggestions
- Share notes via link

### Won't Have (Explicit Out-of-Scope)

- Team / group workspace
- Real-time collaboration
- User self-hosting (users run their own instance)
- Scanned / image-only PDF support
- LaTeX equation rendering
- Mobile native app

---

## Feature Requirements

### AUTH — Authentication & User Session

**FR-001** User can create an account with email and password.

**FR-002** User can sign in with Google OAuth (single-click, no separate registration required).

**FR-003** Session persists across browser restarts via JWT with silent refresh — user does not re-authenticate unless token expires or they sign out.

**FR-004** Each user's data is isolated — no cross-user data access is possible at the API level.

**FR-005** User can sign out explicitly; session is cleared on sign-out.

**FR-006 [Pre-launch gate — not v1 dogfooding scope]** Users can delete their own account and all associated data (PDFs, notes, saved explanations, user record) via a self-serve account deletion flow. During the dogfooding phase, account deletion is handled by admin action. This FR is a hard gate for any public launch.

---

### READER — PDF Upload & In-Browser Render

**FR-010** User can upload a PDF file from local disk (max file size: 50 MB).

**FR-011** Uploaded PDF renders in-browser on the reading view. First page must be visible within 3 seconds on a 5 MB file measured at 10 Mbps download bandwidth (the dogfooding measurement baseline).

**FR-012** User can navigate pages: next, previous, direct page number input.

**FR-013** Unsupported file types (scanned/image-only PDFs, password-protected PDFs, non-PDF files) display a clear, human-readable error message explaining the limitation. No silent failure.

**FR-014** Reading position (last viewed page) is saved server-side per user per paper. When the user reopens a paper, they are returned to the last page they viewed. Position is not stored in browser local storage — it persists across browser restarts and cache clears. Cross-device sync optimization is v1.5.

---

### EXPLAIN — AI Bilingual Explain

**FR-020** User can select any text in the rendered PDF. A contextual action popup appears.

**FR-021** User triggers "Explain" on selected text. System returns a bilingual explanation popup without navigating away from the reading view.

**FR-022** Explanation popup contains all three of:
- **Vietnamese explanation** — plain language, no unexplained jargon, written for a Vietnamese reader with intermediate English proficiency
- **English technical context** — definition in the relevant domain, common usage pattern
- **Domain tag** — one of: Security / DevOps / General, inferred from paper content

**FR-023** Explanation quality standard: an explanation is acceptable if a Vietnamese reader with intermediate English proficiency can understand the term without opening another browser tab. A literal dictionary translation with no technical context does not meet this standard.

*Validation:* Quality is validated during dogfooding against real paper vocabulary before public launch. Failing explanations trigger prompt revision.

**FR-024** Popup closes on explicit close action or click-outside.

**FR-025** By default, AI explanations are ephemeral — they appear in the popup and are discarded when the popup is closed.

**FR-025a** The explain popup includes a "Save explanation" action. When triggered, the explanation is saved and attached to the text position — the text excerpt is marked with a visible indicator in the reading view, recoverable when the user returns to that passage.

**FR-026** Explain calls use Claude Haiku via AWS Bedrock. Latency target: p95 response < 4 seconds.

---

### NOTE — Auto Smart Note Generation

**FR-030** When a user reaches the last page of a paper, a non-intrusive banner appears at the bottom of the reading view: *"You've finished this paper. Generate smart note?"* with Confirm and Dismiss actions. ReadFlow never auto-generates smart notes without explicit user confirmation.

**FR-030a** If the user dismisses the banner, it does not reappear during the same reading session. The offer can be re-triggered manually via FR-031.

**FR-031** User can manually trigger smart note generation at any point from the reading view, regardless of reading position. This is the primary trigger path for users who re-read sections after reaching the last page.

**FR-032** Smart note generation is asynchronous. A progress indicator is shown. User may continue browsing the app while generation runs.

**FR-033** Generated smart note follows this fixed structure:
- **What** — What is this paper about? (1–3 sentences)
- **Why** — Why does this problem matter? (1–2 sentences)
- **How** — What approach or method did the authors use? (2–4 sentences)
- **Key Terms** — 5–10 domain-specific terms, each as three separately addressable fields: (1) term in English, (2) Vietnamese explanation in plain language, (3) English technical context (domain usage pattern, 1–2 sentences). Three distinct fields required — not a combined string — to support downstream flashcard generation in v1.5.

**FR-034** Smart note is saved to the user's account, associated with the paper, and viewable from the paper library.

**FR-035** The AI-generated content (What / Why / How / Key Terms) is read-only. Users cannot edit the AI output.

**FR-035a** Below the AI-generated content, each smart note includes an editable "My Notes" section — a freeform text block, empty by default, auto-saved when the user types in it.

**FR-036** Smart note generation uses Claude Sonnet via AWS Bedrock. Quality target: the Key Terms section must meet the same bilingual quality standard as FR-023. Latency target: < 30 seconds p95.

**FR-037** Full paper text is transmitted to AWS Bedrock for smart note generation. The upload screen displays a one-line privacy disclosure alongside the upload action: *"Paper content is processed by AWS Bedrock (Claude) in a controlled AWS environment. Your data is not used to train AI models."* No additional consent flow or full privacy policy is required in v1.

---

### LIBRARY — Paper Library

**FR-040** Authenticated user sees a list of all their uploaded papers on the library view.

**FR-041** Each paper entry displays: title (extracted from PDF metadata; falls back to filename), upload date, reading status (Unread / In Progress / Finished), and smart note availability indicator.

**FR-042** User can open a paper from the library.

**FR-043** User can delete a paper (with confirmation dialog). Deletion removes the PDF file and all associated smart notes and saved explanations.

**FR-044** Basic status filter in v1: Unread / In Progress / Finished. Tag and folder organization are v1.5.

---

### QUOTA — AI Quota Governance

**FR-050a** Free-tier users are allocated 50 AI explain calls per day. Quota resets at UTC 00:00.

**FR-050b** Free-tier users are allocated 5 smart note generations per month. Quota resets on the account creation anniversary date.

**FR-051** When a user has consumed 80% of either quota, a visible non-blocking warning is shown at the point of next AI feature use.

**FR-052** When a quota is exhausted, the relevant AI feature is disabled. The user sees a clear message stating which limit was reached and when it resets.

**FR-053** Quota state is enforced server-side. Client-side display is informational only.

**FR-054** Administrators can adjust quota limits per user for dogfooding, testing, and support purposes.

---

## Non-Functional Requirements

### NFR-DATA — Data Hosting & Privacy

**NFR-D01** All user data — PDFs, notes, user records, session data — is stored exclusively within the operator's AWS account. No data is sent to third-party SaaS platforms outside of AWS. This is a trust requirement: the product's data-sovereignty promise must hold in every component decision, not just at the application layer.

**NFR-D02** AI explain requests transmit only the selected text excerpt to AWS Bedrock — not full paper content.

**NFR-D03** Smart note generation transmits full paper text to AWS Bedrock. This is a disclosed behavior (see FR-037).

**NFR-D04** Analytics, if any, run on infrastructure within the operator's AWS account. No third-party analytics SDK (e.g. Google Analytics, Mixpanel) in v1.

---

### NFR-PERF — Performance

| Operation | Target |
|---|---|
| PDF first page render | < 3s for 5 MB PDF, standard broadband |
| AI explain response | < 4s p95 |
| Smart note generation | < 30s p95 (async with progress indicator) |
| Paper library load | < 1s for up to 100 papers |
| Page navigation | < 500ms per page turn |

---

### NFR-PDF — PDF Compatibility (v1 Constraints)

**NFR-P01** ReadFlow v1 supports text-based PDFs only. "Text-based" means the PDF contains a selectable text layer — not scanned or image-rendered pages.

**NFR-P02** Scanned, image-only, password-protected, and encrypted PDFs are not supported in v1. These must surface a clear, actionable error message rather than rendering incorrectly or silently failing.

**NFR-P03** LaTeX equation rendering is out of scope for v1. Equations render as their raw source or are skipped; this is disclosed as a known limitation.

**NFR-P04** Multi-column academic layouts are handled on a best-effort basis. Degraded text extraction (wrong reading order) is acceptable in v1 if disclosed at render time.

---

### NFR-BROWSER — Browser & Device Support

**NFR-B01** Supported desktop browsers: Chrome 110+, Firefox 110+, Safari 16+, Edge 110+.

**NFR-B02** Mobile responsive layout for library and smart note views. PDF reader on mobile is best-effort in v1; layout degradation is acceptable but must not crash or produce broken UI. Flashcard review (v1.5) is the primary mobile-critical flow — v1 mobile quality is incidental.

---

### NFR-COST — Infrastructure Cost Constraint

**NFR-C01** Monthly AWS cost target during solo validation phase: $15–25.

**NFR-C02** Bedrock cost per active free-tier user must remain < $0.50/month at the quota limits defined in FR-050a and FR-050b. If quota limits are set correctly, this should hold by construction — but must be verified before public launch.

---

## Known Limitations (v1)

These are deliberate v1 constraints, not bugs. Each should be surfaced to users at the appropriate point in the product rather than discovered by accident.

| Limitation | User-facing disclosure |
|---|---|
| Text-based PDFs only | Error message on upload of unsupported format |
| LaTeX equations not rendered | Noted in onboarding / help text |
| Multi-column layout text extraction may be incorrect | Best-effort note shown on render |
| Explanations are ephemeral by default; saveable on explicit user action | Tooltip in explain popup |
| Smart note AI content is read-only; user commentary ("My Notes") is editable | Visual separation in smart note view |
| No flashcard / spaced repetition | Roadmap reference in UI (coming in v1.5) |

---

## Open Questions

All open questions resolved as of 2026-06-06. See `.decision-log.md` for full rationale.

| # | Question | Resolution |
|---|---|---|
| OQ-001 | Smart note trigger | Banner on last page reached, user confirms. Never auto-generate. Manual trigger always available. → FR-030, FR-030a, FR-031 |
| OQ-002 | Reading position in v1 | Included in v1. Server-side per user per paper. Cross-device sync optimization is v1.5. → FR-014 |
| OQ-003 | Bedrock privacy disclosure | One-line disclosure on upload screen. No full consent flow required for v1. → FR-037 |
| OQ-004 | Account deletion | Manual deletion via admin sufficient for dogfooding phase. Self-serve deletion required before public launch — deferred to pre-launch checklist. |
| OQ-005 | Domain tag inference | Inferred automatically from paper content/metadata. Not user-configurable in v1. → FR-022 |
| OQ-006 | Manual sidebar notes | Confirmed deferred to v1.5. |
