# 🏦 Meridian Bank — Retail Banking Demo System

> A full-stack, **realistic fictional retail bank** built as a sandbox for exploring **agentic AI** — customer-service agents, back-office automation agents, and data/analytics agents — against real banking workflows.

**Meridian Bank is a fictional demo institution. All data is synthetic.**

![Stack](https://img.shields.io/badge/React-19-61dafb) ![tRPC](https://img.shields.io/badge/tRPC-11-2596be) ![Drizzle](https://img.shields.io/badge/Drizzle_ORM-0.45-c5f74f) ![MySQL](https://img.shields.io/badge/MySQL-8-4479a1) ![TypeScript](https://img.shields.io/badge/TypeScript-5-3178c6)

---

## Why This Exists

Building AI agents for banking requires something that **behaves like a real bank**: dense data, real workflows, role-based permissions, and consequences (transfers move balances, approvals change states, every action hits an audit log). Meridian Bank is that environment:

- 🤖 **Agent-ready API** — 34 end-to-end type-safe tRPC procedures (`auth.*`, `customer.*`, `staff.*`) that map 1:1 to banking operations
- 🏦 **Two full portals** — customer online banking + staff back-office console
- 📊 **Rich seeded data** — 28 customers, 58 accounts, **997 transactions**, loans in every lifecycle state, KYC cases, AML alerts, hash-chained audit log
- 🔐 **Role-based access** — customer / teller / relationship manager / admin, enforced server-side
- 🧾 **Maker-checker workflows** — enhanced KYC, large loans, and wires requiring second approval

---

## Screens & Modules

### Front Office — Customer Portal (light, navy/gold)

| Route | Module |
| --- | --- |
| `/` | Marketing landing page (GSAP scroll storytelling) |
| `/login` | Customer login |
| `/banking` | Dashboard — balance cards, spending insights, notifications |
| `/banking/accounts` | Account list grouped by type |
| `/banking/accounts/:id` | Account detail — searchable/filterable/paginated ledger, e-statements |
| `/banking/transfers` | 3-step transfer wizard (own accounts + payees) with live balance updates |
| `/banking/cards` | Debit/credit cards — freeze/unfreeze (live), limits |
| `/banking/loans` | Loans with amortization schedules + application wizard with payment calculator |
| `/banking/profile` | Profile, security, notification preferences, alerts inbox |

### Back Office — Staff Console (dark, teal)

| Route | Module |
| --- | --- |
| `/staff/login` | Staff login with role selector |
| `/staff` | KPI dashboard — deposits, active loans, pending KYC, flagged txns, 30-day chart |
| `/staff/customers` | Customer directory — search, KYC filters, CSV export |
| `/staff/customers/:id` | Customer 360° — 7 tabs (accounts, cards, loans, KYC, txns, notes, audit) |
| `/staff/kyc` | KYC review queue — document viewer, approve/reject, EDD maker-checker |
| `/staff/monitoring` | AML transaction monitoring — alerts, severities, flagged queue, rules |
| `/staff/loans` | Loan approval queue — DTI gauge, approve/reject, amortization generation |
| `/staff/cards` | Card issuance approvals, block/unblock management |
| `/staff/employees` | Employee directory + branch management |
| `/staff/audit` | Hash-chained audit log viewer + maker-checker approvals panel |

---

## Tech Stack

**Frontend:** React 19 · TypeScript · Vite 7 · Tailwind CSS 3.4 · shadcn/ui (Radix) · @trpc/react-query · recharts · Framer Motion · GSAP + Lenis (landing) · Libre Franklin / Inter / JetBrains Mono

**Backend:** Hono (Node 20) · tRPC 11 · Drizzle ORM · zod · MySQL 8 · superjson

**Auth:** Credential-based demo auth — SHA-256 password hashes, opaque session tokens (12h TTL) in a `sessions` table, role-guarded tRPC procedures.

---

## Architecture

```javascript
Browser SPA (React 19 + Vite)
 ├── /            public landing
 ├── /banking/*   customer portal  ──┐
 └── /staff/*     staff console    ──┤  tRPC over HTTP (superjson)
                                     │  header: x-session-token
Hono server (api/boot.ts, :3000)     ▼
 ├── auth.*      login / me / logout
 ├── customer.*  14 procedures (scoped to session customer)
 └── staff.*     17 procedures (teller/manager/admin guards)
        │  Drizzle ORM (type-safe, mysql2)
MySQL 8 — 16 tables:
 branches · customers · users · sessions · employees · accounts
 transactions · payees · cards · loanProducts · loans · loanPayments
 kycCases · alerts · notifications · auditLog (SHA-256 hash chain)
```

---

## Getting Started

```bash
# 1. Install
npm install

# 2. Configure database
cp .env.example .env   # set DATABASE_URL=mysql://user:pass@host:3306/meridian

# 3. Push schema + seed demo data
npm run db:push
npx tsx db/seed.ts

# 4. Run
npm run dev            # dev server w/ HMR → http://localhost:3000
# or production:
npm run build && npm start
```

Useful scripts: `npm run check` (typecheck) · `npm test` (Vitest) · `npm run db:generate` / `db:migrate` · `npx tsx db/sanity.ts` (API smoke test)

---

## 🔑 Demo Credentials

**Customer portal** (`/login`)

| Username | Password |
| --- | --- |
| `elena.vasquez@demo.meridian` | `demo1234` |

**Staff console** (`/staff/login`)

| Username | Password | Role |
| --- | --- | --- |
| `marcus.chen@meridian.bank` | `admin1234` | Admin |
| `priya.sharma@meridian.bank` | `manager1234` | Relationship Manager |
| `jordan.ellis@meridian.bank` | `teller1234` | Teller |

---

## Seeded Demo Data

Time frame: seeded "today" = **Feb 14, 2025**; transactions span Nov 2024 – Feb 2025.

- **3 branches** — Downtown HQ, Westside, Harbor Point
- **28 customers** — incl. anchor customer **Elena Vasquez** (checking $12,450.33 · savings $48,920.15 @ 4.35% APY · platinum card $2,340.18/$15,000 · mortgage LN-2019-0442 $312,400 remaining)
- **997 transactions** — payroll, rent, Whole Foods, Shell, Netflix, Zelle… plus a flagged $9,800 international wire that drives an end-to-end AML structuring alert
- **9 loans** in every state (pending → approved/rejected → active → delinquent/paid_off) with 40 amortized payments
- **9 KYC cases** (4 open, incl. 1 enhanced high-risk) · **6 AML alerts** (up to critical) · **50-entry hash-chained audit log**

Reseed anytime: `npx tsx db/seed.ts` (idempotent wipe + reload).

---

## 🤖 Using It for Agentic Exploration

The tRPC API is the agent surface — every procedure is a potential tool:

| Agent persona | Ground on |
| --- | --- |
| Customer-service agent | `customer.dashboardSummary`, `accountDetail`, `transfer`, `setCardStatus`, `listLoans`, `applyForLoan` |
| Back-office ops agent | `staff.kycQueue`/`reviewKyc`, `listAlerts`/`updateAlert`, `loanQueue`/`decideLoan`, `cardRequests` |
| Compliance/audit agent | `staff.auditLog`, `monitoringTxns`, `flaggedQueue` |
| Analytics agent | read-only queries over the 997-txn ledger + audit trail |

Router definitions with zod input schemas: `api/customer.ts`, `api/staff.ts`, `api/auth.ts` — self-documenting tool contracts for function-calling agents.

---

## Project Structure

```javascript
api/            Hono + tRPC backend (routers, guards, session context)
db/             Drizzle schema (16 tables), relations, idempotent seeder, sanity tests
contracts/      Shared enums/types crossing the client-server boundary
src/
 ├── pages/     19 routes (landing, banking/*, staff/*)
 ├── components/ shared (StatusBadge, MoneyText, AccountNumber, KpiCard, DataTable)
 │               + banking/ & staff/ shells
 ├── hooks/      useSession (localStorage session)
 └── providers/  tRPC client (attaches x-session-token)
design/         19 page-level design documents (source of truth for UI)
```

---

## Disclaimer

Demo/simulation only — not affiliated with any real financial institution. SHA-256 password hashing and specimen KYC documents are demo-grade, **not production security**. Some UI actions (employee management, rule toggles, SAR filing) are intentionally simulated and marked "(demo)".
