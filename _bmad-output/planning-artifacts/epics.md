---
stepsCompleted: [1, 2]
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
