# PRD Quality Review — ReadFlow

## Overall verdict

This is an unusually honest, tight PRD for a solo-developer MVP: the thesis is explicit, trade-offs are named, and most FRs have testable edges. The primary weakness for story creation is a cluster of under-specified "done" conditions in the EXPLAIN and NOTE sections, three FRs whose acceptance criteria rely on adjectives rather than thresholds, and a handful of deferred concerns (account deletion, multi-column disclosure UX, domain-tag inference mechanism) that will create scope ambiguity mid-sprint.

---

## 1. Decision-readiness — adequate

The PRD names the thesis, states key exclusions explicitly, and marks all OQs as resolved with resolution pointers. Quota numbers and cost targets are concrete. The deliberate v1-exclusions paragraph in Scope calls out the trade-off honestly (flashcards deferred, reading hypothesis first). The dogfooding-first launch strategy is stated as a decision, not a hedge.

Weaknesses: The FR-023 quality standard ("a Vietnamese reader with intermediate English proficiency can understand the term without opening another browser tab") is a human-judgment gate with no specified rater, cadence, or failure action beyond "prompt revision." For a solo builder it works in dogfooding, but the mechanism should be named more precisely. OQ-004 (account deletion) is listed as "resolved" but the resolution is "manual via admin — deferred to pre-launch checklist," which means there is no checklist yet; this is an unresolved item dressed as resolved.

### Findings

- **medium** Validation mechanism for FR-023 is under-specified (§ EXPLAIN FR-023) — "tested against ≥10 real terms" with no pass/fail criterion, no named rater, and no definition of what triggers a "prompt revision." A solo builder can live with this but it is not a testable requirement; it is a personal commitment. *Fix:* Add a binary pass/fail criterion: e.g., "Explanation is rated acceptable by the builder for ≥8 of 10 tested terms before onboarding additional users."
- **medium** OQ-004 resolution is fictional (§ Open Questions) — "manual deletion via admin, self-serve deletion deferred to pre-launch checklist" is not a resolution; it is a deferral without a checklist or trigger. *Fix:* Either create the pre-launch checklist item explicitly, or add a FR that gates public launch on self-serve account deletion.
- **low** Cost guardrail NFR-C02 says "must be verified before public launch" but names no verification method (§ NFR-COST). *Fix:* Add one sentence: verified via a cost simulation run with max-quota usage (50 explains × Haiku token cost + 5 smart notes × Sonnet token cost) before opening to additional users.

---

## 2. Substance over theater — strong

This is the PRD's strongest dimension. The persona (Minh) drives specific feature decisions — the Jobs-to-be-Done table maps directly to the three product pillars, and the dogfooding-first strategy is a consequence of the primary user being the builder himself. The competitive table in the addendum is honest about what was and was not web-validated. The vision statement ("read a technical paper, understand it, capture it, keep it") is product-specific and cannot swap into another PRD unchanged. NFRs have numeric thresholds, not adjectives.

One theater risk: the "Controlled data hosting" differentiator is framed as a trust differentiator for Vietnamese users wary of foreign SaaS — but no evidence is cited that this is a real user concern vs. a developer value projection. It may be correct, but it is an [ASSUMPTION] that isn't tagged.

### Findings

- **low** Controlled hosting as a user-trust differentiator is an untagged assumption (§ Product Vision, pillar 2) — the claim that Vietnamese tech learners specifically value this is plausible but unconfirmed. *Fix:* Add [ASSUMPTION] tag or note "to be validated in dogfooding interviews."
- **low** Domain tag inference (FR-022, FR-025a domain tag) names three values (Security / DevOps / General) but does not specify how inference works or what the failure mode is when inference is wrong — which is likely for interdisciplinary papers. This is not theater, but it is incomplete. *Fix:* Add one sentence to FR-022: "If domain cannot be inferred with confidence, tag defaults to General."

---

## 3. Strategic coherence — strong

The PRD has a clear thesis: validate the read → note loop before building the retain loop. Feature prioritization follows this: explain and smart notes are v1; flashcards and spaced repetition are v1.5. Success metrics are tied directly to the hypothesis (core loop completion rate, explain retention, return rate). Counter-metrics address cost sustainability and the AI-offloading anti-pattern. The Won't Have list is credible and specific. The Deliberate v1 Exclusions paragraph names what was left out and why.

The one coherence gap: the 7-day return rate metric (≥40%) and the Smart note access rate (≥50%) are healthy metrics for any product; they do not specifically validate that *bilingual AI explain* is what drives value. None of the metrics isolates the bilingual/Vietnamese dimension — if English-only explain performed identically, the metrics would pass. This is not a flaw for a solo dogfooding phase, but it means the v1 metrics do not differentiate ReadFlow's thesis from a generic PDF AI reader.

### Findings

- **medium** No metric validates the bilingual/Vietnamese thesis specifically (§ Success Metrics) — all five primary metrics would pass for an English-only tool, meaning v1 metrics cannot confirm whether the Vietnamese-language AI is the value driver. *Fix:* Add one qualitative data point target: "During dogfooding, ≥1 direct observation or interview note confirming Vietnamese explanation reduced tab-switching."
- **low** Reading flow proxy metric (explain-to-finish rate ≥60%) is weakly defined — "% of sessions with ≥1 explain call that reach the last page" counts sessions where the user also happened to finish, not sessions where explain *caused* them to finish. *Fix:* Note that this is a proxy and name the confound (user may have finished regardless of explain).

---

## 4. Done-ness clarity — adequate (weakest dimension for story creation)

Most FRs have testable edges. The quota numbers (FR-050: 50/day, 5/month), time bounds (FR-026: p95 < 4s, FR-036: p95 < 30s), and explicit UI behaviors (FR-030, FR-030a, FR-031) are engineer-readable. FR-013, FR-024, FR-043 have clear confirmation/error behaviors. FR-004 ("no cross-user data access is possible at the API level") is a strong security testability statement.

Failures:

1. **FR-011** says "First page must be visible within 3 seconds on a 5 MB file over standard broadband" — "standard broadband" is untestable. How many Mbps? Is this a CI test or a real-device test?
2. **FR-014** says "Cross-device sync optimization is v1.5" but the FR itself requires server-side persistence — no scope boundary is drawn for what "optimization" means vs. the already-required behavior.
3. **FR-033** specifies smart note structure with sentence-count bounds (1–3, 1–2, 2–4, 5–10 terms) — these are good — but "bilingual definitions (Vietnamese + English)" for Key Terms does not specify format (separate fields vs. combined string vs. "term — EN: ... VI: ..."). This will cause rework when a developer implements the note schema.
4. **FR-035** says "AI-generated content is read-only. Users cannot edit the AI output." This is clear. But **FR-035a** says the "My Notes" section is "auto-saved when the user types in it" — no autosave trigger is specified (on keystroke? on blur? after 2 seconds of idle?). For a solo build this matters because it determines whether a debounce timer is required.
5. **NFR-B02** says "mobile responsive layout for library and smart note views" — no breakpoints or screen-size floor is stated.

### Findings

- **high** FR-011 "standard broadband" is untestable in CI (§ READER FR-011) — no Mbps floor, no test environment specification. *Fix:* Replace with "< 3s on a simulated 10 Mbps connection in local test environment, or < 3s measured on developer's own broadband during dogfooding."
- **high** FR-033 Key Terms format is unspecified (§ NOTE FR-033) — "bilingual definitions (Vietnamese + English)" does not define the data structure. A developer will make an arbitrary choice that UX will then have to work around. *Fix:* Specify: "Each Key Term entry contains: term (string), Vietnamese definition (string), English definition (string), stored as structured fields, not as a single concatenated string."
- **medium** FR-035a autosave trigger is unspecified (§ NOTE FR-035a) — "auto-saved when the user types" is ambiguous. *Fix:* Add "debounced: saves 2 seconds after the user stops typing, or on blur, whichever comes first."
- **medium** FR-014 cross-device sync scope gap (§ READER FR-014) — requiring server-side persistence already enables cross-device sync; calling it "optimization" in v1.5 is unclear. *Fix:* Specify: "v1 includes server-side persistence. Cross-device sync is considered complete in v1. The v1.5 item refers to conflict resolution and multi-tab behavior only."
- **low** NFR-B02 mobile breakpoints absent (§ NFR-BROWSER) — "responsive layout" without a breakpoint floor creates UX/dev ambiguity. *Fix:* Add minimum tested viewport: "≥ 375px width (iPhone SE floor)."

---

## 5. Scope honesty — strong

The Won't Have list is specific and motivated (team workspace, self-hosting, scanned PDFs, LaTeX, mobile native app). The Known Limitations table is a deliberate disclosure mechanism — this is excellent practice. The Deliberate v1 Exclusions paragraph names flashcards as the most impactful deferral and explains why. The addendum's Build Capacity Analysis section is unusually transparent about the 12–24 hour budget and names the highest-risk items.

Minor gaps: account deletion (OQ-004) is deferred without a trigger; LaTeX equations "render as their raw source or are skipped" (NFR-P03) — "or are skipped" is two different behaviors with different user impacts; which one is it?

### Findings

- **medium** NFR-P03 LaTeX behavior is ambiguous: "render as raw source or are skipped" are different outcomes (§ NFR-PDF NFR-P03). A user who sees "$$\sum_{i=0}^{n} f(x_i)$$" instead of the equation is in a different situation than a user who sees nothing. *Fix:* Pick one: "equations render as raw LaTeX source text (not rendered)" is the more useful default for a tech reader.
- **low** FR-022 domain tag has no stated fallback behavior when inference fails (§ EXPLAIN FR-022) — see also Dimension 2. Repeated here as a scope-honesty issue because "General" as a default is a scoping decision that should be explicit. *Fix:* State the fallback explicitly in FR-022.

---

## 6. Downstream usability — adequate

**For UX:** The user journey is reconstructible from Minh's persona + the Jobs-to-be-Done table + the FR sequence. FR IDs are stable and cross-referenced (OQ table points to FRs, NFR-D02/D03 point to FR-037). The Known Limitations table doubles as a disclosure specification, which a UX designer can directly consume. The smart note structure (What / Why / How / Key Terms + My Notes) is concrete enough for UI design.

**For story creation:** FR IDs are sequential by feature group (AUTH 001-005, READER 010-014, EXPLAIN 020-026, NOTE 030-037, LIBRARY 040-044, QUOTA 050-054). The sub-FR pattern (FR-025a, FR-025b, FR-030a, FR-035a) is used consistently and logically for scoping that arrived late. This is clean.

**Glossary drift:** The PRD uses "smart note" and "Smart Note" interchangeably (capitalization inconsistent across §Scope, §NOTE, §LIBRARY). The term "highlights" appears in FR-043 ("Deletion removes the PDF file and all associated smart notes and highlights") but highlights as a feature are not defined in v1 — FR-025a/25b define "saved explanations" with "visible indicator," not a highlights system. This creates terminology ambiguity: is a saved explanation a "highlight"? The Known Limitations table also says "Explanations are ephemeral by default; saveable on explicit user action" which aligns with FR-025a — but FR-043's "highlights" suggests a separate concept that was never specified.

**For architecture:** The addendum's tech stack section is correctly separated from the PRD and appropriately positioned as "for reference." The polyrepo structure and CI/CD pipeline are clear. The SM-2 spec in the addendum is well-formed for v1.5 implementation.

### Findings

- **high** FR-043 references "highlights" as a deletable artifact but highlights are not a defined v1 feature (§ LIBRARY FR-043) — the only v1 "marking" feature is saved explanations (FR-025a/25b). A developer implementing deletion logic will not know whether to delete a "highlights" table, a "saved_explanations" table, or both. *Fix:* Replace "highlights" in FR-043 with "saved explanations" to match FR-025a terminology. If a separate highlights table is intended, define it.
- **medium** "Smart note" capitalization is inconsistent throughout the PRD — used as "smart note," "Smart Note," and "smart notes" in different sections. *Fix:* Standardize to lowercase "smart note" (consistent with FR-033 header and product usage).
- **low** FR-025b references "hover or tap on the marked text" for saved explanations — this conflates desktop and mobile interaction patterns in a single FR. For story creation, these are separate implementation items. *Fix:* Split: "On desktop: hover. On mobile: tap. Both reveal the saved explanation."

---

## 7. Shape fit — adequate

This is a consumer product with a named protagonist (Minh) whose journey is concrete and load-bearing: the Jobs-to-be-Done table maps directly to v1 scope, and the explain-to-finish-rate metric is derived from Minh's specific reading behavior. The persona is doing real work here, not decorative work.

The PRD's shape is a tight feature-spec document, which fits the solo-developer + 6-week budget reality. The addendum correctly offloads architecture and v1.5 detail. The Known Limitations table functions as a user-facing disclosure map — this is the right shape for a product that needs to set expectations early.

Shape concern: the "dogfooding-first" strategy means the product's first real user is also the only developer. This is honest and pragmatic. But it creates a shape gap: there is no user research section, no usability acceptance criterion, and no plan for how Minh (the persona, not the builder) would experience the product differently from the builder. For the 6-week MVP this is acceptable. For any subsequent user onboarding, the PRD will need a UJ (user journey) section that traces Minh's first session end to end — currently the FRs are modular but there is no narrative path through them.

### Findings

- **medium** No end-to-end user journey traces Minh's first session (§ entire PRD) — the FRs cover all the pieces but a UX designer or story writer cannot reconstruct the first-run experience (onboarding → first upload → first explain → smart note offer → note view) without assembling the FRs manually. *Fix:* Add a short "First Session User Journey" section (4–6 steps, Minh as protagonist) that sequences the FRs into a flow. This is load-bearing for UX handoff.
- **low** The FR set has no empty-state or error-state coverage for LIBRARY — FR-040 shows "all uploaded papers" but there is no FR for the zero-paper state (Minh's first visit after account creation). *Fix:* Add FR-040a: "When the library is empty, user sees a prompt to upload their first paper with a direct call to action."

---

## Mechanical notes

- **FR-043 terminology mismatch** — "highlights" is not defined in v1 scope. See Dimension 6, high finding.
- **OQ-004 false resolution** — listed as resolved but the resolution is "deferred to pre-launch checklist." No checklist exists in the PRD. This OQ is open.
- **FR-025 / FR-025a / FR-025b numbering** — the sub-FR pattern is clear and consistent. No issue.
- **NFR-P03 ambiguity** — "render as their raw source or are skipped" — pick one behavior.
- **Addendum status** — the addendum correctly flags competitive intelligence as "web-validated research was not available during PRD authoring session." This is honest; no fix required.
- **Decision log reference** — the PRD references `.decision-log.md` for full OQ rationale (§ Open Questions) but this file is not linked or listed in the repo structure in the addendum. Downstream users (UX, story author) will not know where to find it. *Fix:* Add the decision log path to the repo structure table in the addendum.
- **No ID assigned to Known Limitations rows** — the Known Limitations table has no IDs, making it impossible to cross-reference from stories. *Fix:* Add KL-001 through KL-006 IDs to the table rows.
