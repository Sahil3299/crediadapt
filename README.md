# CrediAdapt — AI-Powered Personalized Loan Recommendation System

CrediAdapt is a multi-agent, AI-powered loan recommendation and underwriting decision console for loan officers, risk analysts, and applicants. Instead of returning a single binary **approve / reject**, it produces a personalized offer — loan amount, interest rate, tenure, and concessions — backed by a transparent, auditable reasoning trail.

---

## Table of Contents

- [Overview](#overview)
- [Architecture](#architecture)
- [Agent Pipeline](#agent-pipeline)
- [Tech Stack](#tech-stack)
- [Getting Started](#getting-started)
  - [1. Supabase Setup](#1-supabase-setup)
  - [2. Environment Configuration](#2-environment-configuration)
  - [3. Run Locally](#3-run-locally)
- [Application Routes](#application-routes)
- [API Surface & Security](#api-surface--security)
- [Production Build](#production-build)

---

## Overview

Loan eligibility and loan **affordability** are not the same question. A customer can qualify for a loan on paper and still be set up to struggle with it. CrediAdapt is built around a more useful question:

> *Given this customer's income, obligations, credit profile, and applicable policies — what loan amount, rate, and tenure actually make sense for them?*

Every recommendation is decomposed into seven specialized responsibilities (profiling, affordability, risk, policy, offer, simulation, explanation), executed by dedicated agents rather than a single opaque model — so each decision stays interpretable, testable, and auditable end-to-end.

---

## Architecture

```
Frontend (Next.js 15 App Router + TypeScript + Tailwind + TanStack Query)
       │
       │ HTTP / JSON requests  (Bearer JWT via Supabase Auth)
       ▼
Python FastAPI Facade  (api_server.py)  ──► Supabase Auth verifies JWT
       │
       ├─► Customer Profiling Agent (LLM)         — structures raw applicant data
       ├─► Affordability Agent                    — FOIR ≤ 60% capacity check
       ├─► Risk ML Agent                          — XGBoost default model + SHAP factors
       ├─► Policy RAG Agent + Offer/Discount Agent (LLM) — rate concessions
       ├─► Loan Simulator Agent                   — EMI, interest, amortization, early closure
       └─► Compliance & Explanation Agent (LLM)   — guardrailed audit rationale
       │
       └─► Supabase Postgres  (applications, audit_log, documents)
           └─► Supabase Storage (private bucket for uploaded documents)
```

## Agent Pipeline

| # | Agent | Responsibility |
|---|-------|-----------------|
| 1 | **Customer Profiling** | Transforms raw applicant input into a structured financial profile |
| 2 | **Affordability (FOIR)** | Computes Fixed Obligation to Income Ratio; flags whether the requested structure is affordable, and generates counter-offers when it isn't |
| 3 | **Risk (ML)** | XGBoost-based default-risk classification (LOW / MEDIUM / HIGH), explained with SHAP — deliberately excludes gender as a feature |
| 4 | **Policy RAG** | Retrieves applicable concessions/schemes from a policy knowledge base, decoupled from the ML risk model so policy changes don't require retraining |
| 5 | **Offer / Discount** | Combines affordability + risk + policy into a concrete offer: amount, rate, tenure, concessions |
| 6 | **Loan Simulator** | Computes EMI, total interest, total payable, and supports what-if tenure comparisons and early-closure scenarios |
| 7 | **Compliance & Explanation** | Produces a human-readable rationale for the decision; LLM output is validated against the underlying numbers before being shown, with a rule-based fallback on mismatch |

Every state change is written to an **append-only audit log**, and rejected/counter-offer decisions automatically generate a rule-based **Adverse Action Notice** derived from the affordability/risk figures.

---

## Tech Stack

| Layer | Technologies |
|---|---|
| Frontend | Next.js 15 (App Router), TypeScript, Tailwind CSS, TanStack Query |
| Backend | Python, FastAPI |
| Auth & Data | Supabase Auth (JWT), Supabase Postgres (with Row-Level Security), Supabase Storage |
| ML / Risk | XGBoost, SHAP |
| LLM / Agents | Gemini, LangChain-style multi-agent orchestration, RAG (policy retrieval) |
| Data Processing | Pandas, NumPy |

---

## Getting Started

### 1. Supabase Setup

1. Create a project at [supabase.com](https://supabase.com).
2. From **Settings → API**, copy the `Project URL`, `anon` key, `service_role` key, and **JWT secret**.
3. Run `Hackthon/sql/schema.sql` in the Supabase SQL editor. This provisions:
   - `profiles` — roles `applicant` / `loan_officer`, auto-created on signup
   - `applications` — backed by a `record` JSONB column plus `status`
   - `audit_log` — append-only
   - `documents` — metadata for Supabase Storage
   - Postgres RLS policies scoping applications/documents/audit to the owning user or loan officers
4. **Storage:** the `crediadapt-documents` private bucket is created automatically on first upload (`database.ensure_bucket`).

> **Note:** Free-tier Supabase projects pause after 7 days of API inactivity. Data is retained — resume manually from the dashboard.

### 2. Environment Configuration

**Backend** — `Hackthon/.env`

```env
GOOGLE_API_KEY=your-google-api-key
SUPABASE_URL=https://YOUR-PROJECT-REF.supabase.co
SUPABASE_SERVICE_ROLE_KEY=your-service-role-key
SUPABASE_JWT_SECRET=your-supabase-jwt-secret
FRONTEND_ORIGIN=http://localhost:3000
```

**Frontend** — `frontend/.env.local`

```env
NEXT_PUBLIC_API_URL=http://localhost:8000
NEXT_PUBLIC_SUPABASE_URL=https://YOUR-PROJECT-REF.supabase.co
NEXT_PUBLIC_SUPABASE_ANON_KEY=your-anon-key
```

See `Hackthon/.env.example` and `frontend/.env.example` for the complete list of variables.

### 3. Run Locally

**Step 0 — Create an account**
Register via `/register` and choose a role:
- **Loan Officer** — full console access (operations queue, audit trail, approvals)
- **Applicant** — visibility limited to their own applications

**Step 1 — Start the backend**

```bash
cd c:\loan_system\Hackthon
.venv\Scripts\activate
pip install -r requirements.txt
uvicorn api_server:app --reload --port 8000
```

API runs at `http://localhost:8000`; Swagger docs at `http://localhost:8000/docs`.

**Step 2 — Start the frontend**

```bash
cd c:\loan_system\frontend
npm install
npm run dev
```

Open `http://localhost:3000`. Unauthenticated visitors are redirected to `/login`.

---

## Application Routes

| Route | Description | Backend Integration | Auth |
|---|---|---|---|
| `/login`, `/register` | Supabase Auth screens | — | public |
| `/dashboard` | Loan officer summary dashboard | `GET /api/health`, `GET /api/applications` | user |
| `/intake` | 5-step customer intake form | `POST /api/recommendations` | user |
| `/customers` | Enterprise customer application table | `GET /api/customers` | user (scoped) |
| `/customers/[id]` | Customer 360 profile & financials | `GET /api/recommendations/{id}` | user (scoped) |
| `/recommendations/[id]` | **Core recommendation dashboard** | `GET /api/recommendations/{id}` | user (scoped) |
| `/recommendations/[id]/why` | Explainability, SHAP risk factors, adverse action notice | `GET /api/explainability/{id}` | user (scoped) |
| `/simulator` | Interactive what-if loan simulator | `POST /api/simulator` | user |
| `/foreclosure` | Early closure / foreclosure console | `POST /api/foreclosure` | user |
| `/approvals` | Underwriter review + decision queue | `GET /api/approvals`, `POST /api/approvals/{id}/decision` | loan officer |
| `/audit` | Append-only audit trail | `GET /api/audit` | loan officer |
| `/documents` | Document upload / listing per application | `GET/POST /api/documents/{id}` | user (scoped) |
| `/settings` | API health, policy config, model integrity (AUC / confusion matrix) | `GET /health`, `GET /api/model/metrics` | user / officer |

---

## API Surface & Security

- Every `/api/*` endpoint requires an `Authorization: Bearer <supabase-access-token>` header, except `/health` and `/api/health`.
- Access is role-scoped:
  - **Applicants** — create recommendations; view/simulate only their own applications; upload documents to their own applications.
  - **Loan officers** — full queue access, decisioning, audit trail, model metrics and retraining.
- Every state change writes an immutable row to `audit_log`.
- Rejected or counter-offer decisions automatically surface a rule-based **Adverse Action Notice**, generated from the affordability/risk numbers.
- Gemini-generated explanations are validated against the actual loan/rate/EMI/FOIR figures before being shown, with an automatic fallback to rule-based text on any mismatch.

---

## Production Build

```bash
npm run build
npm run start
```
