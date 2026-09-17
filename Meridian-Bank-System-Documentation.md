# Meridian Bank — Retail Banking Demo System

## Complete Technical & Functional Documentation

**Version:** 1.0 (build `45223e9`, platform version ID `02a8546`)
**Date:** September 2026
**Purpose:** Full-stack, realistic fictional retail banking system used as a sandbox for developing and demonstrating AI agents (customer-service agents, back-office automation agents, data-query agents).

> **Disclaimer:** Meridian Bank is a fictional demo institution. All customers, balances, transactions, and documents are synthetic. Not affiliated with any real bank.

---

## 1. Executive Summary

Meridian Bank is a production-grade-looking demo banking platform with **two portals in one application**:

| Portal | Audience | Theme | Modules |
|---|---|---|---|
| **Front Office** (customer portal) | Retail banking customers | Light, deep navy `#0B2545` + gold `#C9A227` | Dashboard, accounts, transactions, transfers, cards, loans, profile |
| **Back Office** (staff console) | Bank employees (Teller / Relationship Manager / Admin) | Dark slate `#0F1620` + teal `#2DD4A8` | KPI dashboard, customer 360°, KYC review, AML monitoring, loan approvals, card management, employees/branches, audit log |

**Live data, not mock-ups:** every screen is wired to a real tRPC API backed by a MySQL database seeded with 28 customers, 58 accounts, 997 transactions, 9 loans, 9 KYC cases, 6 AML alerts, and a 50-entry hash-chained audit log. Transfers move real balances; approvals change real workflow states; every action writes an audit record.

---

## 2. System Architecture

### 2.1 High-Level Architecture

```
┌──────────────────────────────────────────────────────────────────┐
│                         Browser (SPA)                            │
│  React 19 + TypeScript + Vite  ·  Tailwind CSS 3.4 + shadcn/ui   │
│                                                                  │
│  /  (public landing)    /banking/* (customer)  /staff/* (staff)  │
│         │                     │                      │           │
│         └───────────┬─────────┴──────────────────────┘           │
│                     │  tRPC client (@trpc/react-query)           │
│                     │  header: x-session-token                   │
└─────────────────────┼────────────────────────────────────────────┘
                      ▼  HTTP (superjson-encoded)
┌──────────────────────────────────────────────────────────────────┐
│                 Hono Node Server (port 3000)                     │
│  api/boot.ts → serves SPA (dist/) + mounts /trpc                 │
│                                                                  │
│  api/router.ts                                                   │
│   ├── auth.*      (login / me / logout)                          │
│   ├── customer.*  (14 procedures — scoped to logged-in customer) │
│   └── staff.*     (17 procedures — role-guarded)                 │
│                                                                  │
│  api/middleware.ts  →  public / protected / customer / staff /   │
│                        manager procedure guards                  │
│  api/context.ts     →  resolves session from x-session-token     │
└─────────────────────┼────────────────────────────────────────────┘
                      ▼  Drizzle ORM (type-safe queries, mysql2)
┌──────────────────────────────────────────────────────────────────┐
│                     MySQL 8 (portal-provisioned)                 │
│  16 tables — branches, customers, users, sessions, employees,    │
│  accounts, transactions, payees, cards, loanProducts, loans,     │
│  loanPayments, kycCases, alerts, notifications, auditLog         │
└──────────────────────────────────────────────────────────────────┘
```

### 2.2 Request Flow (example: customer transfer)

1. React page calls `trpc.customer.transfer.useMutation({ fromAccountId, payeeId, amount, description })`.
2. tRPC client attaches `x-session-token` from `localStorage` (`meridian_session`).
3. Hono receives the request; `api/context.ts` looks up the token in the `sessions` table and loads the user (role + customerId).
4. `customerProcedure` guard rejects if the session is missing or the role is not `customer`.
5. The procedure validates ownership, account status, and sufficient funds; inserts **paired Dr/Cr transaction rows**, updates both account balances, and appends a hash-chained `auditLog` entry — all via Drizzle ORM.
6. React Query invalidation refreshes the accounts/transactions lists; a Sonner toast confirms.

### 2.3 Authentication & Authorization

- **Credential-based demo auth** (deliberately *not* OAuth): seeded username/password pairs, SHA-256 password hashes in the `users` table.
- Login issues an **opaque session token** (crypto-random, 12-hour TTL) stored in the `sessions` table and the browser's localStorage.
- Role-based guards: `customerProcedure` (role = customer), `staffProcedure` (teller/manager/admin), `managerProcedure` (manager/admin — used for loan decisions and enhanced-KYC reviews).
- Frontend guards mirror this: `BankingLayout` redirects non-customers to `/login`; `StaffLayout` redirects non-staff to `/staff/login`.

---

## 3. Technology Stack

### 3.1 Frontend

| Layer | Technology | Version |
|---|---|---|
| Framework | React + TypeScript | 19.2 |
| Build tool | Vite | 7.2.4 |
| Styling | Tailwind CSS + shadcn/ui (40+ Radix-based components) | 3.4.19 |
| Routing | react-router (BrowserRouter) | 7.6 |
| Server state | @trpc/react-query + @tanstack/react-query | 11.8 / 5.90 |
| Serialization | superjson (decimals arrive as strings; Dates preserved) | 2.2 |
| Charts | recharts | 2.15 |
| Animation | Framer Motion (portals), GSAP + ScrollTrigger (landing), Lenis smooth scroll | 13.4 / 3.15 / 1.3 |
| Forms | react-hook-form + zod resolvers | 7.70 |
| Icons / dates / toasts | lucide-react, date-fns, sonner | — |
| Fonts | Libre Franklin (display), Inter (UI), JetBrains Mono (data/IDs) | Google Fonts |

### 3.2 Backend

| Layer | Technology | Version |
|---|---|---|
| HTTP server | Hono on @hono/node-server | 4.8 / 1.14 |
| API | tRPC server (end-to-end type-safe) | 11.8 |
| ORM | Drizzle ORM + drizzle-kit | 0.45 |
| DB driver | mysql2 | 3.14 |
| Validation | zod (every mutation/parameterized query input-validated) | 4.3 |
| Database | MySQL 8 (portal-provisioned) | — |
| Runtime | Node.js | 20 |

### 3.3 npm Scripts

| Command | Purpose |
|---|---|
| `npm run dev` | Dev server with HMR (port 3000) |
| `npm run build` | Production build (Vite + esbuild bundles `api/boot.ts` → `dist/`) |
| `npm start` | Run production server (`dist/boot.js`) |
| `npm run db:push` | Sync Drizzle schema to MySQL |
| `npm run db:generate` / `db:migrate` | Migration SQL generation / apply |
| `npx tsx db/seed.ts` | Idempotent wipe + reseed of demo data |
| `npm run check` / `test` / `lint` | Type-check / Vitest / ESLint |

---

## 4. Repository Layout

```
app/
├── api/                        # Backend (Hono + tRPC)
│   ├── boot.ts                 # Server entry (SPA + /trpc)
│   ├── router.ts               # appRouter: auth / customer / staff
│   ├── auth.ts                 # login, me, logout
│   ├── customer.ts             # 14 customer-facing procedures
│   ├── staff.ts                # 17 staff-facing procedures
│   ├── middleware.ts           # procedure guards (public/protected/customer/staff/manager)
│   ├── context.ts              # session resolution from x-session-token
│   ├── lib/                    # framework internals (do not modify)
│   └── queries/                # connection.ts, audit.ts helpers
├── db/
│   ├── schema.ts               # 16 tables (Drizzle, MySQL)
│   ├── relations.ts            # Drizzle relations
│   ├── seed.ts                 # idempotent demo-data seeder
│   └── sanity.ts               # API smoke-test script
├── contracts/types.ts          # Shared enums/types (roles, statuses, SessionUser)
├── src/
│   ├── main.tsx                # TRPCProvider wiring
│   ├── App.tsx                 # All 19 routes
│   ├── hooks/useSession.ts     # Session state (localStorage 'meridian_session')
│   ├── providers/trpc.tsx      # tRPC client (attaches x-session-token)
│   ├── components/
│   │   ├── Navbar/Footer/Layout     # public landing shell
│   │   ├── shared/                  # StatusBadge, MoneyText, AccountNumber, KpiCard, DataTable
│   │   ├── banking/BankingLayout    # customer portal shell (navy sidebar)
│   │   └── staff/StaffLayout        # staff console shell (dark sidebar)
│   └── pages/
│       ├── Home.tsx                 # marketing landing
│       ├── Login.tsx                # customer login
│       ├── banking/                 # 7 customer pages
│       └── staff/                   # 10 staff pages
├── public/                     # Generated brand assets (logos, card art, KYC specimen docs, avatars)
├── design/                     # 19 design documents (source of truth for UI)
└── drizzle.config.ts
```

---

## 5. Database Schema (16 tables)

All tables use `bigint unsigned auto_increment` primary keys; monetary values are `decimal(14,2)`; FK columns are `bigint unsigned`.

| Table | Key columns | Notes |
|---|---|---|
| `branches` | code (BR-001…), name, address, phone | 3 seeded branches |
| `customers` | customerNo (CUS-####), name, email, phone, dob, address, ssnLast4, **kycStatus** (verified/pending/under_review/rejected), branchId | 28 seeded |
| `users` | username, passwordHash (SHA-256), **role** (customer/teller/manager/admin), customerId, lastLoginAt | 5 seeded logins |
| `sessions` | token, userId, expiresAt | 12-hour TTL |
| `employees` | employeeNo, name, role, branchId, status, hiredAt | 6 seeded |
| `accounts` | accountNumber, last4, **type** (checking/savings/credit/mortgage), balance, apy, creditLimit, status (active/frozen/closed), customerId, branchId | 58 seeded |
| `transactions` | txnId (TXN-YYYY-MMDD-####), accountId, counterAccountId, **type** (transfer/deposit/withdrawal/fee/interest/payment/purchase), amount, direction (dr/cr), merchant, category, **status** (posted/pending/flagged), flagReason, occurredAt | 997 seeded, Nov 2024–Feb 2025 |
| `payees` | customerId, name, accountNumber, bank | transfer counterparties |
| `cards` | last4, type (debit/credit), network, holder, status (active/frozen/blocked/pending), creditLimit, availableCredit, expiresAt | 8 seeded incl. frozen/blocked/pending |
| `loanProducts` | code, name, rateApr, min/maxAmount, termMonths | Personal / Auto / 30-yr Mortgage |
| `loans` | loanNo (LN-YYYY-####), principal, remainingBalance, rateApr, monthlyPayment, purpose, **status** (pending/approved/rejected/active/delinquent/paid_off), decidedBy, branchId | 9 seeded — every lifecycle state |
| `loanPayments` | loanId, dueDate, amount, principalPart, interestPart, status (paid/due/overdue) | 40 seeded (amortized schedules) |
| `kycCases` | caseNo, customerId, level (standard/enhanced), status, idDocType, riskRating (low/medium/high), reviewedBy, rejectionReason | 9 seeded, 4 open |
| `alerts` | alertNo (AML-…), **type** (structuring/high_velocity/unusual_amount/geo_anomaly), severity (low/medium/high/critical), status (open/investigating/resolved/dismissed), transactionId | 6 seeded |
| `notifications` | customerId, title, body, type, read | in-app alerts inbox |
| `auditLog` | actorType (customer/staff/system), actorName, action, entityType, entityId, detail, **prevHash, hash** (SHA-256 chain) | tamper-evident log; 50 seeded + every mutation appends |

---

## 6. API Reference (tRPC routers)

### 6.1 `auth.*`
| Procedure | Guard | Description |
|---|---|---|
| `login(username, password)` | public | Verifies credentials, creates session, returns `{ token, user }` |
| `me` | protected | Returns current session user |
| `logout` | protected | Deletes session |

### 6.2 `customer.*` (scoped to the logged-in customer)
| Procedure | Description |
|---|---|
| `dashboardSummary` | Accounts + balances, 8 recent transactions, unread notifications |
| `listAccounts` | All customer accounts grouped by type |
| `accountDetail(id, page, pageSize, search, type, direction, from, to)` | Account + filtered/paginated ledger + running balance |
| `listCards` / `setCardStatus(id, status)` | Cards; freeze/unfreeze |
| `listLoans` / `listLoanProducts` | Loans with next payment + amortization schedule; product catalog |
| `applyForLoan(productId, amount, termMonths, purpose)` | Creates pending loan + audit entry; validates product bounds |
| `listPayees` / `transfer(fromAccountId, toAccountId/payeeId, amount, description)` | Paired Dr/Cr postings, balance updates, audit entry |
| `listNotifications` / `markNotificationRead` | Alerts inbox |
| `profile` / `updateProfile` | Customer profile management |

### 6.3 `staff.*` (role-guarded)
| Procedure | Guard | Description |
|---|---|---|
| `dashboardKpis` | staff | Total deposits, active loans, pending KYC, flagged txns, open alerts, 30-day deposit series |
| `listCustomers(search, status, page)` | staff | Directory with pagination |
| `customer360(id)` | staff | Full customer file: accounts, cards, loans, KYC, transactions, audit trail |
| `kycQueue(status)` / `reviewKyc(caseId, approve, reason)` | staff / **manager for enhanced** | KYC workflow |
| `monitoringTxns(filters, page)` / `flaggedQueue` | staff | Transaction monitoring |
| `listAlerts` / `updateAlert(id, status)` | staff | AML alerts |
| `loanQueue(status)` / `decideLoan(loanId, approve, reason)` | staff / **manager** | Approvals; approval generates full amortization schedule |
| `cardRequests` / `decideCardRequest` / `listCards` / `setCardStatus` | staff | Card issuance + block/unblock |
| `listEmployees` / `listBranches` | staff | Org data |
| `auditLog(page, filters)` | staff | Hash-chained log viewer feed |

---

## 7. Application Pages (19 routes)

### 7.1 Public & Front Office (customer portal)
| Route | Page | Highlights |
|---|---|---|
| `/` | Marketing landing | GSAP pinned product carousel, rates marquee, Ken Burns hero, trust/security sections |
| `/login` | Customer login | Split-screen, demo credential chips, error shake, 2FA-style UX |
| `/banking` | Dashboard | Balance count-up cards, spending donut, recent activity, quick actions, notifications |
| `/banking/accounts` | Accounts | Grouped by type, masked numbers with reveal/copy |
| `/banking/accounts/:id` | Account detail | Server-filtered/paginated ledger, e-statements tab, expandable rows |
| `/banking/transfers` | Transfers | 3-step wizard (own accounts / payees), review & confirm, receipt |
| `/banking/cards` | Cards | 3D-tilt card art, live freeze/unfreeze, payment modal |
| `/banking/loans` | Loans | Repayment schedules, payment history, application wizard with live payment calculator |
| `/banking/profile` | Profile | Inline editing, security settings, notification preferences, alerts inbox |

### 7.2 Back Office (staff console — dark theme)
| Route | Page | Highlights |
|---|---|---|
| `/staff/login` | Staff login | Role selector with quick-fill demo accounts |
| `/staff` | Staff dashboard | KPI cards w/ sparklines, 30-day deposits chart, live work-queue previews |
| `/staff/customers` | Customer directory | Debounced search, KYC filters, server pagination, CSV export |
| `/staff/customers/:id` | Customer 360° | Left summary rail + 7 tabs (Overview, Accounts, Cards, Loans, KYC, Transactions, Notes/Audit) |
| `/staff/kyc` | KYC review queue | Document viewer (specimen ID + utility bill), checklist, approve/reject with reason, EDD maker-checker (tellers blocked) |
| `/staff/monitoring` | Transaction monitoring | Severity-colored AML alerts, detail drawer with rule info, flagged transaction strip, detection-rules tab |
| `/staff/loans` | Loan approvals | DTI gauge, credit-score bar, system recommendation, >$50k maker-checker banner, amortization preview |
| `/staff/cards` | Card management | Issuance request approvals, freeze/block/unblock with confirm dialogs |
| `/staff/employees` | Employees & branches | Directory with role/branch filters, branch cards, add-employee modal (demo-simulated) |
| `/staff/audit` | Audit log | Filterable hash-chained log (prevHash→hash chips), maker-checker pending approvals panel, CSV export |

---

## 8. Demo Data (Seeded Content)

**Time frame:** seeded "today" = **February 14, 2025**; transactions span **November 2024 – February 2025**.

### 8.1 Counts
| Entity | Count |
|---|---|
| Branches | 3 (Downtown HQ BR-001, Westside BR-002, Harbor Point BR-003) |
| Customers | 28 |
| Login users | 5 (1 customer + 3 staff + 1 extra) |
| Employees | 6 |
| Accounts | 58 |
| Transactions | **997** (Elena's checking alone: 67) |
| Cards | 8 (active, frozen, blocked, 2 pending issuance) |
| Loans | 9 — every state: pending, approved, rejected, active, delinquent, paid_off |
| Loan payments | 40 (paid history + upcoming dues) |
| KYC cases | 9 (4 pending/under-review incl. 1 enhanced high-risk, 1 rejected) |
| AML alerts | 6 (severities up to critical; open + investigating) |
| Notifications | 6 (3 unread) |
| Audit entries | 50 (hash-chained, incl. maker-checker pending wire) |

### 8.2 Primary Demo Customer — Elena Vasquez
Customer since 2016, KYC **Verified**:
| Product | Number | Balance / Detail |
|---|---|---|
| Everyday Checking | •••• 4821 | **$12,450.33** |
| High-Yield Savings | •••• 9034 | **$48,920.15** @ 4.35% APY |
| Platinum Rewards Credit Card | •••• 7712 | $2,340.18 used of $15,000 limit |
| 30-yr Fixed Mortgage | LN-2019-0442 | $312,400.00 remaining (24 payments made, 12 upcoming) |

Realistic transaction texture: payroll deposits (1st/15th), rent, Whole Foods, Shell, Netflix, Amazon, Con Edison, Starbucks, CVS, Zelle — plus a flagged **$9,800 international wire (INT'L TRANSFER SARL)** that drives the AML structuring alert end-to-end across portals.

### 8.3 Staff Personas
| Name | Role | Console permissions |
|---|---|---|
| Marcus Chen | **Admin** | Everything, incl. maker-checker approvals, enhanced KYC |
| Priya Sharma | **Relationship Manager** | Loan decisions, KYC reviews, customer management |
| Jordan Ellis | **Teller** | Read queues, card ops; blocked from manager-only decisions (server-enforced) |

---

## 9. How to Log In (Demo Credentials)

### Customer portal → `/login`
| Username | Password |
|---|---|
| `elena.vasquez@demo.meridian` | `demo1234` |

### Staff console → `/staff/login`
| Username | Password | Role |
|---|---|---|
| `marcus.chen@meridian.bank` | `admin1234` | Admin |
| `priya.sharma@meridian.bank` | `manager1234` | Relationship Manager |
| `jordan.ellis@meridian.bank` | `teller1234` | Teller |

Login pages display these credentials as quick-fill chips for friction-free demos. Sessions last 12 hours.

---

## 10. Design System

| | Front Office | Back Office |
|---|---|---|
| Canvas | `#F4F6F9` light | `#0F1620` dark, panels `#161F2B` |
| Primary | Navy `#0B2545` | Teal `#2DD4A8` |
| Accent | Gold `#C9A227` | Amber/red alert states |
| Typography | Libre Franklin headings, Inter body | Inter UI, **JetBrains Mono** for IDs/amounts |
| Density | 1280px content, `rounded-xl` cards | 44px table rows, zebra, sticky headers |
| Motion | Framer Motion fade/rise, balance count-ups | Snappy 0.15s, right-slide drawers, toasts |
| Shared | StatusBadge (12 states), MoneyText (tabular, ± coloring), AccountNumber (mask + reveal + copy), KpiCard, DataTable | Same components, dark variants |

Trust details everywhere: masked account numbers, tabular-nums currency, FDIC-style demo disclaimer, routing number 021000089 (demo), "Last sign-in" header, session/audit realism.

---

## 11. How It Was Built (Process)

1. **Research** — evaluated 15+ open-source banking projects (Apache Fineract, Mifos, Bank of Anthos, etc.); borrowed Fineract's domain model, Bank of Anthos's ledger pattern, and the Czech retail-banking dataset's realism. Conclusion: build fresh on a modern stack rather than fork legacy code.
2. **Design-first** — a dedicated designer agent produced 19 page-level design documents (`design/`) with exact palettes, data anchors, and animation specs before any code.
3. **Scaffold** — landing page + shared components (StatusBadge/MoneyText/AccountNumber/KpiCard/DataTable) + generated brand assets.
4. **Backend graft** — tRPC + Drizzle + Hono + MySQL grafted via backend-building-swarm; 16-table schema, 3 routers (34 procedures), custom credential auth, idempotent seeder.
5. **Parallel page agents** — three agents built front-office (8 pages), back-office core (5 pages), and risk/ops (6 pages) simultaneously in isolated git worktrees.
6. **Integration** — octopus merge, route wiring in `App.tsx`, production build gate, versioned delivery (`build_version`, dynamic full-stack).

---

## 12. Known Limitations & Demo Simulations

- A few UI elements are intentionally demo-simulated (marked "(demo)" in toasts): employee add/edit, detection-rule toggles, SAR escalation, password change.
- Card financial fields (APR, rewards) use design-spec display values; card status changes are fully live.
- Branch snapshot and channel-mix donut on the staff dashboard use static design data (no endpoint).
- This is a demo system: SHA-256 password hashing and specimen KYC documents are not production-grade security.
- Interest accrual, statement PDF generation, and real payment rails are out of scope.

## 13. Suggested Next Steps (for agent demos)

- **Customer-service agent**: ground on `customer.*` procedures (balances, transactions, cards, loans) + notifications.
- **Back-office agent**: drive `staff.*` workflows — KYC review, alert triage, loan recommendations.
- **Analytics agent**: read-only SQL/tRPC over the 997-transaction ledger and audit log.
- Optional extensions: MCP tool wrapper over the tRPC API, statement PDF export, more seeded customers via `db/seed.ts`.
