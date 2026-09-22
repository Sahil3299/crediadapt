# LOANWISE — Enterprise Personalized Loan Recommendation System

**LoanWise** is an AI-powered personalized loan recommendation and underwriting decision console built for loan officers, risk analysts, and applicants.

---

## 1. Architecture Overview

```
Frontend (Next.js 15 App Router + TS + Tailwind + TanStack Query)
       │
       │ HTTP / JSON API requests (Bearer JWT from Supabase Auth)
       ▼
Python FastAPI Facade (Hackthon/api_server.py)  → Supabase Auth verifies JWT
       │
       ├─► CustomerProfilingAgentLLM (Profile Extraction)
       ├─► AffordabilityAgent (FOIR <= 60% Math & Capacity)
       ├─► RiskMLAgent (XGBoost Default Model & SHAP Factors — no gender feature)
       ├─► OfferDiscountAgentLLM & PolicyRAGAgent (Rate Concessions)
       ├─► LoanSimulatorAgent (EMI, Interest, Amortization, Early Closure)
       └─► ComplianceExplanationAgentLLM (Audit Rationale — guardrailed)
       │
       └─► Supabase Postgres (applications, audit_log, documents)
           └─► Supabase Storage (private bucket for uploaded documents)
```

---

## 2. Supabase Setup (one time)

1. Create a project at https://supabase.com.
2. From **Settings → API**: copy `Project URL`, `anon` key, `service_role` key, and **JWT secret**.
3. Run `Hackthon/sql/schema.sql` in the Supabase SQL editor. This creates:
   - `profiles` (roles: `applicant` / `loan_officer`, auto-created on signup),
   - `applications` (backed by a `record` JSONB column plus `status`),
   - `audit_log` (append-only),
   - `documents` (metadata for Supabase Storage),
   - Postgres RLS policies scoping applications/documents/audit to the owning user or loan officers.
4. Storage: the `loanwise-documents` private bucket is created automatically on first upload (`database.ensure_bucket`).

> Free-tier Supabase projects pause after 7 days with no API activity. Data is retained; resume manually from the dashboard.

---

## 3. Environment Configuration

### Backend (`Hackthon/.env`)

```env
GOOGLE_API_KEY=your-google-api-key
SUPABASE_URL=https://YOUR-PROJECT-REF.supabase.co
SUPABASE_SERVICE_ROLE_KEY=your-service-role-key
SUPABASE_JWT_SECRET=your-supabase-jwt-secret
FRONTEND_ORIGIN=http://localhost:3000
```

### Frontend (`frontend/.env.local`)

```env
NEXT_PUBLIC_API_URL=http://localhost:8000
NEXT_PUBLIC_SUPABASE_URL=https://YOUR-PROJECT-REF.supabase.co
NEXT_PUBLIC_SUPABASE_ANON_KEY=your-anon-key
```

See `Hackthon/.env.example` and `frontend/.env.example` for the full list.

---

## 4. How to Run Locally

### Step 0: Create roles

Register via `/register`. Choose **Loan Officer** or **Applicant**. Loan officers see the full console (operations queue, audit, approvals); applicants only see their own applications.

### Step 1: Start Python Backend

```bash
cd c:\loan_system\Hackthon
# Activate virtual environment
.venv\Scripts\activate
# Install new deps
pip install -r requirements.txt
# Start FastAPI server
uvicorn api_server:app --reload --port 8000
```

The API will run at `http://localhost:8000`. Swagger OpenAPI docs are available at `http://localhost:8000/docs`.

### Step 2: Start Next.js Frontend

```bash
cd c:\loan_system\frontend
# Install dependencies if needed
npm install
# Start development server
npm run dev
```

Open `http://localhost:3000` in your browser. Unauthenticated visitors are redirected to `/login`.

---

## 5. Application Routes

| Route | Description | Backend Integration | Auth |
|---|---|---|---|
| `/login`, `/register` | Supabase Auth screens | — | public |
| `/dashboard` | Loan officer summary dashboard | `GET /api/health`, `GET /api/applications` | user |
| `/intake` | 5-Step Customer Intake entry form | `POST /api/recommendations` | user |
| `/customers` | Enterprise customer application table | `GET /api/customers` | user (scope) |
| `/customers/[id]` | Customer 360 profile & financials | `GET /api/recommendations/{id}` | user (scope) |
| `/recommendations/[id]` | **Core Recommendation Dashboard** | `GET /api/recommendations/{id}` | user (scope) |
| `/recommendations/[id]/why` | Explainability & SHAP risk factors + adverse action notice | `GET /api/explainability/{id}` | user (scope) |
| `/simulator` | Interactive What-If Loan Simulator | `POST /api/simulator` | user |
| `/foreclosure` | Early Closure / Foreclosure console | `POST /api/foreclosure` | user |
| `/approvals` | Underwriter review + decision queue | `GET /api/approvals`, `POST /api/approvals/{id}/decision` | loan officer |
| `/audit` | Append-only audit trail | `GET /api/audit` | loan officer |
| `/documents` | Document upload / list per application | `GET/POST /api/documents/{id}` | user (scope) |
| `/settings` | API health, policy config, model integrity (AUC / confusion matrix) | `GET /health`, `GET /api/model/metrics` | user / officer |

---

## 6. Public API Surface

All `/api/*` endpoints now require a `Authorization: Bearer <supabase-access-token>` header except `/health` and `/api/health`. Protected by role:

- **Applicants**: create recommendations, view/simulate only their own applications, upload documents to their own applications.
- **Loan officers**: full queue, decisions, audit trail, model metrics/retrain.

Every state change writes an immutable `audit_log` row. Rejected/counter-offer decisions surface a rule-based **Adverse Action Notice** (`adverse_action`) generated from affordability/risk numbers, and Gemini explanations are validated against the real loan/rate/EMI/FOIR before being shown (fallback to rule-based text on mismatch).

## 7. Production Build

To test and compile the production build:

```bash
npm run build
npm run start
```
