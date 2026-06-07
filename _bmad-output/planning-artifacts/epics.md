---
stepsCompleted: [1, 2, 3]
inputDocuments:
  - _bmad-output/planning-artifacts/prds/prd-readflow-app-2026-06-06/prd.md
  - _bmad-output/planning-artifacts/architecture.md
  - _bmad-output/planning-artifacts/patterns.md
  - _bmad-output/planning-artifacts/prds/prd-readflow-app-2026-06-06/addendum.md
---

# ReadFlow — Epic Breakdown

## Overview

This document provides the complete epic and story breakdown for ReadFlow, decomposing requirements from the PRD and Architecture into implementable stories.

**Constraints:**
- Solo developer, 2–4 hours/week with AI coding assist (Claude Code + Cursor)
- Each story completable in one sitting (max 2–3 hours)
- Stories are strictly sequential within an epic — no parallel dependencies
- Each story produces a working, testable output — no "setup only" stories

---

## Requirements Inventory

### Functional Requirements

**AUTH — Authentication & User Session**

FR-001: User can create an account with email and password.
FR-002: User can sign in with Google OAuth (single-click, no separate registration required).
FR-003: Session persists across browser restarts via JWT with silent refresh — user does not re-authenticate unless token expires or they sign out.
FR-004: Each user's data is isolated — no cross-user data access is possible at the API level.
FR-005: User can sign out explicitly; session is cleared on sign-out.
FR-006: [Pre-launch gate — not v1 dogfooding scope] Self-serve account deletion. Users can delete their own account and all associated data.

**READER — PDF Upload & In-Browser Render**

FR-010: User can upload a PDF file from local disk (max 50 MB).
FR-011: Uploaded PDF renders in-browser. First page visible within 3 seconds on a 5 MB file at 10 Mbps.
FR-012: User can navigate pages: next, previous, direct page number input.
FR-013: Unsupported file types (scanned/image-only, password-protected, non-PDF) display a clear human-readable error. No silent failure.
FR-014: Reading position (last viewed page) saved server-side per user per paper. Cross-device sync is v1.5.

**EXPLAIN — AI Bilingual Explain**

FR-020: User can select any text in the rendered PDF. A contextual action popup appears.
FR-021: User triggers "Explain" on selected text. Bilingual explanation popup appears without navigating away from the reading view.
FR-022: Explanation popup contains: (a) Vietnamese explanation in plain language, (b) English technical context with domain usage, (c) domain tag inferred from paper content (Security / DevOps / General).
FR-023: Explanation quality standard: acceptable if a Vietnamese reader with intermediate English can understand the term without opening another browser tab. Literal dictionary translation = reject.
FR-024: Popup closes on explicit close action or click-outside.
FR-025: By default, AI explanations are ephemeral — discarded when popup closes.
FR-025a: Explain popup includes "Save explanation" action — saved with text position anchor; marked with visible indicator in reading view.
FR-026: Explain calls use Claude Haiku via AWS Bedrock. Latency target: p95 < 4 seconds.

**NOTE — Auto Smart Note Generation**

FR-030: When user reaches last page, non-intrusive banner appears: "You've finished this paper. Generate smart note?" with Confirm and Dismiss. Never auto-generates.
FR-030a: If dismissed, banner does not reappear in same session.
FR-031: Manual trigger for smart note generation always available from reading view.
FR-032: Smart note generation is asynchronous. Progress indicator shown. User may continue browsing while generation runs.
FR-033: Fixed structure: What (1–3 sentences) / Why (1–2 sentences) / How (2–4 sentences) / Key Terms (5–10 terms, each as 3 separately addressable fields: term_en, definition_vi, context_en).
FR-034: Smart note saved to user account, associated with paper, viewable from library.
FR-035: AI-generated content (What/Why/How/Key Terms) is read-only.
FR-035a: Editable "My Notes" section below AI content. Freeform text, empty by default, auto-saved.
FR-036: Smart note generation uses Claude Sonnet via AWS Bedrock. p95 < 30 seconds.
FR-037: Upload screen displays one-line disclosure: "Paper content is processed by AWS Bedrock (Claude) in a controlled AWS environment. Your data is not used to train AI models."

**LIBRARY — Paper Library**

FR-040: Authenticated user sees list of all uploaded papers.
FR-041: Each paper entry displays: title (PDF metadata → filename fallback), upload date, reading status (Unread/In Progress/Finished), smart note availability indicator.
FR-042: User can open a paper from the library.
FR-043: User can delete a paper (with confirmation dialog). Deletion cascades: removes PDF from S3, smart notes, saved explanations.
FR-044: Status filter: Unread / In Progress / Finished.

**QUOTA — AI Quota Governance**

FR-050a: Free tier: 50 AI explain calls per day. Resets at UTC 00:00.
FR-050b: Free tier: 5 smart note generations per month. Resets on account creation anniversary.
FR-051: When user has consumed 80% of either quota, visible non-blocking warning shown at next AI feature use.
FR-052: When quota is exhausted, relevant AI feature is disabled. Clear message stating which limit was reached and when it resets.
FR-053: Quota state enforced server-side. Client display is informational only.
FR-054: Administrators can adjust quota limits per user.

---

### Non-Functional Requirements

NFR-D01: All user data stored exclusively in operator's AWS account. No data sent to third-party SaaS outside AWS.
NFR-D02: AI explain requests transmit only the selected text excerpt to Bedrock — not full paper content.
NFR-D03: Smart note generation transmits full paper text to Bedrock (disclosed via FR-037).
NFR-D04: No third-party analytics SDK in v1 (no Google Analytics, Mixpanel, etc.).
NFR-PERF-01: PDF first page render < 3s for 5 MB at standard broadband.
NFR-PERF-02: AI explain response p95 < 4 seconds.
NFR-PERF-03: Smart note generation p95 < 30 seconds (async).
NFR-PERF-04: Paper library load < 1 second for up to 100 papers.
NFR-PERF-05: Page navigation < 500ms per turn.
NFR-P01: Text-based PDFs only (selectable text layer required).
NFR-P02: Scanned/image-only/password-protected PDFs must show clear actionable error.
NFR-P03: LaTeX equation rendering out of scope v1.
NFR-P04: Multi-column layouts handled best-effort.
NFR-B01: Supported browsers: Chrome 110+, Firefox 110+, Safari 16+, Edge 110+.
NFR-B02: Mobile responsive layout for library and smart note views. PDF reader mobile is best-effort.
NFR-C01: Monthly AWS cost target $15–25 during solo validation.
NFR-C02: Bedrock cost per active free-tier user < $0.50/month.

---

### Additional Requirements (Architecture & Patterns)

**Infrastructure:**
- Polyrepo structure: readflow-web / readflow-api / readflow-infra / readflow-gitops / readflow-docs
- k3s on single t3.small EC2 (ap-southeast-1) running API + PostgreSQL StatefulSet
- Terraform IaC with S3 remote state; bootstrap/ directory is one-time manual apply, never in CI
- ArgoCD GitOps; Helm values per environment (dev / prod)
- CI/CD: GitHub Actions → Trivy scan → build → push to ECR (Cosign-signed) → bump image tag in gitops repo → ArgoCD syncs
- PostgreSQL PVC must carry annotation `argocd.argoproj.io/sync-options: Prune=false` — never pruned by ArgoCD
- EBS snapshot policy (AWS DLM) required before public launch

**API & Auth:**
- JWT: access token 15min (in-memory), refresh token 7 days (httpOnly cookie)
- Google OAuth: server-side callback at `/auth/google/callback` — no Clerk / Auth0
- `ENVIRONMENT=production` disables FastAPI OpenAPI docs (/docs, /redoc) — never exposed in prod
- All API routes except `/auth/*` require `Authorization: Bearer {token}`
- CORS restricted to readflow-web domain only

**PDF Pipeline:**
- PyMuPDF (fitz) extracts text server-side on upload via FastAPI BackgroundTask
- Extracted text stored in `papers.text_content` (PostgreSQL)
- S3 presigned URLs (1h expiry) for PDF delivery — frontend fetches directly, no API proxy
- Explain endpoint receives `selected_text` from client — does NOT re-read from S3
- Smart note reads `papers.text_content` from DB — does NOT re-download from S3

**AI Service:**
- `ai_service.py` is the sole boto3 caller for Bedrock — no other file imports boto3 for Bedrock
- Prompt templates stored as `.txt` files in `app/services/prompts/` — not inline strings
- Bedrock credentials via EC2 IAM instance profile — no credential files in container

**Smart Note Async:**
- `POST /papers/{id}/generate-note` → inserts `note_jobs(status=pending)` → schedules BackgroundTask → returns `{ job_id }`
- `GET /note-jobs/{job_id}/status` → returns `{ status, note_id? }`
- Frontend polls every 3s via React Query `refetchInterval`

**Schema:**
- v1 tables: users, papers, saved_explanations, smart_notes, note_jobs, quota_daily, quota_monthly
- flashcards table present in v1 schema — no endpoints, supports v1.5 without breaking migration
- All tables include `user_id` for row-level isolation

**Implementation Patterns:**
- All JSON responses wrapped in `{ data, meta: { request_id } }` envelope
- Error codes: SCREAMING_SNAKE_CASE exhaustive enum (UNAUTHORIZED, FORBIDDEN, NOT_FOUND, QUOTA_EXCEEDED, PDF_UNSUPPORTED, PDF_TOO_LARGE, BEDROCK_ERROR, NOTE_JOB_FAILED, VALIDATION_ERROR, INTERNAL_ERROR)
- SSE event format: `{ type: "token"|"section"|"done"|"error", payload: ... }`
- DB migrations: additive-only in v1; `verb_noun_context` naming convention; never edit existing migrations
- Environment variables: same names in `.env.local`, `docker-compose.yml`, k3s Secrets; `.env.example` committed; `.env.local` git-ignored

---

### UX Design Requirements

No UX design document. UX refinement will occur during dogfooding. Component names are explicit in architecture:
- `PDFViewer.tsx` — PDF.js integration, text selection, page navigation
- `ExplainPopup.tsx` — SSE stream display, save action
- `SmartNoteBanner.tsx` — last-page trigger + confirm/dismiss
- `SmartNoteView.tsx` — read-only What/Why/How/Key Terms + editable My Notes
- `PaperCard.tsx` — library entry with status badge and note indicator
- `UploadDialog.tsx` — file picker + Bedrock disclosure text

---

### FR Coverage Map

| FR | Primary Epic | Description |
|---|---|---|
| FR-001 | Epic 2 | Email registration — `/auth/register` endpoint |
| FR-002 | Epic 2 | Google OAuth — `/auth/google/callback` server-side flow |
| FR-003 | Epic 2 + Epic 3 | JWT issuance (API), silent refresh + client storage (Web) |
| FR-004 | Epic 2 | Row-level user isolation via `user_id` on all queries |
| FR-005 | Epic 2 | Sign-out clears refresh token cookie |
| FR-006 | Epic 7 | Pre-launch gate — admin deletion sufficient for dogfooding |
| FR-010 | Epic 4 | PDF upload `POST /papers` multipart, 50 MB limit |
| FR-011 | Epic 4 | PDF.js in-browser render, first page < 3s |
| FR-012 | Epic 4 | Page navigation (next/prev/direct input) |
| FR-013 | Epic 4 | Unsupported file error — clear human-readable message |
| FR-014 | Epic 4 | Server-side reading position save + restore on open |
| FR-020 | Epic 5 | Text selection → contextual action popup |
| FR-021 | Epic 5 | Explain trigger → bilingual popup, no navigation |
| FR-022 | Epic 5 | Popup content: Vietnamese explanation + English context + domain tag |
| FR-023 | Epic 5 | Explanation quality standard (validated during dogfooding) |
| FR-024 | Epic 5 | Popup close on explicit close or click-outside |
| FR-025 | Epic 5 | Ephemeral by default — discarded on popup close |
| FR-025a | Epic 5 | "Save explanation" action + visible anchor indicator |
| FR-026 | Epic 5 | Claude Haiku via Bedrock, p95 < 4s |
| FR-030 | Epic 6 | Last-page smart note banner with Confirm/Dismiss |
| FR-030a | Epic 6 | Dismissed banner does not reappear in same session |
| FR-031 | Epic 6 | Manual smart note trigger always available in reader |
| FR-032 | Epic 6 | Async generation with progress indicator; user can browse while running |
| FR-033 | Epic 6 | Fixed note structure: What/Why/How + Key Terms (3-field schema) |
| FR-034 | Epic 6 | Note saved to account, linked to paper, viewable from library |
| FR-035 | Epic 6 | AI content (What/Why/How/Key Terms) is read-only |
| FR-035a | Epic 6 | Editable "My Notes" freeform block, auto-saved |
| FR-036 | Epic 6 | Claude Sonnet via Bedrock, p95 < 30s |
| FR-037 | Epic 4 | One-line Bedrock disclosure on upload screen |
| FR-040 | Epic 3 | Library view — empty state on scaffold |
| FR-041 | Epic 4 + Epic 7 | Paper card (basic in E4); full status + note indicator polish in E7 |
| FR-042 | Epic 4 | Open paper from library |
| FR-043 | Epic 7 | Delete paper with confirmation + cascade (S3 + notes + saved explanations) |
| FR-044 | Epic 7 | Status filter: Unread / In Progress / Finished |
| FR-050a | Epic 5 | 50 explain calls/day quota, resets UTC 00:00 |
| FR-050b | Epic 6 | 5 note generations/month quota, resets on anniversary |
| FR-051 | Epic 5 + Epic 6 | 80% quota warning at point of use |
| FR-052 | Epic 5 + Epic 6 | Quota exhausted: feature disabled + reset time shown |
| FR-053 | Epic 5 | Server-side quota enforcement (pattern established, extended to notes in E6) |
| FR-054 | Epic 7 | Admin quota adjustment per user |

---

## Epic List

### Epic 1: Infrastructure Bootstrap
Provision all AWS infrastructure, establish the GitOps pipeline, and validate end-to-end deployment. **Done when ArgoCD is running on k3s and successfully syncs a hello-world Deployment.**

No direct product FRs — this is enabling infrastructure. Addresses NFR-D01 (all data stays in operator AWS account), NFR-C01 (t3.small cost-constrained), and CI/CD security requirements from the architecture.

**Key deliverables:** Terraform bootstrap (S3 state backend + DynamoDB lock), VPC + EC2 t3.small + k3s install, S3 bucket (PDFs), ECR repo + lifecycle policy, IAM instance profile (Bedrock + S3 + ECR), ArgoCD installed on cluster, hello-world app synced via ArgoCD.

---

### Epic 2: API Foundation — Auth & Database Schema
Scaffold `readflow-api` with all database tables and working auth endpoints. **Done when `POST /auth/register` and `POST /auth/login` return a valid JWT pair.**

**FRs covered:** FR-001, FR-002, FR-003, FR-004, FR-005

**Key deliverables:** FastAPI app with all 8 SQLAlchemy models (including `flashcards` for v1.5 readiness), Alembic initial migration, bcrypt password auth, Google OAuth server-side callback, JWT access + refresh token pair, row-level isolation enforced, Dockerfile + docker-compose for local dev parity.

---

### Epic 3: Web Foundation — Auth UI & Library Shell
Scaffold `readflow-web` with auth flow (login/register) and a protected library page. **Done when a logged-in user sees the library page (empty state) in the browser.**

**FRs covered:** FR-003 (client-side JWT handling + silent refresh), FR-040 (library view)

**Key deliverables:** Next.js 15 App Router scaffold with shadcn/ui + React Query, login/register forms wired to API, protected route pattern, JWT in-memory with refresh token cookie, library page showing empty state.

---

### Epic 4: PDF Pipeline — Upload, Render & Reading Position
Users can upload a PDF, read it page by page in-browser, and have their reading position remembered. **Done when user can upload a paper, navigate pages, and reopen to the last page viewed.**

**FRs covered:** FR-010, FR-011, FR-012, FR-013, FR-014, FR-037, FR-041 (basic paper card), FR-042

**Key deliverables:** `POST /papers` multipart upload → S3 store → PyMuPDF BackgroundTask text extraction, presigned URL generation, `PDFViewer.tsx` with PDF.js, page navigation, unsupported file error handling, server-side position save/restore, `UploadDialog.tsx` with Bedrock disclosure text, paper card in library.

---

### Epic 5: Explain Feature — SSE Streaming Popup & Save
Users can select text, see a bilingual explanation stream in a popup, and optionally save it. **Done when the full explain + save explanation flow works end-to-end with quota enforcement.**

**FRs covered:** FR-020, FR-021, FR-022, FR-023, FR-024, FR-025, FR-025a, FR-026, FR-050a, FR-051, FR-052, FR-053

**Key deliverables:** `POST /explain/stream` SSE endpoint with `StreamingResponse`, `quota_daily` table + `QuotaService.check_and_increment()`, `POST /papers/{id}/saved-explanations` endpoint, `ExplainPopup.tsx` with SSE stream rendering, `useExplainStream` hook, 80% warning + exhaustion message in UI.

---

### Epic 6: Smart Note Feature — Async Generation & Editable Notes
Users can confirm smart note generation and see the structured What/Why/How/Key Terms note with an editable My Notes section. **Done when the full async generation → polling → note view → My Notes editing flow works.**

**FRs covered:** FR-030, FR-030a, FR-031, FR-032, FR-033, FR-034, FR-035, FR-035a, FR-036, FR-050b, FR-051 (note quota), FR-052 (note quota)

**Key deliverables:** `POST /papers/{id}/generate-note` → `note_jobs` insert → `BackgroundTask: generate_note_task`, `GET /note-jobs/{job_id}/status` polling endpoint, `PATCH /smart-notes/{id}/my-notes` auto-save endpoint, `SmartNoteBanner.tsx` (last-page + manual trigger), `SmartNoteView.tsx` (read-only AI + editable My Notes), React Query poll every 3s.

---

### Epic 7: Library & Polish
Complete remaining must-have FRs: paper deletion with cascade, status filter, quota dashboard, admin quota override, mobile responsive pass, known limitation disclosures. **Done when all must-have FRs are covered and the product is ready for dogfooding.**

**FRs covered:** FR-006 (admin deletion), FR-041 (full: status badge + note indicator), FR-043, FR-044, FR-054

**NFRs addressed:** NFR-B02 (mobile responsive library + notes), NFR-PERF-04 (library load < 1s), NFR-P03/P04 (LaTeX + multi-column disclosures)

**Key deliverables:** `DELETE /papers/{id}` with cascade (S3 + smart_notes + saved_explanations), status filter query param, full paper card with status badge + note indicator, admin quota endpoint, mobile responsive pass on library and smart note views.

---

### Epic 8: Production Hardening
Establish DevSecOps posture: security scanning, signed images, policy enforcement, secrets management, observability, and validated runbooks. **Done when the platform story is demonstrable for CV/interview.**

**No new product FRs** — operational excellence. Addresses NFR-D01 (deeper: OPA/Secrets Manager layer), NFR-C01/C02 (SLO visibility), CI/CD security architecture requirements.

**Key deliverables:** Trivy scan in GitHub Actions CI, Cosign image signing on ECR push, OPA policies for k3s admission control, AWS Secrets Manager (or Vault) for production secrets, CloudWatch SLO dashboard (explain latency p95, note success rate, quota utilization), k3s-node-replacement runbook tested end-to-end including StatefulSet finalizer handling.

---

## Epic 1: Infrastructure Bootstrap

Provision all AWS infrastructure and establish the GitOps pipeline. Done when ArgoCD is running on k3s and successfully syncs a hello-world Deployment.

**FRs covered:** None directly — enabling infrastructure.
**NFRs addressed:** NFR-D01 (all data in operator AWS, ap-southeast-1), NFR-C01 (t3.small cost target).

---

### E1-S1: Bootstrap Terraform Remote State

As a developer,
I want a remote Terraform state backend provisioned in S3 with a DynamoDB lock table,
So that all subsequent Terraform runs are safe, collaborative, and recoverable.

**What to build:** Create the `readflow-infra/bootstrap/` one-time manual module that provisions the S3 state bucket and DynamoDB lock table. Wire the `envs/dev/` backend block to use this bucket. The `bootstrap/` directory must carry a README that explicitly warns it is never run in CI.

**Acceptance Criteria:**

**Given** the developer has AWS credentials configured with sufficient IAM permissions
**When** they run `cd readflow-infra/bootstrap && terraform init && terraform apply`
**Then** an S3 bucket named `readflow-tfstate-<aws-account-id>` is visible via `aws s3 ls`
**And** `aws dynamodb describe-table --table-name readflow-tfstate-lock` returns `ACTIVE` status

**Given** the bootstrap is complete
**When** developer runs `cd readflow-infra/envs/dev && terraform init`
**Then** Terraform initializes successfully using the S3 backend (output confirms remote state backend)
**And** a second `terraform init` produces no errors (idempotent)

**Local verification (no full stack needed):**
```
aws s3 ls | grep readflow-tfstate
aws dynamodb describe-table --table-name readflow-tfstate-lock --query 'Table.TableStatus'
cd readflow-infra/envs/dev && terraform init
```

**Tasks:**
- Create `readflow-infra/bootstrap/main.tf` — S3 bucket (versioning on, server-side encryption AES-256) + DynamoDB table (PAY_PER_REQUEST billing, `LockID` hash key)
- Create `readflow-infra/bootstrap/README.md` — explicit warning: "Run once manually. Never add to CI/CD."
- Create `readflow-infra/envs/dev/main.tf` — `terraform { backend "s3" { bucket, key, region, dynamodb_table } }`
- Create `readflow-infra/.terraform.lock.hcl` after init (commit this)
- Run `terraform apply` in bootstrap, then `terraform init` in `envs/dev`

**Estimated time:** 1.5 hours
**Depends on:** None

---

### E1-S2: Provision VPC and Network Security Groups

As a developer,
I want a VPC with a public subnet, internet gateway, and security groups provisioned via Terraform,
So that the EC2 instance has network connectivity and only necessary ports are exposed.

**What to build:** Create the `readflow-infra/modules/vpc/` Terraform module that provisions a VPC (10.0.0.0/16), a public subnet (10.0.1.0/24), an internet gateway, a route table, and a security group. The security group must allow inbound SSH (22), k3s API (6443), HTTP (80), HTTPS (443), and NodePort range (30000–32767). All outbound is open.

**Acceptance Criteria:**

**Given** `terraform apply` completes in `envs/dev`
**When** developer runs `aws ec2 describe-vpcs --filters Name=tag:Project,Values=readflow-dev`
**Then** exactly one VPC is returned with CIDR `10.0.0.0/16` and state `available`

**Given** the VPC is provisioned
**When** developer runs `aws ec2 describe-security-groups --filters Name=tag:Project,Values=readflow-dev`
**Then** a security group exists with inbound rules for ports 22, 6443, 80, 443, and range 30000–32767

**Given** the module is applied
**When** developer runs `terraform plan` a second time
**Then** output shows `No changes. Infrastructure is up-to-date.`

**Local verification (no full stack needed):**
```
aws ec2 describe-vpcs --filters "Name=tag:Project,Values=readflow-dev" --query 'Vpcs[0].State'
aws ec2 describe-subnets --filters "Name=tag:Project,Values=readflow-dev" --query 'Subnets[0].CidrBlock'
terraform plan  # must show 0 changes
```

**Tasks:**
- Create `readflow-infra/modules/vpc/main.tf` — VPC, subnet, IGW, route table + association, security group with listed inbound rules
- Create `readflow-infra/modules/vpc/variables.tf` — `project_name`, `env`, `vpc_cidr`, `subnet_cidr`, `region`
- Create `readflow-infra/modules/vpc/outputs.tf` — `vpc_id`, `subnet_id`, `security_group_id`
- Wire `module "vpc"` into `readflow-infra/envs/dev/main.tf` with `ap-southeast-1` region
- Add resource tags: `Project = "readflow-${var.env}"`
- Run: `terraform plan && terraform apply`

**Estimated time:** 1.5 hours
**Depends on:** E1-S1

---

### E1-S3: Provision EC2 Instance and Install k3s

As a developer,
I want a t3.small EC2 instance with k3s installed and a 20 GB gp3 EBS data volume,
So that the Kubernetes cluster is running and ready to host application workloads.

**What to build:** Create the `readflow-infra/modules/compute/` Terraform module that launches a t3.small EC2 instance (Ubuntu 22.04 LTS) using a user-data script that installs k3s. The module attaches a 20 GB gp3 EBS volume (mounted at `/data`) for PostgreSQL persistence. An Elastic IP is allocated and associated with the instance.

**Acceptance Criteria:**

**Given** `terraform apply` completes
**When** developer runs `aws ec2 describe-instances --filters Name=tag:Name,Values=readflow-dev-node --query 'Reservations[0].Instances[0].State.Name'`
**Then** output is `"running"`

**Given** the instance is running (allow 3–5 min for user-data to complete)
**When** developer SSHes in (`ssh -i readflow-dev.pem ubuntu@<eip>`) and runs `kubectl get nodes`
**Then** output shows one node with `STATUS=Ready` and `ROLES=control-plane,master`

**When** developer runs `kubectl get pods -n kube-system`
**Then** `coredns` and `metrics-server` pods are in `Running` state

**When** developer runs `lsblk`
**Then** a 20 GB disk is visible (separate from the root volume) and mounted at `/data`

**Local verification (no full stack needed):**
```
aws ec2 describe-instances --filters "Name=tag:Name,Values=readflow-dev-node" \
  --query 'Reservations[0].Instances[0].State.Name'
# Then SSH:
ssh -i ~/.ssh/readflow-dev.pem ubuntu@$(terraform output -raw instance_public_ip)
kubectl get nodes   # must show Ready
lsblk               # must show /data mount
```

**Tasks:**
- Create `readflow-infra/modules/compute/main.tf` — EC2 t3.small (Ubuntu 22.04 AMI), EIP, key pair reference, EBS 20 GB gp3 volume + attachment + mount in user-data
- Create user-data script: `curl -sfL https://get.k3s.io | sh -s - --write-kubeconfig-mode 644` + `mkdir -p /data && mount /dev/xvdf /data` (with fstab entry)
- Create `readflow-infra/modules/compute/variables.tf` — `vpc_id`, `subnet_id`, `sg_id`, `key_name`, `instance_type`
- Create `readflow-infra/modules/compute/outputs.tf` — `instance_public_ip`, `instance_id`
- Wire compute module into `envs/dev/main.tf`
- Run: `terraform apply` → SSH → `kubectl get nodes`

**Estimated time:** 2.5 hours
**Depends on:** E1-S2

---

### E1-S4: Provision S3 Bucket, ECR Repository, and IAM Instance Profile

As a developer,
I want the PDF S3 bucket, ECR repository, and EC2 IAM instance profile provisioned via Terraform,
So that the application can store PDFs, push/pull container images, and call Bedrock — all without hardcoded credentials.

**What to build:** Create Terraform modules for: (1) the `readflow-dev-pdfs` S3 bucket (versioning on, private, server-side encryption), (2) the `readflow-api` ECR repository (lifecycle policy: keep last 10 images), and (3) an IAM instance profile for the EC2 instance with a policy granting `s3:*` on the PDFs bucket, `ecr:*` on the readflow-api repo, and `bedrock:InvokeModel` + `bedrock:InvokeModelWithResponseStream` on all resources.

**Acceptance Criteria:**

**Given** `terraform apply` completes
**When** developer runs `aws s3 ls | grep readflow-dev-pdfs`
**Then** the bucket appears in the list

**When** developer runs `aws ecr describe-repositories --repository-names readflow-api`
**Then** the repository is returned with `repositoryUri` in output

**When** developer SSHes to the EC2 instance and runs `aws s3 ls s3://readflow-dev-pdfs`
**Then** the command succeeds without error (IAM instance profile grants access, no credentials file needed)

**When** developer runs on instance: `aws ecr get-login-password --region ap-southeast-1 | docker login --username AWS --password-stdin <ecr-uri>`
**Then** login succeeds

**Local verification (no full stack needed):**
```
aws s3 ls | grep readflow-dev-pdfs
aws ecr describe-repositories --repository-names readflow-api --query 'repositories[0].repositoryUri'
aws iam get-instance-profile --instance-profile-name readflow-dev-ec2-profile \
  --query 'InstanceProfile.Roles[0].RoleName'
# Then SSH to instance:
aws s3 ls s3://readflow-dev-pdfs   # must succeed with no credentials file
```

**Tasks:**
- Create `readflow-infra/modules/storage/main.tf` — S3 bucket with versioning, AES-256 SSE, `block_public_acls = true`
- Create `readflow-infra/modules/ecr/main.tf` — ECR repo, lifecycle policy JSON (expire images count > 10)
- Create `readflow-infra/modules/iam/main.tf` — IAM role (`ec2.amazonaws.com` trust), inline policy for S3+ECR+Bedrock, instance profile, `aws_iam_instance_profile_association` to EC2
- Wire all three modules into `envs/dev/main.tf`
- Run: `terraform apply` → verify from AWS Console or CLI → SSH to instance and test

**Estimated time:** 2 hours
**Depends on:** E1-S3

---

### E1-S5: Bootstrap ArgoCD and Sync Hello-World Deployment

As a developer,
I want ArgoCD installed on k3s and a hello-world application syncing from the `readflow-gitops` repository,
So that the GitOps pipeline is proven end-to-end before any real application code is deployed.

**What to build:** Install ArgoCD via its stable manifest on the k3s cluster. Create the `readflow-gitops` repository with the hello-world Helm chart skeleton and the ArgoCD Application CRD. Apply the Application and confirm ArgoCD syncs the hello-world pod to the `readflow-dev` namespace. This is the E1 done-when condition.

**Acceptance Criteria:**

**Given** ArgoCD is installed and the gitops repo is accessible
**When** developer runs `kubectl get pods -n argocd`
**Then** all ArgoCD pods (`argocd-server`, `argocd-application-controller`, `argocd-repo-server`, `argocd-dex-server`) show `Running`

**When** developer runs `kubectl port-forward svc/argocd-server -n argocd 8080:443` and opens `https://localhost:8080`
**Then** the ArgoCD login UI loads in the browser

**When** developer applies the hello-world Application CRD and waits for sync
**When** developer runs `kubectl get applications -n argocd`
**Then** `readflow-hello-world` shows `STATUS=Synced` and `HEALTH=Healthy`

**When** developer runs `kubectl get pods -n readflow-dev`
**Then** at least one pod from the hello-world Deployment is in `Running` state

**Local verification (no full stack needed):**
```
# Run on EC2 via SSH:
kubectl get pods -n argocd
kubectl get applications -n argocd -o jsonpath='{.items[0].status.sync.status}'
# Must output: Synced
kubectl get pods -n readflow-dev
# Must show: Running
```

**Tasks:**
- SSH to instance: `kubectl create namespace argocd`
- `kubectl apply -n argocd -f https://raw.githubusercontent.com/argoproj/argo-cd/stable/manifests/install.yaml`
- Create `readflow-gitops/` repo structure:
  - `apps/hello-world/Chart.yaml` (Helm chart metadata)
  - `apps/hello-world/values.yaml` (replicas: 1, image: nginx:alpine)
  - `apps/hello-world/templates/deployment.yaml` + `service.yaml`
  - `namespaces/readflow-dev.yaml` (Namespace manifest)
  - `argocd/app-hello-world.yaml` (ArgoCD Application CRD — source: gitops repo, dest: readflow-dev namespace)
- `kubectl apply -f readflow-gitops/argocd/app-hello-world.yaml`
- Verify: `kubectl get applications -n argocd` shows Synced

**Estimated time:** 2 hours
**Depends on:** E1-S4

---

## Epic 2: API Foundation — Auth & Database Schema

Scaffold `readflow-api` with all database tables and working auth endpoints. Done when `POST /auth/register` and `POST /auth/login` return a valid JWT pair, Google OAuth callback returns a token, Alembic migration runs clean on a fresh DB, and the Docker image builds and passes Trivy scan in CI.

**FRs covered:** FR-001, FR-002, FR-003, FR-004, FR-005

---

### E2-S1: Scaffold readflow-api with Project Structure and Service Stubs

As a developer,
I want the full `readflow-api` project skeleton committed and the server starting successfully,
So that all subsequent stories add to a consistent, well-structured codebase without reorganizing files later.

**What to build:** Initialize the FastAPI project with the full directory structure from the architecture spec — `routers/`, `models/`, `schemas/`, `services/`, `tasks/`. Create `ai_service.py` with the complete public interface (`explain_stream` and `generate_smart_note` raising `NotImplementedError`) and the `prompts/` directory with placeholder `.txt` files. Include `docker-compose.yml`, `Dockerfile`, and `.env.example`.

**Acceptance Criteria:**

**Given** the repo is cloned and `.env.local` is created from `.env.example`
**When** developer runs `uvicorn app.main:app --reload --port 8000`
**Then** the server starts without error

**When** developer runs `curl http://localhost:8000/`
**Then** response is `200` with body `{"data": {"status": "ok"}, "meta": {"request_id": "<uuid>"}}`

**When** developer runs `python -m pytest tests/ -v`
**Then** all tests pass (smoke test: `import app.main` succeeds + `GET /` returns 200)

**When** developer calls `ai_service.explain_stream("test", "Security")` in a Python REPL
**Then** `NotImplementedError` is raised — stub exists and enforces the boundary

**When** developer checks `app/services/prompts/`
**Then** `explain_system.txt` and `note_system.txt` both exist with non-empty placeholder comments

**Local verification (no full stack needed):**
```bash
cd readflow-api
cp .env.example .env.local    # fill JWT_SECRET_KEY with any dev string
uvicorn app.main:app --reload --port 8000
# separate terminal:
curl http://localhost:8000/   # must return {"data":{"status":"ok"},...}
python -m pytest tests/ -v    # must pass
```

**Tasks:**
- Create `readflow-api/pyproject.toml` with all deps: `fastapi>=0.115`, `uvicorn[standard]`, `sqlalchemy>=2.0`, `alembic`, `pydantic-settings`, `python-jose[cryptography]`, `passlib[bcrypt]`, `boto3`, `pymupdf`, `python-multipart`, `google-auth-oauthlib`, `itsdangerous`; dev: `pytest`, `httpx`, `pytest-asyncio`, `pytest-cov`
- Create directory skeleton: `app/{main.py, config.py, dependencies.py, routers/__init__.py, models/__init__.py, schemas/__init__.py, schemas/response.py, services/__init__.py, services/ai_service.py, services/prompts/explain_system.txt, services/prompts/note_system.txt, tasks/__init__.py}`
- `app/main.py` — FastAPI init, CORS (`FRONTEND_URL` env var), `X-Request-ID` middleware, global exception handler → `INTERNAL_ERROR` envelope, `GET /` health endpoint
- `app/schemas/response.py` — `SuccessResponse[T]`, `ErrorResponse`, `ErrorDetail` generics per Pattern 1
- `app/services/ai_service.py` — `AIService` class: `explain_stream()` and `generate_smart_note()` raise `NotImplementedError`; singleton `ai_service = AIService(...)`
- `app/services/prompts/explain_system.txt` — `# EXPLAIN SYSTEM PROMPT — content written in E5-S3`
- `app/services/prompts/note_system.txt` — `# NOTE SYSTEM PROMPT — content written in E6-S3`
- `docker-compose.yml` — `postgres:16-alpine` + `api` service matching Pattern 4 env vars exactly
- `Dockerfile` — multi-stage: `python:3.12-slim` builder (pip install) + runtime stage (non-root user `appuser`)
- `.env.example` — all vars documented with inline comments
- `tests/conftest.py` — empty; `tests/test_smoke.py` — `GET /` returns 200

**Estimated time:** 2 hours
**Depends on:** None (E2 entry point; E1 is infra-only with no code dependency)

---

### E2-S2: Create Initial Database Schema Covering All v1 Tables

As a developer,
I want all 8 v1 database tables created by a single clean Alembic migration,
So that the schema is established atomically and subsequent stories can insert data without migration dependency issues.

**What to build:** Write SQLAlchemy ORM models for all 8 v1 tables: `users`, `papers`, `smart_notes`, `saved_explanations`, `note_jobs`, `quota_daily`, `quota_monthly`, and `flashcards`. Generate the Alembic initial migration with `--autogenerate`, review it, and run it against a local postgres instance. `flashcards` is created now with no endpoints — satisfies the v1.5 schema readiness requirement (ADR-004).

**Acceptance Criteria:**

**Given** postgres is running via `docker compose up -d postgres`
**When** developer runs `alembic upgrade head`
**Then** the command completes without error

**When** developer runs `psql $DATABASE_URL -c "\dt"`
**Then** all 8 tables appear: `users`, `papers`, `smart_notes`, `saved_explanations`, `note_jobs`, `quota_daily`, `quota_monthly`, `flashcards`, plus `alembic_version`

**When** developer runs `alembic check`
**Then** output is `No new upgrade operations detected.` (zero schema drift between models and DB)

**When** developer runs `alembic downgrade base && alembic upgrade head`
**Then** both commands complete without error (clean round-trip)

**When** developer inspects the `flashcards` table schema
**Then** columns `id, paper_id, user_id, term_en, definition_vi, context_en, interval_days, ease_factor, due_date, repetitions, created_at` all exist with correct types

**Local verification (no full stack needed):**
```bash
docker compose up -d postgres
alembic upgrade head
alembic check                                    # must show: No new upgrade operations
psql $DATABASE_URL -c "\dt"                      # must list all 8 tables
alembic downgrade base && alembic upgrade head   # must complete cleanly
```

**Tasks:**
- Create `app/models/user.py` — `users(id UUID PK default gen_random_uuid(), email TEXT UNIQUE NOT NULL, password_hash TEXT, google_id TEXT, created_at TIMESTAMPTZ default now())`
- Create `app/models/paper.py` — `papers(id, user_id FK→users, title TEXT, s3_key TEXT, text_content TEXT, status VARCHAR default 'unread', last_page INT default 1, created_at)`
- Create `app/models/smart_note.py` — `smart_notes(id, paper_id FK→papers, user_id FK→users, what TEXT, why TEXT, how TEXT, key_terms_json JSONB, my_notes TEXT default '', created_at, updated_at)`
- Create `app/models/saved_explanation.py` — `saved_explanations(id, paper_id FK, user_id FK, text_excerpt TEXT, text_position_json JSONB, explanation_vi TEXT, explanation_en TEXT, domain_tag VARCHAR, created_at)`
- Create `app/models/note_job.py` — `note_jobs(id, paper_id FK, user_id FK, status VARCHAR default 'pending', note_id FK→smart_notes nullable, error_msg TEXT, created_at, updated_at)`
- Create `app/models/quota.py` — `quota_daily(id, user_id FK, date DATE, explain_count INT default 0, UNIQUE(user_id, date))`; `quota_monthly(id, user_id FK, year_month VARCHAR(7), note_count INT default 0, UNIQUE(user_id, year_month))`
- Create `app/models/flashcard.py` — `flashcards(id, paper_id FK, user_id FK, term_en TEXT, definition_vi TEXT, context_en TEXT, interval_days INT default 1, ease_factor FLOAT default 2.5, due_date DATE, repetitions INT default 0, created_at)`
- Configure `alembic/env.py` to import `Base.metadata` from all models
- Run: `alembic revision --autogenerate -m "create_all_v1_tables"` → review generated file → `alembic upgrade head`

**Estimated time:** 2 hours
**Depends on:** E2-S1

---

### E2-S3: Implement Email Auth Endpoints and JWT get_current_user Dependency

As a developer testing the API,
I want to register an account and receive a JWT that gates all protected endpoints,
So that the auth boundary is verified working before any product feature is built on top of it.

**What to build:** Implement `POST /auth/register` (creates user with bcrypt-hashed password, returns JWT pair), `POST /auth/login` (validates credentials, returns JWT pair), `POST /auth/refresh` (rotates access token from httpOnly refresh cookie), and `GET /me` (protected — used to verify the `get_current_user` dependency works). Access token: 15 min, in-memory. Refresh token: 7 days, httpOnly cookie.

**Acceptance Criteria:**

**Given** `docker compose up` is running
**When** developer runs `curl -s -X POST http://localhost:8000/auth/register -H "Content-Type: application/json" -d '{"email":"test@example.com","password":"SecurePass1!"}'`
**Then** response is `201` with `{"data": {"access_token": "<jwt>", "token_type": "bearer"}, "meta": {...}}`
**And** response headers include `Set-Cookie: refresh_token=<value>; HttpOnly; SameSite=Lax`

**When** developer calls `POST /auth/login` with same credentials
**Then** response is `200` with a new valid `access_token`

**When** developer calls `GET /me` with `Authorization: Bearer <access_token>`
**Then** response is `200` with `{"data": {"user_id": "<uuid>", "email": "test@example.com"}, "meta": {...}}`

**When** developer calls `GET /me` with a tampered or expired token
**Then** response is `401` with `{"error": {"code": "UNAUTHORIZED", ...}, "meta": {...}}`

**When** developer calls `POST /auth/register` with the same email twice
**Then** second call returns `409` with a clear message (not a 500 or DB constraint error leaking)

**Local verification (no full stack needed):**
```bash
docker compose up -d
TOKEN=$(curl -s -X POST http://localhost:8000/auth/login \
  -H "Content-Type: application/json" \
  -d '{"email":"test@example.com","password":"SecurePass1!"}' | jq -r '.data.access_token')
curl -s http://localhost:8000/me -H "Authorization: Bearer $TOKEN" | jq .
# must return 200 with email field
curl -s http://localhost:8000/me -H "Authorization: Bearer invalid.token.here" | jq .
# must return 401 UNAUTHORIZED
pytest tests/test_auth.py -v   # all tests green
```

**Tasks:**
- Create `app/schemas/auth.py` — `RegisterRequest(email, password)`, `LoginRequest`, `TokenResponse(access_token, token_type)`
- Create `app/routers/auth.py` — `POST /auth/register`, `POST /auth/login`, `POST /auth/refresh`, `GET /me`
- Update `app/dependencies.py` — `get_current_user(token: str = Depends(oauth2_scheme)) -> User`: decode JWT, look up user, raise `HTTP 401 UNAUTHORIZED` on any failure
- Create JWT helpers in `app/core/security.py` — `create_access_token(data, expires_delta=15min)`, `create_refresh_token()`, `decode_token(token)`
- Add `get_db()` session dependency in `dependencies.py`
- Wire `auth` router into `app/main.py`
- Create `tests/test_auth.py` — test register (201 + cookie), login (200), duplicate email (409), bad token (401), `GET /me` happy path; use `pytest-asyncio` + `httpx.AsyncClient`

**Estimated time:** 2.5 hours
**Depends on:** E2-S2

---

### E2-S4: Implement Google OAuth Server-Side Callback and Token Issuance

As a developer,
I want the Google OAuth server-side flow to exchange a code for a user profile and return a JWT identical to email auth,
So that Google sign-in works without any third-party auth service touching user data.

**What to build:** Implement `GET /auth/google` (redirects to Google consent screen with HMAC-signed `state` parameter) and `GET /auth/google/callback` (receives `code` + `state`, validates state, exchanges code for Google profile via `google-auth-oauthlib`, upserts user in DB by email, returns JWT pair). A user with an existing email-registered account gets `google_id` added — no duplicate user created.

**Acceptance Criteria:**

**Given** `GOOGLE_CLIENT_ID` and `GOOGLE_CLIENT_SECRET` are set in `.env.local`
**When** developer opens `http://localhost:8000/auth/google` in a browser
**Then** they receive a `302` redirect to `accounts.google.com/o/oauth2/auth` with `client_id`, `redirect_uri`, `state`, and `scope=openid email` in the URL

**When** Google redirects to `GET /auth/google/callback?code=<code>&state=<valid-state>`
**Then** handler exchanges code for profile, upserts user in DB
**And** returns `200` with `{"data": {"access_token": "<jwt>", "token_type": "bearer"}, "meta": {...}}`
**And** sets `refresh_token` httpOnly cookie (identical behaviour to email auth)

**When** `GET /auth/google/callback` is called with a missing or tampered `state`
**Then** response is `400` with `{"error": {"code": "VALIDATION_ERROR", "message": "Invalid OAuth state"}, "meta": {...}}`

**When** the Google profile email matches an existing email-registered user
**Then** no duplicate user is created; existing user's `google_id` column is updated

**Given** a JWT returned from the Google callback
**When** developer calls `GET /me` with `Authorization: Bearer <jwt>`
**Then** response is `200` — same protected route works for both auth methods

**Local verification (no full stack needed):**
```bash
# Test redirect (no Google credentials needed for this):
curl -v http://localhost:8000/auth/google
# Expect: HTTP 302  Location: accounts.google.com/o/oauth2/auth?...

# Test callback + upsert logic with mocked profile:
pytest tests/test_oauth.py -v   # must pass (mocked google exchange)
```

**Tasks:**
- Add `google-auth-oauthlib`, `itsdangerous` to `pyproject.toml`
- Create `app/services/oauth_service.py` — `build_authorization_url() -> (url, state)`, `exchange_code_for_profile(code, state) -> {email, google_id, name}` wrapping `google.oauth2`; `state` is HMAC-signed with `JWT_SECRET_KEY` via `itsdangerous.URLSafeSerializer` (no session DB needed)
- Add to `app/routers/auth.py`: `GET /auth/google` (call `build_authorization_url`, return redirect), `GET /auth/google/callback` (validate state, exchange code, upsert user, return JWT)
- Upsert logic in callback: `SELECT user WHERE email = profile.email` → if exists, `UPDATE google_id`; if not, `INSERT` new user (no password_hash)
- Create `tests/test_oauth.py` — `monkeypatch` / `mock.patch` on `oauth_service.exchange_code_for_profile` to return fixed profile; test new user creation, existing user upsert, tampered state rejection, JWT returned

**Estimated time:** 2.5 hours
**Depends on:** E2-S3

---

### E2-S5: Build GitHub Actions CI Pipeline with pytest, Migration Check, and Trivy Scan

As a developer,
I want every push to run tests, verify schema integrity, and scan the Docker image for vulnerabilities,
So that the `main` branch always contains a tested, deployable, and secure artifact.

**What to build:** Create `.github/workflows/api-ci.yml` with three sequential jobs: `test` (postgres service container → `alembic upgrade head` → `alembic check` → `pytest`), `migration-guard` (fails if any existing file under `alembic/versions/` was edited vs `main`), and `build` (Docker build → `trivy image --exit-code 1 --severity HIGH,CRITICAL` → ECR push on `main` only). Workflow triggers on `push` and `pull_request` to `main`.

**Acceptance Criteria:**

**Given** a PR to `main` in `readflow-api`
**When** the CI workflow runs
**Then** all three jobs (`test`, `migration-guard`, `build`) complete green in GitHub Actions

**When** developer introduces a deliberate test failure in `tests/test_auth.py` and pushes
**Then** the `test` job fails and subsequent jobs do not run

**When** developer edits any existing file under `alembic/versions/` and pushes
**Then** the `migration-guard` job fails with message: `ERROR: Existing migration <filename> was edited. Create a new migration instead.`

**When** the `Dockerfile` is temporarily changed to use a known-vulnerable base image
**Then** the `build` job fails with Trivy exit code 1

**When** all jobs pass on `main`
**Then** the Docker image is tagged with `${{ github.sha }}` and pushed to ECR
**And** the `migration-guard` job has verified no existing migrations were modified

**Local verification (no full stack needed):**
```bash
# Mirror the test job locally:
docker compose up -d postgres
alembic upgrade head && alembic check
pytest tests/ -v --cov=app

# Mirror the build + scan job locally:
docker build -t readflow-api:local .
trivy image --exit-code 1 --severity HIGH,CRITICAL readflow-api:local

# Mirror migration guard locally:
git diff origin/main --name-only | grep "alembic/versions/"
# must return empty output on a clean branch
```

**Tasks:**
- Create `.github/workflows/api-ci.yml`:
  - `test` job: `services: postgres:16-alpine`, env `DATABASE_URL` pointing to service, steps: checkout → pip install → `alembic upgrade head` → `alembic check` → `pytest --cov=app --cov-report=term`
  - `migration-guard` job: bash script — `git diff origin/main --name-only | grep alembic/versions/` → for each matched file, `git show origin/main:<file>` succeeds → exit 1 with error message
  - `build` job (needs: [test, migration-guard]): `docker build -t readflow-api:${{ github.sha }} .` → `trivy image --exit-code 1 --severity HIGH,CRITICAL readflow-api:${{ github.sha }}` → on `main`: `aws ecr get-login-password | docker login` → `docker push`
- Update `Dockerfile` to ensure base is `python:3.12-slim` with no HIGH/CRITICAL known CVEs at time of writing; add non-root `appuser`
- Add `pytest-cov` to dev deps in `pyproject.toml`
- Create `.github/workflows/README.md` — one-line description of each job and when it runs
- Push a branch → verify all three jobs green in Actions tab before merging

**Estimated time:** 1.5 hours
**Depends on:** E2-S4

---

## Epic 3: Web Foundation — Auth UI & Library Shell

Scaffold `readflow-web` with auth flow and protected library page. Done when a logged-in user sees the library page (empty state) in a real browser after registering with email.

**FRs covered:** FR-003 (client-side JWT handling + silent refresh), FR-040 (library view — empty state)

---

### E3-S1: Scaffold readflow-web with Next.js App Router and Local Dev Stack

As a developer,
I want the full `readflow-web` project scaffold running locally alongside `readflow-api` and postgres via docker-compose,
So that I can test the complete auth flow in a real browser without any Kubernetes involvement.

**What to build:** Run `create-next-app` with TypeScript, Tailwind, ESLint, App Router, and `src/` directory. Install shadcn/ui and React Query. Create the typed API client (`src/lib/api.ts`) with the `ApiSuccess<T>`, `ApiError`, and `isApiError()` types from Pattern 1. Create a `docker-compose.yml` that runs `readflow-web`, `readflow-api`, and `postgres` together for full local development.

**Acceptance Criteria:**

**Given** `docker compose up` is run from the working directory
**When** developer opens `http://localhost:3000`
**Then** the Next.js page loads without console errors

**When** developer opens `http://localhost:8000/`
**Then** API health response `{"data": {"status": "ok"}, "meta": {...}}` is returned (confirms API is reachable from web container)

**When** developer inspects `src/lib/api.ts`
**Then** `ApiSuccess<T>`, `ApiError`, `isApiError()`, and a `fetchApi()` wrapper are defined and TypeScript-valid per Pattern 1.1

**When** developer runs `npm run lint && npx tsc --noEmit`
**Then** both commands exit with code `0` — no lint errors, no type errors

**Local verification (no full stack needed — just local docker):**
```bash
docker compose up
# Browser: http://localhost:3000 — page loads, no console errors
# Browser: http://localhost:8000/ — API health OK
cd readflow-web && npm run lint && npx tsc --noEmit  # must both pass
```

**Tasks:**
- `npx create-next-app@latest readflow-web --typescript --tailwind --eslint --app --src-dir --import-alias "@/*"`
- `npx shadcn@latest init` — style: default, color: slate, CSS variables: yes
- Install `@tanstack/react-query` and `@tanstack/react-query-devtools`
- Create `src/lib/api.ts` — `type ApiSuccess<T>`, `type ApiError`, `function isApiError()`, `async function fetchApi(path, init?)` using `NEXT_PUBLIC_API_URL` base; returns typed union
- Create `src/lib/query-client.ts` — `QueryClient` singleton, `staleTime: 30_000`
- Wrap `src/app/layout.tsx` with `<QueryClientProvider client={queryClient}>`
- Create `docker-compose.yml` — services: `postgres:16-alpine`, `api` (build from `../readflow-api`), `web` (build from `.`); `web` depends on `api`, `api` depends on `postgres`; env vars match Pattern 4 exactly
- Create `readflow-web/.env.example` — `NEXT_PUBLIC_API_URL=http://localhost:8000`
- Set `"strict": true` in `tsconfig.json`

**Estimated time:** 2 hours
**Depends on:** E2-S1 (readflow-api must exist for docker-compose)

---

### E3-S2: Build Login and Register Pages with Email Auth API Integration

As a developer,
I want to register a new account via the `/register` page and be redirected to `/library` in a real browser,
So that the email auth flow is proven working end-to-end before any UI polish or middleware is added.

**What to build:** Create `/login` and `/register` pages under `src/app/(auth)/`. Each page renders a shadcn/ui form. On submit, call `POST /auth/register` or `POST /auth/login` via React Query mutations. Store the returned `access_token` in an `AuthContext` that wraps the app. On success, redirect to `/library`. Show field-level and API-level error messages inline. On page load, attempt `POST /auth/refresh` silently to restore session from the httpOnly refresh cookie.

**Acceptance Criteria:**

**Given** `docker compose up` is running with a fresh database
**When** developer navigates to `http://localhost:3000/register` and submits a valid email and password
**Then** `POST /auth/register` is called, `access_token` is stored in `AuthContext`, browser redirects to `/library`
**And** browser DevTools → Application → Cookies shows `refresh_token` httpOnly cookie set by the API

**When** developer navigates to `/login` and submits credentials for the registered account
**Then** `POST /auth/login` is called, token stored, redirect to `/library`

**When** developer refreshes the browser while on `/library`
**Then** `POST /auth/refresh` is called silently on `AuthContext` mount; access token is restored from the refresh cookie without redirecting to `/login`

**When** developer submits the register form with an email that already exists
**Then** an inline error message appears ("An account with this email already exists") — no page navigation

**When** developer submits the form with an invalid email format
**Then** client-side validation catches it before the API call — error shown inline on the field

**Local verification (full auth flow in browser):**
```bash
docker compose up
# 1. Browser: http://localhost:3000/register → fill form → submit
#    Expect: redirect to /library
# 2. Open DevTools → Application → Cookies → refresh_token cookie visible (HttpOnly checked)
# 3. Refresh page → must stay on /library (silent refresh worked)
# 4. Try registering same email again → must show inline error, no navigation
```

**Tasks:**
- Create `src/context/auth-context.tsx` — `AuthContext` with `{ accessToken, user, login(token), logout(), isLoading }`; on mount: `POST /auth/refresh` → if 200, `login(newToken)`; if 401, set `isLoading = false` (stay unauthenticated)
- Create `src/app/(auth)/layout.tsx` — minimal layout (no nav)
- Create `src/app/(auth)/login/page.tsx` — shadcn `Form` + `Input` (email, password) + `Button`; React Query `useMutation` → `POST /auth/login`; on success: `login(token)` + `router.push('/library')`; on error: inline message from `response.error.message`
- Create `src/app/(auth)/register/page.tsx` — same pattern with `POST /auth/register`; client-side validation: email format, password min-length 8
- Create `src/hooks/use-auth.ts` — `useAuth()` wrapping `useContext(AuthContext)`
- Wrap `src/app/layout.tsx` with `<AuthProvider>`

**Estimated time:** 2 hours
**Depends on:** E3-S1

---

### E3-S3: Implement Next.js Route Protection Middleware

As a developer,
I want unauthenticated users redirected to `/login` and authenticated users redirected away from auth pages,
So that route protection is enforced at the edge before any React component renders — no content flash.

**What to build:** Create `src/middleware.ts` that checks the `refresh_token` httpOnly cookie as the server-side proxy for "user is authenticated." Routes under `/(app)` redirect to `/login` if the cookie is absent. Routes under `/(auth)` redirect to `/library` if the cookie is present. The middleware must not make any API calls — cookie presence check only.

**Acceptance Criteria:**

**Given** the user is NOT logged in (no `refresh_token` cookie)
**When** they navigate to `http://localhost:3000/library`
**Then** browser redirects to `/login` — no flash of library content visible

**Given** the user IS logged in (`refresh_token` cookie present)
**When** they navigate to `http://localhost:3000/login`
**Then** browser redirects to `/library`

**Given** the user is logged in and on `/library`
**When** they refresh the page
**Then** page renders normally — no redirect loop

**Given** the middleware runs on a request
**When** developer inspects Network tab in DevTools
**Then** no API calls are made by middleware — zero XHR/fetch from the middleware layer

**When** developer runs `npx tsc --noEmit`
**Then** `src/middleware.ts` compiles without type errors

**Local verification (no full stack needed — just start `npm run dev`):**
```bash
npm run dev   # start web only, no docker needed for this test
# Test 1 — incognito browser (no cookies):
#   http://localhost:3000/library → must redirect to /login instantly (no flash)
# Test 2 — after login (refresh_token cookie set):
#   http://localhost:3000/login → must redirect to /library
# Test 3 — DevTools Network tab: navigate between pages, confirm no API calls from middleware
npx tsc --noEmit   # must exit 0
```

**Tasks:**
- Create `src/middleware.ts`:
  - Import `NextRequest`, `NextResponse`
  - Check `request.cookies.get('refresh_token')` presence
  - If route matches `/(app)/(.*)` pattern and no cookie → `NextResponse.redirect(new URL('/login', request.url))`
  - If route matches `/(auth)/(.*)` pattern and cookie present → `NextResponse.redirect(new URL('/library', request.url))`
  - Export `config.matcher: ['/(app)/(.*)', '/(auth)/(.*)']`
- Reorganise `src/app/` directory into route groups:
  - `src/app/(auth)/` — login, register, auth/callback
  - `src/app/(app)/` — library (and future reader, notes pages)
- Create `src/app/(app)/layout.tsx` — placeholder `{children}` layout (nav added in E7)

**Estimated time:** 1.5 hours
**Depends on:** E3-S2

---

### E3-S4: Handle Google OAuth Browser Redirect and Token Storage

As a developer,
I want the Google OAuth flow to end with the user on the library page with a valid token in `AuthContext`,
So that both auth methods (email and Google) arrive at the same authenticated state.

**What to build:** Add a "Sign in with Google" link to the login page that navigates to `API_URL/auth/google`. Create `src/app/(auth)/auth/callback/page.tsx` — a client component that reads `access_token` (or `error`) from URL query params after the API redirects back from Google. On success: call `login(token)` + redirect to `/library`. On error: show error message + link back to `/login`. Update the API's Google callback to redirect to `FRONTEND_URL/auth/callback?access_token=<jwt>` on success.

**Acceptance Criteria:**

**Given** `GOOGLE_CLIENT_ID` and `GOOGLE_CLIENT_SECRET` are set in `readflow-api/.env.local`
**When** developer clicks "Sign in with Google" on the `/login` page
**Then** browser navigates to `http://localhost:8000/auth/google` which issues a `302` to Google's consent screen

**Given** Google redirects to the API callback and the API redirects to `http://localhost:3000/auth/callback?access_token=<jwt>`
**When** the callback page mounts
**Then** `login(token)` is called and browser navigates to `/library`

**When** developer navigates to `/auth/callback?error=oauth_failed`
**Then** an error message is displayed ("Sign-in failed. Please try again.") with a "Back to login" link
**And** no redirect to `/library` occurs

**When** developer navigates to `/auth/callback` with no query params at all
**Then** page redirects to `/login`

**Given** a JWT stored from Google callback
**When** developer calls any protected API endpoint with that token via `fetchApi()`
**Then** the request succeeds (same token format as email auth)

**Local verification:**
```bash
docker compose up
# With real Google credentials in readflow-api/.env.local:
# Browser: http://localhost:3000/login → click "Sign in with Google"
# Expect: Google consent → redirect back → /library loaded

# Without Google credentials — test error path:
# Browser: http://localhost:3000/auth/callback?error=oauth_failed
# Expect: error message + "Back to login" link
# Browser: http://localhost:3000/auth/callback (no params)
# Expect: redirect to /login
```

**Tasks:**
- Add "Sign in with Google" `<a href={`${process.env.NEXT_PUBLIC_API_URL}/auth/google`}>` to `src/app/(auth)/login/page.tsx`
- Create `src/app/(auth)/auth/callback/page.tsx` — `'use client'`; `useSearchParams()` to read `access_token` and `error`; on mount: if token → `login(token)` + `router.replace('/library')`; if error → show error + link to `/login`; if neither → `router.replace('/login')`
- Update `readflow-api/app/routers/auth.py` `GET /auth/google/callback` — on success redirect to `settings.FRONTEND_URL + '/auth/callback?access_token=' + access_token` (small addition to E2-S4 work)
- Add `FRONTEND_URL` env var to `readflow-api/.env.example` and `docker-compose.yml` (if not already present)

**Estimated time:** 1.5 hours
**Depends on:** E3-S3

---

### E3-S5: Build Library Page with Authenticated Empty State and Upload CTA

As a logged-in user,
I want to see a library page with a clear empty state and an "Add your first paper" prompt,
So that the core navigation destination exists and is ready for real paper data in E4.

**What to build:** Create `src/app/(app)/library/page.tsx`. Paper list data comes from `src/lib/mock-data.ts` (empty array by default, with a commented example) — no `GET /papers` API call in this story, as that endpoint doesn't exist yet. When the mock list is empty, render `<LibraryEmptyState>`. When non-empty, render `<PaperCard>` components. The upload CTA button is a visual placeholder only — no dialog or upload logic until E4.

**Acceptance Criteria:**

**Given** the user is logged in and on `http://localhost:3000/library`
**When** `src/lib/mock-data.ts` exports `mockPapers = []`
**Then** the page renders with a heading ("My Library"), empty state illustration or icon, the message "No papers yet", and an "Add your first paper" `<Button>`

**Given** the user is NOT logged in
**When** they navigate to `/library`
**Then** middleware redirects to `/login` — library content never renders (verified by middleware in E3-S3)

**Given** `src/lib/mock-data.ts` is edited to export 2 mock paper objects
**When** the page hot-reloads
**Then** 2 `<PaperCard>` components are visible, each showing `title`, `upload_date`, a status badge, and a note indicator

**When** developer clicks "Add your first paper"
**Then** nothing happens (button is a visual placeholder — no dialog, no navigation)

**When** developer runs `npx tsc --noEmit`
**Then** all new files type-check without errors

**Local verification (full auth flow in browser):**
```bash
docker compose up
# 1. Register at /register → redirected to /library
#    Must see: "My Library" heading + "No papers yet" + "Add your first paper" button
# 2. Incognito → http://localhost:3000/library → must redirect to /login
# 3. Edit src/lib/mock-data.ts → add 2 mock objects → save
#    Must see: 2 paper cards rendered (hot-reload)
npx tsc --noEmit   # must exit 0
```

**Tasks:**
- Create `src/lib/mock-data.ts` — `export const mockPapers: Paper[] = []`; include commented example paper object: `/* { id: 'mock-1', title: 'TLS 1.3 Paper', upload_date: '2026-06-06', status: 'unread', has_note: false } */`
- Create `src/types/api.ts` — TypeScript interfaces: `Paper { id: string, title: string, upload_date: string, status: 'unread' | 'in_progress' | 'finished', has_note: boolean }`, `SmartNote`, `ExplainRequest` (mirrors backend Pydantic schemas)
- Create `src/app/(app)/library/page.tsx` — reads `mockPapers`; if empty → `<LibraryEmptyState />`; else → `<div>` with `mockPapers.map(p => <PaperCard key={p.id} paper={p} />)`
- Create `src/components/library/LibraryEmptyState.tsx` — upload icon (lucide-react `Upload`), `<p>No papers yet</p>`, `<Button onClick={() => {}}>Add your first paper</Button>`
- Create `src/components/library/PaperCard.tsx` — shadcn `<Card>` with `title`, `upload_date` (formatted), status `<Badge>` (variant by status), `has_note` indicator (dot or icon)

**Estimated time:** 1.5 hours
**Depends on:** E3-S4

---

### E3-S6: Build GitHub Actions CI for readflow-web

As a developer,
I want every push to `readflow-web` to validate lint, TypeScript types, and a successful production build,
So that type regressions and broken builds are caught immediately before code reaches `main`.

**What to build:** Create `.github/workflows/web-ci.yml` in `readflow-web` with three sequential steps: ESLint (`next lint`), TypeScript type-check (`tsc --noEmit`), and Next.js production build (`next build`). No test step — no business logic to test in E3. Workflow triggers on `push` and `pull_request` to `main`. Must pass green before E3 is considered done.

**Acceptance Criteria:**

**Given** a PR to `main` in `readflow-web`
**When** the CI workflow runs
**Then** all three steps (lint, type-check, build) complete green

**When** developer introduces a TypeScript type error and pushes
**Then** the `type-check` step fails with a clear error message in the Actions log

**When** developer introduces an ESLint violation and pushes
**Then** the `lint` step fails

**When** developer introduces a broken import that makes `next build` fail
**Then** the `build` step fails

**The workflow does NOT run `npm test`** — no test requirement in E3; tests will be added in later epics when business logic exists.

**Local verification (mirrors CI exactly):**
```bash
cd readflow-web
npm run lint          # must exit 0
npx tsc --noEmit      # must exit 0
npm run build         # must complete successfully
# Push branch → GitHub Actions tab → all 3 steps green
```

**Tasks:**
- Create `.github/workflows/web-ci.yml`:
  - `on: push + pull_request` to `main`, `paths: ['readflow-web/**']`
  - Job `ci`, `runs-on: ubuntu-latest`
  - Steps: checkout → `npm ci` → `npm run lint` → `npx tsc --noEmit` → `npm run build`
  - Build env: `NEXT_PUBLIC_API_URL: http://localhost:8000` (required for `next build` to resolve env reference without error)
- Confirm `package.json` has `"lint": "next lint"` and `"build": "next build"` scripts (created by `create-next-app`, verify not removed)
- Push all E3 code to a branch → verify all 3 steps green in GitHub Actions tab

**Estimated time:** 1 hour
**Depends on:** E3-S5

---

## Epic 4: PDF Pipeline — Upload, Render & Reading Position

Users can upload a PDF, read it page by page in-browser, and have their reading position remembered. Done when a user uploads a real academic paper, reads it page by page, and returns to the same page after closing and reopening.

**FRs covered:** FR-010, FR-011, FR-012, FR-013, FR-014, FR-037, FR-041 (basic paper card), FR-042

---

### E4-S1: Implement PDF Upload Endpoint with PyMuPDF Extraction, S3 Storage, and Bedrock Disclosure

As a user,
I want to upload a PDF from my local disk and see it appear in my library,
So that I have a paper ready to read and the system has extracted its text for future AI features.

**What to build:** `POST /papers` multipart endpoint that validates the file (PDF, ≤50MB, text-based), uploads to S3, inserts a `papers` record, and schedules a `BackgroundTask` to extract text via PyMuPDF and write it to `papers.text_content`. Returns `paper_id`, `title` (from PDF metadata or filename), and `presigned_url`. The frontend `UploadDialog.tsx` must display the Bedrock disclosure text (FR-037) before the user can submit. Tests use a committed real academic PDF fixture.

**Acceptance Criteria:**

**Given** a valid text-based PDF ≤50MB
**When** `POST /papers` is called with it (`multipart/form-data`)
**Then** response is `201` with `{"data": {"paper_id": "<uuid>", "title": "<extracted-or-filename>", "presigned_url": "<s3-url>"}, "meta": {...}}`
**And** the file is visible in S3 at key `pdfs/<paper_id>.pdf`

**Given** the upload completes and the BackgroundTask runs
**When** developer queries `SELECT length(text_content) FROM papers WHERE id = '<paper_id>'` after 10 seconds
**Then** `length` is greater than 100 (non-empty extraction)

**Given** a scanned/image-only PDF (no text layer) is uploaded
**When** `POST /papers` is called with it
**Then** response is `422` with `{"error": {"code": "PDF_UNSUPPORTED", "message": "This PDF does not contain a selectable text layer..."}, "meta": {...}}`

**Given** a file larger than 50MB is uploaded
**Then** response is `422` with `{"error": {"code": "PDF_TOO_LARGE", ...}, "meta": {...}}`

**Given** the user opens the UploadDialog in the browser
**When** they see the dialog
**Then** the disclosure text is visible in the dialog body before any upload button can be clicked: *"Paper content is processed by AWS Bedrock (Claude) in a controlled AWS environment. Your data is not used to train AI models."*
**And** the "Upload" button is disabled until a file is selected

**Given** `tests/fixtures/sample.pdf` — a real text-based academic PDF (≤500KB)
**When** `pytest tests/test_pdf_service.py -v` runs
**Then** `test_extract_text_returns_content` passes: extracted text is a non-empty string containing recognisable words from the PDF

**Local verification (no full k8s needed — docker compose only):**
```bash
docker compose up -d
TOKEN=$(curl -s -X POST http://localhost:8000/auth/login \
  -H "Content-Type: application/json" \
  -d '{"email":"test@example.com","password":"SecurePass1!"}' | jq -r '.data.access_token')
curl -s -X POST http://localhost:8000/papers \
  -H "Authorization: Bearer $TOKEN" \
  -F "file=@tests/fixtures/sample.pdf" | jq .
# Expect: 201 with paper_id + presigned_url
aws s3 ls s3://readflow-dev-pdfs/pdfs/     # file must appear
sleep 10
psql $DATABASE_URL -c "SELECT length(text_content) FROM papers LIMIT 1;"
# Expect: > 0
pytest tests/test_pdf_service.py -v        # must pass
```

**Tasks:**
- Create `app/services/pdf_service.py` — `validate_pdf(file_bytes) -> None`: open with `fitz.open()`, raise `PDF_UNSUPPORTED` if page count 0 or no text layer on first page; `extract_text(file_bytes) -> str`: concatenate `page.get_text()` for all pages; `generate_presigned_url(s3_key: str, expiry: int = 3600) -> str`
- Create `app/tasks/extract_text.py` — `async def extract_text_task(paper_id, s3_key, db)`: download from S3 → `pdf_service.extract_text()` → `UPDATE papers SET text_content = ?`
- Create `app/routers/papers.py` — `POST /papers`: size check (413 before reading) → `UploadFile` → `pdf_service.validate_pdf()` → S3 `put_object(pdfs/<paper_id>.pdf)` → `INSERT papers(user_id, title, s3_key, status='unread', last_page=1)` → `BackgroundTasks.add_task(extract_text_task, ...)` → `generate_presigned_url()` → return 201
- Wire `papers` router into `app/main.py`
- Add `tests/fixtures/sample.pdf` — commit a real but small (≤500KB) text-based academic PDF (e.g., a freely available CS paper from arxiv.org)
- Create `tests/test_pdf_service.py` — `test_extract_text_returns_content`: calls `pdf_service.extract_text()` on `sample.pdf` bytes, asserts `len(result) > 100`; `test_validate_pdf_rejects_empty`: creates 1-page image-only PDF with `reportlab`, asserts `PDF_UNSUPPORTED` raised
- Create `src/components/library/UploadDialog.tsx` — shadcn `<Dialog>` triggered by "Add your first paper" `<Button>`; body: `<Input type="file" accept=".pdf" onChange={setFile}`; disclosure `<p className="text-sm text-muted-foreground">Paper content is processed by AWS Bedrock...</p>`; `<Button disabled={!file} onClick={handleUpload}>Upload</Button>`; on success: `queryClient.invalidateQueries(['papers'])` + close dialog; on error: show `error.message` inline
- Update `src/components/library/LibraryEmptyState.tsx` — open `UploadDialog` when "Add your first paper" is clicked (wire dialog open state)

**Estimated time:** 3 hours
**Depends on:** E3-S5

---

### E4-S2: Build PDFViewer Component with PDF.js Rendering and Page Navigation

As a user,
I want to open a paper and read it page by page in the browser,
So that I can read my uploaded academic paper without leaving the app or downloading it.

**What to build:** Create `/reader/[paperId]` page and the `PDFViewer.tsx` component. Fetch the paper detail (including `presigned_url`) from `GET /papers/{id}`. Load the PDF in-browser via `pdfjs-dist`, render each page to `<canvas>` with a text-layer overlay (required for E5 text selection). Implement previous/next buttons and direct page number input. Test this with a real multi-page academic paper — not a synthetic test file.

**Acceptance Criteria:**

**Given** the user navigates to `/reader/<paper_id>`
**When** the page loads (measured with DevTools Network throttled to "Fast 3G" on a 5MB PDF)
**Then** the first page is visible within 3 seconds

**When** the user clicks "Next page"
**Then** the next page renders within 500ms (no full reload)

**When** the user clicks "Previous page" on page 2
**Then** page 1 renders

**When** the user types `5` in the page number input and presses Enter
**Then** page 5 renders (if paper has ≥5 pages)

**When** the user is on page 1 and clicks "Previous page"
**Then** the button is disabled — no action, no error

**When** the user is on the last page and clicks "Next page"
**Then** the button is disabled — no action, no error

**When** the user selects text on a rendered page
**Then** text is selectable in the browser (text layer is active, not a pure canvas image render)

**Developer AC — real paper test:** Developer must verify this story using a real multi-page academic PDF (≥10 pages, e.g., the TLS 1.3 RFC or a security/DevOps paper). All pages must render without errors. Multi-column layouts are best-effort (degraded order is acceptable, as per NFR-P04).

**Given** the PDF is fetched
**When** inspecting DevTools Network tab
**Then** the PDF is fetched directly from the S3 presigned URL — no API proxy is involved

**Local verification:**
```bash
docker compose up
# 1. Upload a real academic PDF (≥10 pages) via UploadDialog
# 2. Click paper in library → navigates to /reader/<id>
# 3. DevTools Performance → first page render < 3s on throttled connection
# 4. Click Next/Prev repeatedly → all pages render correctly
# 5. Type page number → jumps correctly
# 6. Select text on any page → browser selection highlights text
# 7. DevTools Network → PDF loaded from S3 URL (not localhost:8000)
```

**Tasks:**
- Install `pdfjs-dist`; create `src/lib/pdf-worker.ts` — `pdfjsLib.GlobalWorkerOptions.workerSrc = '/pdf.worker.min.js'`; copy worker to `public/` in `next.config.ts` using `CopyPlugin` or `next.config.ts` custom webpack
- Add backend endpoint to `app/routers/papers.py`: `GET /papers/{paper_id}` — verify `paper.user_id == current_user.id` (403 if not) → call `pdf_service.generate_presigned_url(paper.s3_key)` → return `PaperDetail` schema (id, title, presigned_url, last_page, status)
- Create `src/app/(app)/reader/[paperId]/page.tsx` — `useQuery(['paper', paperId], () => fetchApi(`/papers/${paperId}`))` → loading skeleton → pass `presigned_url` + `last_page` to `<PDFViewer>`
- Create `src/components/reader/PDFViewer.tsx` — `'use client'`; `pdfjsLib.getDocument(pdfUrl)` on mount; render `currentPage` to `<canvas ref>` via `page.render({ canvasContext, viewport })`; render text layer via `pdfjsLib.renderTextLayer()` over canvas; accept `initialPage`, `onPageChange(page: number)` props; internal `currentPage` state
- Create `src/components/reader/PageNavigation.tsx` — prev/next `<Button>` (disabled at boundaries) + `<Input type="number">` (clamped 1–totalPages, on Enter/blur navigate) + `<span>{currentPage} / {totalPages}</span>`

**Estimated time:** 2.5 hours
**Depends on:** E4-S1

---

### E4-S3: Persist and Restore Reading Position with Debounced Updates

As a user,
I want my reading position saved automatically and restored when I reopen a paper,
So that I never lose my place even after closing the browser.

**What to build:** Add `PATCH /papers/{id}/position` to update `papers.last_page` and transition status (`unread` → `in_progress` on first page turn). In the frontend reader, call this endpoint when the user changes pages, debounced to 300ms so rapid navigation does not flood the API. On paper open, `GET /papers/{id}` returns `last_page` and the reader initialises PDF.js at that page.

**Acceptance Criteria:**

**Given** the user navigates to page 7 in the reader and closes the browser tab
**When** they reopen `/reader/<paper_id>`
**Then** PDF.js opens directly to page 7 (position restored from server, not from browser storage)

**Given** the user rapidly clicks "Next page" 5 times within 2 seconds
**When** observing the Network tab in DevTools
**Then** at most 2 `PATCH /papers/{paper_id}/position` requests are fired (debounce suppresses intermediate calls)

**When** `PATCH /papers/{paper_id}/position` is called with `{"last_page": 3}`
**Then** response is `200` with `{"data": {"last_page": 3}, "meta": {...}}`
**And** `SELECT last_page FROM papers WHERE id = '<paper_id>'` returns `3`

**When** the paper status is `unread` and user navigates to page 2
**Then** `SELECT status FROM papers WHERE id = '<paper_id>'` returns `in_progress`
**And** subsequent navigations do not reset status back to `unread`

**When** developer calls `PATCH /papers/<other_user_paper_id>/position` with a different user's token
**Then** response is `403` with `{"error": {"code": "FORBIDDEN", ...}, "meta": {...}}`

**Local verification:**
```bash
docker compose up
# 1. Open a paper → navigate to page 7
# 2. Close browser tab → reopen /reader/<paper_id>
#    Expect: opens on page 7
# 3. Rapidly click Next 5 times in 2 seconds
#    DevTools Network → count PATCH requests → must be ≤ 2
# 4. Verify DB:
psql $DATABASE_URL -c "SELECT last_page, status FROM papers LIMIT 1;"
#    Expect: last_page=7, status='in_progress'
```

**Tasks:**
- Add to `app/routers/papers.py`: `PATCH /papers/{paper_id}/position` — `Depends(get_current_user)` → `SELECT paper WHERE id = ? AND user_id = ?` (404 if not found, 403 if user mismatch) → `UPDATE papers SET last_page = :page, status = CASE WHEN status = 'unread' AND :page > 1 THEN 'in_progress' ELSE status END WHERE id = :paper_id`
- Create `app/schemas/paper.py` additions — `PositionUpdate(last_page: int = Field(ge=1))`, `PositionResponse(last_page: int)`
- Update `src/app/(app)/reader/[paperId]/page.tsx` — pass `last_page` from query result as `initialPage` to `<PDFViewer>`; wire `onPageChange` prop to a debounced `PATCH /papers/{paperId}/position` call using `useCallback` + `useRef` cancel pattern (300ms debounce — no lodash dependency, plain `setTimeout`/`clearTimeout`)
- Add `tests/test_papers.py` — `test_position_update_persists`, `test_position_update_forbidden_other_user`, `test_status_transitions_to_in_progress`

**Estimated time:** 2 hours
**Depends on:** E4-S2

---

### E4-S4: Wire Library Page to Real GET /papers API

As a user,
I want my uploaded papers to appear in the library with their real title, upload date, and reading status,
So that I can see and manage all my papers in one place.

**What to build:** Add `GET /papers` to the API (returns the authenticated user's papers ordered by `created_at DESC`, with a `has_note` flag derived from `smart_notes`). Replace `mockPapers` in the library page with a real React Query `useQuery`. Add loading skeleton cards and an error state with retry. Wire `PaperCard` click to navigate to `/reader/[paperId]`. After a successful upload, invalidate the papers query so the new paper appears without a manual refresh.

**Acceptance Criteria:**

**Given** the user has uploaded at least one paper
**When** they navigate to `/library`
**Then** their uploaded paper appears with the correct `title`, `upload_date` (formatted as e.g. "Jun 6, 2026"), `status` badge (Unread/In Progress/Finished), and `has_note` indicator

**When** the user uploads a new paper via `UploadDialog`
**Then** the library list refreshes automatically without a page reload and the new paper appears at the top

**When** the library is fetching data
**Then** skeleton placeholder cards are shown — not a blank page or a full-page spinner

**When** the API returns a network error
**Then** an error message appears with a "Try again" button that re-fires the query

**When** the user clicks a paper card
**Then** browser navigates to `/reader/<paper_id>`

**Given** a user with zero uploaded papers
**When** they navigate to `/library`
**Then** `<LibraryEmptyState>` renders — driven by an empty API response array, not the mock

**Local verification:**
```bash
docker compose up
# 1. Upload 2 papers via UploadDialog
#    Library must show both immediately (no manual refresh)
# 2. Click a paper → must navigate to /reader/<id>
# 3. Log out → create new account → library must show empty state
# 4. Throttle network in DevTools → loading skeletons must appear before data
# 5. Kill the API container mid-load → error state with "Try again" must appear
# DevTools Network: GET /papers called on library load, visible in XHR tab
```

**Tasks:**
- Add to `app/routers/papers.py`: `GET /papers` — `Depends(get_current_user)` → `SELECT papers.*, (SELECT COUNT(*) > 0 FROM smart_notes WHERE paper_id = papers.id AND user_id = papers.user_id) AS has_note FROM papers WHERE user_id = ? ORDER BY created_at DESC`; return `{"data": [...], "meta": {"total": N, "request_id": "..."}}`
- Create `app/schemas/paper.py` — `PaperListItem(id, title, upload_date: datetime, status, has_note: bool)` Pydantic response model
- Update `src/app/(app)/library/page.tsx` — replace `mockPapers` with `useQuery({ queryKey: ['papers'], queryFn: () => fetchApi('/papers') })`; `isLoading` → render 3× `<PaperCardSkeleton>`; `isError` → `<ErrorState onRetry={() => refetch()} />`; `data?.data?.length === 0` → `<LibraryEmptyState>`; else → `data.data.map(p => <PaperCard key={p.id} paper={p} />)`
- Create `src/components/library/PaperCardSkeleton.tsx` — shadcn `<Skeleton>` blocks matching `PaperCard` height/layout
- Create `src/components/library/ErrorState.tsx` — error message + `<Button onClick={onRetry}>Try again</Button>`
- Update `src/components/library/PaperCard.tsx` — add `onClick={() => router.push(`/reader/${paper.id}`)}` on the card; format `upload_date` with `Intl.DateTimeFormat`
- Update `src/components/library/UploadDialog.tsx` — after successful upload mutation: `queryClient.invalidateQueries({ queryKey: ['papers'] })`
- Delete `src/lib/mock-data.ts` (no longer needed)

**Estimated time:** 2 hours
**Depends on:** E4-S3

---

## Epic 5: Explain Feature — SSE Streaming Popup & Save

Users can select text in a paper, see a bilingual explanation stream progressively in a popup, and optionally save it. Done when the full explain + save flow works end-to-end with quota enforcement.

**FRs covered:** FR-020, FR-021, FR-022, FR-023, FR-024, FR-025, FR-025a, FR-026, FR-050a, FR-051, FR-052, FR-053

---

### E5-S1: Implement QuotaService with check_and_increment() and get_quota_status()

As a developer,
I want a tested `QuotaService` that atomically checks and increments quota before any Bedrock call,
So that the quota boundary is enforced at the architectural layer, not as an afterthought on individual endpoints.

**What to build:** Create `app/services/quota_service.py` with two methods. `check_and_increment(user_id, quota_type)` checks whether the user has quota remaining and atomically increments the counter — using a database-level upsert to prevent race conditions between concurrent requests. If the limit is exceeded, it raises `QuotaExceededException` with reset time. `get_quota_status(user_id)` returns current usage, limits, and percentage for both quota types. The `QuotaType` enum covers `EXPLAIN` (daily, resets UTC 00:00) and `NOTE` (monthly, resets on account creation anniversary).

**Acceptance Criteria:**

**Given** a user with `explain_count = 0` for today
**When** `quota_service.check_and_increment(user_id, QuotaType.EXPLAIN)` is called
**Then** no exception is raised and `SELECT explain_count FROM quota_daily WHERE user_id=? AND date=today` returns `1`

**When** called 50 times cumulatively for the same user on the same day
**Then** the 50th call succeeds — `explain_count` reaches `50`

**When** called a 51st time
**Then** `QuotaExceededException` is raised with `{"limit": 50, "reset_at": "<next UTC midnight ISO8601>"}`
**And** `explain_count` remains at `50` — the failed call does not increment

**Given** two concurrent requests call `check_and_increment` simultaneously for the same user
**When** both complete
**Then** `explain_count` increments by exactly 2 — no double-count, no lost update (atomic DB upsert, not read-then-write)

**When** `get_quota_status(user_id)` is called after 40 explain calls
**Then** returns `{"explain_daily": {"used": 40, "limit": 50, "percent": 80.0, "resets_at": "<ISO8601>"}, "note_monthly": {"used": 0, "limit": 5, "percent": 0.0, "resets_at": "<ISO8601>"}}`

**When** `QuotaExceededException` is raised inside a FastAPI route
**Then** the global exception handler maps it to `HTTP 429` with `{"error": {"code": "QUOTA_EXCEEDED", "message": "...", "detail": {"limit": 50, "resets_at": "..."}}, "meta": {...}}`

**Local verification (no full stack needed — docker compose postgres only):**
```bash
docker compose up -d postgres
pytest tests/test_quota_service.py -v
# All 4 tests must pass:
# test_check_allows_under_limit
# test_check_blocks_at_limit_and_does_not_increment
# test_check_increments_atomically (two concurrent calls → count +2)
# test_get_quota_status_returns_correct_percentages
```

**Tasks:**
- Create `app/services/quota_service.py`:
  - `class QuotaType(str, Enum): EXPLAIN = "explain"; NOTE = "note"`
  - `class QuotaExceededException(Exception): limit, reset_at`
  - `check_and_increment(user_id, quota_type, db)` — for EXPLAIN: `INSERT INTO quota_daily(user_id, date, explain_count) VALUES(?, today, 1) ON CONFLICT(user_id, date) DO UPDATE SET explain_count = quota_daily.explain_count + 1 RETURNING explain_count`; if returned value > 50: raise `QuotaExceededException(limit=50, reset_at=next_utc_midnight())`; NOTE: same pattern with `quota_monthly` + limit 5 + anniversary reset
  - `get_quota_status(user_id, db) -> QuotaStatus`: SELECT both quota tables, calculate percent, calculate reset times
- Register `QuotaExceededException` handler in `app/main.py` → HTTP 429 `QUOTA_EXCEEDED` envelope
- Create `tests/test_quota_service.py` — all 4 tests; concurrent test uses `asyncio.gather` with two coroutines calling `check_and_increment`

**Estimated time:** 1.5 hours
**Depends on:** E4-S1 (needs the `papers` router pattern; quota tables exist from E2-S2)

---

### E5-S2: Author and Validate Explain System Prompt Against Real Paper Excerpts

As a developer,
I want `prompts/explain_system.txt` to produce explanations that a Vietnamese reader can understand without opening another browser tab,
So that the core value of the product — bilingual in-context understanding — is encoded in source-controlled, iterable production code.

**What to build:** Replace the placeholder in `app/services/prompts/explain_system.txt` with a production-quality system prompt. The prompt must instruct the model to produce three structured output sections: `vi_complete` (Vietnamese explanation, no unexplained jargon), `en_context` (English technical domain context, not a dictionary definition), and `domain_tag` (Security|DevOps|General inferred from content). Quality is validated by running the prompt against 3 real paper excerpts using a local test script — not mocked, not synthetic. The prompt is iterated until all 3 excerpts pass the quality bar defined in FR-023.

**Acceptance Criteria:**

**Given** the prompt in `explain_system.txt` and the term "certificate transparency log" from a TLS 1.3 paper
**When** developer calls `python scripts/test_prompt.py "certificate transparency log" Security`
**Then** the `vi_complete` output is in Vietnamese (contains Vietnamese diacritical characters), explains the concept in plain terms understandable without opening Wikipedia, and does not simply transliterate "certificate transparency log" phonetically

**Given** the term "OCSP stapling" (PKI/certificate domain)
**When** `python scripts/test_prompt.py "OCSP stapling" Security` runs
**Then** the `en_context` output contains domain-specific information: what OCSP stapling does, why it exists (performance/privacy trade-off vs. standard OCSP), and where it appears in the TLS handshake — not a generic definition of "stapling"

**Given** the term "eBPF program" (kernel/observability domain)
**When** `python scripts/test_prompt.py "eBPF program" DevOps` runs
**Then** `domain_tag` is `"DevOps"` (not `"Security"` or `"General"`)
**And** `vi_complete` references the term in the context of Linux kernel tracing or observability, not as a generic program

**Given** the final committed prompt
**When** developer inspects `explain_system.txt`
**Then** the prompt uses `{selected_text}` and `{domain_hint}` as placeholders (`.format()` compatible)
**And** the prompt specifies the output section delimiter format so `ai_service.py` can parse `vi_complete`, `en_context`, and `domain_tag` sections
**And** no model IDs, API keys, or hardcoded credentials appear in the file

**The `scripts/test_prompt.py` file is committed and documented** but is NOT added to CI — it requires live Bedrock credentials and is a developer tool, not an automated test.

**Local verification (requires AWS credentials for Bedrock — not in docker compose):**
```bash
export AWS_ACCESS_KEY_ID=... AWS_SECRET_ACCESS_KEY=...
python scripts/test_prompt.py "certificate transparency log" Security
# Inspect output: vi_complete must be understandable Vietnamese, not garbled
python scripts/test_prompt.py "OCSP stapling" Security
# Inspect en_context: must mention TLS handshake or certificate verification, not generic
python scripts/test_prompt.py "eBPF program" DevOps
# Inspect domain_tag: must be "DevOps"
```

**Tasks:**
- Write `app/services/prompts/explain_system.txt` — system prompt with:
  - Role framing: expert technical translator for Vietnamese InfoSec/DevOps learners with intermediate English
  - Task: produce exactly 3 sections: `[VI_START]...[VI_END]`, `[EN_START]...[EN_END]`, `[DOMAIN: Security|DevOps|General]`
  - Quality instructions: `vi_complete` must be self-contained, use Vietnamese grammatical structure (not English word order), no unexplained English jargon; `en_context` must be domain-specific (1–2 sentences on usage in relevant technical domain, not a general definition)
  - Variable slots: `Selected text: {selected_text}` and `Domain context: {domain_hint}`
- Create `scripts/test_prompt.py` — reads `explain_system.txt`, substitutes variables, calls `boto3.bedrock_runtime.converse(Haiku, messages=[...], system=[...])`, prints raw output; accepts `selected_text` and `domain_hint` as CLI args
- Iterate prompt in `explain_system.txt` until all 3 excerpts pass; each iteration is a git commit with message like `prompt: improve vi_complete for technical jargon`

**Estimated time:** 2 hours
**Depends on:** E5-S1

---

### E5-S3: Implement SSE Streaming Endpoint for Bilingual Explain

As a developer,
I want `POST /explain/stream` to return a progressive SSE stream of bilingual explanation events,
So that the frontend can render the Vietnamese explanation token-by-token as it arrives from Bedrock.

**What to build:** Implement `ai_service.explain_stream()` (replacing the `NotImplementedError` stub from E2-S1) and `POST /explain/stream`. Flow: quota check (first, before everything else) → validate request → call `ai_service.explain_stream()` which calls `boto3.converse_stream(Haiku)` → yield SSE events per Pattern 1.3. All tests use the `mock_ai_service` fixture — no live Bedrock calls in CI.

**Acceptance Criteria:**

**Given** a valid token and quota remaining
**When** `POST /explain/stream` is called with `{"selected_text": "certificate transparency log", "paper_id": "<uuid>", "domain_hint": "Security"}`
**Then** response status is `200` with `Content-Type: text/event-stream`
**And** response header `X-Accel-Buffering: no` is present
**And** response header `Cache-Control: no-cache` is present

**When** consuming the SSE stream to completion
**Then** the following event sequence is received (order of token events varies; section events appear after their tokens):
- One or more `data: {"type": "token", "content": "..."}` events
- `data: {"type": "section", "section": "vi_complete", "content": "<full Vietnamese text>"}` event
- `data: {"type": "section", "section": "en_context", "content": "<full English context>"}` event
- `data: {"type": "section", "section": "domain_tag", "content": "Security"|"DevOps"|"General"}` event
- `data: {"type": "done"}` as the final event

**Given** the user's daily explain quota is exhausted
**When** `POST /explain/stream` is called
**Then** response is `429` with `{"error": {"code": "QUOTA_EXCEEDED", ...}, "meta": {...}}` — the stream never opens, no SSE events emitted

**Given** `boto3.converse_stream` raises an exception mid-stream (mocked)
**When** the stream is being consumed
**Then** a `data: {"type": "error", "code": "BEDROCK_ERROR", "message": "Explanation service temporarily unavailable."}` event is emitted
**And** the stream closes — no hanging connection, no spinner-forever on client

**Given** `ai_service.explain_stream()` is called
**When** developer greps for boto3 in `app/routers/explain.py`
**Then** no boto3 import is found — only `ai_service.explain_stream()` is called from the router

**Local verification (mocked Bedrock — no AWS credentials needed):**
```bash
docker compose up -d
TOKEN=$(...)
curl -N -s http://localhost:8000/explain/stream \
  -H "Authorization: Bearer $TOKEN" \
  -H "Content-Type: application/json" \
  -d '{"selected_text":"TLS handshake","paper_id":"<uuid>","domain_hint":"Security"}'
# Expect: stream of data: {...} lines ending with data: {"type":"done"}
pytest tests/test_explain.py -v
# All 3 tests must pass using mock_ai_service fixture
```

**Tasks:**
- Implement `AIService.explain_stream(selected_text, domain_hint) -> AsyncGenerator[SSEEvent, None]` in `app/services/ai_service.py`:
  - Calls `self._client.converse_stream(modelId=settings.BEDROCK_EXPLAIN_MODEL, messages=[{role: user, content: prompt.format(...)}], system=[{text: explain_system_prompt}])`
  - Parses Bedrock stream `"contentBlockDelta"` events → yields `SSEEvent(type="token", content=chunk_text)`
  - Detects section delimiters (`[VI_START]`/`[VI_END]`, `[EN_START]`/`[EN_END]`, `[DOMAIN: ...]`) → yields `SSEEvent(type="section", section=..., content=...)`
  - On `boto3` exception → yields `SSEEvent(type="error", code="BEDROCK_ERROR", message="...")`, returns
  - Final yield: `SSEEvent(type="done")`
- Create `app/routers/explain.py`:
  - `POST /explain/stream`: `Depends(get_current_user)` → `quota_service.check_and_increment(user_id, QuotaType.EXPLAIN, db)` (raises 429 if over limit) → `return StreamingResponse(stream_generator(...), media_type="text/event-stream", headers={"X-Accel-Buffering": "no", "Cache-Control": "no-cache"})`
  - `async def stream_generator(text, domain, ai_svc)`: `async for event in ai_svc.explain_stream(text, domain): yield f"data: {event.model_dump_json()}\n\n"`
- Create `app/schemas/explain.py` — `ExplainRequest(selected_text: str = Field(min_length=1, max_length=2000), paper_id: UUID, domain_hint: str = "General")`
- Wire `explain` router into `app/main.py`
- Update `tests/conftest.py` — `mock_ai_service` fixture: returns async generator of `[SSEEvent(type="section", section="vi_complete", content="giải thích test"), SSEEvent(type="section", section="en_context", content="English test context"), SSEEvent(type="section", section="domain_tag", content="Security"), SSEEvent(type="done")]`
- Create `tests/test_explain.py` — `test_explain_stream_returns_events`, `test_explain_stream_quota_exceeded_returns_429`, `test_explain_stream_bedrock_error_emits_error_event`

**Estimated time:** 2.5 hours
**Depends on:** E5-S2

---

### E5-S4: Build ExplainPopup Component with SSE Stream Consumption and All Edge Cases

As a user reading a paper,
I want to select a technical term and see a bilingual explanation stream progressively into a popup without leaving the reading view,
So that I understand the term in context without losing my place in the paper.

**What to build:** Create `useExplainStream()` hook (fetch + `ReadableStream`, not `EventSource` — POST body requires fetch) and `ExplainPopup.tsx`. The popup is positioned relative to the text selection using `getBoundingClientRect()` with viewport-edge detection. All five edge cases from the story rules have explicit AC.

**Acceptance Criteria:**

**AC-1 — Normal stream completion:**
**Given** the user selects "certificate transparency log" and clicks "Explain"
**When** the stream runs to completion
**Then** Vietnamese text appears progressively (token by token, not all at once)
**And** English context appears after `vi_complete`
**And** the domain tag badge is shown ("Security")
**And** the "Save explanation" button becomes enabled only after `{"type": "done"}` is received

**AC-2 — Selection cleared before stream finishes:**
**Given** a stream is running (Vietnamese tokens appearing)
**When** the user clicks elsewhere on the page to clear their text selection
**Then** `AbortController.abort()` is called on the stream, the popup closes cleanly
**And** no error is thrown, no console error appears, no zombie fetch continues in the background

**AC-3 — New selection while stream is running:**
**Given** a stream is running for term A
**When** the user selects term B and clicks "Explain"
**Then** the stream for term A is aborted first
**And** the popup resets to loading state
**And** a new stream for term B starts

**AC-4 — Bedrock error mid-stream:**
**Given** the SSE stream emits `{"type": "error", "code": "BEDROCK_ERROR", "message": "..."}`
**When** the frontend receives this event
**Then** the popup shows "Explanation failed. Try again." with a retry button — not a spinner, not a blank state, not a generic browser error

**AC-5 — Selection near viewport bottom:**
**Given** the user selects text on the last visible line of the PDF (selection `bottom > window.innerHeight - 200`)
**When** the popup renders
**Then** the popup appears above the selection anchor, not below it
**And** the popup is not cut off by the viewport edge

**Local verification:**
```bash
docker compose up
# AC-1: Select "certificate transparency log" → Explain
#   Expect: tokens appear one by one in popup
# AC-2: Start a stream → immediately click elsewhere
#   DevTools Network: fetch to /explain/stream shows "cancelled"
# AC-3: Select term A → start streaming → select term B → Explain
#   Expect: popup resets, new stream appears
# AC-4: Temporarily modify mock to return error event
#   Expect: "Explanation failed. Try again." shown
# AC-5: Scroll to near bottom of a PDF page → select last line of text → Explain
#   Expect: popup renders above selection, fully visible
```

**Tasks:**
- Create `src/hooks/use-explain-stream.ts`:
  - State: `{ vi: string, en: string, domainTag: string, isStreaming: boolean, error: string | null }`
  - `explain(selectedText, paperId, domainHint)`: abort any running stream → create new `AbortController` → `fetch(POST /explain/stream, { signal: controller.signal, body: ... })` → `response.body.getReader()` → `TextDecoder` loop → split chunks on `\n\n` → parse `data: {...}` JSON → dispatch to state by event type
  - `abort()`: calls `controller.abort()`, resets state to idle
  - On `error.name === 'AbortError'`: silently reset state (not an error)
  - On SSE `type === "error"`: set `state.error = event.message`, set `isStreaming = false`
  - On `type === "done"`: set `isStreaming = false`
- Create `src/components/reader/ExplainPopup.tsx`:
  - Positioned with `style={{ position: 'fixed', left: rect.left, top: rect.bottom + 8 }}` (default below)
  - Viewport flip: `if (rect.bottom + popupHeight > window.innerHeight - 20) top = rect.top - popupHeight - 8`
  - Renders: loading skeleton (while `isStreaming && !vi`) → progressive `vi` text as it grows → `en` section → `domainTag` badge → "Save explanation" `<Button disabled={isStreaming}`
  - On AC-4 (`error` state): error message + `<Button onClick={() => explain(...)}>Try again</Button>`
  - Closes on: ESC keydown, `mousedown` outside popup (via `useEffect` document listener), selection cleared (detected via `selectionchange` event)
- Wire text selection in `PDFViewer.tsx`:
  - `document.addEventListener('mouseup', handleMouseUp)` — if `window.getSelection().toString().trim().length > 2`: show "Explain" button floated at selection coords; on button click: `explainHook.explain(selectedText, paperId, domainHint)`
  - `document.addEventListener('selectionchange', handleSelectionChange)` — if `selection.toString() === ''` and streaming: `explainHook.abort()`

**Estimated time:** 2.5 hours
**Depends on:** E5-S3

---

### E5-S5: Implement Save Explanation with Text Position Anchor and Underline Indicator

As a user,
I want to save an explanation and see a subtle underline on that passage when I return to the paper,
So that I can find and revisit important explanations without re-triggering the AI popup.

**What to build:** Add `POST /papers/{id}/saved-explanations` and `GET /papers/{id}/saved-explanations` endpoints. The "Save explanation" button in `ExplainPopup` calls the POST endpoint after streaming completes. On success, the button shows "Saved ✓". On reader load, saved explanations are fetched and text passages with saved explanations receive a CSS underline indicator in the PDF text layer.

**Acceptance Criteria:**

**Given** a stream has completed and the "Save explanation" button is enabled
**When** the user clicks "Save explanation"
**Then** `POST /papers/{paper_id}/saved-explanations` is called with `{ text_excerpt, text_position_json: { page, char_offset }, explanation_vi, explanation_en, domain_tag }`
**And** response is `201` with `{"data": {"saved_explanation_id": "<uuid>"}, "meta": {...}}`
**And** the button changes to "Saved ✓" (disabled, non-clickable — no duplicate save on re-click)

**When** developer queries `SELECT text_excerpt, explanation_vi, domain_tag FROM saved_explanations WHERE paper_id = '<id>'`
**Then** all three columns are non-empty for the saved row

**Given** the user has saved an explanation and reopens the paper
**When** the reader page loads and fetches `GET /papers/{paper_id}/saved-explanations`
**Then** the text passage is marked with a visible underline indicator in the PDF text layer (CSS `border-bottom: 1px solid currentColor; opacity: 0.6` or equivalent)

**Given** a user saves an explanation for the same text excerpt twice
**Then** both `201` responses succeed — no unique constraint blocks duplicate saves

**When** developer calls `POST /papers/<other_user_paper_id>/saved-explanations` with a different user's token
**Then** response is `403 FORBIDDEN` with `{"error": {"code": "FORBIDDEN", ...}, "meta": {...}}`

**Local verification:**
```bash
docker compose up
# 1. Stream explanation → wait for completion → click "Save explanation"
#    Button must show "Saved ✓"
# 2. Check DB:
psql $DATABASE_URL -c "SELECT text_excerpt, domain_tag FROM saved_explanations LIMIT 1;"
# 3. Close reader → reopen /reader/<id>
#    Saved text must show underline in PDF text layer
curl -s http://localhost:8000/papers/<id>/saved-explanations \
  -H "Authorization: Bearer $TOKEN" | jq '.data | length'
# Must return 1
```

**Tasks:**
- Add to `app/routers/papers.py`:
  - `POST /papers/{paper_id}/saved-explanations` — verify `paper.user_id == current_user.id` (403) → `INSERT saved_explanations(paper_id, user_id, text_excerpt, text_position_json, explanation_vi, explanation_en, domain_tag)` → return 201 `{ saved_explanation_id }`
  - `GET /papers/{paper_id}/saved-explanations` — verify ownership → `SELECT * FROM saved_explanations WHERE paper_id=? ORDER BY created_at` → return list
- Create `app/schemas/explanation.py` — `SaveExplanationRequest(text_excerpt: str, text_position_json: dict, explanation_vi: str, explanation_en: str, domain_tag: str)`, `SavedExplanationItem`
- Update `src/components/reader/ExplainPopup.tsx`:
  - "Save explanation" `<Button>` calls `fetchApi(POST /papers/{paperId}/saved-explanations, { body: { text_excerpt, text_position_json: { page: currentPage, char_offset: selectionOffset }, explanation_vi: state.vi, explanation_en: state.en, domain_tag: state.domainTag } })`
  - On 201: button text → "Saved ✓", `disabled={true}`, clear loading state; on error: show inline error
- Update `src/app/(app)/reader/[paperId]/page.tsx`:
  - `useQuery(['saved-explanations', paperId], () => fetchApi(`/papers/${paperId}/saved-explanations`))` — on load
  - Pass `savedExplanations` to `PDFViewer`
- Update `src/components/reader/PDFViewer.tsx`:
  - After text layer renders per page: for each `savedExplanation` where `text_position_json.page === currentPage`, find text layer `<span>` elements containing `text_excerpt` → add CSS class `border-b border-primary/60`

**Estimated time:** 2 hours
**Depends on:** E5-S4

---

### E5-S6: Build Quota Warning Banner and Explain Hard Block UI

As a user,
I want to see a warning when I'm approaching my explain limit and a clear message when I've hit it,
So that I understand why the feature is unavailable and when it will reset.

**What to build:** Add `GET /quota/status` endpoint (returns current usage for both quota types). Build `QuotaWarningBanner.tsx` shown in the reader when daily explain usage is ≥80%. When the quota is fully exhausted (100%), the "Explain" trigger is disabled and a block message replaces the popup. Quota status is fetched on reader page load and after each explain call.

**Acceptance Criteria:**

**Given** the user has used 40 of 50 daily explain calls (80%)
**When** they are in the reader
**Then** `QuotaWarningBanner` is visible with text: "You've used 40/50 daily explanations. Resets at midnight UTC."
**And** the banner is non-blocking — the user can still trigger explains

**Given** the user has used all 50 daily explain calls
**When** they select text and attempt to trigger "Explain"
**Then** the "Explain" button is visually disabled (not clickable, cursor not-allowed)
**And** the text "Daily explain limit reached. Resets at midnight UTC." is shown inline near the selection — no popup opens, no API call is made

**When** the user's usage is at 79%
**Then** `QuotaWarningBanner` is NOT rendered (threshold is exactly 80%, not 79%)

**When** `GET /quota/status` is called with a valid auth token
**Then** response is `200` with `{"data": {"explain_daily": {"used": N, "limit": 50, "percent": P, "resets_at": "<ISO8601>"}, "note_monthly": {"used": M, "limit": 5, "percent": Q, "resets_at": "<ISO8601>"}}, "meta": {...}}`

**Given** the user completes an explain call
**When** the call resolves
**Then** `queryClient.invalidateQueries(['quota'])` is triggered and the banner re-evaluates (count updated without page refresh)

**Local verification:**
```bash
docker compose up
# Set explain_count to 40:
psql $DATABASE_URL -c "UPDATE quota_daily SET explain_count=40 WHERE user_id='<id>' AND date=CURRENT_DATE;"
# Navigate to /reader/<id> → QuotaWarningBanner must appear
# Set to 50:
psql $DATABASE_URL -c "UPDATE quota_daily SET explain_count=50 WHERE user_id='<id>' AND date=CURRENT_DATE;"
# Select text → Explain button must be disabled; block message must appear
curl -s http://localhost:8000/quota/status \
  -H "Authorization: Bearer $TOKEN" | jq .
# Expect: 200 with explain_daily and note_monthly
```

**Tasks:**
- Create `app/routers/quota.py` — `GET /quota/status`: `Depends(get_current_user)` → `quota_service.get_quota_status(user_id, db)` → return quota status for both types in `{ data, meta }` envelope
- Wire `quota` router into `app/main.py`
- Create `src/components/reader/QuotaWarningBanner.tsx` — renders only when `quotaStatus.explain_daily.percent >= 80 && percent < 100`; amber warning bar: "You've used {used}/{limit} daily explanations. Resets at midnight UTC."; `percent >= 100` branch: render red banner "Daily explain limit reached. Resets at midnight UTC." (no dismiss, persistent)
- Update `src/app/(app)/reader/[paperId]/page.tsx` — `useQuery(['quota'], () => fetchApi('/quota/status'))` on page load; render `<QuotaWarningBanner quotaStatus={quotaStatus} />`; after each `explainMutation.onSuccess`: `queryClient.invalidateQueries(['quota'])`
- Update `ExplainPopup` trigger in `PDFViewer.tsx` — check `quotaStatus?.explain_daily.percent >= 100` before allowing "Explain": if exhausted → show inline `<span className="text-destructive text-xs">Daily limit reached. Resets at midnight UTC.</span>` instead of opening popup; no fetch fired

**Estimated time:** 1.5 hours
**Depends on:** E5-S5

---

## Epic 6: Smart Note Feature — Async Generation, Polling & Editor

Users can generate a structured Smart Note for any paper, view the AI-generated What/Why/How/Key Terms output, and write their own notes alongside it. Done when the user confirms smart note generation and sees What/Why/How/Key Terms with an editable My Notes section that auto-saves.

**FRs covered:** FR-030, FR-031, FR-032, FR-033, FR-034, FR-035, FR-036, FR-050b

---

### E6-S1: Author and Validate Note Generation System Prompt Against a Full Paper

As a developer,
I want `prompts/note_system.txt` to produce a structured Smart Note that passes a human quality bar on a real paper,
So that the AI output is source-controlled, iterable, and proven useful before the async pipeline is built around it.

**What to build:** Replace the placeholder in `app/services/prompts/note_system.txt` with a production-quality system prompt. The prompt instructs the model to produce a JSON object with four keys: `what` (what the paper establishes), `why` (why it matters for practitioners), `how` (how the technique/approach works), and `key_terms` (array of `{term, definition_vi, context_en}`). Quality is validated against a full paper's extracted text using a local test script. The prompt is iterated until the output passes the quality bar for all four sections.

**Acceptance Criteria:**

**Given** the prompt in `note_system.txt` and the full extracted text of the uploaded `sample.pdf` (from `tests/fixtures/`)
**When** developer runs `python scripts/test_note_prompt.py tests/fixtures/sample.pdf`
**Then** the output is valid JSON parseable without errors

**When** developer inspects the `what` key
**Then** it contains 2–4 sentences describing the paper's core thesis or finding, not a generic restatement of the abstract

**When** developer inspects the `why` key
**Then** it contains 1–3 sentences explaining practical relevance for an InfoSec/DevOps practitioner — not "this paper is important"

**When** developer inspects the `how` key
**Then** it contains a description of the method, technique, or approach — not a copy-paste of the `what` section

**When** developer inspects the `key_terms` array
**Then** it contains 3–10 items, each with non-empty `term`, `definition_vi` (Vietnamese, with diacritical characters), and `context_en` (one sentence of domain-specific usage context)

**Given** the final committed prompt
**When** developer inspects `note_system.txt`
**Then** the prompt uses `{paper_text}` as the only placeholder
**And** it specifies that output must be a single valid JSON object (no prose wrapping, no markdown fences)
**And** no model IDs, API keys, or hardcoded credentials appear in the file

**The `scripts/test_note_prompt.py` file is committed** but NOT added to CI — requires live Bedrock credentials.

**Local verification (requires AWS credentials for Bedrock):**
```bash
export AWS_ACCESS_KEY_ID=... AWS_SECRET_ACCESS_KEY=...
python scripts/test_note_prompt.py tests/fixtures/sample.pdf
# Inspect output: must be valid JSON, all 4 keys present
# Inspect key_terms: definition_vi must contain Vietnamese diacriticals (ă, ơ, ề, etc.)
# Inspect what/why/how: must be specific to the paper content, not generic
```

**Tasks:**
- Write `app/services/prompts/note_system.txt` — system prompt with:
  - Role framing: expert technical summarizer for Vietnamese InfoSec/DevOps learners
  - Task: produce a single JSON object with keys `what`, `why`, `how`, `key_terms`; `key_terms` is an array of `{term: string, definition_vi: string, context_en: string}`
  - Constraints: output must be raw JSON only — no markdown, no prose before/after; `definition_vi` must be Vietnamese (not transliteration); `key_terms` must contain 3–10 items from the paper's actual vocabulary
  - Variable slot: `Paper text: {paper_text}`
- Create `scripts/test_note_prompt.py` — reads `note_system.txt`, reads PDF text from provided file path via `app/services/pdf_service.extract_text()`, calls `boto3.bedrock_runtime.converse(Sonnet, ...)`, prints raw JSON output; accepts PDF path as CLI arg
- Iterate prompt in `note_system.txt` until all quality ACs pass; each iteration is a git commit with message like `prompt: fix key_terms definition_vi quality`

**Estimated time:** 2 hours
**Depends on:** E5-S2 (explain prompt story establishes the scripts/ pattern and Bedrock call structure)

---

### E6-S2: Implement Async Note Generation Endpoint and BackgroundTask

As a user,
I want to trigger smart note generation for a paper and have it run in the background while I keep reading,
So that I am not blocked for the 15–30 seconds it takes Claude Sonnet to produce the structured note.

**What to build:** Implement `AIService.generate_smart_note()` (replacing the `NotImplementedError` stub from E2-S1). Add `POST /papers/{id}/note/generate` which creates a `note_jobs` record and schedules a `BackgroundTask` to call Bedrock (Sonnet) and write the result back to `smart_notes`. The job lifecycle is: `pending` → `running` → `done` or `failed`. Tests use a mocked `ai_service` fixture.

**Acceptance Criteria:**

**Given** a valid token, a paper with non-empty `text_content`, and `note_monthly` quota remaining
**When** `POST /papers/{paper_id}/note/generate` is called
**Then** response is `202` with `{"data": {"job_id": "<uuid>", "status": "pending"}, "meta": {...}}`
**And** `SELECT status FROM note_jobs WHERE id = '<job_id>'` returns `pending`

**Given** the BackgroundTask runs to completion (mocked `ai_service` returns valid JSON)
**When** developer queries DB 1 second after the 202
**Then** `SELECT status FROM note_jobs WHERE id = '<job_id>'` returns `done`
**And** `SELECT what, why, how FROM smart_notes WHERE paper_id = '<paper_id>'` returns non-empty values
**And** `SELECT COUNT(*) FROM key_terms WHERE smart_note_id = '<note_id>'` is ≥ 3

**Given** the user's monthly note quota is exhausted (5 notes used)
**When** `POST /papers/{paper_id}/note/generate` is called
**Then** response is `429` with `{"error": {"code": "QUOTA_EXCEEDED", ...}, "meta": {...}}` — no job created

**Given** the paper has `text_content = NULL` (extraction still running)
**When** `POST /papers/{paper_id}/note/generate` is called
**Then** response is `422` with `{"error": {"code": "VALIDATION_ERROR", "message": "Paper text extraction is still in progress. Please wait and retry."}, "meta": {...}}`

**Given** `boto3.converse` raises an exception inside the BackgroundTask
**When** the task completes
**Then** `SELECT status FROM note_jobs WHERE id = '<job_id>'` returns `failed`
**And** no `smart_notes` row is inserted for this job

**Given** a job already exists for this paper with status `pending` or `running`
**When** `POST /papers/{paper_id}/note/generate` is called again
**Then** response is `409` with `{"error": {"code": "VALIDATION_ERROR", "message": "Note generation already in progress."}, "meta": {...}}`

**Local verification (mocked Bedrock):**
```bash
docker compose up -d
TOKEN=$(...)
curl -s -X POST http://localhost:8000/papers/<paper_id>/note/generate \
  -H "Authorization: Bearer $TOKEN" | jq .
# Expect: 202 with job_id
sleep 2
psql $DATABASE_URL -c "SELECT status FROM note_jobs WHERE paper_id='<paper_id>';"
# Expect: done
psql $DATABASE_URL -c "SELECT what IS NOT NULL FROM smart_notes WHERE paper_id='<paper_id>';"
# Expect: t
pytest tests/test_note_generation.py -v
```

**Tasks:**
- Implement `AIService.generate_smart_note(paper_text: str) -> dict` in `app/services/ai_service.py`:
  - Reads `note_system.txt`, formats `prompt.format(paper_text=paper_text[:15000])` (truncate to avoid token limits)
  - Calls `self._client.converse(modelId=settings.BEDROCK_NOTE_MODEL, messages=[...], system=[...])`
  - Parses response text as JSON; returns dict with keys `what`, `why`, `how`, `key_terms`
  - Raises `ValueError` if JSON invalid or required keys missing
- Create `app/tasks/generate_note.py` — `async def generate_note_task(job_id, paper_id, user_id, db, ai_service)`:
  - `UPDATE note_jobs SET status='running' WHERE id=job_id`
  - `paper_text = SELECT text_content FROM papers WHERE id=paper_id`
  - Call `ai_service.generate_smart_note(paper_text)` → on exception: `UPDATE note_jobs SET status='failed'`; return
  - `INSERT smart_notes(paper_id, user_id, what, why, how)` → get `note_id`
  - `INSERT key_terms(smart_note_id, term, definition_vi, context_en)` for each item in `result['key_terms']` (bulk insert)
  - `UPDATE note_jobs SET status='done', smart_note_id=note_id`
- Add to `app/routers/papers.py`: `POST /papers/{paper_id}/note/generate`:
  - Verify paper ownership (403) → check `text_content IS NOT NULL` (422 if null) → `quota_service.check_and_increment(NOTE)` (429 if exceeded)
  - Check for in-flight job: `SELECT id FROM note_jobs WHERE paper_id=? AND status IN ('pending','running')` → if found: 409
  - `INSERT note_jobs(paper_id, user_id, status='pending')` → `BackgroundTasks.add_task(generate_note_task, ...)` → return 202 `{job_id, status}`
- Create `tests/test_note_generation.py` — `test_generate_returns_202`, `test_generate_quota_exceeded_returns_429`, `test_generate_paper_not_extracted_returns_422`, `test_generate_duplicate_job_returns_409`, `test_background_task_writes_smart_note` (uses `mock_ai_service` fixture returning fixed dict)

**Estimated time:** 2.5 hours
**Depends on:** E6-S1

---

### E6-S3: Implement Note Job Status Polling Endpoint

As a user,
I want the frontend to know when my smart note is ready without refreshing the page,
So that I can keep reading and the note view opens automatically when generation completes.

**What to build:** Add `GET /papers/{id}/note/status` which returns the current job status and `smart_note_id` when done. The frontend polls this endpoint every 3 seconds using React Query's `refetchInterval`. When status is `done`, polling stops and the note view is rendered. When status is `failed`, polling stops and an error state is shown.

**Acceptance Criteria:**

**Given** a job with status `pending`
**When** `GET /papers/{paper_id}/note/status` is called
**Then** response is `200` with `{"data": {"status": "pending", "job_id": "<uuid>", "smart_note_id": null}, "meta": {...}}`

**Given** a job with status `done`
**When** `GET /papers/{paper_id}/note/status` is called
**Then** response is `200` with `{"data": {"status": "done", "job_id": "<uuid>", "smart_note_id": "<uuid>"}, "meta": {...}}`

**Given** no job exists for this paper
**When** `GET /papers/{paper_id}/note/status` is called
**Then** response is `200` with `{"data": {"status": "none", "job_id": null, "smart_note_id": null}, "meta": {...}}`
**And** response code is `200` — not 404 (absence of a job is a valid state, not an error)

**Given** a job with status `failed`
**When** `GET /papers/{paper_id}/note/status` is called
**Then** response is `200` with `{"data": {"status": "failed", "job_id": "<uuid>", "smart_note_id": null}, "meta": {...}}`

**When** developer calls `GET /papers/<other_user_paper_id>/note/status` with a different user's token
**Then** response is `403 FORBIDDEN`

**Local verification:**
```bash
docker compose up -d
TOKEN=$(...)
# After triggering generate:
curl -s http://localhost:8000/papers/<paper_id>/note/status \
  -H "Authorization: Bearer $TOKEN" | jq '.data.status'
# Expect: "pending" then "running" then "done" as you poll
# For a new paper with no job:
curl -s http://localhost:8000/papers/<new_paper_id>/note/status \
  -H "Authorization: Bearer $TOKEN" | jq '.data.status'
# Expect: "none"
pytest tests/test_note_status.py -v
```

**Tasks:**
- Add to `app/routers/papers.py`: `GET /papers/{paper_id}/note/status`:
  - Verify paper ownership (403)
  - `SELECT id, status, smart_note_id FROM note_jobs WHERE paper_id=? ORDER BY created_at DESC LIMIT 1`
  - If no row: return `{"status": "none", "job_id": null, "smart_note_id": null}`
  - Else: return `{"status": job.status, "job_id": job.id, "smart_note_id": job.smart_note_id}`
- Create `app/schemas/note.py` — `NoteJobStatus(status: Literal['none','pending','running','done','failed'], job_id: UUID | None, smart_note_id: UUID | None)`
- Create `tests/test_note_status.py` — `test_status_returns_none_when_no_job`, `test_status_returns_pending`, `test_status_returns_done_with_note_id`, `test_status_forbidden_other_user`

**Estimated time:** 1.5 hours
**Depends on:** E6-S2

---

### E6-S4: Build SmartNoteBanner with Last-Page Detection, Dismiss Logic, and Manual Trigger

As a user,
I want a non-intrusive prompt to generate a Smart Note when I reach the last page of a paper,
So that note generation is suggested at the natural completion moment without interrupting my reading.

**What to build:** Create `SmartNoteBanner.tsx` — a dismissible banner shown in the reader when the user reaches the last page AND no smart note exists for that paper. The banner offers "Generate Smart Note" and a dismiss ("✕") button. The user can also trigger generation manually from a button in the reader header at any time. Once generation is triggered (or dismissed), the banner does not appear again in the current session. Clicking "Generate Smart Note" calls `POST /papers/{id}/note/generate` and shows a spinner in place of the banner.

**Acceptance Criteria:**

**Given** the user is on the last page of a paper
**When** the reader renders and the note status is `none` (no note, no job)
**Then** `SmartNoteBanner` is visible with text "You've reached the end — generate a Smart Note?" and two buttons: "Generate Smart Note" and "✕"

**Given** the user is on page 3 of a 10-page paper
**When** the reader renders
**Then** `SmartNoteBanner` is NOT visible (only shown on last page)

**Given** the user dismisses the banner with "✕"
**When** they navigate away and return to the same paper in the same browser session
**Then** the banner does NOT reappear (dismissed state is held in React component state, not localStorage)

**Given** the user clicks "Generate Smart Note"
**When** `POST /papers/{paper_id}/note/generate` returns 202
**Then** the banner is replaced by a spinner with text "Generating your Smart Note…"
**And** the reader page begins polling `GET /papers/{paper_id}/note/status` every 3 seconds

**Given** the reader header contains a "Generate Note" icon button
**When** the user clicks it while on any page (not just the last)
**Then** same behaviour as clicking "Generate Smart Note" from the banner — 202 call + spinner + polling starts

**Given** a smart note already exists for the paper (status `done`)
**When** the user reaches the last page
**Then** `SmartNoteBanner` is NOT shown (note already generated)

**Local verification:**
```bash
docker compose up
# 1. Open a paper → navigate to last page
#    Expect: SmartNoteBanner appears
# 2. Click "✕" → navigate away → return to paper → last page
#    Expect: banner does NOT reappear
# 3. Navigate to last page of paper with no existing note → click "Generate Smart Note"
#    Expect: spinner replaces banner; DevTools Network: POST to /note/generate then GET /note/status every 3s
# 4. Click "Generate Note" button in reader header from page 3 (not last page)
#    Expect: same behaviour as above
```

**Tasks:**
- Create `src/components/reader/SmartNoteBanner.tsx`:
  - Props: `onGenerate: () => void`, `onDismiss: () => void`, `isGenerating: boolean`
  - `isGenerating=false`: render amber/info banner with "You've reached the end — generate a Smart Note?", `<Button onClick={onGenerate}>Generate Smart Note</Button>`, `<Button variant="ghost" onClick={onDismiss}>✕</Button>`
  - `isGenerating=true`: render spinner + "Generating your Smart Note…" (no dismiss button)
- Update `src/app/(app)/reader/[paperId]/page.tsx`:
  - `const [dismissed, setDismissed] = useState(false)` — session-only dismissal state
  - `showBanner = currentPage === totalPages && noteStatus.status === 'none' && !dismissed && !isGenerating`
  - `handleGenerate()`: call `fetchApi(POST /papers/{paperId}/note/generate)` → set `isGenerating = true` → `queryClient.invalidateQueries(['note-status', paperId])`
  - Pass `currentPage` down to `PDFViewer` via `onPageChange` prop already wired (E4-S3)
  - Render `<SmartNoteBanner>` when `showBanner`; render `<SmartNoteBanner isGenerating={true}>` when `isGenerating && noteStatus.status !== 'done'`
  - `useQuery(['note-status', paperId], ..., { refetchInterval: isGenerating ? 3000 : false })` — polling only when generating
  - On status `done`: set `isGenerating = false`; on status `failed`: set `isGenerating = false`, show toast error "Note generation failed. Try again."
- Add "Generate Note" icon button to `src/components/reader/PageNavigation.tsx` — `<Button variant="ghost" size="icon" onClick={onGenerateNote}><FileText className="h-4 w-4" /></Button>`; disabled when `isGenerating`

**Estimated time:** 2 hours
**Depends on:** E6-S3

---

### E6-S5: Build SmartNoteView with My Notes Editor and Auto-Save

As a user,
I want to read the AI-generated Smart Note and write my own notes alongside it,
So that I have one place where AI analysis and my personal understanding live together for each paper.

**What to build:** Create `SmartNoteView.tsx` — a side-panel or full page that displays the structured Smart Note (What/Why/How sections + Key Terms list) and a `MyNotesEditor` textarea for free-form personal notes. My Notes content is fetched from `GET /papers/{id}/note` (`smart_notes.my_notes` column) on load and auto-saved via `PATCH /papers/{id}/note/my-notes` with 1000ms debounce on keypress. Key Terms are displayed as a scrollable list with Vietnamese definition and English context per term.

**Acceptance Criteria:**

**Given** note status is `done` and the user is in the reader
**When** they click "View Smart Note" (or it opens automatically after generation)
**Then** `SmartNoteView` renders with four visible sections: "What", "Why", "How", and "Key Terms"
**And** each of `what`, `why`, `how` is non-empty text from the `smart_notes` table
**And** the Key Terms section shows a list where each row has: `term` (bold), `definition_vi` (Vietnamese text), `context_en` (English usage context)

**Given** the Smart Note view is open
**When** the user types in the My Notes textarea
**Then** no API call is fired for at least 1000ms after the last keystroke
**And** after 1000ms of inactivity: `PATCH /papers/{paper_id}/note/my-notes` is called with `{"my_notes": "<current text>"}`
**And** response is `200` with `{"data": {"updated": true}, "meta": {...}}`
**And** a subtle "Saved" indicator appears after the successful PATCH

**Given** the user closes and reopens the Smart Note view (without page reload)
**When** `SmartNoteView` mounts
**Then** the My Notes textarea shows the last saved content — fetched fresh from `GET /papers/{id}/note`

**Given** `PATCH /papers/{paper_id}/note/my-notes` returns a network error
**When** the debounced save fires
**Then** a subtle error indicator appears ("Save failed — retrying…") — the textarea remains editable, the user is not blocked

**Given** no smart note exists for the paper (status not `done`)
**When** "View Smart Note" is clicked
**Then** the button is disabled (grayed out) — no navigation to `SmartNoteView`

**Local verification:**
```bash
docker compose up
# 1. Generate a smart note → wait for status done → click "View Smart Note"
#    Expect: SmartNoteView with What/Why/How/Key Terms sections populated
# 2. Type in My Notes textarea
#    DevTools Network: no PATCH for 1s after typing; PATCH fires after 1s idle
# 3. Refresh page → open SmartNoteView again → My Notes content must be restored
# 4. Kill API mid-save → error indicator must appear without blocking textarea
psql $DATABASE_URL -c "SELECT my_notes FROM smart_notes WHERE paper_id='<id>';"
# Must match what was typed
```

**Tasks:**
- Add to `app/routers/papers.py`:
  - `GET /papers/{paper_id}/note` — verify ownership → `SELECT sn.what, sn.why, sn.how, sn.my_notes FROM smart_notes sn WHERE sn.paper_id=?` + `SELECT term, definition_vi, context_en FROM key_terms WHERE smart_note_id=sn.id ORDER BY id` → return combined `SmartNoteDetail` schema; 404 if no note
  - `PATCH /papers/{paper_id}/note/my-notes` — verify ownership → `UPDATE smart_notes SET my_notes=:content WHERE paper_id=:paper_id AND user_id=:user_id` → return `200 {"data": {"updated": true}, "meta": {...}}`
- Create `app/schemas/note.py` additions — `KeyTerm(term, definition_vi, context_en)`, `SmartNoteDetail(what, why, how, my_notes, key_terms: list[KeyTerm])`
- Create `src/components/reader/SmartNoteView.tsx`:
  - `useQuery(['smart-note', paperId], () => fetchApi(`/papers/${paperId}/note`))` on mount
  - Sections: `<section><h3>What</h3><p>{note.what}</p></section>` × 3 (what/why/how)
  - Key Terms: `<ul>{note.key_terms.map(t => <li key={t.term}><strong>{t.term}</strong><span lang="vi">{t.definition_vi}</span><span>{t.context_en}</span></li>)}</ul>`
  - My Notes: `<textarea value={myNotes} onChange={e => { setMyNotes(e.target.value); debouncedSave(e.target.value) }}>`; `debouncedSave` is a `useRef` holding a `setTimeout` handle (1000ms debounce, no lodash); on save success: show "Saved" for 2s; on save error: show "Save failed — retrying…" with auto-retry after 3s
- Update `src/app/(app)/reader/[paperId]/page.tsx` — render `<SmartNoteView>` when `noteStatus.status === 'done'`; add "View Smart Note" `<Button disabled={noteStatus.status !== 'done'}>` to reader header

**Estimated time:** 2.5 hours
**Depends on:** E6-S4

---

## Epic 7: Library & Polish — Deletion, Filters, Quota Dashboard & Mobile

Users can manage their paper library (delete papers, filter by status), see their quota usage on the reader, and use the app comfortably on a phone. Done when a user can delete a paper with all its data, filter the library by reading status, and view the app on a 375px-wide screen without horizontal scroll.

**FRs covered:** FR-006, FR-041 (full status filter), FR-043, FR-044, FR-054

---

### E7-S1: Implement Paper Deletion with Cascade and Library Confirmation Dialog

As a user,
I want to delete a paper and have all its associated data removed,
So that I can keep my library tidy and know no orphaned data remains.

**What to build:** Add `DELETE /papers/{id}` to the API. Deletion cascades through: S3 file removal, then DB delete (PostgreSQL FK `ON DELETE CASCADE` handles notes, saved_explanations, note_jobs, key_terms, quota_daily is NOT deleted). In the frontend, a confirmation dialog must be shown before deletion fires. After deletion, the papers query is invalidated and the library re-renders without the deleted paper.

**Acceptance Criteria:**

**Given** the user clicks "Delete" on a paper card
**When** the confirmation dialog appears
**Then** the dialog shows the paper title and the text "This will permanently delete the paper and all its notes and saved explanations."
**And** two buttons are shown: "Cancel" and "Delete permanently"

**When** user clicks "Cancel"
**Then** the dialog closes, no API call is made, paper remains in library

**When** user clicks "Delete permanently"
**Then** `DELETE /papers/{paper_id}` is called
**And** response is `200` with `{"data": {"deleted": true}, "meta": {...}}`
**And** the paper disappears from the library immediately (optimistic removal or query invalidation)

**Given** `DELETE /papers/{paper_id}` completes
**When** developer queries the DB
**Then** `SELECT COUNT(*) FROM papers WHERE id='<id>'` returns `0`
**And** `SELECT COUNT(*) FROM smart_notes WHERE paper_id='<id>'` returns `0`
**And** `SELECT COUNT(*) FROM saved_explanations WHERE paper_id='<id>'` returns `0`
**And** the file no longer exists in S3 at key `pdfs/<paper_id>.pdf`

**When** developer calls `DELETE /papers/<other_user_paper_id>` with a different user's token
**Then** response is `403 FORBIDDEN`

**Local verification:**
```bash
docker compose up
# 1. Upload a paper → generate a note → save an explanation
# 2. Click Delete on the paper card → confirm in dialog
# 3. Verify library no longer shows the paper
psql $DATABASE_URL -c "SELECT COUNT(*) FROM papers WHERE id='<id>';"
# Expect: 0
psql $DATABASE_URL -c "SELECT COUNT(*) FROM smart_notes WHERE paper_id='<id>';"
# Expect: 0
aws s3 ls s3://readflow-dev-pdfs/pdfs/<paper_id>.pdf
# Expect: no output (file gone)
pytest tests/test_papers.py::test_delete_paper_cascade -v
```

**Tasks:**
- Add `ON DELETE CASCADE` FK constraints to `smart_notes`, `saved_explanations`, `note_jobs`, `key_terms` referencing `papers(id)` — new Alembic migration `add_cascade_delete_to_paper_fks`
- Add to `app/routers/papers.py`: `DELETE /papers/{paper_id}`:
  - Verify ownership (403)
  - `s3_client.delete_object(Bucket=settings.S3_BUCKET, Key=f'pdfs/{paper_id}.pdf')`
  - `DELETE FROM papers WHERE id=:paper_id` (cascades via FK)
  - Return `200 {"data": {"deleted": true}, "meta": {...}}`
- Create `src/components/library/DeletePaperDialog.tsx` — shadcn `<AlertDialog>` with paper title in description body; "Cancel" maps to `AlertDialogCancel`; "Delete permanently" maps to `AlertDialogAction` which calls `deleteMutation.mutate(paper.id)`
- Update `src/components/library/PaperCard.tsx` — add kebab menu (`DropdownMenu`) with "Delete" item → opens `<DeletePaperDialog>`; on mutation success: `queryClient.invalidateQueries({ queryKey: ['papers'] })`
- Add `tests/test_papers.py::test_delete_paper_cascade` — uploads paper, inserts mock smart_note + saved_explanation, calls DELETE, asserts all three tables empty and S3 call made (mocked boto3)

**Estimated time:** 2 hours
**Depends on:** E6-S5

---

### E7-S2: Add Status Filter to Library and Mark-as-Finished Action

As a user,
I want to filter my library by reading status and mark a paper as finished,
So that I can focus on what I'm actively reading and track my completion.

**What to build:** Add query param filtering to `GET /papers?status=unread|in_progress|finished`. Add `PATCH /papers/{id}/status` to manually set status to `finished` (the only manual override — `unread` → `in_progress` is automated by reading position). In the frontend, add filter tabs to the library page (All / Unread / In Progress / Finished). Add a "Mark as finished" option to the paper card menu.

**Acceptance Criteria:**

**Given** the user has papers with status `unread`, `in_progress`, and `finished`
**When** developer calls `GET /papers?status=unread`
**Then** only papers with `status='unread'` are returned
**And** `GET /papers` (no filter) returns all papers ordered by `created_at DESC`

**When** `PATCH /papers/{paper_id}/status` is called with `{"status": "finished"}`
**Then** response is `200` with `{"data": {"status": "finished"}, "meta": {...}}`
**And** `SELECT status FROM papers WHERE id='<id>'` returns `finished`

**When** `PATCH /papers/{paper_id}/status` is called with `{"status": "unread"}`
**Then** response is `422` with `{"error": {"code": "VALIDATION_ERROR", "message": "Manual status can only be set to 'finished'."}, "meta": {...}}`

**Given** the library page is open
**When** user clicks the "In Progress" tab
**Then** only papers with `status='in_progress'` render; other papers are hidden
**And** the active tab is visually highlighted

**When** user clicks "Mark as finished" from the paper card menu
**Then** `PATCH /papers/{id}/status` is called; on success: paper's status badge updates to "Finished"

**Local verification:**
```bash
docker compose up
# Manually set statuses:
psql $DATABASE_URL -c "UPDATE papers SET status='in_progress' WHERE id='<id1>';"
psql $DATABASE_URL -c "UPDATE papers SET status='finished' WHERE id='<id2>';"
# Test filter:
curl -s "http://localhost:8000/papers?status=in_progress" \
  -H "Authorization: Bearer $TOKEN" | jq '.data | length'
# Click "In Progress" tab in browser → only in_progress papers visible
# Click "Mark as finished" on unread paper → badge changes to "Finished"
```

**Tasks:**
- Update `app/routers/papers.py` `GET /papers` — add optional `status: str | None = Query(None)` param; if provided: append `WHERE status = :status` to query
- Add `PATCH /papers/{paper_id}/status` — verify ownership → validate `status == 'finished'` (422 otherwise) → `UPDATE papers SET status='finished'` → return 200
- Create new Alembic migration to add index on `papers(user_id, status)` for filter performance — `add_index_papers_user_status`
- Update `src/app/(app)/library/page.tsx` — add filter state `const [filter, setFilter] = useState<'all'|'unread'|'in_progress'|'finished'>('all')`; pass to query: `queryKey: ['papers', filter]`, `queryFn: () => fetchApi(`/papers${filter !== 'all' ? `?status=${filter}` : ''}`)`; render filter tabs using shadcn `<Tabs>` component above paper list
- Update `src/components/library/PaperCard.tsx` — add "Mark as finished" to `DropdownMenu` (only shown when `status !== 'finished'`); `PATCH /papers/{paper.id}/status` mutation; on success: `queryClient.invalidateQueries({ queryKey: ['papers'] })`

**Estimated time:** 1.5 hours
**Depends on:** E7-S1

---

### E7-S3: Build Quota Dashboard in Reader Header and Limitation Disclosures

As a user,
I want to see my quota usage at a glance and understand any known limitations of the AI features,
So that I can plan my usage and am not surprised when something doesn't work as expected.

**What to build:** Move quota status display to the reader header as a compact badge (e.g., "42/50 explains"). Add a "Limitations" info popover accessible from the reader header that discloses the known limitations listed in FR-054: multi-column PDF layout, image-only PDFs, non-InfoSec/DevOps domain accuracy. The quota badge updates after each explain call (query invalidation already wired in E5-S6).

**Acceptance Criteria:**

**Given** the user is in the reader and has used 42 of 50 daily explains
**When** they look at the reader header
**Then** a compact badge shows "42/50 explains today"
**And** the badge turns amber when `percent >= 80` and red when `percent >= 100`

**When** user clicks the "ⓘ Limitations" button in the reader header
**Then** a popover opens with the following disclosures (each as a bullet):
- "Multi-column PDF layouts may render in incorrect reading order."
- "Image-only PDFs (scanned documents) are not supported — text must be selectable."
- "AI explanations are optimised for InfoSec and DevOps topics. Accuracy may vary for other domains."
- "Smart Note generation may take up to 30 seconds depending on paper length."

**When** the user completes an explain call
**Then** the quota badge count increments without a page refresh (React Query invalidation)

**When** the monthly note quota is `5/5` (exhausted)
**Then** the "Generate Note" button in the reader header is disabled
**And** a tooltip on hover shows "Monthly note limit reached. Resets on [reset date]."

**Local verification:**
```bash
docker compose up
# 1. Set explain_count to 42 via SQL
psql $DATABASE_URL -c "UPDATE quota_daily SET explain_count=42 WHERE user_id='<id>' AND date=CURRENT_DATE;"
# Reader header must show "42/50 explains today"
# 2. Click "ⓘ Limitations" → popover must show all 4 bullet points
# 3. Trigger an explain → badge must increment to 43 without page refresh
# 4. Set note quota to 5:
psql $DATABASE_URL -c "UPDATE quota_monthly SET note_count=5 WHERE user_id='<id>';"
# "Generate Note" button must be disabled with tooltip
```

**Tasks:**
- Create `src/components/reader/QuotaBadge.tsx` — compact `<span>` showing `{used}/{limit} explains today`; className: default text-muted, `percent >= 80` → amber, `percent >= 100` → red destructive; consumes `quotaStatus` prop (already fetched in reader page)
- Create `src/components/reader/LimitationsPopover.tsx` — shadcn `<Popover>` triggered by `<Button variant="ghost" size="sm">ⓘ Limitations</Button>`; content: `<ul>` with 4 `<li>` items verbatim from AC; no close button needed (click outside closes)
- Update `src/components/reader/PageNavigation.tsx` — add `<QuotaBadge>` and `<LimitationsPopover>` to the header bar; add `disabled` + `<Tooltip>` on "Generate Note" button when `noteQuota.percent >= 100`
- Update `src/app/(app)/reader/[paperId]/page.tsx` — pass `quotaStatus` to `<PageNavigation>`; ensure `quotaStatus` query is already invalidated after explain (verified in E5-S6 — no new wiring needed)

**Estimated time:** 1.5 hours
**Depends on:** E7-S2

---

### E7-S4: Mobile Responsive Pass for Library and Reader

As a user on a phone,
I want the library and reader to be usable on a 375px-wide screen without horizontal scroll,
So that I can manage my papers and read on mobile without a broken layout.

**What to build:** Audit and fix layout issues on 375px viewport for the library page, reader page, and Smart Note view. The goal is no horizontal scroll and all interactive elements reachable by thumb. PDF rendering on mobile is best-effort (zoom/scroll acceptable) but the surrounding UI must be correct. Use Tailwind responsive prefixes (`sm:`, `md:`) throughout — no custom media queries.

**Acceptance Criteria:**

**Given** browser DevTools set to "iPhone SE" (375×667px) or equivalent
**When** user navigates to `/library`
**Then** no horizontal scrollbar is present
**And** paper cards stack in a single column
**And** the "Add your first paper" button and filter tabs are fully visible and tappable

**When** user navigates to `/reader/<paper_id>`
**Then** the reader header (quota badge, limitations, generate note button) wraps gracefully or scrolls vertically without horizontal overflow
**And** PDF canvas is contained within the viewport (may require horizontal scroll within the PDF area itself — this is acceptable)
**And** `SmartNoteBanner` (if shown) is fully visible and not clipped

**When** user navigates to the Smart Note view
**Then** the What/Why/How sections stack vertically
**And** the Key Terms list is scrollable
**And** the My Notes textarea is at least 120px tall and full-width

**When** developer runs `npm run build`
**Then** build completes without errors — all responsive changes are type-safe

**Local verification:**
```bash
npm run dev
# Chrome DevTools → Toggle device toolbar → iPhone SE (375×667)
# Test each page:
# 1. /library — no horizontal scroll, cards single-column, filter tabs visible
# 2. /reader/<id> — header visible, no horizontal overflow on outer shell
# 3. Smart Note view — sections stacked, Key Terms scrollable, textarea full-width
npm run build  # must exit 0
```

**Tasks:**
- Audit `src/app/(app)/library/page.tsx` and `PaperCard.tsx`:
  - Ensure `max-w-full` on card container; `flex-col` on mobile, `flex-row` on `sm:` for card metadata
  - Tabs: `overflow-x-auto` on tabs container if needed, or stack with `flex-wrap`
- Audit `src/components/reader/PageNavigation.tsx`:
  - Header: `flex-wrap gap-2` so buttons wrap on narrow viewport; badge moves to second line if needed
- Audit `src/components/reader/SmartNoteBanner.tsx`:
  - `flex-col sm:flex-row` on banner; buttons stack vertically on mobile
- Audit `src/components/reader/SmartNoteView.tsx`:
  - Key Terms `<ul>`: `overflow-y-auto max-h-64`; My Notes `<textarea>`: `min-h-[120px] w-full`
- Run DevTools mobile emulation on all three pages; fix any remaining `overflow-x` issues with `overflow-hidden` or `min-w-0` on flex children

**Estimated time:** 1.5 hours
**Depends on:** E7-S3

---

## Epic 8: Production Hardening — Security, Observability & Documentation

The system meets production security and observability standards. Done when: (1) Trivy blocks a vulnerable image in CI, (2) OPA rejects a pod without resource limits, (3) Grafana SLO dashboard shows 7-day error budget, (4) README contains an architecture diagram. All four conditions are required.

**FRs covered:** FR-060, FR-061, FR-062, NFR-S01, NFR-S02, NFR-O01, NFR-O02

---

### E8-S1: Enforce OPA Policy — Reject Pods Without Resource Limits

As a platform operator,
I want OPA Gatekeeper to reject any pod that doesn't declare CPU and memory limits,
So that a misconfigured deployment can never starve other workloads on the k3s node.

**What to build:** Write an OPA Gatekeeper `ConstraintTemplate` and `Constraint` that rejects any pod missing `resources.limits.cpu` or `resources.limits.memory`. Add resource limits to all ReadFlow Helm manifests. Verify with a deliberately misconfigured pod manifest that OPA blocks it.

**Acceptance Criteria:**

**Given** OPA Gatekeeper is installed on the k3s cluster
**When** developer applies the `ConstraintTemplate` and `Constraint` from `readflow-gitops/policies/`
**Then** `kubectl get constrainttemplate` shows the template as established

**When** developer applies a test pod manifest with no `resources.limits`
**Then** `kubectl apply` is rejected with a message containing "Resource limits are required"

**When** developer applies the ReadFlow API and web Helm releases
**Then** both deployments apply without OPA rejection (resource limits are present in all containers)

**Given** the policy is active
**When** developer inspects `readflow-gitops/policies/require-resource-limits.yaml`
**Then** the file contains both a `ConstraintTemplate` and a `Constraint` in a single file separated by `---`

**Local verification:**
```bash
# Apply policy:
kubectl apply -f readflow-gitops/policies/require-resource-limits.yaml
# Test rejection:
kubectl apply -f readflow-gitops/policies/test-no-limits-pod.yaml
# Expect: admission webhook denied: Resource limits are required
# Verify ReadFlow deployments pass:
helm upgrade --install readflow-api readflow-gitops/charts/api/
kubectl rollout status deployment/readflow-api
# Expect: successfully rolled out
```

**Tasks:**
- Create `readflow-gitops/policies/require-resource-limits.yaml`:
  - `ConstraintTemplate` kind: `K8sRequireResourceLimits`; Rego rule: `deny[msg]` if any container in `input.review.object.spec.containers` is missing `resources.limits.cpu` or `resources.limits.memory`
  - `Constraint` kind: `K8sRequireResourceLimits`, name: `require-resource-limits`, `spec.match.kinds: [{apiGroups:[""], kinds:["Pod"]}]`
- Create `readflow-gitops/policies/test-no-limits-pod.yaml` — a minimal pod manifest with no `resources` block; used to verify rejection; NOT committed to be auto-applied (label with `app=opa-test`)
- Update all container specs in `readflow-gitops/charts/api/values.yaml` and `readflow-gitops/charts/web/values.yaml` — add `resources: { limits: { cpu: "500m", memory: "512Mi" }, requests: { cpu: "100m", memory: "128Mi" } }` (adjust to fit t3.small budget)
- Verify: apply policy → apply test pod (expect rejection) → helm upgrade (expect success) → document result in commit message

**Estimated time:** 2 hours
**Depends on:** E1-S4 (k3s cluster and ArgoCD must exist)

---

### E8-S2: Harden CI — Trivy Blocks High/Critical Vulnerabilities and Add SBOM Output

As a developer,
I want the CI pipeline to fail if the Docker image has any HIGH or CRITICAL CVEs, and produce an SBOM for audit,
So that no vulnerable image is ever pushed to ECR and we have a traceable artifact inventory.

**What to build:** Verify and tighten the Trivy step in `.github/workflows/api-ci.yml` (added in E2-S5) to also produce an SBOM in CycloneDX JSON format as a GitHub Actions artifact. Add the same Trivy + SBOM step to `readflow-web`'s CI. Demonstrate that a deliberately vulnerable base image causes CI failure.

**Acceptance Criteria:**

**Given** `.github/workflows/api-ci.yml` contains the Trivy step from E2-S5
**When** the `build` job runs on a clean image with no HIGH/CRITICAL CVEs
**Then** Trivy exits with code `0` and the job passes
**And** a `sbom.json` artifact is uploaded to the Actions run (CycloneDX format)

**When** developer temporarily changes the `Dockerfile` base to `python:3.8` (known vulnerable)
**Then** Trivy exits with code `1`, the `build` job fails, and no image is pushed to ECR

**Given** `.github/workflows/web-ci.yml`
**When** the `build` job runs
**Then** it includes a `trivy image --exit-code 1 --severity HIGH,CRITICAL` step for the Next.js image
**And** a `sbom-web.json` artifact is uploaded

**Given** CI passes on `main`
**When** developer checks the GitHub Actions run
**Then** two artifacts are downloadable: `sbom-api.json` and `sbom-web.json`

**Local verification:**
```bash
# API image:
docker build -t readflow-api:local .
trivy image --exit-code 1 --severity HIGH,CRITICAL readflow-api:local
trivy image --format cyclonedx --output sbom.json readflow-api:local
cat sbom.json | jq '.metadata.component.name'
# Expect: no exit 1, sbom.json created

# Simulate vulnerable base (temporary edit to Dockerfile):
# Change FROM to python:3.8 → build → trivy → must exit 1
```

**Tasks:**
- Update `readflow-api/.github/workflows/api-ci.yml` `build` job — after existing Trivy scan step, add:
  ```yaml
  - name: Generate SBOM
    run: trivy image --format cyclonedx --output sbom-api.json readflow-api:${{ github.sha }}
  - name: Upload SBOM
    uses: actions/upload-artifact@v4
    with:
      name: sbom-api
      path: sbom-api.json
  ```
- Update `readflow-web/.github/workflows/web-ci.yml` `build` job — add Docker build step, Trivy scan, and SBOM upload (same pattern, output `sbom-web.json`)
- Verify Trivy version pinned in CI (e.g., `aquasecurity/trivy-action@0.20.0`) to avoid breaking version upgrades
- Push branch with clean images → verify both artifacts appear in Actions run

**Estimated time:** 1.5 hours
**Depends on:** E2-S5 (api-ci.yml must exist)

---

### E8-S3: Deploy Grafana SLO Dashboard with 7-Day Error Budget

As a platform operator,
I want a Grafana dashboard showing the 7-day error budget for ReadFlow's core API endpoints,
So that I can see at a glance whether the system is within its reliability target without manual log review.

**What to build:** Deploy Prometheus + Grafana to the k3s cluster via ArgoCD. Configure FastAPI to expose Prometheus metrics via `prometheus-fastapi-instrumentator`. Create a Grafana dashboard (provisioned as code in `readflow-gitops/dashboards/`) with: request rate, error rate (5xx), p95 latency, and a 7-day error budget panel (target: 99% availability → 1.68 hours budget per 7 days). SLO alert fires when error budget is below 10%.

**Acceptance Criteria:**

**Given** Prometheus and Grafana are deployed to the cluster
**When** developer opens Grafana at `http://<node-ip>:3001`
**Then** the "ReadFlow SLO" dashboard is visible without manual import (provisioned from `readflow-gitops/dashboards/readflow-slo.json`)

**When** the dashboard loads
**Then** four panels are visible: "Request Rate (req/min)", "Error Rate (% 5xx)", "p95 Latency (ms)", "7-Day Error Budget (%)"

**When** developer calls `POST /explain/stream` 10 times (with valid token)
**Then** the "Request Rate" panel shows a spike within 60 seconds (Prometheus scrape interval)

**Given** the SLO alert rule is active
**When** developer temporarily returns 500 errors from the API to exhaust >90% of the error budget
**Then** the alert fires and is visible in Grafana Alerting → Active alerts

**Given** FastAPI is running
**When** developer calls `GET /metrics`
**Then** response contains `http_requests_total` and `http_request_duration_seconds` metrics in Prometheus text format

**Local verification:**
```bash
# After ArgoCD syncs:
kubectl get pods -n monitoring
# Expect: prometheus-*, grafana-* running
kubectl port-forward svc/grafana 3001:80 -n monitoring &
# Browser: http://localhost:3001 → ReadFlow SLO dashboard → 4 panels visible
curl http://localhost:8000/metrics | grep http_requests_total
# Expect: metric line present
```

**Tasks:**
- Add `prometheus-fastapi-instrumentator` to `readflow-api/pyproject.toml`; instrument in `app/main.py`: `Instrumentator().instrument(app).expose(app, endpoint='/metrics')`
- Create `readflow-gitops/apps/monitoring.yaml` — ArgoCD `Application` pointing to a kube-prometheus-stack Helm chart (community chart) with minimal values: single replica, no persistent storage (simplicity over durability for solo validation)
- Create `readflow-gitops/dashboards/readflow-slo.json` — Grafana dashboard JSON (export from Grafana UI after manual setup, then commit): 4 panels with PromQL queries:
  - Request rate: `rate(http_requests_total{job="readflow-api"}[1m]) * 60`
  - Error rate: `rate(http_requests_total{job="readflow-api",status=~"5.."}[5m]) / rate(http_requests_total{job="readflow-api"}[5m]) * 100`
  - p95 latency: `histogram_quantile(0.95, rate(http_request_duration_seconds_bucket{job="readflow-api"}[5m])) * 1000`
  - Error budget: `(1 - rate(http_requests_total{status=~"5.."}[7d]) / rate(http_requests_total[7d])) / 0.01 * 100` (100% = full budget remaining)
- Create `readflow-gitops/dashboards/provisioning/dashboards.yaml` — Grafana provisioning config pointing to the dashboard JSON file
- Create SLO alert rule in `readflow-gitops/dashboards/alert-rules.yaml` — fires when error budget below 10% for 5 minutes

**Estimated time:** 3 hours
**Depends on:** E1-S4

---

### E8-S4: Write README Architecture Diagram and Deployment Runbook

As a new contributor (or future-self after 3 months away),
I want a README with an architecture diagram and a deployment runbook,
So that I can understand the system topology and deploy a new version without reconstructing the steps from memory.

**What to build:** Write `README.md` in `readflow-docs` (or root) with: a Mermaid architecture diagram showing all five repos and their relationships (web → api → Bedrock, api → S3/RDS, infra → k3s, gitops → ArgoCD), a "How to deploy" section, a "Local development" section, and a "Known limitations" section. The diagram must render correctly in GitHub's Mermaid renderer.

**Acceptance Criteria:**

**Given** the README is pushed to GitHub
**When** developer opens the README on `github.com`
**Then** the Mermaid diagram renders without errors — no "Syntax error" displayed by GitHub

**When** developer inspects the diagram
**Then** the following components are visible and connected:
- `readflow-web` (Next.js) → `readflow-api` (FastAPI)
- `readflow-api` → AWS Bedrock (Claude Haiku + Sonnet)
- `readflow-api` → Amazon S3 (PDF storage)
- `readflow-api` → PostgreSQL (data)
- `readflow-infra` (Terraform) → k3s on EC2
- `readflow-gitops` (ArgoCD) → k3s deployments
- GitHub Actions → ECR → ArgoCD (CI/CD flow)

**When** developer reads the "How to deploy" section
**Then** it contains: pre-requisites (AWS account, kubectl context, ECR repos), bootstrap steps (`terraform apply` in `bootstrap/`), and a step to trigger a deploy by pushing to `main`

**When** developer reads the "Local development" section
**Then** it contains: one command to start the full local stack (`docker compose up`) and one command to run all tests (`pytest` + `npm run lint`)

**When** developer reads the "Known limitations" section
**Then** the four limitations from FR-054 appear verbatim (same text as the in-app Limitations popover)

**Local verification:**
```bash
# Push README to GitHub
# Open https://github.com/<org>/readflow-docs/blob/main/README.md
# Verify: Mermaid diagram renders, all 7 component connections visible
# Verify: Deploy section complete, Local dev section complete, Limitations section present
```

**Tasks:**
- Create `readflow-docs/README.md` (or `README.md` in repo root if readflow-docs doesn't exist locally):
  - `## Architecture` section with Mermaid diagram (`\`\`\`mermaid` block):
    ```
    graph LR
      web[readflow-web<br/>Next.js] --> api[readflow-api<br/>FastAPI]
      api --> bedrock[AWS Bedrock<br/>Claude Haiku + Sonnet]
      api --> s3[Amazon S3<br/>PDF Storage]
      api --> pg[(PostgreSQL<br/>Data)]
      infra[readflow-infra<br/>Terraform] --> k3s[k3s on EC2<br/>ap-southeast-1]
      gitops[readflow-gitops<br/>ArgoCD] --> k3s
      ghactions[GitHub Actions] --> ecr[Amazon ECR]
      ecr --> gitops
    ```
  - `## How to Deploy` — pre-reqs, bootstrap Terraform, push-to-deploy flow
  - `## Local Development` — `docker compose up` command, `pytest` and `npm run lint` commands
  - `## Known Limitations` — 4 bullets from FR-054 verbatim
- Push to GitHub and verify Mermaid renders on the GitHub UI before marking story done

**Estimated time:** 1.5 hours
**Depends on:** E8-S3
