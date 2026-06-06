# ReadFlow PRD — Addendum

> This file captures depth that belongs in downstream documents (architecture, solution design) or that earned a place but does not fit the PRD body. It is not a substitute for the PRD — the PRD is canonical for product requirements.

---

## Tech Stack Decisions

*These are architectural decisions for reference. They inform implementation but are not product requirements. Full ADRs should live in `readflow-docs`.*

### Frontend
- **Framework:** Next.js (React)
- **PDF rendering:** PDF.js (in-browser, no server-side render required)

### Backend
- **Framework:** FastAPI (Python)
- **Responsibilities:** REST API for user data, AI orchestration, quota management, file management

### AI Layer
- **Provider:** AWS Bedrock
- **Inline explain:** Claude Haiku — latency-optimized, cost-efficient for high-frequency calls
- **Smart note generation:** Claude Sonnet — quality-optimized for structured long-form output
- **Future (v1.5):** Claude Sonnet for flashcard generation from Key Terms

### Storage
- **File storage:** Amazon S3 — PDF files
- **Relational data:** PostgreSQL (AWS RDS) — user records, notes, highlights, flashcards, quota counters

### Infrastructure
- **Platform:** AWS
- **Compute:** k3s on t3.medium (cost-optimized; full EKS deferred until scaling requires it)
- **Cost target:** $15–25/month during solo validation phase
- **IaC:** Terraform (VPC, EKS/k3s, RDS, S3, IAM)
- **GitOps:** ArgoCD with Helm values per environment (dev / prod)

### CI/CD Pipeline
```
git push
  → GitHub Actions (test + Trivy security scan + build container image)
  → push to Amazon ECR (tag: git SHA, signed with Cosign)
  → bump image tag in readflow-gitops repo
  → ArgoCD syncs to k3s cluster
```

### Repository Structure (Polyrepo, GitHub)
| Repo | Purpose |
|---|---|
| `readflow-web` | Next.js frontend |
| `readflow-api` | FastAPI backend |
| `readflow-infra` | Terraform modules (VPC, compute, RDS, S3, IAM) |
| `readflow-gitops` | ArgoCD manifests, Helm values per env |
| `readflow-docs` | BMAD artifacts, ADRs, runbooks |

---

## SM-2 Algorithm Behavioral Specification

*For v1.5 reference. Captured here rather than in the PRD because flashcards are deferred from v1.*

When a user reviews a flashcard, they rate it on a 4-point scale:

| Rating | Meaning | Interval behavior |
|---|---|---|
| **Again** | Complete failure to recall | Interval resets to 1 day |
| **Hard** | Recalled with significant difficulty | Interval increases slowly (factor < 1) |
| **Good** | Recalled correctly with some effort | Interval increases at standard SM-2 rate |
| **Easy** | Recalled instantly with no effort | Interval increases faster (bonus factor) |

Additional constraints:
- No card is shown more than once per calendar day regardless of rating
- Cards new to the user are introduced in order of Key Term appearance in the paper
- "Again" rating does not cascade — card is re-queued for the next day, not shown again in the same session

---

## Competitive Intelligence

*Captured from product brief. Web-validated research was not available during PRD authoring session.*

### Why ReadFlow's niche exists

The tools that solve parts of this problem are either general-purpose or US-market-focused:

| Tool | Strengths | Gap |
|---|---|---|
| SciSpace | Inline AI explain, academic focus, citation lookup | No Vietnamese, US-hosted |
| RemNote | PDF + flashcard + spaced repetition | General-purpose, no bilingual support, no domain tuning |
| Scholarcy | Structured summary + flashcard export | $9.99/month, no Vietnamese, no SR integration |
| Mochi / Space / Retain | Strong SM-2/FSRS spaced repetition | No integrated PDF reader |

### ReadFlow's defensible position

Three properties that co-occur only in ReadFlow:
1. **Domain specificity** — prompts tuned for InfoSec and DevOps vocabulary
2. **Bilingual output** — Vietnamese + English, not translation but contextual explanation
3. **Controlled hosting** — operator-owned AWS, no data leaving a trusted boundary

None of these competitors will pursue this combination because:
- The Vietnamese tech learner market is too small for VC-backed SaaS economics
- Self-hosted positioning cannibalizes SaaS revenue models
- Domain-specific prompt tuning requires ongoing investment that doesn't scale across general markets

This is the canonical "too small for them, large enough for you" niche.

### Risk: scope creep toward general tool
The competitive pressure from RemNote and SciSpace may push product decisions toward "also works for English users" or "also works for general topics." This should be resisted in v1. Narrow focus is the differentiator, not a limitation.

---

## Build Capacity Analysis

*Context for scope decisions.*

| Parameter | Value |
|---|---|
| Developer count | 1 (solo) |
| Hours per week | 2–4 |
| Target MVP timeline | 6 weeks |
| Total build budget | ~12–24 hours |
| AI coding assist | Claude Code + Cursor |

The 12–24 hour envelope significantly constrains what can ship in v1. The scope trim (DL-003) was derived from this constraint. With AI coding assistance, throughput per hour is higher than traditional solo development — but architecture, debugging, and integration still dominate time budgets.

Highest-risk items by time complexity:
1. PDF.js integration with text selection and popup overlay
2. AWS Bedrock API integration with quota tracking
3. Auth flow (email + Google OAuth)
4. Smart note generation prompt engineering to meet bilingual quality standard

Lowest-risk items (AI-assistable boilerplate):
- Paper library CRUD
- Quota counter logic
- Basic error states
