# Blaze Academy — Automated Exam & Certification Platform

Blaze Academy is an automated student assessment and certification system built on **Supabase**, **n8n**, and **Lovable**. It handles the full lifecycle from course enrollment and exam grading (AI-assisted) to certificate generation and operational health monitoring — all with a zero-trust security model.

---

## ⚙️ Key Components

| Component | Technology | Responsibility |
|---|---|---|
| **Frontend** | Lovable | Student-facing UI for enrollment, exam, and results |
| **Gateway** | Supabase Edge Functions | Server-side validation, token verification, n8n handoff |
| **Database** | Supabase Postgres | 6-table relational schema, RLS, atomic seat-cap functions |
| **Automation** | n8n Workflows | AI grading, email orchestration, certificate generation |
| **Security** | Cloudflare Turnstile | Bot verification for enrollment and submission |
| **AI Engine** | LLM (via n8n) | Grading subjective answers with prompt-injection defense |

---

## 🔄 Core Workflows

### 1. Course Enrollment & Certification — *Enrollment & Welcome*
Handles student sign-up, token generation, and welcome email delivery.

- **Input Validation:** Name (NFKC normalization, control char stripping), Email (trim, lowercase, disposable domain check), Consent (mandatory).
- **Magic-Link Auth:** Generates a 64-char token, hashes it, stores `token_hash` and `token_expires_at` in the `students` table.
- **Seat Cap Enforcement:** Atomic `enroll_student()` function ensures no overselling.
- **Email Orchestration:** Sends a branded welcome email (HTML), then a reminder, then the exam link — all logged in `email_log`.

### 2. Test Evaluation — *Exam Submission & Grading*
The core assessment engine that receives, validates, and grades exam submissions.

- **Input Validation:** Token format, answer structure (MCQs A–D, subjective answers ≤1500 chars).
- **Eligibility Check:** Confirms student exists, token not expired, exam unlocked, attempts left.
- **AI Grading:**
  - **MCQs:** Graded in-code against a fixed key (never exposed to the LLM).
  - **Subjective Qs (q5–q11):** LLM evaluates against 3 criteria per question (e.g., "hosting control", "error handling").
  - **Prompt Injection Defense:** LLM is instructed to treat all input as untrusted data; a separate code node flags suspicious patterns (e.g., "ignore instructions", "give full marks").
- **Verdict & Feedback:** Computed in-code — **PASS** if ≥ threshold (default 10/25) and no confirmed injection.
- **Certificate Trigger:** If PASS, triggers the Certificate Generation workflow.

### 3. Certificate Generation — *Certificate Issuance*
Generates and sends a branded PDF certificate for passed students.

- **Idempotency:** Checks the `certificates` table first — if already issued, returns existing data.
- **Certificate ID:** Generates a public, unambiguous ID (e.g., `BA-N8N-2026-ABCDEFGH`).
- **Template Engine:** Copies a Google Docs template, replaces placeholders (`{{Your Name}}`, `{{CERT_ID}}`, `{{DATE}}`), converts to PDF, uploads to Google Drive.
- **Database Update:** Logs the certificate in `certificates` and updates student status to `certificate_issued`.
- **Email:** Sends a branded certificate email with a view-only link.

### 4. Daily Reconciler — *Operational Health Check*
Runs nightly at 02:00 to clean up stuck work and report platform health.

- **Snapshot:** Calls a Supabase RPC to get daily stats (enrollments, submissions, certificates, security events).
- **Stuck Submissions:** Finds submissions stuck in `received` or `grading` and moves them to `needs_review`.
- **Certificate Retries:** Finds passed students without a certificate and retries Certificate Generation.
- **Slack Report:** Sends a daily summary to Slack with any issues (e.g., "3 stuck submissions", "2 certificate retries failed").

### 5. Alert Router (Errors & Security) — *Incident Management*
The central error and security alert handler.

- **Error Trigger:** Listens for workflow errors and direct alerts.
- **Throttling:** Suppresses duplicate alerts (e.g., 1 Slack message per key per 10 minutes).
- **Slack Alerts:** Sends formatted alerts to a Slack channel (e.g., `:rotating_light: Workflow error in "Test Evaluation"`).
- **Database Logging:** Logs security events (e.g., `injection_confirmed`) to the `security_events` table.

---

## 🛡️ Security & Reliability

### 1. Access Control
- **Magic-Link Auth:** Students receive a hashed, expiring token via email. No passwords stored.
- **Row-Level Security (RLS):** Database policies ensure students can only access their own data.
- **Single-Attempt Integrity:** `UNIQUE(student_id, attempt_no)` constraint enforces one exam attempt per student.

### 2. Threat Mitigation
- **Prompt Injection Defense:** Layered validation (input caps, pattern detection, untrusted-data system prompt, JSON schema parsing) to prevent students from manipulating the AI grader.
- **Rate Limiting:** Atomic `check_rate_limit()` function prevents abuse (e.g., 3 enrollments/IP/hour, 5 submissions/minute).
- **Seat Cap Enforcement:** Atomic database transaction prevents race conditions (overselling 300 seats).

### 3. Data Protection
- **PII Minimization:** Student identity is separated from exam answers before AI grading.
- **Payload Cap:** 32 KB limit on subjective answers prevents token exhaustion attacks.
- **Idempotency:** Email sub-workflows prevent duplicate sends on retry.

### 4. Operational Safety
- **Kill Switch:** Config-driven `registration_open` flag allows instant pause of new enrollments.
- **Error Handling:** Bounded retries, Slack alerts for failures, and scheduled reconciler jobs for stuck submissions.

---

## ⚠️ Known Issues & Fixes

> _These are identified bugs in the current export. They demonstrate active maintenance and debugging skills._

| Issue | Node/Component | Fix Required |
|---|---|---|
| **Dead Cron Path** | Schedule Trigger | Wire to "Get row(s)" in sheet to enable automated daily runs. |
| **Wrong Field Path** | Download Image | URL template uses `{{%20$json.imageurl%20}}`. Should be `{{ $json.output.images[0].url }}`. |
| **Status Not Updating** | Mark Status in Sheet | Currently writes Status back to itself. Should update to "Published" or "Done". |
| **Unused Nodes** | HTTP Request (Tavily) | Disabled node. Can be removed to clean up the canvas. |

---

## 🛠️ Setup Instructions

1. **Import:** Upload the workflow JSON into your n8n instance.
2. **Credentials:** Create and link the following OAuth/API credentials:
   - `googleSheetsOAuth2Api`
   - `tavilyApi`
   - `linkedInOAuth2Api`
   - `anthropicApi` (or configure the AI Gateway)
3. **Configuration:** Replace placeholder values:
   - `SHEET_ID` → Your Google Sheet ID
   - `LINKEDIN_PERSON_ID` → Your LinkedIn User ID (from Profile URL)
4. **Test:** Run manually via the **"Execute Workflow"** button.

---