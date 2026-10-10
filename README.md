RadarPay — MSME Cash-Flow Early-Warning Dashboard

RadarPay helps small Indian businesses (MSMEs) see cash problems before they happen. It models how each customer actually pays, forecasts the next 60 days of cash with uncertainty, and calculates the indicative interest and tax exposure from paying registered micro and small suppliers late. Then it tells you whom to chase and whom to pay first.

Everything runs in the browser. CSV files are parsed locally and are never uploaded. Imported data is kept only in that browser's local storage.

Disclaimer: Interest (MSMED Act Section 16) and tax (Income Tax Section 43B(h)) figures are indicative estimates, not legal or tax advice. Have a Chartered Accountant verify them before relying on them.

What RadarPay does
The problem
Customers pay late, so expected receipts arrive after their due dates.
Suppliers have statutory deadlines. For registered micro and small suppliers, payment is due by the "appointed day" (the agreed credit period, capped at 45 days; 15 days if nothing was agreed). Paying late attracts compound interest at 3× the RBI bank rate (Section 16), and the expense may lose its tax deduction if unpaid by year end (Section 43B(h)).

Most owners can't see how these two combine over the coming weeks. RadarPay does.

Pages
Page	Purpose
Overview	Open receivables and payables, probability-weighted late amount, accrued interest, 43B(h) exposure, and a 60-day cash forecast chart (P10 / P50 / P90 bands).
Import ledger	Upload receivables and payables as CSV, map columns manually if needed, and set the opening cash balance.
Cash actions	Two ranked queues: customers to chase and suppliers to pay.
What if?	Shift one buyer's expected payment timing (−30 to +90 days) and re-run the same seeded forecast. No ledger data is changed.
Payment notice	Drafts a polite or firm payment-reminder with an indicative interest annexure.
Model check	Backtests the model on held-out paid invoices against a "pays on the due date" baseline.
Assumptions	Shows the RBI rate history, tax rate, cash buffer and model limitations.

A fictional demo ledger (eight buyer behaviour patterns, six suppliers) loads on first open so you can explore before importing real data.

How it works (artifacts/payradar/src/lib/engine.ts)
Buyer behaviour model. For each buyer, payment times of already-paid invoices are fitted to a log-normal distribution. Buyers with little history are shrunk toward the overall average.
Cash forecast. A seeded Monte Carlo simulation (about 800 paths) draws a payment date for each open invoice and places supplier payments on their deadlines. The output is P10 / P50 / P90 cash per day, the daily probability of breaching the minimum buffer, and the first likely shortfall date. The fixed seed makes results reproducible.
Statutory exposure. Appointed day per supplier, Section 16 interest (3× bank rate, monthly rests), and Section 43B(h) exposure, applied only to suppliers marked as registered micro or small.
Action ranking. Customers are ranked by late probability, amount and accrued interest. Suppliers are ranked by deadline proximity, exposure and whether paying fits within the P10 cash buffer.
Backtest. A time-ordered holdout compares the learned model's error (MAE in days) against the due-date-only baseline and reports P10–P90 interval coverage.
CSV formats

Dates use YYYY-MM-DD.

Receivables: invoice_id, buyer, amount, invoice_date, acceptance_date, agreed_credit_days, paid_date, paid_amount
Payables: invoice_id, vendor, amount, invoice_date, acceptance_date, agreed_credit_days, vendor_category, udyam_registered, paid_date

Rows are merged by invoice ID (matching IDs are replaced). A receivables template can be downloaded from the Import page.

Run & Operate

Prerequisites: Node.js 24 (20+ should work) and pnpm 10.16+ (npm install -g pnpm@latest). The project refuses npm and yarn.

bash
pnpm install --no-frozen-lockfile      # first-time install
Web app (the dashboard)

The Vite config requires PORT and BASE_PATH:

bash
PORT=18955 BASE_PATH=/ pnpm --filter @workspace/payradar run dev      # dev server
PORT=18955 BASE_PATH=/ pnpm --filter @workspace/payradar run build    # production build

Then open http://localhost:18955. The production build is static and is written to artifacts/payradar/dist/public. When hosting it, rewrite all paths to /index.html.

API server (optional)

The dashboard does not call the API server. It currently exposes a health check only (/api/healthz) and needs Postgres to start.

bash
DATABASE_URL=postgres://... PORT=8080 pnpm --filter @workspace/api-server run dev
Other commands
pnpm run typecheck — full typecheck across all packages
pnpm run build — typecheck and build all packages
pnpm --filter @workspace/api-spec run codegen — regenerate API hooks and Zod schemas from the OpenAPI spec
pnpm --filter @workspace/db run push — push DB schema changes (dev only)
Required env for the API/DB packages: DATABASE_URL — Postgres connection string
Stack
pnpm workspaces, Node.js 24, TypeScript 5.9
Web: React 19, Vite, Tailwind CSS 4, wouter (routing), TanStack Query, Radix UI components, custom SVG charts
API: Express 5
DB: PostgreSQL + Drizzle ORM
Validation: Zod (zod/v4), drizzle-zod
API codegen: Orval (from the OpenAPI spec)
Build: esbuild (API server bundle)
Where things live
Path	What's there
artifacts/payradar/	The dashboard (React + Vite)
artifacts/payradar/src/App.tsx	All pages and UI
artifacts/payradar/src/lib/engine.ts	Forecasting, statutory calculations, ranking, backtest, CSV parsing, demo data
artifacts/api-server/	Express API server
artifacts/mockup-sandbox/	Component preview sandbox (design tooling, not part of the product)
lib/api-spec/openapi.yaml	API contract (source of truth)
lib/api-client-react/, lib/api-zod/	Generated API hooks and Zod schemas
lib/db/src/schema/	Database schema (Drizzle)
pnpm-workspace.yaml	Workspace layout and shared dependency versions (catalog)
Architecture decisions
Local-first. All parsing and calculation happens client-side, which keeps financial data private and avoids needing a backend to use the product.
Deterministic forecasting. The Monte Carlo uses a fixed seed, so the same ledger always gives the same forecast. This keeps What-if comparisons meaningful.
Indicative, transparent assumptions. Rates, tax rate and cash buffer are visible on the Assumptions page, and every statutory figure is labelled as needing professional verification.
Spec-driven API. Client hooks and validators are generated from openapi.yaml, so the contract lives in one place.
Gotchas
PORT and BASE_PATH must both be set, or the Vite config throws on startup.
Use pnpm, not npm or yarn.
The API server and lib/db throw at startup if DATABASE_URL is missing.
Forecast and backtest quality depends on how many paid invoices exist per buyer. With small samples, treat results as a warning signal, not a guarantee.
RBI bank-rate history lives in DEFAULT_RATES in engine.ts. Add a new entry when the rate changes.
