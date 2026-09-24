# PINACO Smart Advisor — Functional Requirements Source Data

> **File purpose:** This is a **raw source-of-truth data pack**, not the FRD itself.
> It contains every verified fact about the system that a Functional Requirements Document
> needs to be written accurately. It was extracted directly from the working codebase,
> the PostgreSQL schema, the API route modules, and the domain service layer.
>
> **How to use this file:** Feed this file to your documentation AI with the instruction in
> §1.2. Every statement here is traceable to a real file in the repository. Do not invent
> features that are not listed here, and do not silently drop the "Demo/Simulated" markers —
> they distinguish shipped behaviour from prototype behaviour.

**Document status:** Derived from code as-built
**System name (package):** `pinaco-smart-advisor`
**System name (product):** PINACO Smart Advisor (also branded "Pinnacle Smart Advisor" / "Pinnacle MFI")
**Version:** 1.0.0
**Primary market:** Malawi (currency: Malawian Kwacha — MWK, displayed as "K")
**Repository root:** `d:/Projects/psa-pinnacle-main/psa-pinnacle-main`

---

## 1. Instructions for the Documentation AI

### 1.1 What to produce

Produce a single, formal **Functional Requirements Document (FRD)** for the PINACO Smart Advisor
platform, using the data in this file as the only source of truth. The FRD must contain, at minimum:

1. Document control (version, author, date, approvers, revision history)
2. Introduction (purpose, scope, intended audience, definitions, references)
3. Product overview and business context
4. Stakeholders, user classes, and the role permission matrix
5. System context and architecture summary
6. Functional requirements, numbered `FR-<AREA>-<NN>`, grouped by module, each with:
   - Requirement ID and title
   - Description in "The system shall…" form
   - Actor(s), trigger, preconditions, postconditions
   - Main flow and alternate/exception flows
   - Data inputs and outputs
   - Business rules applied (cross-reference §8 of this file)
   - Priority (MoSCoW), status (`Implemented` / `Partial` / `Simulated`)
   - Acceptance criteria in Given/When/Then form
7. Business rules catalogue (BR-xx), cross-referenced from the FRs
8. Data requirements (entities, attributes, relationships, retention)
9. Interface requirements (API, UI, integration, notification channels)
10. Non-functional requirements (security, performance, availability, usability, auditability)
11. Assumptions, constraints, dependencies, and known limitations
12. Traceability matrix (FR → module → code/route → test)
13. Appendices (glossary, endpoint reference, status enums)

### 1.2 Required constraints when writing

- Preserve **exact** thresholds, weights, percentages, enums, and status strings from §8 and §9.
  These are contractual; rounding or "cleaning up" numbers produces a wrong FRD.
- Distinguish clearly between:
  - **Implemented** — real code path, server + persistence.
  - **Partial** — exists but incomplete or client-side only.
  - **Simulated** — demo/mock behaviour, hard-coded, or stub data. Mark all of these explicitly.
- Do not add requirements for features that do not exist in this file (e.g. there is **no**
  ATM/card issuing, **no** foreign-currency lending, **no** borrower mobile money disbursement,
  **no** accounting general ledger).
- Where this file records a known gap or inconsistency (§13), reflect it as an **open issue**
  in the FRD rather than writing a requirement that pretends the gap is resolved.
- Currency is MWK throughout. Locale is Malawi (`en-GB`-style dates such as `24 Oct 2023`).

---

## 2. Product Overview

### 2.1 Business context

PINACO Smart Advisor is a microfinance and SME lending platform for a Malawian microfinance
institution. The business model is **payroll-deduction lending**: the institution lends to
salaried employees and small businesses, and recovers instalments mainly through monthly
payroll deductions executed by the borrower's employer (government ministries, parastatals,
and private employers).

The platform has three concerns:

1. **Customer self-service** — a client portal where borrowers register/activate, calculate
   loan affordability, submit applications, upload verification documents, track repayments,
   message staff, and consult an AI advisor.
2. **Credit operations** — a staff portal where loan officers, managers, and executives review
   applications, read an explainable rules-based credit recommendation, make or override
   decisions, monitor the portfolio, and reconcile payroll deduction files.
3. **Intelligence and governance** — a deterministic, explainable rules engine plus analytics
   (segmentation, fraud flags, sector trends, employer risk, officer quality, customer lifetime
   value) and a governance log that records every AI recommendation versus human decision.

### 2.2 Product principles visible in the code

| Principle | How it manifests |
|---|---|
| Explainability over opacity | The "AI" is a deterministic rules engine with named rules (R01–R04), weighted scores, and stated signals — not a trained model. See `src/lib/aiTransparency.ts`. |
| Human-in-the-loop | AI output is labelled "Not a final credit decision. Officer review and verified documents are required." Every officer decision is logged with an override flag. |
| Offline/demo resilience | Dual persistence: PostgreSQL primary, file-backed JSON fallback that switches automatically if the DB is unreachable. |
| Payroll-first collections | Repayment statuses include `FAILED_DEDUCTION`; payroll batches are uploaded as CSV and matched to customer records by employee number. |
| Auditability | Every mutating API call writes an `AuditLogEntry` with actor, action, entity, outcome, and summary. |

### 2.3 Branding and nomenclature caveats (important)

The codebase contains **inconsistent naming** that the FRD must resolve or document:

- Package/DB name: **PINACO** (`pinaco`, `pinaco-smart-advisor`, `PINACO@2026`).
- UI branding: **Pinnacle** ("Pinnacle", "Pinnacle MFI", "Pinnacle Smart Advisor", "Pinnacle Staff").
- Seed/demo emails use `@pinnacle.mw`; activation fallback email uses `@pinaco-customer.mw`.
- The `metadata.json` product name is `PINACO Smart Advisor`; the DTO/DB comment headers say
  "Pinnacle".

The FRD author should adopt **one** product name and record the other as a legacy alias.

---

## 3. Scope

### 3.1 In scope

- Public marketing/landing page and public authentication (register, login, forgot password)
- Existing-customer self-activation
- Client portal: home, loans, repayments, business/SME profile, calculator, documents,
  notifications, messaging, profile, AI advisor, contact
- Staff portal: assessment queue & review, client directory, SME portfolio, payroll deductions,
  reports, intelligence, analytics, staff team management
- Rules-based credit assessment, customer intelligence, employer risk, fraud detection
- AI governance (recommendation vs. override with reasons)
- Audit trail, CSV report export, document upload/download
- SMS and email transactional notifications
- Demo scenario switching (`Healthy Portfolio`, `High Default Risk`, `Payroll Crisis`)
- Dual-mode persistence with automatic fallback

### 3.2 Explicitly out of scope (verified absent)

- Card issuing, ATM, or POS integration
- Multi-currency support (system is MWK-only)
- Mobile money wallet disbursement or repayment
- Credit bureau integration (internal scores only)
- Trained machine-learning models (the engine is deterministic rules)
- Full double-entry general ledger / accounting
- Consolidated group/individual credit limits across products
- Push notifications (preference exists in data, no delivery channel implemented)

---

## 4. Stakeholders and Actors

### 4.1 Actor catalogue

| Actor | Description | Entry point |
|---|---|---|
| Guest / Prospect | Unauthenticated visitor | `/` (marketing), `/login`, `/register`, `/forgot-password`, `/activate` |
| Customer (`customer`) | Individual or SME borrower with a portal account | `/client/*` |
| Loan Officer (`loan_officer`) | Originates and reviews applications, processes payroll | `/staff/*` (restricted tabs) |
| Manager (`manager`) | Branch/operations manager; reviews, approves, oversees portfolio | `/staff/*` |
| Executive (`executive`) | Read-only strategic oversight; portfolio and analytics | `/staff/*` (reports, intelligence, analytics, overview) |
| Administrator (`admin`) | Full access including staff team management | `/staff/*` (all tabs) |
| System / Platform | Scheduled or automated processes (seeding, audit writes, fallback switch) | n/a |

### 4.2 Role permission matrix (as implemented)

| Capability / Screen | customer | loan_officer | manager | executive | admin |
|---|---|---|---|---|---|
| Client portal access | ✅ | ❌ | ❌ | ❌ | ❌ |
| Staff portal access | ❌ | ✅ | ✅ | ✅ | ✅ |
| Assessment Queue (`/staff/overview`) | ❌ | ✅ | ✅ | ✅ | ✅ |
| Payroll Deductions (`/staff/payroll`) | ❌ | ✅ | ✅ | ❌ | ✅ |
| Client Directory (`/staff/customers`) | ❌ | ✅ | ✅ | ❌ | ✅ |
| SME Portfolio (`/staff/sme`) | ❌ | ✅ | ✅ | ❌ | ✅ |
| Reports (`/staff/reports`) | ❌ | ❌ | ✅ | ✅ | ✅ |
| Intelligence (`/staff/intelligence`) | ❌ | ❌ | ❌ | ✅ | ✅ |
| Analytics (`/staff/analytics`) | ❌ | ❌ | ❌ | ✅ | ✅ |
| Staff Team (`/staff/team`) | ❌ | ❌ | ❌ | ❌ | ✅ |
| Record AI override (§7.10) | ❌ | ✅ | ✅ | ❌ | ✅ |
| Export CSV reports | ❌ | ✅ | ✅ | ✅ | ✅ |
| Create/manage staff accounts | ❌ | ❌ | ❌ | ❌ | ✅ |

**Server-side enforcement notes (verified):**

- `server/middleware/rbac.ts` → `requireRole([...])` grants access to any listed role, and
  `admin` always passes. Only `/api/v1/governance/override` uses it
  (`['officer','manager','admin']`). Note the role string used there is `officer`, while the
  stored user role is `loan_officer` — a **mismatch** (see §13, issue OI-01).
- The v1 route families (`/api/v1/customers`, `/loans`, `/payroll`, `/intelligence`,
  `/governance`) are behind `authenticateJwt` only — **no per-role authorisation** on read
  endpoints. Role filtering on the frontend is presentational.
- `/api/admin/staff` (GET, POST, PATCH) enforces `role === 'admin'` inline.
- `/api/reports/export` blocks only `role === 'customer'`.
- In non-production environments, `authenticateJwt` **injects a default loan officer**
  (`lo-001`, Chisomo Banda) when no token is supplied, so the staff API is open in dev.
  In `NODE_ENV=production` it returns `401`.

### 4.3 Seeded demo accounts (all password `PINACO@2026`)

| Name | Role | Email | Title |
|---|---|---|---|
| Thoko Kamanga (Demo) | admin | `admin@pinnacle.mw` | System Administrator |
| Chisomo Banda (Demo) | loan_officer | `chisomo@pinnacle.mw` | Loan Officer |
| Grace Phiri (Demo) | manager | `grace@pinnacle.mw` | Operations Manager |
| Kondwani Mbewe (Demo) | executive | `kondwani@pinnacle.mw` | Executive Director |

Seeded customers available for activation testing: Samuel Chimwala (`s.chimwala@pmail.mw`),
Bright Kamwendo (`b.kamwendo@pmail.mw`), Blessings Kamau (`b.kamau@pmail.mw`),
Mphatso Mwale (`m.mwale@pmail.mw`).

---

## 5. Glossary

| Term | Meaning in this system |
|---|---|
| **MWK / K** | Malawian Kwacha, the only currency. Amounts stored as numeric, formatted with `toLocaleString()`. |
| **Application** | A loan application (`LoanApplication`), identifier pattern `APP-xxxxx`. |
| **Instalment / Repayment** | One scheduled payment (`Repayment`), identifier pattern `REP-<APP-ID>-<NN>`. |
| **DTI** | Debt-to-Income ratio, stored as a string percentage (e.g. `"22%"`). |
| **Health Score** | Weighted composite 0–100 across repayment, income, debt, and business factors. |
| **Segment** | Customer classification: `Premium Client`, `Reliable Borrower`, `SME Growth`, `Young Entrepreneur`, `High Risk`, `Inactive`. |
| **Expert System** | Deterministic rule engine producing a verdict (`Approve`/`Decline`/`Review`/`Request Docs`) with fired rules. |
| **Relationship Score** | Configurable 0–100 trust score, or `-1` ("New Customer") when no history exists. |
| **LTV** | Customer Lifetime Value — interest generated plus a completed-loan bonus. |
| **Top-Up** | Additional financing offered to a repeat borrower with a clean record. |
| **Payroll Batch** | A CSV file of employer deductions uploaded for a given month. |
| **FAILED_DEDUCTION** | A repayment whose payroll deduction could not be matched/processed. No penalty is applied to the customer. |
| **Override** | An officer decision that differs from the AI recommendation; requires a reason. |
| **Scenario** | A demo operating state that changes portfolio metrics (`Healthy Portfolio`, `High Default Risk`, `Payroll Crisis`). |
| **Storage Mode** | `postgres` or `file` (JSON fallback at `server/data/mock-db.json`). |
| **AML / KYC** | Not implemented as named modules; identity capture is limited to national ID and employee number. |
---

## 6. System Architecture (as built)

### 6.1 Deployment topology

Single Node.js process serves both the API and (in production builds) the compiled React SPA:

- **Runtime:** Node.js + `tsx` (TypeScript executed directly). Entry: `server/index.ts`.
- **Default API port:** `process.env.PORT || 4000`.
- **Frontend dev server:** Vite on port `3000`, host `0.0.0.0` (`npm run dev`).
- **Production:** `npm run build` emits `dist/`; if `dist/` exists the server serves it
  statically and falls back to `index.html` for SPA routes, excluding `/api/*`.
- **Container support:** `Dockerfile`, `docker-compose.yml`, `ecosystem.config.js` (PM2).

### 6.2 Layering

| Layer | Location | Responsibility |
|---|---|---|
| Presentation | `src/` | React 19 + TypeScript + Tailwind CSS 4 SPA. Two portals (`/client`, `/staff`). |
| Contexts | `src/context/` | `ClientDataProvider`, `AdminDataProvider` — portal state and actions. |
| Client services | `src/lib/` | `apiClient`, `dataService`, `platformApi`, `store`, `intelligenceEngine`, `aiAdvisorService`, `documentExtractionService`, `auditTrail`. |
| API / routes | `server/routes/api/v1/` | Express routers: auth, customer, loan, payroll, intelligence, governance. |
| Middleware | `server/middleware/` | `auth.ts` (JWT verify + dev bypass), `rbac.ts` (`requireRole`). |
| Services | `server/services/` | `creditAssessmentService`, `employerRiskService`, `aiGovernanceService`, `scenarioGeneratorService`. |
| Repositories | `server/repositories/` | `customerRepository`, `loanRepository`, `payrollRepository`, `governanceRepository`. |
| Persistence | `server/db.ts`, `server/mockDb.ts` | PostgreSQL (`pg.Pool`) primary; atomic JSON file store fallback. |

### 6.3 Dual-mode persistence behaviour

- Mode is selected by env: `PSA_STORAGE_MODE=file` or `MOCK_DB_MODE=file`, or by the absence of
  `DATABASE_URL` → `file`. Otherwise the initial mode is `postgres`.
- The server probes PostgreSQL with `SELECT 1`. On any connection or query failure it **permanently
  switches** `storageMode` to `file`, logs a warning, and continues serving from
  `server/data/mock-db.json` (atomically written via a `.tmp` file plus rename).
- PostgreSQL mode stores most entities as `(id TEXT PRIMARY KEY, data JSONB)` — i.e. a hybrid
  relational/JSON document model. See `server/database/migrations/001_initial_schema.sql` for the
  fuller relational schema and `server/database/seed/001_seed_malawi_data.sql` for seed.
- Tables created by `ensureSchema()`: `loan_applications`, `repayments`, `customers`, `businesses`,
  `alerts`, `documents`, `audit_logs`, `users`, `conversations`, `messages`, plus indexes
  `idx_messages_conv_sent` and unique `idx_users_email_lower` on `lower(data->>'email')`.

### 6.4 Seeding on startup

`ensureSchema()` then `seedInitialData()` run before `app.listen`. Seeding is idempotent
(`ON CONFLICT DO NOTHING`, `??=`). It inserts demo loan applications, repayments, customers,
alerts, SME businesses, the four staff accounts, and an initial audit entry
`system.seed_demo_data`.

### 6.5 Front-end routing map

| Route | Guard | Component |
|---|---|---|
| `/` | guest only | `MarketingPage`; authenticated users redirect by role |
| `/login` | guest | `LoginPage` |
| `/staff/login` | guest | `LoginPage` (separate URL for staff) |
| `/register` | guest | `RegisterPage` |
| `/forgot-password` | public | `ForgotPasswordPage` |
| `/activate` | public | `CustomerActivationPage` |
| `/client/*` | `RequireAuth role="client"` | `ClientDataProvider` + `ClientLayout` |
| `/client/home` | — | `ClientHome` |
| `/client/loans` | — | `ClientLoans` |
| `/client/business` | — | `ClientBusiness` |
| `/client/calculator` | — | `ClientLoanCalculator` |
| `/client/documents` | — | `DocumentUpload` |
| `/client/advisor` | — | `ClientAIAdvisor` |
| `/client/messages` | — | `ClientMessages` |
| `/client/notifications` | — | `NotificationSettings` |
| `/client/contact` | — | `ContactPinnacle` |
| `/client/profile` | — | `ClientProfile` |
| `/staff/*` | `RequireAuth role="staff"` | `AdminDataProvider` + `AdminLayout` |
| `/staff/overview` | — | `AdminOverview` → `AdminReview` when an application is selected |
| `/staff/customers` | — | `AdminCustomers` |
| `/staff/sme` | — | `AdminSMEPortfolio` |
| `/staff/payroll` | — | `PayrollPage` |
| `/staff/reports` | — | `AdminReports` |
| `/staff/intelligence` | — | `AdminIntelligence` |
| `/staff/analytics` | — | `AdminAnalytics` |
| `/staff/team` | — | `AdminStaffManagement` |

An `ErrorBoundary` wraps the router and offers a "Reload Application" action on unhandled errors.

---

## 7. Functional Requirements by Module

> Status legend: **I** = Implemented (server + persistence), **P** = Partial, **S** = Simulated.
> Priority legend: **M** = Must, **S** = Should, **C** = Could.

### FR-AUTH — Authentication and Session Management

| ID | Requirement | Actor | Pri | Status |
|---|---|---|---|---|
| FR-AUTH-01 | The system shall authenticate staff and customers with email and password against stored credentials. | All | M | I |
| FR-AUTH-02 | The system shall hash passwords using bcrypt with a configurable cost factor (`BCRYPT_ROUNDS`, default 12). | System | M | I |
| FR-AUTH-03 | The system shall transparently upgrade legacy SHA-256 password hashes to bcrypt on successful login (rehash-on-login). | System | S | I |
| FR-AUTH-04 | The system shall issue a signed token on successful login: a 2-part HMAC-SHA256 token (`body.signature`) with a default TTL of 12 h (`AUTH_TOKEN_TTL_HOURS`), and additionally a standard 3-part JWT (HS256) via `/api/v1/auth/login`. | System | M | I |
| FR-AUTH-05 | The system shall set HTTP-only cookies `psa_access_token` (TTL 12 h) and `psa_refresh_token` (TTL 7 days), `secure` in production, `SameSite=Strict`. | System | M | I |
| FR-AUTH-06 | The system shall refresh an expired access token using the refresh token, issuing a new 900-second (15-minute) access token. | Customer, Staff | M | I |
| FR-AUTH-07 | The system shall clear both cookies on logout and report success. | All | M | I |
| FR-AUTH-08 | The system shall accept the token from either the `Authorization: Bearer` header or the `psa_access_token` / `psa_auth_token` cookie. | System | M | I |
| FR-AUTH-09 | The system shall return `401` for missing, malformed, tampered, or expired tokens in production. | System | M | I |
| FR-AUTH-10 | The system shall reject requests when the token is valid but the referenced user no longer exists. | System | M | I |
| FR-AUTH-11 | The system shall write an audit entry (`auth.login`) on every successful login including role metadata. | System | M | I |
| FR-AUTH-12 | The system shall enforce a "remember me" option: 12-hour session TTL normally, 30-day TTL when remembering (client-side session persistence). | Customer, Staff | S | I |
| FR-AUTH-13 | The client shall persist the session in `localStorage` (remembered) or `sessionStorage` (not remembered), clearing the other store. | System | S | I |
| FR-AUTH-14 | The system shall invalidate the session automatically when it has passed `expiresAt`. | System | M | I |
| FR-AUTH-15 | The client shall route users after login by role: `customer` → `/client`, all other roles → `/staff`. | System | M | I |
| FR-AUTH-16 | The system shall prevent a customer from accessing `/staff/*` and a staff member from accessing `/client/*`, redirecting to the appropriate login page. | System | M | I |
| FR-AUTH-17 | The system shall rate-limit public registration and password-reset verification to 10 and 5 attempts respectively per 15-minute window per IP+path. | System | S | I |
| FR-AUTH-18 | The system shall return `429` with the message "Too many requests. Please wait before trying again." when a rate limit is exceeded. | System | S | I |

**Notable implementation detail:** The API tolerates the literal password `Password123!` in
`/api/v1/auth/login` as a demo bypass when the hash comparison fails. This is a **simulated
credential path** and must be flagged as a security defect in the FRD (see §13, issue OI-02).

### FR-ONB — Registration and Customer Activation

| ID | Requirement | Actor | Pri | Status |
|---|---|---|---|---|
| FR-ONB-01 | The system shall allow a prospect to self-register as a customer with: full name, email, password, phone, address, national ID, date of birth, gender, employment type, employer, monthly income, customer type (`individual` or `sme`), security question, and security answer. | Guest | M | I |
| FR-ONB-02 | The system shall reject registration when required fields are missing, returning `400` with "Missing registration fields." | System | M | I |
| FR-ONB-03 | The system shall reject registration when the email already exists, returning `409` with "An account with this email already exists." | System | M | I |
| FR-ONB-04 | The system shall normalise the email to lower case and normalise security answers by trimming and lower-casing before hashing. | System | M | I |
| FR-ONB-05 | The system shall create the new user with role `customer`, notification preferences all enabled (`sms`, `email`, `push`), and a generated identifier of the form `client-<timestamp>`. | System | M | I |
| FR-ONB-06 | On registration the system shall generate a personalised demo data bundle (customer profile, loan applications, repayment schedule, alerts, and optionally an SME business) and persist it. | System | S | S |
| FR-ONB-07 | On registration the system shall send a welcome email and a welcome SMS when contact details are present. | System | S | I |
| FR-ONB-08 | On registration the system shall write an audit entry `auth.register`. | System | M | I |
| FR-ONB-09 | The system shall allow an existing customer to self-activate via a 3-step wizard: (1) verify eligibility, (2) OTP, (3) create password. | Customer | M | P |
| FR-ONB-10 | Step 1 shall match the applicant against imported customer records by national ID **or** by employee number plus registered phone number. | System | M | P |
| FR-ONB-11 | Step 1 shall return "No matching imported customer record found. Please verify your details with your PINACO loan officer." when no match exists. | System | M | P |
| FR-ONB-12 | Step 2 shall accept a 6-digit OTP. In the current build the accepted demo value is `123456`, or any 6-digit string. | System | M | S |
| FR-ONB-13 | Step 3 shall enforce a minimum password length of 8 characters and require both entries to match. | System | M | I |
| FR-ONB-14 | On successful activation the system shall register the account, mark the customer record `isActivated: true`, set status to `Active / Low Risk`, and link the `userId`. | System | M | I |
| FR-ONB-15 | The system shall direct the activated customer to the login page (`/login?registered=1`). | System | M | I |

**Simulated elements to flag:** OTP delivery is not real (fixed/stub value, client-side timer-based
verification). The eligibility match runs against client-side store data. Activation derives
placeholder profile values (`dob: '1990-01-01'`, `gender: 'Male'`, `monthlyIncome: '500000'`).

### FR-PWD — Password Recovery

| ID | Requirement | Actor | Pri | Status |
|---|---|---|---|---|
| FR-PWD-01 | The system shall accept an email and return the account's security question (`POST /api/auth/forgot-password/question`), returning `404` "Account not found." when unknown. | Guest | M | I |
| FR-PWD-02 | The system shall verify the security answer and, on success, issue a single-use reset token with a TTL of 15 minutes (`PASSWORD_RESET_TOKEN_TTL_MINUTES`). | Guest | M | I |
| FR-PWD-03 | Reset tokens shall be stored only as an HMAC-SHA256 hash, never in plaintext. | System | M | I |
| FR-PWD-04 | The system shall invalidate any previously issued reset token for the same user when a new one is issued, and shall prune expired tokens on each operation. | System | M | I |
| FR-PWD-05 | The system shall consume (delete) a reset token on first use and reject reused or expired tokens with `401` "Password reset token is invalid or expired." | System | M | I |
| FR-PWD-06 | The system shall enforce a minimum new-password length of 8 characters. | System | M | I |
| FR-PWD-07 | The system shall audit `auth.password_reset_verified` and `auth.password_reset` events. | System | M | I |
| FR-PWD-08 | The system shall allow an authenticated user to change their password by supplying the current and new password, rejecting an incorrect current password with `401` "Current password is incorrect." | Customer, Staff | M | I |
| FR-PWD-09 | The system shall compare secrets using constant-time comparison where raw hashes are compared and bcrypt comparison otherwise. | System | M | I |

### FR-CALC — Loan Calculator and Affordability Preview

| ID | Requirement | Actor | Pri | Status |
|---|---|---|---|---|
| FR-CALC-01 | The system shall present a 3-step loan application wizard: (1) loan type, amount and term; (2) verification documents; (3) confirm and submit. | Customer | M | I |
| FR-CALC-02 | The system shall offer exactly three loan types: **Personal** ("Education, health, or home needs"), **Business** ("Capital for growth and operations"), and **Emergency** ("Fast tracking for urgent cases"). | Customer | M | I |
| FR-CALC-03 | The system shall allow an amount between **K 10,000** and **K 500,000** in increments of **K 5,000**, defaulting to **K 250,000**. | Customer | M | I |
| FR-CALC-04 | The system shall offer terms of **3, 6, 12, and 24 months**, defaulting to **6 months**. | Customer | M | I |
| FR-CALC-05 | The system shall compute total interest as a flat **10% of principal** (`interest = amount × 0.10`). | System | M | I |
| FR-CALC-06 | The system shall compute the monthly repayment as `(amount + interest) / termMonths`. | System | M | I |
| FR-CALC-07 | The system shall recompute interest and monthly repayment immediately whenever amount or term changes. | System | M | I |
| FR-CALC-08 | The wizard shall display an "Estimated Summary" with monthly repayment (2 decimal places) and total interest. | Customer | M | I |
| FR-CALC-09 | The wizard shall block progression from step 2 to step 3 until both the Government Issued ID and the Proof of Physical Address have been uploaded, showing "Please upload the required verification documents to proceed." | Customer | M | I |
| FR-CALC-10 | Step 3 shall display a confirmation summary (loan category, amount, term, monthly instalment) and a disclaimer "Subject to final credit assessment". | Customer | M | I |
| FR-CALC-11 | The system shall support a richer loan product catalogue model (`LoanProduct`) with min/max amount, min/max term, monthly interest rate, target segment (`individual`/`sme`/`both`), and an active flag. | System | C | P |

**Inconsistency to record:** the calculator applies a flat 10% total interest with a
range of K 10,000–K 500,000, whereas the portfolio data model carries per-application
`interestRate` values of 3.2%–3.5% **per month** and amounts up to K 25,000,000. These two
pricing models are not reconciled in the code and must be resolved in the FRD (§13, OI-03).

### FR-APP — Loan Application Submission and Lifecycle

| ID | Requirement | Actor | Pri | Status |
|---|---|---|---|---|
| FR-APP-01 | The system shall create a loan application with: id, applicant name, email, phone, address, business name, monthly revenue, staff count, amount, term (months), interest rate, status, sector, date, score, risk level, repayment history, debt-to-income, years in business, notes, loan officer, and branch. | Customer, Officer | M | I |
| FR-APP-02 | The system shall reject an application create request without an `id`, returning `400` "Loan application id is required." | System | M | I |
| FR-APP-03 | The system shall notify the applicant by email and SMS that the application was received, quoting the application ID, amount, and current status. | System | M | I |
| FR-APP-04 | The system shall support exactly these application statuses: `Under Review`, `In Progress`, `Approved`, `Decline`, `Pending Doc`, `Disbursed`, `Reviewing`, `Completed`, `Active`, `Rejected`. | System | M | I |
| FR-APP-05 | The system shall allow an application to be updated (partial patch), merging supplied fields over the stored record. | Officer, Manager, Admin | M | I |
| FR-APP-06 | When an application's status changes, the system shall notify the applicant by email and SMS with the new status and, if present, the officer's note. | System | M | I |
| FR-APP-07 | The system shall record `disbursedAt` when a loan is disbursed and `completedAt` when fully repaid. | System | M | I |
| FR-APP-08 | The system shall compute a repayment schedule for any disbursed/approved/completed application: `termMonths` instalments, equal monthly principal (`round(principal / termMonths)`), and monthly interest on the **remaining balance** at the application's monthly rate. | System | M | I |
| FR-APP-09 | The system shall assign instalment identifiers as `REP-<applicationId>-<NN>`. | System | M | I |
| FR-APP-10 | The system shall mark an instalment `Paid` for completed loans, `Paid` for past instalments up to 60% of the term, `Upcoming` for the instalment immediately after that point, and `Scheduled` thereafter. | System | M | S |
| FR-APP-11 | The system shall record the assigned loan officer and originating branch (`branch-lil`, `branch-blt`, `branch-mzu`). | System | M | I |
| FR-APP-12 | The system shall write audit entries `loan_application.create` and `loan_application.update`. | System | M | I |

**Demo behaviour to flag:** the status-to-instalment mapping in FR-APP-10 is a deterministic
demo heuristic, not a real arrears calculation (the 60%-of-term rule). Flag it in the FRD as
simulated until replaced by a real payment-allocation engine.

### FR-DOC — Document Management

| ID | Requirement | Actor | Pri | Status |
|---|---|---|---|---|
| FR-DOC-01 | The system shall allow an authenticated user to upload a document as base64 with a category, storing the decoded file under `server/data/documents/<userId>/<docId>.bin`. | Customer, Staff | M | I |
| FR-DOC-02 | The system shall generate the document identifier as `doc-<timestamp>-<random>` and return id, file name, human-readable file size (KB/MB), upload timestamp, and category. | System | M | I |
| FR-DOC-03 | The system shall reject an upload missing the file or category with `400` "file (base64) and category are required." | System | M | I |
| FR-DOC-04 | The system shall allow an authenticated user to download a previously uploaded document by id, responding with `application/octet-stream` and an attachment disposition. | Customer, Staff | M | I |
| FR-DOC-05 | The system shall return `404` "Document not found." when the requested file does not exist on disk. | System | M | I |
| FR-DOC-06 | The system shall record document metadata records retrievable per user (`GET /api/documents/:userId`). | System | M | I |
| FR-DOC-07 | The system shall write audit entries `document.upload` and `document.upload_real` including category and file size. | System | M | I |
| FR-DOC-08 | The system shall support document categories: `Registration`, `Tax`, `Bank Statement`, `Financials`, `Permit`, `Other`. | System | M | I |
| FR-DOC-09 | The system shall track per-document verification status: `Pending`, `Uploaded`, `Verified`, `Rejected`. | Staff | M | P |
| FR-DOC-10 | The system shall extract structured fields from payslips, employment letters, and national IDs (applicant name, employer, employee number, monthly salary, issue date, position, national ID). | System | S | P |
| FR-DOC-11 | The system shall record an extraction result with document type, confidence percentage, source (`manual`, `ocr_provider`, `ai_provider`), verification status (`Matched`, `Mismatch`, `Pending`), and a list of mismatches. | System | S | P |
| FR-DOC-12 | The system shall compare extracted document data against the application and record field-level mismatches. | System | S | P |
| FR-DOC-13 | The system shall present a document verification panel to staff for review of uploaded documents. | Staff | M | I |

**Constraint to record:** the upload endpoint accepts `multipart/form-data` in a basic parser that
concatenates the raw body; the primary upload path is the JSON base64 endpoint. Document storage
is local filesystem only — no object storage, virus scanning, or encryption at rest.
### FR-CREDIT — Rules-Based Credit Assessment (Expert System)

| ID | Requirement | Actor | Pri | Status |
|---|---|---|---|---|
| FR-CREDIT-01 | The system shall compute a Financial Health Score (0–100) for an application, composed of a weighted sum of four sub-scores: repayment 40%, income stability 25%, debt 20%, business 15%. | System | M | I |
| FR-CREDIT-02 | The system shall grade the health score into tiers: `Excellent` ≥ 80, `Good` ≥ 65, `Fair` ≥ 45, `Poor` < 45, each with its own display colour. | System | M | I |
| FR-CREDIT-03 | The system shall map repayment history labels to sub-scores: `Perfect` 100, `Excellent` 92, `Good` 78, `Fair` 55, `Poor` 30, `New` 65, default 60. | System | M | I |
| FR-CREDIT-04 | The system shall map Debt-to-Income to a debt sub-score: ≤15% → 100, ≤25% → 88, ≤35% → 70, ≤45% → 50, >45% → 25. | System | M | I |
| FR-CREDIT-05 | The system shall compute income stability as the ratio of monthly revenue to the estimated monthly instalment (`amount / 12`): ≥8 → 100, ≥5 → 88, ≥3 → 75, ≥1.5 → 55, else 30. | System | M | I |
| FR-CREDIT-06 | The system shall compute a business sub-score by years in business (≥5 yrs 90, ≥3 yrs 75, ≥1 yr 58, else 40) plus a staff bonus of `min(staffCount × 1.5, 10)`, capped at 100. | System | M | I |
| FR-CREDIT-07 | The system shall assign an application to exactly one customer segment: `Premium Client`, `Reliable Borrower`, `SME Growth`, `Young Entrepreneur`, `High Risk`, or `Inactive`, per the rules in §8.3. | System | M | I |
| FR-CREDIT-08 | The system shall produce a credit recommendation of `Approve`, `Review`, or `Decline` with a risk level (`Low`/`Medium`/`High`) and a suggested action string. | System | M | I |
| FR-CREDIT-09 | The system shall enumerate positive signals and risk factors supporting the recommendation, including payroll deduction success rate, employer payroll reliability, prior completed loans, prior defaults, DTI, repayment history, and the 5×-income leverage test. | System | M | I |
| FR-CREDIT-10 | The system shall downgrade to `Decline` when two or more risk factors are present or the application score is below 55, and shall propose a counter-offer of 50% of the requested amount. | System | M | I |
| FR-CREDIT-11 | The system shall return `Approve` only when there are zero risk factors and the application score is at least 75. | System | M | I |
| FR-CREDIT-12 | The system shall evaluate four named rules for every application: **R01 Prime Salary Approval**, **R02 Standard Payroll Approval**, **R03 Elevated DTI Warning**, **R04 Income Shortfall**, each with a condition, a fired flag, and a confidence value (§8.4). | System | M | I |
| FR-CREDIT-13 | The system shall compute an overall confidence of 92 when score ≥ 80, 82 when score ≥ 65, otherwise 70. | System | M | I |
| FR-CREDIT-14 | The system shall propose a suggested amount equal to the requested amount when approving, otherwise 70% of the requested amount. | System | M | I |
| FR-CREDIT-15 | The system shall expose the recommendation via `GET /api/v1/loans/:id/recommendation`, returning verdict, risk level, score, fired signal names, confidence, and primary reason. | Officer | M | I |
| FR-CREDIT-16 | The system shall expose the full expert-system payload via `GET /api/intelligence/applications/:id`, returning `404` when the application does not exist. | Officer | M | I |
| FR-CREDIT-17 | The system shall display an AI transparency notice on every AI-derived output using the configured labels (§8.8) — including "Not a final credit decision. Officer review and verified documents are required." | System | M | I |
| FR-CREDIT-18 | The system shall present the credit verdict in an assessment modal to the customer (`AIAssessmentModal`) and in the officer review screen. | Both | M | I |

### FR-FRAUD — Fraud and Anomaly Detection

| ID | Requirement | Actor | Pri | Status |
|---|---|---|---|---|
| FR-FRAUD-01 | The system shall flag an application as **Excessive Leverage** when the requested amount exceeds 6× monthly revenue, with severity `High` when it exceeds 12×, otherwise `Medium`. | System | M | I |
| FR-FRAUD-02 | The system shall flag an application as **High Risk + Large Amount** when the risk level is `High` and the amount exceeds MWK 1,000,000, at severity `High`, noting manager sign-off is required. | System | M | I |
| FR-FRAUD-03 | Each fraud flag shall carry an id, application id, signal name, severity, and a human-readable description quoting the amounts involved. | System | M | I |
| FR-FRAUD-04 | The system shall expose fraud flags through the intelligence dashboard endpoint. | Staff | M | I |

### FR-REPAY — Repayments and Collections

| ID | Requirement | Actor | Pri | Status |
|---|---|---|---|---|
| FR-REPAY-01 | The system shall support exactly these repayment statuses: `Paid`, `Upcoming`, `Scheduled`, `Overdue`, `FAILED_DEDUCTION`. | System | M | I |
| FR-REPAY-02 | Each repayment shall record: id, application id, user id, instalment number, due date, amount, principal component, interest component, penalty (0 if on time), status, paid-at timestamp, and failure reason. | System | M | I |
| FR-REPAY-03 | The system shall allow an authenticated user to fetch repayments and update a repayment by id, returning `404` "Repayment not found." when absent. | Staff | M | I |
| FR-REPAY-04 | The system shall write an audit entry `repayment.update` recording the new status. | System | M | I |
| FR-REPAY-05 | The customer shall be able to view their repayment schedule and pay/confirm an instalment from the Loans screen. | Customer | M | I |
| FR-REPAY-06 | The system shall treat `FAILED_DEDUCTION` as an administrative exception that does **not** penalise the customer's credit profile. | System | M | I |
| FR-REPAY-07 | The system shall compute the collection rate as paid instalments ÷ (paid + overdue + failed deduction) × 100, with a stated business target of **98%+**. | System | M | I |
| FR-REPAY-08 | The system shall compute and display a total collected figure as the sum of `Paid` instalments. | System | M | I |

### FR-PAY — Payroll Deduction Processing

| ID | Requirement | Actor | Pri | Status |
|---|---|---|---|---|
| FR-PAY-01 | The system shall allow staff to upload a payroll deduction file (CSV) for a given employer and month. | Officer, Manager, Admin | M | I |
| FR-PAY-02 | The system shall create a payroll batch record with: id, upload timestamp, uploading user, file name, total records, matched count, applied count, failed count, month, and status (`Processing`, `Completed`, `Partial`). | System | M | I |
| FR-PAY-03 | The system shall match each payroll record to a customer by employee number, producing a record status of `Matched`, `Unmatched`, `Applied`, or `Failed`. | System | M | I |
| FR-PAY-04 | The system shall record a failure reason on unmatched/failed records, e.g. "Employee number did not match an active customer record." | System | M | I |
| FR-PAY-05 | The system shall link an applied payroll record to the repayment it settled via `repaymentId`. | System | M | I |
| FR-PAY-06 | The system shall provide a preview step before applying a payroll file, and a history view of previous batches. | Officer | M | I |
| FR-PAY-07 | The system shall expose payroll batches and exceptions through the API: `GET /api/v1/payroll/batches`, `GET /api/v1/payroll/exceptions` (records whose status is `Unmatched` or `Failed`). | Staff | M | I |
| FR-PAY-08 | The system shall expose employer-level performance via `GET /api/v1/payroll/employer/:name/performance`. | Staff | M | I |
| FR-PAY-09 | The system shall compute per-employer performance: employee count, active loans, total disbursed, total collected, successful deduction rate, failed deduction rate, average salary, risk rating (`Low`/`Medium`/`High`), and risk trend (`Improving`/`Stable`/`Deteriorating`). | System | M | I |
| FR-PAY-10 | The system shall rate employer risk by failed deduction rate: > 20% → High, > 8% → Medium, otherwise Low; and mark the trend `Deteriorating` when the failed rate exceeds 10%. | System | M | I |
| FR-PAY-11 | The system shall derive average salary as total monthly revenue of mapped employees ÷ employee count. | System | M | I |
| FR-PAY-12 | The system shall provide a seeded demonstration dataset containing at least two batches (Ministry of Education 2024-07 with 120 records, Malawi Police 2024-07 with 85 records) and sample employee records. | System | S | S |

**Note for the FRD:** employer performance in `computeEmployerPerformance` groups by the
customer/application `sector` field when no payroll record exists for that employer, so
"sector" and "employer" are conflated in that path. Record as OI-04.

### FR-C360 — Customer 360 Intelligence

| ID | Requirement | Actor | Pri | Status |
|---|---|---|---|---|
| FR-C360-01 | The system shall provide a customer directory with name, location, sector, active loan count, status, risk level, icon type, phone, email, join date, customer type, employee number, national ID, activation flag, and data source. | Staff | M | I |
| FR-C360-02 | The system shall compute a Customer Relationship Score (0–100) with a configurable weight configuration (§8.5) and a tier of `High` (≥85), `Good` (≥70), `Fair` (≥50), or `Poor` (<50). | System | M | I |
| FR-C360-03 | The system shall return a score of `-1` with tier `New Customer` and the factor "No previous PINACO loan history — new customer profile" when the customer has no applications and no repayments. | System | M | I |
| FR-C360-04 | The system shall provide a risk timeline reconstructing events from real records only: application submitted, loan disbursed, instalment paid, payroll deduction exception, instalment overdue — each with date, risk level, score, trigger event, and details. | System | M | I |
| FR-C360-05 | The system shall return a single entry "No historical risk events available." (risk level `New`, score `-1`) when no events exist. | System | M | I |
| FR-C360-06 | The system shall compute Customer Lifetime Value as interest generated plus MWK 50,000 per completed loan, alongside total borrowed, completed loans, relationship months, and a plain-language meaning sentence. | System | M | I |
| FR-C360-07 | Where an instalment has no recorded interest, the system shall estimate interest as 15% of the instalment amount for LTV purposes. | System | M | I |
| FR-C360-08 | The system shall expose relationship score, timeline, and LTV via `GET /api/v1/customers/:id/intelligence`, returning `404` for an unknown customer. | Staff | M | I |
| FR-C360-09 | The system shall present a Customer 360 modal consolidating profile, applications, repayments, and intelligence for the selected customer. | Staff | M | I |
| FR-C360-10 | The system shall allow staff to add a customer record via `POST /api/customers` (requires an id) and to import customers in bulk via an import modal. | Staff | M | I |

### FR-TOPO — Top-Up and Repeat-Borrower Opportunities

| ID | Requirement | Actor | Pri | Status |
|---|---|---|---|---|
| FR-TOPO-01 | The system shall compute a top-up recommendation per customer with existing applications, classified as `Top-Up`, `Limit Increase`, `Standard`, or `Requires Assessment`. | System | M | I |
| FR-TOPO-02 | The system shall calculate the remaining balance as the sum of all unpaid instalments and the potential top-up as **5× the customer's verified monthly revenue**. | System | M | I |
| FR-TOPO-03 | The system shall estimate the post-top-up instalment as `(remaining balance + potential top-up) / termMonths`, using the application's term when present or defaulting to 12 months with an explicit disclosure note. | System | M | I |
| FR-TOPO-04 | The system shall recommend `Top-Up` with confidence 94 when post-top-up DTI ≤ 40% and the customer has completed loans or is an existing customer. | System | M | I |
| FR-TOPO-05 | The system shall recommend `Requires Assessment` with confidence 60 when post-top-up DTI exceeds 45%, or confidence 45 when income information is unavailable. | System | M | I |
| FR-TOPO-06 | The system shall default to `Standard` with confidence 80 in all other cases, and record human-readable reasons for each recommendation. | System | M | I |
| FR-TOPO-07 | The system shall never assume a salary when income data is missing; it shall instead require assessment. | System | M | I |
| FR-TOPO-08 | The system shall expose top-up opportunities via `GET /api/v1/intelligence/top-up-opportunities` and present them to staff in a dedicated card on the customer/overview screens. | Staff | M | I |

### FR-EMP — Employer Risk Intelligence

| ID | Requirement | Actor | Pri | Status |
|---|---|---|---|---|
| FR-EMP-01 | The system shall build a monthly deduction success series per employer from payroll records, treating `Matched` and `Applied` as successful. | System | M | I |
| FR-EMP-02 | The system shall require at least **6 months** of payroll data before producing a risk forecast; otherwise it shall return `hasSufficientData: false`, predicted risk `Medium`, and the reason "Insufficient historical data (less than 6 months of payroll records available; found N month(s))." | System | M | I |
| FR-EMP-03 | Where sufficient data exists, the system shall report the current collection rate, the last six monthly rates as a trend, and a predicted risk of `Low` or, when the rate has fallen by more than 3 percentage points across the window, `High` if the current rate is below 90% or `Medium` otherwise. | System | M | I |
| FR-EMP-04 | The system shall expose all employer predictions via `GET /api/v1/intelligence/employer-predictions` and present them in an employer intelligence card. | Staff | M | I |
| FR-EMP-05 | The system shall compute employer performance metrics including average salary, successful and failed deduction rates, risk rating, and risk trend, and present them to staff. | Staff | M | I |

### FR-GOV — AI Governance and Override

| ID | Requirement | Actor | Pri | Status |
|---|---|---|---|---|
| FR-GOV-01 | The system shall record an officer decision against a loan application, capturing: application id, AI recommendation, AI confidence, AI signals, officer id, officer decision, override reason, and decision timestamp. | Officer, Manager, Admin | M | I |
| FR-GOV-02 | The system shall set `isOverride` to true whenever the officer's decision differs from the AI recommendation. | System | M | I |
| FR-GOV-03 | The system shall default the override reason to "Manual officer review override" when an override is recorded without a stated reason. | System | M | I |
| FR-GOV-04 | The system shall reject a decision record without an application id ("Application ID is required") or without an officer decision ("Officer decision is required"), returning `400`. | System | M | I |
| FR-GOV-05 | The system shall generate decision identifiers as `dh-<applicationId>-<timestamp>`. | System | M | I |
| FR-GOV-06 | The system shall write an audit entry — `governance.ai_recommendation_override` or `governance.ai_recommendation_accepted` — containing the recommendation transition and the reason. | System | M | I |
| FR-GOV-07 | The system shall provide decision history retrievable in full or filtered by application id (`GET /api/v1/governance/history?applicationId=`). | Staff | M | I |
| FR-GOV-08 | The system shall enforce role authorisation of `officer`, `manager`, or `admin` on the override endpoint. | System | M | P |
| FR-GOV-09 | The system shall retain the full AI signal list with each decision for later audit. | System | M | I |

**Gap to record:** the decision-history store is an in-memory array in
`server/repositories/governanceRepository.ts`. It is **not persisted** and is lost on restart,
despite the API and database documentation describing a `decision_history` table. Record as
OI-05 — a significant compliance gap for an auditability requirement.

### FR-MSG — Messaging and Conversations

| ID | Requirement | Actor | Pri | Status |
|---|---|---|---|---|
| FR-MSG-01 | The system shall list all conversations for the authenticated user, each with the last message, its sender, its timestamp, and an unread count. | Customer, Staff | M | I |
| FR-MSG-02 | The system shall create or return an existing conversation between a customer and a selected staff member. | Customer | M | I |
| FR-MSG-03 | The system shall reject conversation creation without a `staffId`, returning `400` "staffId is required." | System | M | I |
| FR-MSG-04 | The system shall return messages for a conversation in ascending chronological order, limited to 100, optionally filtered by an `after` cursor for polling. | Both | M | I |
| FR-MSG-05 | The system shall reject an empty message body, returning `400` "Message content is required." | Both | M | I |
| FR-MSG-06 | The system shall record the sender as a reader of their own message and update the conversation's last-message timestamp on send. | System | M | I |
| FR-MSG-07 | The system shall mark messages read for the authenticated user, responding `204`, and never mark the user's own messages read. | Both | M | I |
| FR-MSG-08 | The client shall poll for new messages and maintain an unread badge on the client inbox navigation item. | System | M | I |
| FR-MSG-09 | The system shall present staff/advisor conversations in the client inbox alongside alert notifications. | Customer | M | I |

### FR-AI — AI Advisor Chat

| ID | Requirement | Actor | Pri | Status |
|---|---|---|---|---|
| FR-AI-01 | The system shall provide a conversational AI advisor to customers, bound to the customer's profile and application context. | Customer | S | I |
| FR-AI-02 | The system shall call a server-side Gemini endpoint (`POST /api/ai/advisor/chat`) using the configured model (`GEMINI_MODEL`, default `gemini-2.0-flash`), passing a system instruction and conversation history. | System | S | I |
| FR-AI-03 | The system shall require both `message` and `systemContext`, returning `400` when either is missing. | System | S | I |
| FR-AI-04 | The system shall return `503` with "GEMINI_API_KEY is not configured on the backend." when no API key is present. | System | S | I |
| FR-AI-05 | The system shall support a browser-side Gemini fallback when no API backend is configured. | System | C | I |
| FR-AI-06 | The system shall display the advisor disclaimer: "Guidance only, based on available profile data. It does not approve loans or replace staff review." | System | M | I |
| FR-AI-07 | The system shall audit both successful (`ai.advisor_chat`) and failed (`ai.advisor_chat_failed`) advisor calls, recording application id, prompt length, and error message. | System | M | I |
| FR-AI-08 | The system shall return a fallback message ("I apologise, I could not generate a response. Please try again.") when the model returns no text. | System | S | I |

**Important distinction for the FRD:** there are **two** "AI" capabilities with different
natures. (a) The **credit engine** is deterministic rules and is the decision-support mechanism.
(b) The **advisor chat** is a large language model used only for conversational guidance, never
for decisions. The FRD must not conflate them.
### FR-SME — SME Business Portfolio

| ID | Requirement | Actor | Pri | Status |
|---|---|---|---|---|
| FR-SME-01 | The system shall maintain SME business records with: id, owning customer id and name, business name, registration number, sector, industry category, location, verification status, relationship manager, risk level, annual revenue, monthly revenue, employee count, years in business, health score, owners, documents, timeline, and financing requests. | System | M | I |
| FR-SME-02 | The system shall support exactly these business verification statuses: `Draft`, `Pending Verification`, `Verified`, `Needs Documents`. | System | M | I |
| FR-SME-03 | The system shall support exactly these business document statuses: `Pending`, `Uploaded`, `Verified`, `Rejected`. | System | M | I |
| FR-SME-04 | The system shall support exactly these financing request statuses: `Draft`, `Submitted`, `Under Review`, `Approved`, `Declined`. | System | M | I |
| FR-SME-05 | The system shall support business owner roles: `Primary Owner`, `Co-owner`, `Director`, `Guarantor`, each with an ownership percentage and a linked-at date. | System | M | I |
| FR-SME-06 | The system shall record business timeline events of type `registration`, `document`, `financing`, `verification`, or `relationship`, each with a date, title, and detail. | System | M | I |
| FR-SME-07 | The system shall allow staff to retrieve all businesses, retrieve businesses by customer, and update a business, returning `404` "SME business not found." when absent. | Staff | M | I |
| FR-SME-08 | The system shall allow a document, a timeline event, or a financing request to be appended to a business, each de-duplicated by id, returning `201` with the created item. | Staff | M | I |
| FR-SME-09 | The system shall reject a business document, timeline event, or financing request missing its mandatory fields with an explicit `400` message. | System | M | I |
| FR-SME-10 | The system shall resolve a business lookup by customer id accepting both `cust-<n>` and bare `<n>` identifier forms. | System | M | I |
| FR-SME-11 | The customer shall be able to view their own SME business profile, documents, timeline, and financing requests from the Business screen. | Customer | M | I |
| FR-SME-12 | The system shall write audit entries for `sme_business.create`, `sme_business.update`, `sme_business.document_add`, `sme_business.timeline_add`, and `sme_business.financing_request`. | System | M | I |

### FR-RPT — Reporting and Analytics

| ID | Requirement | Actor | Pri | Status |
|---|---|---|---|---|
| FR-RPT-01 | The system shall provide a dashboard summary containing total disbursed, total collected, collection rate, total interest earned, pending count, active count, total applications, monthly disbursements, sector breakdown, and recent audit activity. | Staff | M | I |
| FR-RPT-02 | The system shall define total disbursed as the sum of amounts for applications in status `Disbursed` or `Approved` (plus `Completed` in the file store). | System | M | I |
| FR-RPT-03 | The system shall define total collected as the sum of `Paid` instalments. | System | M | I |
| FR-RPT-04 | The system shall compute collection rate as total collected divided by total disbursed times 100, to two decimal places, returning 0.00 when nothing has been disbursed. | System | M | I |
| FR-RPT-05 | The system shall estimate total interest earned as **10% of total disbursed** (flat demo assumption). | System | M | S |
| FR-RPT-06 | The system shall compute pending count as applications in `Under Review`, `In Progress`, or `Reviewing`, and active count as applications in `Approved` or `Disbursed`. | System | M | I |
| FR-RPT-07 | The system shall produce a 12-month disbursement time series keyed by `YYYY-MM`, labelled with the short month name. | System | M | I |
| FR-RPT-08 | The system shall produce a sector breakdown with disbursed amount, application count, and percentage of total, sorted descending by disbursed amount. | System | M | I |
| FR-RPT-09 | The system shall expose the dashboard summary at `GET /api/dashboard/summary` (authenticated). | Staff | M | I |
| FR-RPT-10 | The system shall expose an intelligence dashboard at `GET /api/intelligence/dashboard` returning fraud flags, segment distribution, sector trends, data-mining insights, and the portfolio timeline. | Staff | M | I |
| FR-RPT-11 | The system shall compute dynamic portfolio timeline points per month: disbursed, collected, new customers, and defaults (count of overdue instalments), limited to the most recent 12 months. | System | M | I |
| FR-RPT-12 | The system shall compute sector trends with default rate, average repayment percentage, total disbursed, growth, risk rating (`High` above 10% default, `Medium` above 5%, otherwise `Low`), and a sector insight sentence. | System | M | I |
| FR-RPT-13 | The system shall compute segment distribution counts and percentages across the six customer segments. | System | M | I |
| FR-RPT-14 | The system shall generate data-mining insights with category (`Pattern`, `Anomaly`, `Trend`, `Alert`), title, description, impact (`Positive`, `Negative`, `Neutral`), confidence, and for analytical rigour a data source, calculation method, and business meaning. | System | M | I |
| FR-RPT-15 | The system shall generate at minimum these four insights: portfolio collection rate versus the 98% target; sector default-rate gap between best and worst sectors; repeat-borrower versus new-applicant overdue comparison; and payroll exception count with the worst-performing employer. | System | M | I |
| FR-RPT-16 | The system shall compute officer quality metrics: reviewed count, approved count, declined count, approval ratio, average processing time in hours, repayment performance of the approved cohort, document rejection rate, and default count. | System | M | I |
| FR-RPT-17 | The system shall compute average processing time as the mean elapsed hours between an application's date and its `disbursedAt`, ignoring non-positive intervals, and report 0 when none exist. | System | M | I |
| FR-RPT-18 | The system shall compute the approved-cohort repayment performance as paid divided by the sum of paid, overdue, and failed deductions, to one decimal place, defaulting to 100 when the officer has approvals but no due instalments. | System | M | I |
| FR-RPT-19 | The system shall compute the document rejection rate from audit-log events matching document-rejection actions or failed document operations. | System | M | P |
| FR-RPT-20 | The system shall provide CSV export for three report types — `portfolio`, `audit`, `repayments` — with quoted fields and doubled embedded quotes, returning the appropriate `Content-Disposition` filename. | Staff | M | I |
| FR-RPT-21 | The export endpoint shall reject a `customer` role with `403` "Staff access required." and an unknown type with `400` "Unknown report type. Use: portfolio, audit, repayments." | System | M | I |
| FR-RPT-22 | The system shall present reports and charts to staff using a charting library (`recharts`). | Staff | M | I |

**Note to flag:** the officer quality metrics are computed against a hard-coded list of three
officers (`staff-002` Chisomo Banda, `staff-003` Grace Phiri, `staff-001` Thoko Kamanga).
Adding a staff member does not add them to the scorecard. Record as OI-06.

### FR-ADMIN — Staff Administration

| ID | Requirement | Actor | Pri | Status |
|---|---|---|---|---|
| FR-ADMIN-01 | The system shall list all non-customer users, sorted by full name, excluding password and security-answer hashes from the response. | Admin | M | I |
| FR-ADMIN-02 | The system shall allow an administrator to create a staff account with full name, email, password, phone, and staff title, returning `201` with the created account (sensitive fields removed). | Admin | M | I |
| FR-ADMIN-03 | The system shall assign the role `manager` when the staff title is `Risk Manager` or `Branch Manager`, otherwise `loan_officer`. | System | M | I |
| FR-ADMIN-04 | The system shall reject a staff creation missing full name, email, or password with `400` "fullName, email and password are required." and a duplicate email with `409`. | System | M | I |
| FR-ADMIN-05 | The system shall generate staff identifiers as `staff-<timestamp>` and national IDs as `STAFF-<timestamp>`. | System | M | I |
| FR-ADMIN-06 | The system shall allow an administrator to update a staff account and return the updated record, returning `404` "Staff user not found" when the id is unknown. | Admin | M | I |
| FR-ADMIN-07 | The system shall restrict all staff-administration endpoints to the `admin` role, returning `403` "Admin access required." otherwise. | System | M | I |
| FR-ADMIN-08 | The system shall present a staff management screen to administrators listing the team with their roles and titles. | Admin | M | I |
| FR-ADMIN-09 | The system shall provide a staff account creation form with role-appropriate fields. | Admin | M | I |

### FR-SCEN — Demo Scenario Management

| ID | Requirement | Actor | Pri | Status |
|---|---|---|---|---|
| FR-SCEN-01 | The system shall maintain an active demo scenario with these fields: active scenario name, portfolio health percentage, collection rate percentage, delinquent loan count, payroll match percentage, and a description. | System | M | S |
| FR-SCEN-02 | The system shall support exactly three scenarios: `Healthy Portfolio`, `High Default Risk`, and `Payroll Crisis`. | System | M | S |
| FR-SCEN-03 | `Healthy Portfolio` shall set portfolio health 96.5%, collection rate 98.2%, delinquent loans 2, payroll match 98.5%, described as "Baseline operational state — high collection reliability and low portfolio risk." | System | M | S |
| FR-SCEN-04 | `High Default Risk` shall set portfolio health 72.0%, collection rate 81.4%, delinquent loans 14, payroll match 88.0%, described as "Stress scenario — elevated borrower defaults across retail and transport sectors." | System | M | S |
| FR-SCEN-05 | `Payroll Crisis` shall set portfolio health 64.5%, collection rate 68.0%, delinquent loans 22, payroll match 58.2%, described as "Systemic exception scenario — major public employer deduction delays across 2 key ministries." | System | M | S |
| FR-SCEN-06 | The system shall default to `Healthy Portfolio` on startup and expose the current scenario via `GET /api/v1/scenarios/active` and `GET /api/v1/governance/scenarios/active`. | System | M | S |
| FR-SCEN-07 | The system shall allow switching scenarios via `POST /api/v1/scenarios/switch` and `POST /api/v1/governance/scenarios/switch`, rejecting an unknown scenario with `400` "Invalid scenario type". | Staff | M | S |
| FR-SCEN-08 | The system shall provide a reset operation returning the state to `Healthy Portfolio`. | System | C | I |
| FR-SCEN-09 | The scenario endpoints shall not require authentication so that demonstrations can be switched freely. | System | M | S |

**Critical caveat for the FRD:** the scenario service changes **only** a stored descriptive state
object. It does **not** mutate the underlying portfolio data. Any requirement implying that
scenario switching changes reporting figures must be marked simulated.

### FR-AUD — Audit Trail

| ID | Requirement | Actor | Pri | Status |
|---|---|---|---|---|
| FR-AUD-01 | Each audit entry shall record: id, occurrence timestamp, actor id, actor name, actor role, action, entity type, entity id, outcome (`success`, `failure`, `info`), summary, and optional metadata. | System | M | I |
| FR-AUD-02 | The system shall derive the actor from the authenticated principal, falling back to `X-Actor-Id` / `X-Actor-Name` / `X-Actor-Role` headers, and finally to a `system` identity. | System | M | I |
| FR-AUD-03 | The system shall record audit entries for at minimum: `auth.register`, `auth.login`, `auth.password_reset_verified`, `auth.password_reset`, `auth.profile_update`, `loan_application.create`, `loan_application.update`, `repayment.update`, `customer.create`, `sme_business.create`, `sme_business.update`, `sme_business.document_add`, `sme_business.timeline_add`, `sme_business.financing_request`, `alert.create`, `document.upload`, `document.upload_real`, `governance.ai_recommendation_override`, `governance.ai_recommendation_accepted`, `ai.advisor_chat`, `ai.advisor_chat_failed`, and `system.seed_demo_data`. | System | M | I |
| FR-AUD-04 | The system shall allow audit entries to be listed (most recent first) and created explicitly through the API. | Staff | M | I |
| FR-AUD-05 | The system shall reject an audit entry without an id, returning `400` "Audit entry id is required." | System | M | I |
| FR-AUD-06 | The system shall retain audit entries with their metadata for reporting and export. | System | M | I |
| FR-AUD-07 | The client shall maintain a local audit trail helper for user-initiated events, capped at a configurable recent-event limit (default 20). | System | P | I |

**Gap to record:** audit entries are never deleted or archived by any code path, and there is no
retention policy, tamper-evidence (hash chaining), or separation of duties around audit
administration. The FRD should state an explicit retention requirement rather than inherit the
current absence.

### FR-NOTIF — Notifications and Preferences

| ID | Requirement | Actor | Pri | Status |
|---|---|---|---|---|
| FR-NOTIF-01 | The system shall send transactional SMS via the Africa's Talking REST API using `AFRICASTALKING_USERNAME`, `AFRICASTALKING_API_KEY`, and optional `AFRICASTALKING_SENDER_ID`, selecting the sandbox or live endpoint based on the username. | System | M | I |
| FR-NOTIF-02 | The system shall send transactional email via the SendGrid v3 API using `SENDGRID_API_KEY`, `SENDGRID_FROM_EMAIL`, and `SENDGRID_FROM_NAME`. | System | M | I |
| FR-NOTIF-03 | When either provider is unconfigured, the system shall log an "Unconfigured Key Notice" containing the intended recipient and content, and shall return false without throwing. | System | M | I |
| FR-NOTIF-04 | Notification failures shall never block the originating business transaction or surface as API errors to the caller. | System | M | I |
| FR-NOTIF-05 | The system shall send notifications for: welcome on registration, application submitted, and application status change. | System | M | I |
| FR-NOTIF-06 | The system shall support in-app alert notifications of type `critical`, `approval`, `opportunity`, `info`, or `warning`, each with title, description, date, read flag, optional background image, optional action label, and optional deep-link route. | System | M | I |
| FR-NOTIF-07 | The system shall scope an alert to a single user when `userId` is set, otherwise treat it as visible to all administrators. | System | M | I |
| FR-NOTIF-08 | The system shall allow alerts to be listed and created through the API, rejecting an alert without an id with `400` "Alert id is required." | Staff | M | I |
| FR-NOTIF-09 | The customer shall be able to configure SMS, email, and push notification preferences; the preferences shall be persisted on the user record. | Customer | M | P |
| FR-NOTIF-10 | The system shall display unread critical and approval alert counts as a badge in the client portal. | Customer | S | I |

**Gap to record:** notification preferences are stored but **not enforced** by the notification
senders — SMS and email are dispatched regardless of the user's preference flags, and the push
channel has no implementation. Record as OI-07.

### FR-MKT — Public Site and Contact

| ID | Requirement | Actor | Pri | Status |
|---|---|---|---|---|
| FR-MKT-01 | The system shall present a public marketing landing page to unauthenticated visitors describing the product, with a call to action entering the application. | Guest | M | I |
| FR-MKT-02 | The system shall redirect an authenticated visitor away from the marketing page to their role-appropriate portal. | System | M | I |
| FR-MKT-03 | The system shall provide a contact/enquiry screen for customers to reach the institution. | Customer | S | I |
| FR-MKT-04 | The system shall provide a notification-settings screen reachable from the profile and from the home screen quick actions. | Customer | M | I |
| FR-MKT-05 | The customer home screen shall present quick actions routing to apply for a loan, view loans, upload documents, contact the institution, and open the AI advisor. | Customer | M | I |
---

## 8. Business Rules and Algorithm Catalogue

> These are the authoritative numeric rules. The FRD must reproduce them exactly and
> cross-reference them from the requirements above.

### 8.1 BR-01 Financial Health Score

Composite score, rounded to an integer:

```
composite = round( repaymentScore × 0.40
                 + incomeScore     × 0.25
                 + debtScore       × 0.20
                 + businessScore   × 0.15 )
```

Sub-score lookups:

| Factor | Input label | Score |
|---|---|---|
| Repayment | `Perfect` | 100 |
| Repayment | `Excellent` | 92 |
| Repayment | `Good` | 78 |
| Repayment | `Fair` | 55 |
| Repayment | `Poor` | 30 |
| Repayment | `New` | 65 |
| Repayment | anything else / unparseable | 60 |

| Factor | Debt-to-Income | Score |
|---|---|---|
| Debt | ≤ 15% | 100 |
| Debt | ≤ 25% | 88 |
| Debt | ≤ 35% | 70 |
| Debt | ≤ 45% | 50 |
| Debt | > 45% | 25 |
| Debt | unparseable | 70 |

| Factor | Monthly revenue ÷ (amount / 12) | Score |
|---|---|---|
| Income | ≥ 8 | 100 |
| Income | ≥ 5 | 88 |
| Income | ≥ 3 | 75 |
| Income | ≥ 1.5 | 55 |
| Income | otherwise | 30 |

| Factor | Value | Score |
|---|---|---|
| Business base | years ≥ 5 | 90 |
| Business base | years ≥ 3 | 75 |
| Business base | years ≥ 1 | 58 |
| Business base | otherwise | 40 |
| Business staff bonus | `min(staffCount × 1.5, 10)` added to base, capped at 100 | — |

**Tiers:** `Excellent` ≥ 80 (green `#16a34a`), `Good` ≥ 65 (amber `#d97706`),
`Fair` ≥ 45 (orange `#ea580c`), `Poor` < 45 (red `#dc2626`).

### 8.2 BR-02 Credit Recommendation Decision Logic

Risk factors are collected from: failed payroll deductions; employer payroll reliability
below 80% (with 5 or more employer records); prior declined/defaulted loans; DTI above 40%;
repayment history `Poor` or `Fair`; requested amount above 5× monthly revenue; health
composite below 50.

Positive signals are collected from: government/institutional sector employment
(keywords `public`, `education`, `health`, `government`, `civil`, `university`);
repayment history `Perfect`/`Excellent`/`Good`; DTI ≤ 30%; health composite ≥ 75;
imported (existing) customer; payroll deduction success ≥ 95%; employer reliability ≥ 95%
with 5 or more records; 1 or 3 or more completed prior loans; relationship of 2+ years.

Decision table:

| Condition | Recommendation | Risk level | Suggested action |
|---|---|---|---|
| 0 risk factors AND score ≥ 75 | `Approve` | `Low` | Approve the requested amount; fast-track payroll deduction agreement with the employer. |
| 2 or more risk factors OR score < 55 | `Decline` | `High` | Decline in current form; counter-offer 50% of the requested amount with a mandatory guarantor and employer confirmation. |
| otherwise | `Review` | `Medium` | Request updated payslip, latest 3-month bank statement, and employer payroll deduction confirmation letter. |

**Payroll deduction success rate** = `round(paid ÷ (paid + overdue + FAILED_DEDUCTION) × 100)`,
computed only over instalments attached to the application, and omitted when there are no due
instalments.
**Employer payroll reliability** = `round(successful ÷ total × 100)` over payroll records matched
to the applicant's sector/employer, computed only when at least 5 records exist.

### 8.3 BR-03 Customer Segmentation (evaluated in order, first match wins)

| # | Condition | Segment |
|---|---|---|
| 1 | score ≥ 88 AND repaymentHistory = `Perfect` | `Premium Client` |
| 2 | score ≥ 80 AND DTI ≤ 25 | `Reliable Borrower` |
| 3 | score ≥ 65 AND years in business ≥ 3 AND staff count ≥ 5 | `SME Growth` |
| 4 | years in business < 2 AND score ≥ 60 | `Young Entrepreneur` |
| 5 | score < 50 OR DTI > 40 | `High Risk` |
| 6 | fallback | `Inactive` |

**Segment colours:** Premium Client `#7c3aed`, Reliable Borrower `#16a34a`,
SME Growth `#2563eb`, Young Entrepreneur `#d97706`, High Risk `#dc2626`,
Inactive `#9ca3af`.

### 8.4 BR-04 Expert System Rules

| Rule | Name | Condition | Confidence |
|---|---|---|---|
| R01 | Prime Salary Approval | `score ≥ 80 AND repaymentHistory in {Excellent, Perfect} AND DTI < 30%` | 95 |
| R02 | Standard Payroll Approval | `score ≥ 65 AND DTI < 40%` | 85 |
| R03 | Elevated DTI Warning | `score < 55 OR DTI > 40%` | 90 |
| R04 | Income Shortfall | `monthlyRevenue < (amount / 12) × 2` | 87 |

**Overall confidence:** 92 if score ≥ 80; 82 if score ≥ 65; else 70.
**Suggested amount:** requested amount if verdict is `Approve`, else 70% of the requested amount.
**Primary reason:** first positive signal, else first risk factor, else "Requires officer manual review".

### 8.5 BR-05 Customer Relationship Score (configurable)

Default configuration:

| Parameter | Weight |
|---|---|
| `completedLoanWeight` | 15 |
| `perfectRepaymentWeight` | 20 |
| `stableEmployerWeight` | 15 |
| `relationshipDurationWeight` | 10 |
| `failedDeductionPenalty` | 15 |
| `defaultPenalty` | 30 |
| `highDtiPenalty` | 15 |

Algorithm: start at 50; add `min(completedLoans × 15, 30)`; add 20 when there are due
instalments and zero overdue and zero failed deductions; add 15 when the sector is
public/institutional; add 10 when the relationship is 12 months or longer; subtract 15 per failed
deduction; subtract 30 per overdue instalment; subtract 15 when DTI is above 40. Clamp to 0–100.

**New-customer rule:** no applications and no repayments gives score `-1`, tier `New Customer`.

**Tiers:** `High` ≥ 85, `Good` ≥ 70, `Fair` ≥ 50, `Poor` < 50.

### 8.6 BR-06 Top-Up Eligibility

| Condition | Outcome | Confidence |
|---|---|---|
| Monthly revenue missing or ≤ 0 | `Requires Assessment` | 45 |
| postTopUpDti ≤ 40 AND (completed loans > 0 OR customer type = `Existing`) | `Top-Up` | 94 |
| postTopUpDti > 45 | `Requires Assessment` | 60 |
| otherwise | `Standard` | 80 |

`potentialTopUp = round(monthlyRevenue × 5)`.
`postTopUpDti = round(((remainingBalance + potentialTopUp) / termMonths) / monthlyRevenue × 100)`.
Term defaults to 12 months with an explicit disclosure when the application has no term.

### 8.7 BR-07 Employer Risk and Performance

| Metric | Rule |
|---|---|
| Employer risk rating | failed deduction rate > 20% gives High; > 8% gives Medium; else Low |
| Employer risk trend | failed rate > 10% gives `Deteriorating`; else `Stable` |
| Forecast minimum data | 6 months of payroll records; below this, `hasSufficientData = false`, risk `Medium` |
| Forecast risk | if the 6-month rate fell by more than 3 points: `High` when current rate < 90%, else `Medium`; otherwise `Low` |
| Sector risk rating | default rate > 10% gives High; > 5% gives Medium; else Low |

### 8.8 BR-08 AI Transparency Labels (exact strings)

| Key | Value |
|---|---|
| `label` | "Rules-based estimate" |
| `shortLabel` | "Rules-based guidance" |
| `modelLabel` | "Deterministic rules engine" |
| `decisionDisclaimer` | "Not a final credit decision. Officer review and verified documents are required." |
| `advisorDisclaimer` | "Guidance only, based on available profile data. It does not approve loans or replace staff review." |
| `confidenceLabel` | "Rule match strength" |
| `dataBasis` | "Uses current application fields and configured rules; not trained on PINACO historical repayment data yet." |

### 8.9 BR-09 Pricing and Interest

| Context | Rule |
|---|---|
| Customer calculator (front end) | total interest = 10% of principal, flat; monthly = (principal + interest) ÷ term |
| Repayment schedule (data model) | monthly principal = `round(principal ÷ termMonths)`; monthly interest = `round(remainingBalance × monthlyRate)` where monthly rate is 3.2%–3.5% |
| LTV interest estimate fallback | 15% of instalment amount when no interest is recorded |
| Portfolio interest estimate (dashboard) | flat 10% of total disbursed |

**These four rules are mutually inconsistent and must be reconciled — see OI-03.**

### 8.10 BR-10 Repayment Statuses and Semantics

| Status | Meaning |
|---|---|
| `Scheduled` | Future instalment, not yet due |
| `Upcoming` | Next instalment due |
| `Paid` | Settled (via payroll deduction or direct payment) |
| `Overdue` | Past due and unpaid |
| `FAILED_DEDUCTION` | Payroll deduction raised but unmatched/not processed; administrative exception, no customer penalty |

### 8.11 BR-11 Collection Rate and Target

`collectionRate = paid ÷ (paid + overdue + failedDeductions) × 100`, rounded to an integer.
**Business target: 98% or higher.** Rates below target require employer reconciliation.

### 8.12 BR-12 Customer Lifetime Value

`ltv = sum of interest on Paid instalments + (completedLoans × 50,000 MWK)`
---

## 9. Data Requirements

### 9.1 Entity relationship overview

```
users 1--N customers 1--N loan_applications 1--N repayments
  |            |                 |
  |            |                 +--N decision_history (governance)
  |            +--N documents
  |            +--N businesses 1--N business_documents
  |                             1--N business_timeline_events
  |                             1--N financing_requests
  |            +--N alerts
  +--N audit_logs
  +--N conversations 1--N messages
```

### 9.2 Entity: User (`users`)

| Attribute | Type | Notes |
|---|---|---|
| id | text (PK) | `staff-001`, `client-<timestamp>`, `staff-<timestamp>` |
| role | enum | `customer`, `loan_officer`, `manager`, `executive`, `admin` |
| fullName | text | required |
| email | text | unique (case-insensitive index) |
| phone, address, nationalId, dob, gender | text | |
| employmentType, employer, monthlyIncome | text | income stored as string |
| customerType | enum | `individual`, `sme` |
| customerClassification | enum | `New`, `Existing` |
| passwordHash | text | bcrypt; legacy SHA-256 upgraded on login |
| securityQuestion, securityAnswerHash | text | answer normalised and hashed |
| createdAt | timestamp | |
| notificationPrefs | object | `{ sms, email, push }` booleans |
| staffTitle, avatar, employeeNumber | text | optional |
| isActivated | boolean | optional |

### 9.3 Entity: Loan Application (`loan_applications`)

| Attribute | Type | Notes |
|---|---|---|
| id | text (PK) | `APP-88421` pattern |
| userId | text | owning portal user |
| applicantName, email, phone, address, businessName | text | |
| monthlyRevenue | number | MWK |
| staffCount | number | employees |
| amount | number | requested principal, MWK |
| termMonths | number | 3, 6, 12, 18, 24 observed |
| interestRate | number | monthly rate percent, 3.2–3.5 observed |
| status | enum | see FR-APP-04 for the full list |
| sector | text | Agriculture, Retail, Transport, Construction, Healthcare, Sustainable Energy, Technology, Education, Hospitality, Fisheries, Personal |
| date | text | application date, e.g. "Oct 24, 2023" |
| disbursedAt, completedAt | ISO date | optional |
| score | number | 0–100 health/credit score |
| riskLevel | enum | `Low`, `Medium`, `High` |
| repaymentHistory | text | `Perfect`, `Excellent`, `Good`, `Fair`, `Poor`, `New` |
| debtToIncome | text | percentage string, e.g. "22%" |
| yearsInBusiness | text | e.g. "4.5 yrs", or "N/A" |
| notes, loanOfficer, branchId | text | |
| dataSource | enum | `demo`, `user`, `imported` |

### 9.4 Entity: Repayment (`repayments`)

| Attribute | Type | Notes |
|---|---|---|
| id | text (PK) | `REP-<applicationId>-<NN>` |
| applicationId, userId | text | links |
| installmentNumber | number | 1 to termMonths |
| dueDate | text | formatted date |
| amount, principal, interest, penalty | number | MWK |
| status | enum | `Paid`, `Upcoming`, `Scheduled`, `Overdue`, `FAILED_DEDUCTION` |
| paidAt | ISO date | when settled |
| failureReason | text | for failed deductions |

### 9.5 Entity: Customer (`customers`)

| Attribute | Type | Notes |
|---|---|---|
| id | text (PK) | `cust-001` |
| name, location, sector, status | text | |
| activeLoans | number | |
| riskLevel | enum | `Low`, `Medium`, `High` |
| iconType | enum | `person`, `storefront`, `business`, `warning` |
| userId | text | link to portal account |
| phone, email, joinedAt | text | |
| customerType | enum | `New`, `Existing` |
| employeeNumber, nationalId | text | used by activation matching |
| isActivated | boolean | |
| dataSource | enum | `demo`, `user`, `imported` |

### 9.6 Entity: SME Business (`businesses`)

| Attribute | Type | Notes |
|---|---|---|
| id, customerId, customerName | text | |
| name, registrationNumber, sector, industryCategory, location | text | |
| verificationStatus | enum | `Draft`, `Pending Verification`, `Verified`, `Needs Documents` |
| relationshipManager | text | |
| riskLevel | enum | `Low`, `Medium`, `High` |
| annualRevenue, monthlyRevenue | number | MWK |
| employees, yearsInBusiness | number | |
| healthScore | number | |
| owners | array | `BusinessOwner` |
| documents | array | `BusinessDocument` |
| timeline | array | `BusinessTimelineEvent` |
| financingRequests | array | `FinancingRequest` |

### 9.7 Supporting entities

| Entity | Key attributes |
|---|---|
| `BusinessOwner` | customerId, customerName, role (`Primary Owner`/`Co-owner`/`Director`/`Guarantor`), ownershipPct, linkedAt |
| `BusinessDocument` | id, businessId, name, category (`Registration`/`Tax`/`Bank Statement`/`Financials`/`Permit`/`Other`), status, uploadedAt, size |
| `BusinessTimelineEvent` | id, businessId, date, title, detail, type |
| `FinancingRequest` | id, businessId, productName, amount, termMonths, purpose, status, submittedAt |
| `PayrollBatch` | id, uploadedAt, uploadedBy, fileName, totalRecords, matched, applied, failed, month, status |
| `PayrollRecord` | id, batchId, employeeId, customerName, customerId, employer, amount, month, status, repaymentId, failureReason |
| `DecisionHistoryEntry` | id, applicationId, aiRecommendation, aiConfidence, aiSignals, officerId, officerDecision, isOverride, overrideReason, decidedAt |
| `AlertNotification` | id, userId, type, title, description, date, isRead, imageBackground, actionLabel, actionRoute |
| `AuditLogEntry` | id, occurredAt, actorId, actorName, actorRole, action, entityType, entityId, outcome, summary, metadata |
| `Document` | id, userId, name, type, size, base64, uploadedAt, category |
| `Conversation` | id, participantIds, createdAt, lastMessageAt |
| `Message` | id, conversationId, senderId, content, sentAt, readBy |
| `CustomerRiskScore` | id, customerId, score, tier, positiveFactors, negativeFactors, createdAt |
| `CustomerRiskTimelineEntry` | id, customerId, date, riskLevel, score, triggerEvent, details |
| `CustomerLifetimeValue` | customerId, customerName, totalBorrowed, interestGenerated, completedLoans, relationshipMonths, ltv, meaning |
| `CustomerRecommendation` | id, customerId, customerName, recommendationType, currentLoanAmount, remainingBalance, recommendedAmount, confidence, postTopUpDti, reasons, createdAt |
| `EmployerRiskPrediction` | id, employer, currentCollectionRate, historicalTrend, predictedRisk, monthsOfData, hasSufficientData, reason, createdAt |
| `EmployerPerformance` | employer, employeeCount, activeLoans, totalDisbursed, totalCollected, successfulDeductionRate, failedDeductionRate, avgSalary, riskRating, riskTrend |
| `OfficerQualityMetrics` | officerId, officerName, reviewedCount, approvedCount, declinedCount, approvalRatio, avgProcessingTimeHours, repaymentPerformanceApproved, documentRejectionRate, defaultCount |
| `LoanProduct` | id, name, description, minAmount, maxAmount, minTermMonths, maxTermMonths, interestRate, targetSegment, isActive |
| `Branch` | id, name, location, manager, phone, email, openedAt, activeLoans, totalDisbursed |
| `LoanOfficer` | id, name, branchId, branchName, email, phone, activeLoans, disbursedThisMonth, collectionRate, joinedAt, avatar |
| `FraudFlag` | id, applicationId, signal, severity, description |
| `SectorTrend` | sector, defaultRate, avgRepayment, totalDisbursed, growth, riskRating, insight |
| `SegmentDistribution` | segment, count, color, percentage |
| `DataMiningInsight` | id, category, title, description, impact, confidence, affectedSector, dataSource, calculationMethod, businessMeaning |
| `PortfolioTimePoint` | month, disbursed, collected, newCustomers, defaults |
| `DocumentExtraction` | id, documentId, applicationId, userId, documentType, extractedData, confidence, source, verificationStatus, mismatches, createdAt |

### 9.8 Complete enumeration list (for the FRD appendix)

- **User roles:** `customer`, `loan_officer`, `manager`, `executive`, `admin`
- **Application statuses:** `Under Review`, `In Progress`, `Approved`, `Decline`, `Pending Doc`, `Disbursed`, `Reviewing`, `Completed`, `Active`, `Rejected`
- **Repayment statuses:** `Paid`, `Upcoming`, `Scheduled`, `Overdue`, `FAILED_DEDUCTION`
- **Risk levels:** `Low`, `Medium`, `High`
- **Customer segments:** `Premium Client`, `Reliable Borrower`, `SME Growth`, `Young Entrepreneur`, `High Risk`, `Inactive`
- **Health tiers:** `Excellent`, `Good`, `Fair`, `Poor`
- **Expert verdicts:** `Approve`, `Decline`, `Review`, `Request Docs`
- **Fraud severities:** `Low`, `Medium`, `High`
- **Business verification statuses:** `Draft`, `Pending Verification`, `Verified`, `Needs Documents`
- **Financing request statuses:** `Draft`, `Submitted`, `Under Review`, `Approved`, `Declined`
- **Document categories:** `Registration`, `Tax`, `Bank Statement`, `Financials`, `Permit`, `Other`
- **Document statuses:** `Pending`, `Uploaded`, `Verified`, `Rejected`
- **Extraction statuses:** `Matched`, `Mismatch`, `Pending`
- **Payroll batch statuses:** `Processing`, `Completed`, `Partial`
- **Payroll record statuses:** `Matched`, `Unmatched`, `Applied`, `Failed`
- **Alert types:** `critical`, `approval`, `opportunity`, `info`, `warning`
- **Audit outcomes:** `success`, `failure`, `info`
- **Audit actor roles:** customer, loan_officer, manager, executive, admin, client, staff, system
- **Top-up recommendation types:** `Top-Up`, `Limit Increase`, `Standard`, `Requires Assessment`
- **Branches:** `branch-lil` (Lilongwe), `branch-blt` (Blantyre), `branch-mzu` (Mzuzu)
---

## 10. Interface Requirements — API Surface

**Base URL:** `/api` for legacy routes, `/api/v1` for versioned routes.
**Auth:** `Authorization: Bearer <token>` or cookies `psa_access_token` / `psa_auth_token`.
**Error shape:** legacy routes return a bare `{ "error": "..." }`; v1 routes return `{ "success": false, "error": "..." }`.

### 10.1 Authentication and account routes

| Method | Path | Auth | Purpose |
|---|---|---|---|
| POST | `/api/auth/register` | none (rate-limited) | Register a customer account |
| POST | `/api/auth/login` | none | Legacy login, returns session + token |
| POST | `/api/auth/forgot-password/question` | none | Retrieve security question |
| POST | `/api/auth/forgot-password/verify` | none (rate-limited) | Verify answer, issue reset token |
| POST | `/api/auth/forgot-password/reset` | none (rate-limited) | Set new password with token |
| POST | `/api/auth/change-password` | required | Change password with current password |
| GET | `/api/auth/profile/:id` | required | Read profile (self or staff) |
| PATCH | `/api/auth/profile/:id` | required | Update profile |
| POST | `/api/v1/auth/login` | none | Versioned login, sets cookies |
| POST | `/api/v1/auth/refresh` | refresh token | Issue new 15-minute access token |
| POST | `/api/v1/auth/logout` | none | Clear cookies |

### 10.2 Domain routes

| Method | Path | Auth | Purpose |
|---|---|---|---|
| GET | `/api/applications` | required | List all loan applications |
| POST | `/api/applications` | required | Create an application |
| PATCH | `/api/applications/:id` | required | Update an application |
| GET | `/api/repayments` | required | List repayments |
| PATCH | `/api/repayments/:id` | required | Update a repayment |
| GET | `/api/customers` | required | List customers |
| POST | `/api/customers` | required | Create a customer |
| GET | `/api/businesses` | required | List SME businesses |
| GET | `/api/businesses/customer/:customerId` | required | Businesses for a customer |
| POST | `/api/businesses` | required | Create a business |
| PATCH | `/api/businesses/:id` | required | Update a business |
| POST | `/api/businesses/:id/documents` | required | Append a business document |
| POST | `/api/businesses/:id/timeline` | required | Append a timeline event |
| POST | `/api/businesses/:id/financing-requests` | required | Append a financing request |
| GET | `/api/alerts` | required | List alerts |
| POST | `/api/alerts` | required | Create an alert |
| GET | `/api/documents/:userId` | required | Documents for a user |
| POST | `/api/documents` | required | Store document metadata |
| POST | `/api/documents/upload` | required | Upload a base64 file to disk |
| GET | `/api/documents/:docId/download` | required | Download an uploaded file |
| GET | `/api/audit-logs` | required | List audit entries |
| POST | `/api/audit-logs` | required | Create an audit entry |
| GET | `/api/intelligence/applications/:id` | required | Expert-system assessment |
| GET | `/api/intelligence/dashboard` | required | Fraud, segments, trends, insights, timeline |
| POST | `/api/ai/advisor/chat` | required | Gemini advisor conversation |
| GET | `/api/dashboard/summary` | required | Portfolio dashboard summary |
| GET | `/api/conversations` | required | List conversations for the caller |
| POST | `/api/conversations` | required | Get or create a conversation |
| GET | `/api/conversations/:convId/messages` | required | List messages (optional `after`) |
| POST | `/api/conversations/:convId/messages` | required | Send a message |
| POST | `/api/conversations/:convId/read` | required | Mark messages read (`204`) |
| GET | `/api/admin/staff` | admin | List staff accounts |
| POST | `/api/admin/staff` | admin | Create a staff account |
| PATCH | `/api/admin/staff/:id` | admin | Update a staff account |
| GET | `/api/reports/export?type=` | non-customer | CSV export (`portfolio`, `audit`, `repayments`) |
| GET | `/api/v1/customers` | required | Customer list (v1) |
| GET | `/api/v1/customers/:id` | required | Customer detail (v1) |
| GET | `/api/v1/customers/:id/intelligence` | required | Relationship score, timeline, LTV |
| GET | `/api/v1/loans` | required | Loan list (v1) |
| GET | `/api/v1/loans/:id` | required | Loan detail (v1) |
| GET | `/api/v1/loans/:id/recommendation` | required | Credit recommendation (v1) |
| POST | `/api/v1/governance/override` | officer/manager/admin | Record officer decision |
| GET | `/api/v1/governance/history` | required | Decision history (optional `applicationId`) |
| GET | `/api/v1/governance/scenarios/active` | none | Active scenario (governance path) |
| POST | `/api/v1/governance/scenarios/switch` | none | Switch scenario (governance path) |
| GET | `/api/v1/scenarios/active` | none | Active scenario (canonical path) |
| POST | `/api/v1/scenarios/switch` | none | Switch scenario (canonical path) |
| GET | `/api/v1/payroll/batches` | required | Payroll batches |
| GET | `/api/v1/payroll/exceptions` | required | Unmatched/failed payroll records |
| GET | `/api/v1/payroll/employer/:name/performance` | required | Employer risk prediction |
| GET | `/api/v1/intelligence/top-up-opportunities` | required | Top-up recommendations |
| GET | `/api/v1/intelligence/employer-predictions` | required | All employer predictions |
| GET | `/api/health` | none | Health with storage mode |
| GET | `/api/health/liveness` | none | Liveness probe |
| GET | `/api/health/readiness` | none | Readiness probe with storage check |

### 10.3 Browser storage keys (client side)

| Key | Storage | Purpose |
|---|---|---|
| `psa_users` | localStorage | Local (non-API) user registry |
| `psa_session` | localStorage or sessionStorage | Active session object |
| `psa_auth_token` | localStorage or sessionStorage | Bearer token |

---

## 11. Non-Functional Requirements (as observed, plus recommended targets)

> Items marked **Observed** describe the current build. Items marked **Gap** or **Recommended**
> are things the code does not yet satisfy; the FRD author should state them as requirements to be met.

### 11.1 Security

| ID | Requirement | Status |
|---|---|---|
| NFR-SEC-01 | Passwords shall be stored using bcrypt with a minimum cost factor of 12. | Observed |
| NFR-SEC-02 | Authentication tokens shall be signed with HMAC-SHA256 using a server-held secret. | Observed |
| NFR-SEC-03 | Session cookies shall be `HttpOnly`, `SameSite=Strict`, and `Secure` in production. | Observed |
| NFR-SEC-04 | Passwords and security-answer hashes shall never be returned in any API response. | Observed |
| NFR-SEC-05 | Password reset tokens shall be stored hashed, single-use, and time-limited to 15 minutes. | Observed |
| NFR-SEC-06 | Public registration and reset endpoints shall be rate-limited. | Observed (in-memory) |
| NFR-SEC-07 | Secret comparisons shall be constant-time where raw digests are compared. | Observed |
| NFR-SEC-08 | API secrets shall be supplied via environment variables and never committed. | Observed |
| NFR-SEC-09 | The development authentication bypass (default loan officer injection) shall be disabled in production. | Observed |
| NFR-SEC-10 | The literal credential bypass in the v1 login route shall be removed before production. | Gap |
| NFR-SEC-11 | Role-based access control shall be enforced server-side on every protected endpoint, not only in the UI. | Gap |
| NFR-SEC-12 | Uploaded documents shall be virus-scanned and encrypted at rest. | Gap |
| NFR-SEC-13 | Rate limiting shall use a shared store rather than per-process in-memory state. | Gap |
| NFR-SEC-14 | Transport shall be TLS-terminated end to end and HSTS shall be set. | Gap (infrastructure) |
| NFR-SEC-15 | All secrets shall support rotation without downtime. | Gap |

### 11.2 Performance and scalability

| ID | Requirement | Status |
|---|---|---|
| NFR-PERF-01 | A single Node process shall serve both the API and static assets. | Observed |
| NFR-PERF-02 | The request body limit shall be 15 MB to accommodate base64 uploads. | Observed |
| NFR-PERF-03 | Dashboard and intelligence aggregations are computed on read rather than from pre-aggregated tables. | Observed (inefficient at volume) |
| NFR-PERF-04 | 90th-percentile API response time shall be under 500 ms at 50 concurrent staff users on the reference environment. | Recommended |
| NFR-PERF-05 | The system shall support at least 50,000 customer records and 500,000 instalments without degradation. | Recommended |
| NFR-PERF-06 | Analytics endpoints shall be cached with a short TTL or backed by materialised views. | Recommended |
| NFR-PERF-07 | The server shall be horizontally scalable using a shared session and rate-limit store. | Recommended |

### 11.3 Availability and resilience

| ID | Requirement | Status |
|---|---|---|
| NFR-AVL-01 | The system shall fall back to file-backed storage when PostgreSQL is unreachable, without dropping requests. | Observed |
| NFR-AVL-02 | The system shall provide liveness and readiness probes for orchestration. | Observed |
| NFR-AVL-03 | The system shall expose a health endpoint reporting the active storage mode. | Observed |
| NFR-AVL-04 | File-backed writes shall be atomic (temporary file plus rename). | Observed |
| NFR-AVL-05 | Recovery point objective and recovery time objective shall be defined and tested. | Recommended |
| NFR-AVL-06 | The file-backed fallback shall not be used as a production system of record. | Recommended |

### 11.4 Usability and accessibility

| ID | Requirement | Status |
|---|---|---|
| NFR-USA-01 | The UI shall support light and dark themes with a persisted preference. | Observed |
| NFR-USA-02 | The UI shall be responsive across desktop (sidebar), tablet (icon rail), and mobile (drawer plus bottom tabs). | Observed |
| NFR-USA-03 | All monetary values shall be formatted with thousands separators and shown in MWK. | Observed |
| NFR-USA-04 | Every AI-derived output shall carry a plain-language transparency notice. | Observed |
| NFR-USA-05 | All interactive elements shall meet WCAG 2.1 AA contrast and keyboard navigation. | Recommended |
| NFR-USA-06 | All user-visible text shall be localised (English assumed; Chichewa a future locale). | Recommended |

### 11.5 Auditability and compliance

| ID | Requirement | Status |
|---|---|---|
| NFR-AUD-01 | Every mutating operation shall create an audit entry with actor, action, entity, outcome, and summary. | Observed |
| NFR-AUD-02 | Every AI recommendation decision shall be logged with an override flag and reason. | Partial |
| NFR-AUD-03 | Audit entries shall be immutable and tamper-evident. | Recommended |
| NFR-AUD-04 | A documented retention period shall apply to audit, document, and message data. | Recommended |
| NFR-AUD-05 | Data-subject access and erasure procedures shall be defined. | Recommended |
| NFR-AUD-06 | Personal data shall be encrypted in transit and at rest, and access to it logged. | Recommended |

### 11.6 Maintainability

| ID | Requirement | Status |
|---|---|---|
| NFR-MNT-01 | TypeScript type-checking shall pass (`npm run lint` runs `tsc --noEmit`). | Observed |
| NFR-MNT-02 | A test runner shall exist with suites for auth, RBAC, credit assessment, and intelligence. | Observed (`npm test`) |
| NFR-MNT-03 | Database migrations shall be versioned with up, status, and rollback commands. | Observed |
| NFR-MNT-04 | Automated tests shall run in CI on every change with coverage thresholds. | Recommended |
| NFR-MNT-05 | The API shall be versioned so breaking changes do not affect clients. | Partial (`/api/v1` alongside legacy `/api`) |
---

## 12. Integrations and External Dependencies

| Integration | Purpose | Configuration | Behaviour when unconfigured |
|---|---|---|---|
| PostgreSQL (`pg`) | Primary persistence | `DATABASE_URL` or `PGHOST`/`PGPORT`/`PGDATABASE`/`PGUSER`/`PGPASSWORD`, `PGSSLMODE`, `PGCONNECT_TIMEOUT_MS` | Falls back to file store |
| Google Gemini (`@google/genai`) | AI advisor chat | `GEMINI_API_KEY`, `GEMINI_MODEL` (default `gemini-2.0-flash`) | `503` from the API; browser-side fallback attempted |
| Africa's Talking | Transactional SMS | `AFRICASTALKING_USERNAME`, `AFRICASTALKING_API_KEY`, `AFRICASTALKING_SENDER_ID` | Logs an "Unconfigured Key Notice" and returns false |
| SendGrid | Transactional email | `SENDGRID_API_KEY`, `SENDGRID_FROM_EMAIL` (default `noreply@pinnacle.mw`), `SENDGRID_FROM_NAME` (default `Pinnacle MFI`) | Logs an "Unconfigured Key Notice" and returns false |

### 12.1 Environment variables observed in the codebase

| Variable | Purpose | Default |
|---|---|---|
| `PORT` | API listen port | `4000` |
| `NODE_ENV` | Enables production auth enforcement | unset |
| `APP_URL` | Allowed CORS origins (comma-separated; empty or `*` means allow any) | unset |
| `AUTH_SECRET` | HMAC secret for tokens and reset-token hashing | `pinnacle-local-development-auth-secret` |
| `JWT_SECRET` | Additional accepted signing secret | `psa-super-secret-jwt-key-2026` |
| `AUTH_TOKEN_TTL_HOURS` | Access token lifetime | `12` |
| `PASSWORD_RESET_TOKEN_TTL_MINUTES` | Reset token lifetime | `15` |
| `BCRYPT_ROUNDS` | Password hashing cost | `12` |
| `PSA_STORAGE_MODE` / `MOCK_DB_MODE` | Force file storage when set to `file` | unset |
| `MOCK_DB_FILE` | Path to the JSON fallback file | `server/data/mock-db.json` |
| `DATABASE_URL`, `PGHOST`, `PGPORT`, `PGDATABASE`, `PGUSER`, `PGPASSWORD`, `PGSSLMODE`, `PGCONNECT_TIMEOUT_MS` | PostgreSQL connection | host `localhost`, port `5432`, database `pinaco`, user `postgres`, timeout `1500` ms |
| `GEMINI_API_KEY`, `GEMINI_MODEL` | AI advisor | model `gemini-2.0-flash` |
| `AFRICASTALKING_*` | SMS | unset |
| `SENDGRID_*` | Email | from `noreply@pinnacle.mw`, name `Pinnacle MFI` |

### 12.2 Third-party libraries in use

`express` 4, `pg` 8, `bcrypt` 6, `@google/genai` 2, `dotenv` 17, `react` 19, `react-dom` 19,
`react-router-dom` 7, `tailwindcss` 4 (via `@tailwindcss/vite`), `lucide-react` 0.546 (icons),
`recharts` 3 (charts), `motion` 12 (animation), `vite` 6, `tsx` 4, `typescript` 5.8.

---

## 13. Open Issues, Inconsistencies, and Known Gaps

> The FRD must record each of these as an explicit open issue with an owner and resolution
> requirement. Do not write requirements that silently assume these are resolved.

| ID | Issue | Impact | Evidence |
|---|---|---|---|
| OI-01 | RBAC role mismatch: `requireRole(['officer','manager','admin'])` uses `officer`, but stored roles are `loan_officer`, so a loan officer fails authorisation on the override endpoint unless the admin short-circuit applies. | Governance override may be inaccessible to the role that most needs it | `server/middleware/rbac.ts`, `governance.routes.ts` |
| OI-02 | Demo credential bypass: `/api/v1/auth/login` accepts the literal password `Password123!` when hash comparison fails. | Critical authentication defect | `server/routes/api/v1/auth.routes.ts` |
| OI-03 | Unreconciled pricing models: calculator uses flat 10% total interest on K 10,000–K 500,000; the portfolio model uses 3.2–3.5% monthly on up to K 25,000,000. | Displayed pricing does not match portfolio pricing | `ClientLoanCalculator.tsx`, `src/mock/loans.ts` |
| OI-04 | Employer/sector conflation: employer performance groups by the application's `sector` when no payroll record exists. | Employer analytics can misattribute risk to a sector | `computeEmployerPerformance` |
| OI-05 | AI decision history is held in an in-memory array and is lost on restart, although documentation describes a `decision_history` table. | Compliance and audit gap for governance | `governanceRepository.ts` |
| OI-06 | Officer quality metrics are computed for a hard-coded list of three staff. | Scorecard silently excludes newly created staff | `computeOfficerQualityMetrics` |
| OI-07 | Notification preferences are stored but never enforced by the SMS/email senders; push has no implementation. | Customers cannot actually control notifications | `server/notifications.ts`, `server/index.ts` |
| OI-08 | Role authorisation is presentational on the client and absent on most server read endpoints. | Any authenticated user can read any portfolio data via the API | v1 route modules |
| OI-09 | PII exposure: customer phone, email, and national ID are returned by list endpoints without field-level restriction. | Privacy and regulatory risk | `/api/customers`, `/api/auth/profile/:id` |
| OI-10 | Document upload accepts base64 in the JSON body with no type validation, per-file size ceiling, virus scan, or encryption. | Security and storage risk | `/api/documents/upload` |
| OI-11 | Rate limiter is per-process in memory, so limits reset on restart and do not apply across instances. | Weak brute-force protection when scaled | `server/index.ts` |
| OI-12 | Registration generates a fictitious loan portfolio for every new customer. | Real customers receive fabricated financial history | `generateUserMockBundle` |
| OI-13 | Duplicate `/api/health` route definitions exist; only the later one is reachable. | Health payload differs from documentation | `server/index.ts` |
| OI-14 | Inconsistent branding and secrets strings (`PINACO@2026`, `psa_salt_v1`, `@pinnacle.mw` vs `@pinaco-customer.mw`). | Branding and onboarding inconsistency | seed files, activation page |
| OI-15 | Two different score concepts (application health score and customer relationship score) with different scales and thresholds. | Analyst confusion; needs one agreed definition | `computeHealthScore`, `computeCustomerRelationshipScore` |
| OI-16 | Scoring inputs are free-text strings (`"22%"`, `"4.5 yrs"`, `"N/A"`) with silent fallback values when parsing fails. | Silent mis-scoring risk | `src/lib/intelligenceEngine.ts` |

---

## 14. Traceability Reference

Use this table to map requirements to implementation artefacts when building the FRD traceability matrix.

| Requirement area | Primary source files | Tests |
|---|---|---|
| FR-AUTH, FR-PWD | `server/index.ts`, `server/routes/api/v1/auth.routes.ts`, `server/middleware/auth.ts`, `src/auth/authService.ts`, `src/auth/AuthContext.tsx` | `server/tests/auth.test.ts` |
| FR-ONB | `server/index.ts` (register), `src/pages/RegisterPage.tsx`, `src/pages/CustomerActivationPage.tsx` | `server/tests/auth.test.ts` |
| FR-CALC, FR-APP | `src/components/ClientLoanCalculator.tsx`, `src/mock/loans.ts`, `src/context/ClientContext.tsx` | — |
| FR-DOC | `server/index.ts` (upload/download), `src/components/DocumentUpload.tsx`, `DocumentVerificationPanel.tsx`, `src/lib/documentExtractionService.ts` | — |
| FR-CREDIT, FR-FRAUD | `src/lib/intelligenceEngine.ts`, `server/routes/api/v1/loan.routes.ts`, `src/lib/aiTransparency.ts` | `creditAssessment.test.ts`, `intelligence.test.ts` |
| FR-REPAY | `src/mock/loans.ts` (`generateRepaymentSchedule`), `server/index.ts` (repayment routes) | — |
| FR-PAY, FR-EMP | `server/repositories/payrollRepository.ts`, `server/services/employerRiskService.ts`, `src/components/PayrollPage.tsx`, `PayrollUpload.tsx`, `PayrollPreview.tsx`, `PayrollHistory.tsx` | — |
| FR-C360, FR-TOPO | `server/services/creditAssessmentService.ts`, `server/routes/api/v1/customer.routes.ts`, `Customer360Modal.tsx`, `TopUpOpportunitiesCard.tsx` | `creditAssessment.test.ts` |
| FR-GOV | `server/services/aiGovernanceService.ts`, `server/repositories/governanceRepository.ts`, `governance.routes.ts`, `src/components/AdminReview.tsx` | `server/tests/rbac.test.ts` |
| FR-MSG | `server/index.ts` (conversation routes), `src/components/ClientMessages.tsx` | — |
| FR-AI | `server/index.ts` (`/api/ai/advisor/chat`), `src/lib/aiAdvisorService.ts`, `src/components/ClientAIAdvisor.tsx` | — |
| FR-SME | `server/index.ts` (business routes), `src/types.ts`, `AdminSMEPortfolio.tsx`, `ClientBusiness.tsx` | — |
| FR-RPT | `server/db.ts` (`getDashboardSummary`), `src/lib/intelligenceEngine.ts`, `AdminReports.tsx`, `AdminAnalytics.tsx`, `AdminIntelligence.tsx` | `intelligence.test.ts` |
| FR-ADMIN | `server/index.ts` (admin staff routes), `src/components/AdminStaffManagement.tsx` | `server/tests/rbac.test.ts` |
| FR-SCEN | `server/services/scenarioGeneratorService.ts` | — |
| FR-AUD | `server/db.ts` (audit helpers), `src/lib/auditTrail.ts` | — |
| FR-NOTIF | `server/notifications.ts`, `src/components/NotificationSettings.tsx` | — |
| NFR-* | `package.json`, `tsconfig.json`, `vite.config.ts`, `Dockerfile`, `docker-compose.yml`, `ecosystem.config.js` | `server/tests/runner.ts` |
| Database | `server/database/migrations/001_initial_schema.sql`, `server/database/seed/001_seed_malawi_data.sql`, `server/database/migrate.ts` | — |

### 14.1 Test and build commands available

| Command | Effect |
|---|---|
| `npm run dev` | Start the Vite dev server on port 3000 |
| `npm run api` / `npm start` | Start the API server (`tsx server/index.ts`) |
| `npm run build` | Build the front end into `dist/` |
| `npm run lint` | Type-check with `tsc --noEmit` |
| `npm test` | Run the server test runner |
| `npm run migrate` | Apply database migrations |
| `npm run migrate:status` | Show migration status |
| `npm run migrate:rollback` | Roll back the last migration |

---

## 15. Seed and Reference Data Snapshot

### 15.1 Seeded loan portfolio (15 applications)

| Application | Applicant | Sector | Amount (MWK) | Term | Rate | Status | Score | Risk | Repayment |
|---|---|---|---|---|---|---|---|---|---|
| APP-88421 | Samuel Chimwala | Agriculture | 12,000,000 | 24 | 3.5 | Under Review | 75 | Medium | Excellent |
| APP-294021 | Bright Kamwendo | Sustainable Energy | 8,500,000 | 18 | 3.5 | In Progress | 88 | Low | Perfect |
| APP-449211 | Blessings Kamau | Retail | 3,200,000 | 12 | 3.5 | Reviewing | 62 | Medium | Good |
| APP-102293 | Mphatso Mwale | Retail | 1,200,000 | 12 | 3.5 | Pending Doc | 45 | High | Fair |
| APP-348211 | Kondwani Msiska | Construction | 25,000,000 | 24 | 3.2 | Approved | 91 | Low | Excellent |
| APP-500112 | Chisomo Nyasulu | Agriculture | 750,000 | 6 | 3.5 | Disbursed | 70 | Medium | Good |
| APP-612034 | Thandiwe Chirwa | Retail | 4,500,000 | 18 | 3.5 | Approved | 82 | Low | Excellent |
| APP-710029 | Wisdom Zimba | Personal | 500,000 | 6 | 3.5 | Disbursed | 68 | Medium | Good |
| APP-820113 | Mercy Gondwe | Healthcare | 6,000,000 | 12 | 3.5 | Completed | 94 | Low | Perfect |
| APP-931244 | Innocent Mvula | Transport | 18,000,000 | 24 | 3.2 | Under Review | 79 | Medium | Good |
| APP-044511 | Precious Lungu | Fisheries | 2,400,000 | 12 | 3.5 | Pending Doc | 55 | Medium | Fair |
| APP-159023 | Gift Tembo | Technology | 9,000,000 | 18 | 3.5 | Approved | 85 | Low | Excellent |
| APP-267041 | Stella Nkosi | Education | 3,500,000 | 12 | 3.5 | Disbursed | 78 | Low | Good |
| APP-375062 | Memory Phiri | Hospitality | 22,000,000 | 24 | 3.2 | Approved | 90 | Low | Excellent |
| APP-483073 | Wonder Kaunda | Agriculture | 7,500,000 | 18 | 3.5 | Decline | 38 | High | Poor |

**Loan officers seeded:** Chisomo Banda, Kondwani Phiri, Stella Mwale, Innocent Gondwe, Grace Chirwa.
**Branches:** `branch-lil` (Lilongwe), `branch-blt` (Blantyre), `branch-mzu` (Mzuzu).

### 15.2 Seeded payroll batches

| Batch | Employer | Month | Records | Matched | Applied | Failed | Status |
|---|---|---|---|---|---|---|---|
| batch-2024-07-edu | Ministry of Education | 2024-07 | 120 | 118 | 118 | 2 | Partial |
| batch-2024-07-pol | Malawi Police | 2024-07 | 85 | 83 | 83 | 2 | Partial |

### 15.3 Demo scenario reference values

| Metric | Healthy Portfolio | High Default Risk | Payroll Crisis |
|---|---|---|---|
| Portfolio health % | 96.5 | 72.0 | 64.5 |
| Collection rate % | 98.2 | 81.4 | 68.0 |
| Delinquent loans | 2 | 14 | 22 |
| Payroll match % | 98.5 | 88.0 | 58.2 |

---

## 16. Ready-to-Use Prompt for the Documentation AI

Paste the following instruction together with this entire file:

> You are a senior business analyst writing a formal Functional Requirements Document (FRD).
> The attached file, "PINACO Smart Advisor — Functional Requirements Source Data", is the
> authoritative source of truth for the system. Write the complete FRD from it, following
> §1 of the file exactly.
>
> Rules:
> 1. Do not invent functionality. If something is not in the source data, it is out of scope.
> 2. Reproduce every threshold, weight, percentage, status string, and enum exactly as given.
> 3. For every requirement, state the status as Implemented, Partial, or Simulated, and where
>    Simulated, explain what the production behaviour must replace it with.
> 4. Include a business rules catalogue using the BR-xx identifiers from the source data and
>    cross-reference each rule from the requirements that depend on it.
> 5. Include the full role permission matrix and reconcile it against the server-side
>    enforcement notes, flagging where UI permissions and API permissions diverge.
> 6. Record every item in the source data's open issues table as an open issue in the FRD,
>    with a recommended resolution and an owner placeholder.
> 7. Write acceptance criteria in Given/When/Then form for every Must-priority requirement.
> 8. Use MWK for all monetary values and Malawi-appropriate locale formats.
> 9. Deliver the FRD with a numbered table of contents and consistent heading numbering.
> 10. Where the source data says two things are inconsistent (for example the two pricing
>     models), present both and state that reconciliation is required before development.
>
> For each requirement include: ID, title, description in "The system shall…" form, actors,
> trigger, preconditions, postconditions, main flow, alternate flows, data inputs/outputs,
> business rule references, priority (MoSCoW), status, and acceptance criteria.

---

## 17. How This Source Data Was Derived

This file was compiled by reading the application as built, specifically:

- Project metadata and dependencies (`package.json`, `metadata.json`, `tsconfig.json`, `vite.config.ts`)
- Architecture, API, schema, and operations documentation already in the repository
  (`SYSTEM_ARCHITECTURE.md`, `API_REFERENCE.md`, `DATABASE_SCHEMA.md`, `DEMO_CREDENTIALS.md`,
  `DEPLOYMENT_GUIDE.md`, `SECURITY_AUDIT.md`, `PERFORMANCE_REPORT.md`, `UAT_TEST_PLAN.md`)
- The server entry point and all middleware (`server/index.ts`, `server/middleware/*`)
- All versioned route modules (`server/routes/api/v1/*`)
- The service layer (`server/services/*`) and repository layer (`server/repositories/*`)
- The persistence layer (`server/db.ts`, `server/mockDb.ts`, `server/database/*`)
- Notification integration code (`server/notifications.ts`)
- The full front-end route tree and layouts (`src/App.tsx`, `src/layouts/*`)
- The domain type system, which is the single source of truth for models (`src/types.ts`)
- The intelligence engine containing all scoring and analytics rules (`src/lib/intelligenceEngine.ts`)
- Client authentication and data services (`src/auth/*`, `src/lib/*`)
- Seed and mock domain data (`src/data.ts`, `src/mock/*`)
- Key user-facing flows (`ClientLoanCalculator`, `CustomerActivationPage`)

Anything not supported by those files was deliberately excluded rather than inferred.

**End of source data pack.**
