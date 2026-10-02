# PortfolioPilot — Milestone 10: Connect the Portfolio UI and Watchlist

This learning activity connects the React interface to authenticated, persisted portfolio data. It builds on the authentication, transaction APIs, and valuation calculations from milestones 07–09.

The implementation and verification described below were completed on **October 2, 2026**. Setup instructions explain how to reproduce the activity; they are not a claim that every setup command was rerun during milestone 10.

## 1. Purpose and What You Will Learn

The prompt asks the coding agent to replace sample portfolio screens with real API data and build portfolio management dialogs, a buy/sell form, transaction history, holdings, allocation charts, summary cards, and watchlist management.

This matters because a financial interface must show the signed-in user's saved records and trustworthy calculations. A polished screen is not enough if refreshing loses data, another user can access it, or rounding changes the accounting.

You will learn how to:

- Connect React screens to an **API**, the server endpoints that read and change application data.
- Use an authenticated session to enforce **ownership**: Alice's resources belong to Alice, and Bob's belong to Bob.
- Keep money and quantities as decimal strings, such as `"106.8666666667"`, instead of converting them to JavaScript floating-point numbers.
- Display metrics calculated by the server rather than rebuilding accounting formulas in the browser.
- Handle loading, empty data, errors, expired sessions, and stale prices explicitly.
- Combine disabled submission controls with **idempotency**, which lets the server recognize a retry and avoid recording the same trade twice.
- Verify a complete user journey with an **end-to-end test** that drives Chrome against the actual application and database.

The app records purchases and sales that have already happened. It does not place broker orders.

## 2. Implementation Steps Actually Performed

### Step 1: Inspect the project and existing behavior

The agent read `AGENTS.md`, the project state, milestone plan, architecture decision index, frontend, shared contracts, database services, API handlers, and tests. It confirmed that portfolio APIs already enforced ownership, validated the transaction ledger, supported idempotent posting, and calculated valuations on the server. The frontend still displayed portfolio and watchlist fixtures.

A **ledger** is the chronological record of purchases and sales. Its complete history is needed to calculate remaining holdings and weighted average cost correctly.

Version-sensitive Route Handler behavior was checked against installed Next.js documentation and the official reference. Query options and Prisma database operations were checked against installed type definitions. No dependencies were upgraded.

### Step 2: Add shared response contracts and watchlist APIs

The agent extended `packages/contracts/src/index.ts` with schemas for portfolio records, transaction pages, security choices, and watchlist responses. A **schema** describes and validates the expected data shape; this project uses Zod for that validation.

It created `packages/db/src/watchlist-service.ts` and these authenticated routes:

| Endpoint | Purpose |
| --- | --- |
| `GET /api/securities` | List supported USD stocks with symbol, exchange, name, and ID |
| `GET /api/watchlist` | Read the signed-in user's watchlist |
| `POST /api/watchlist` | Add a security; repeated adds reuse the existing entry |
| `PATCH /api/watchlist/:id` | Change an owned entry's security |
| `DELETE /api/watchlist/:id` | Remove an owned entry |

The service derives the owner from the verified session. It rejects browser-supplied owner fields and filters edits/removals by owner and entry ID. ACME on XNAS and ACME on XNYS remain separate securities. An **exchange MIC** is the four-character identifier for a trading venue.

### Step 3: Connect the portfolio and watchlist screens

The main implementation lives in `apps/web/src/portfolio.tsx`. The agent connected the Dashboard and Portfolios routes to authenticated API queries and added:

- Create, rename, and archive portfolio dialogs.
- A buy/sell form with security selection, quantity, price, fees, and UTC execution time.
- Transaction history with ten records per page.
- Holdings, four summary cards, and allocation bars based on server metrics.
- Watchlist add/edit/remove controls with exchange-aware selection.

Archiving makes a portfolio read-only while preserving its history. Transactions remain immutable: the UI does not edit or delete recorded trades.

### Step 4: Preserve precision and handle interaction states

`apps/web/src/lib/decimal-display.ts` uses BigInt and string operations for display formatting. **BigInt** represents integers exactly, which helps avoid floating-point rounding errors. Money displays two decimal places; quantities retain their nonzero fractional digits. The UI keeps raw form inputs and uses server calculations for accounting.

Submission controls use a synchronous lock and disabled fields while a request is pending. An unchanged trade retry keeps its idempotency key while its dialog stays open. This supplements the existing backend safeguards.

Native modal dialogs provide keyboard containment, cancellation, and focus restoration. A server `401 Unauthorized` response clears cached account data and returns the user to sign-in.

### Step 5: Test, correct issues, and document the result

The agent ran:

```powershell
npm ci --ignore-scripts --offline
npm run typecheck
npm run build
npm run check:browser-boundary
npm run test
npm run test:browser --workspace @portfolio-pilot/web
```

The test runs used the dedicated database variables documented below. The offline install worked because the required packages were cached. The first typecheck failed when sandbox permissions prevented Prisma from touching its installed engine cache; the approved retry passed.

Browser testing exposed outdated or ambiguous test selectors, Chrome's date-input normalization, and a test expectation that needed to match the existing transaction API's HTTP 200 response. It also found a real dialog focus-restoration bug under React Strict Mode, which runs extra setup/cleanup checks during development. These issues were corrected before the final successful run.

The agent updated the lesson notes and project state. No database migration, dependency version, lockfile, instruction file, or milestone-plan change was needed.

### Files Created or Modified

| Area | Files |
| --- | --- |
| New frontend code | `apps/web/src/portfolio.tsx`, `apps/web/src/lib/decimal-display.ts` |
| Modified frontend code | `apps/web/src/app.tsx`, `auth.tsx`, `styles.css`, `lib/api-client.ts` |
| Shared contracts | `packages/contracts/src/index.ts` |
| Database service | New `packages/db/src/watchlist-service.ts`; modified `packages/db/src/index.ts` |
| New API routes | `apps/api/app/api/securities/route.ts`, `watchlist/route.ts`, `watchlist/[id]/route.ts` |
| Modified API support | `apps/api/lib/authorization.ts`, `portfolio-http.ts` |
| Tests | New `apps/web/src/lib/decimal-display.test.ts` and `apps/web/e2e/portfolio.spec.ts`; modified `apps/web/e2e/shell.spec.ts` and `apps/api/lib/portfolio.integration.test.ts` |
| Teaching records | `docs/lessons/10-portfolio-ui-and-watchlist.md`, `docs/project-state.md` |

This README was added afterward to explain the completed activity. No existing root README was present.

## 3. Results Achieved

The application now reads saved portfolios and watchlists from PostgreSQL, the authoritative database. Alice and Bob can independently manage their data. Refreshing preserves records; after refreshing, you may need to reselect the portfolio in the shared selector.

The browser acceptance scenario observed the following results for ACME / XNAS / USD:

| UTC execution date | Side | Quantity | Price (USD) | Fees (USD) |
| --- | --- | --- | --- | --- |
| 2025-01-01 00:00 | Buy | 10 | 100 | 2 |
| 2025-01-02 00:00 | Buy | 5 | 120 | 1 |
| 2025-01-03 00:00 | Sell | 6 | 130 | 3 |

```text
Remaining quantity:       9
Remaining cost basis:     USD 961.80
Realized gain / loss:     USD 135.80
Market value:             USD 1,125.00
Unrealized gain / loss:   USD 163.20
Allocation:               100.00%
```

The last three values were verified using a temporary **fresh synthetic quote of USD 125** inserted by the test. Synthetic means invented teaching data, not a current market price. Historical seed quotes normally appear stale, so manual runs correctly show `Unavailable` for market value, unrealized gain/loss, and allocation until a usable fresh quote exists.

Quote rows display UTC timestamps, provider information, currency context, synthetic labels, and freshness status. Sold-out positions remain visible and do not require a quote. Empty portfolios show guidance instead of invented holdings.

### Observed Verification Results

| Check | Final observed result |
| --- | --- |
| Offline pinned install | 347 packages installed; zero audit vulnerabilities reported |
| Workspace typecheck | Passed after the Prisma cache permission retry |
| Workspace production build | Passed |
| Final frontend build | Passed after the dialog/encoding correction, including TypeScript validation |
| Browser dependency boundary | Passed; frontend packages do not depend on server packages |
| Workspace tests with both integration databases enabled | **57 passed, none skipped** |
| Chrome browser tests | **4 passed in 24.3 seconds** |

The browser scenario checked reference metrics, refresh persistence, two-user isolation, canceled/confirmed removal, sold-out holdings, pagination, rename/archive persistence, and return to sign-in after a simulated 401. The other scenarios checked desktop routes, mobile navigation/layout, and a completed mock assistant answer.

## 4. How to Run and Verify

### Prerequisites

- Node.js **24.21.0** and npm **11.19.0**, matching the pinned project toolchain.
- A running local PostgreSQL database with the project's migrations applied and demo data seeded.
- Docker Desktop with Compose if using the supplied PostgreSQL/Redis setup. An existing local database can also be used.
- Google Chrome for the configured Playwright browser tests.
- Available ports: web **5173**, API **3001**, and the database port you configure.

Run all commands from the repository root. Examples use PowerShell. No AI API key or market-data credential is needed for the mock demo. Example database credentials below are public local-development credentials.

### Install and Prepare Local Demo Data

The following is setup guidance for a student environment. Milestone 10 verification reused existing local databases rather than provisioning new infrastructure.

```powershell
node --version
npm --version
npm ci
npm run build:types
npm run infra:start
```

`npm ci` installs the versions in the lockfile. If all packages are already cached, the offline command used during implementation is an alternative:

```powershell
npm ci --ignore-scripts --offline
```

Configure the current terminal, apply migrations, and seed a disposable local demo database:

```powershell
$env:DATABASE_URL='postgresql://portfolio_local:local_only_change_me@127.0.0.1:5432/portfolio_pilot'
$env:REDIS_URL='redis://127.0.0.1:6379'
$env:DATA_MODE='mock'
$env:AGENT_MODE='mock'
$env:DEMO_AUTH_ENABLED='true'
$env:AUTH_BASE_URL='http://localhost:5173'
npm run migrate:deploy --workspace @portfolio-pilot/db
$env:NODE_ENV='development'
$env:ALLOW_DEMO_SEED='true'
npm run seed:demo --workspace @portfolio-pilot/db
Remove-Item Env:ALLOW_DEMO_SEED
npm run dev
```

A **migration** applies the database structure defined by the project. **Seeding** inserts predictable demonstration records. Seed only a local teaching database: the seed also updates existing named demo records. The API example configuration is in [apps/api/.env.example](apps/api/.env.example).

The Vite frontend forwards `/api` requests to Next.js locally. Open exactly **http://localhost:5173** when using the `AUTH_BASE_URL` above; the configured origin must match the address used to sign in.

### Verify the Interface Manually

1. Sign in as **Alice Demo** and open **Portfolios**.
2. Create a portfolio and record the three reference trades from the table above. Choose **ACME · XNAS · USD** and enter UTC dates.
3. Confirm nine shares, USD 961.80 remaining cost basis, and USD 135.80 realized gain. With historical seed quotes, expect stale status and unavailable market valuation.
4. Refresh and reselect your portfolio. Confirm its holdings and transaction history remain.
5. Rename it, then archive it. Confirm the history remains and transaction recording is disabled.
6. Open **Watchlist**, add a security, edit it, and try removal. Cancel once to check focus returns to the triggering button, then confirm removal.
7. Sign out and sign in as **Bob Demo**. Confirm Alice's portfolio and watchlist entries do not appear in Bob's account.

These steps describe expected manual behavior supported by the recorded automated checks. They do not claim an additional manual run was performed while writing this README.

### Run Build and Local Checks

```powershell
npm run typecheck
npm run build
npm run check:browser-boundary
npm run test
```

Without integration database variables, database-dependent suites are skipped. That is not equivalent to the recorded 57-test result.

### Run PostgreSQL Acceptance Tests

Use dedicated disposable databases named **portfolio_m08_verify** and **portfolio_m07_auth_verify** on a loopback host. Apply the existing migrations to both before testing. See [Lesson 08](docs/lessons/08-portfolio-and-transaction-apis.md) for the existing verification-container setup.

For example, when preparing fresh databases in the supplied Compose PostgreSQL service, run these commands once. This is setup guidance, not a new milestone 10 verification result:

```powershell
docker compose exec postgres psql -U portfolio_local -d postgres -c 'CREATE DATABASE portfolio_m08_verify'
docker compose exec postgres psql -U portfolio_local -d postgres -c 'CREATE DATABASE portfolio_m07_auth_verify'
$env:DATABASE_URL='postgresql://portfolio_local:local_only_change_me@127.0.0.1:5432/portfolio_m08_verify'
npm run migrate:deploy --workspace @portfolio-pilot/db
$env:DATABASE_URL='postgresql://portfolio_local:local_only_change_me@127.0.0.1:5432/portfolio_m07_auth_verify'
npm run migrate:deploy --workspace @portfolio-pilot/db
```

Set the test URLs to your database port. The actual successful implementation run used the existing verification server on **5546**:

```powershell
$env:PORTFOLIO_TEST_DATABASE_URL='postgresql://portfolio_local:local_only_change_me@127.0.0.1:5546/portfolio_m08_verify'
$env:AUTH_TEST_DATABASE_URL='postgresql://portfolio_local:local_only_change_me@127.0.0.1:5546/portfolio_m07_auth_verify'
npm run test
```

If you created these databases through the supplied Compose service, replace `5546` with `5432`. Do not point acceptance tests at personal/application data; tests insert or update demo fixtures.

### Run the Browser Scenario Against the Same Test Database

In terminal 1, stop any previous dev server and start the app against the dedicated portfolio database:

```powershell
$env:DATABASE_URL='postgresql://portfolio_local:local_only_change_me@127.0.0.1:5546/portfolio_m08_verify'
$env:DEMO_AUTH_ENABLED='true'
$env:AUTH_BASE_URL='http://127.0.0.1:5173'
$env:DATA_MODE='mock'
$env:AGENT_MODE='mock'
npm run dev
```

In terminal 2:

```powershell
$env:PORTFOLIO_E2E_DATABASE_URL='postgresql://portfolio_local:local_only_change_me@127.0.0.1:5546/portfolio_m08_verify'
npm run test:browser --workspace @portfolio-pilot/web
```

Adjust the database port consistently in both terminals if needed. The API and browser test must use the same database. The reference scenario seeds it, creates temporary test resources and a fresh synthetic quote, then removes its own temporary resources. Existing demo fixtures remain. If `PORTFOLIO_E2E_DATABASE_URL` is absent, that scenario is skipped; the three shell scenarios still need a running seeded API and demo authentication.

## 5. Limitations, Failures, and Unfinished Work

- **Resolved failures:** The first sandboxed typecheck failed with Prisma cache `EPERM` (permission denied). The authorized retry succeeded. Earlier browser runs failed for the test issues and dialog focus bug described above; the final four scenarios passed.
- **Remaining warnings:** Vite reports TanStack `use client` directive warnings, and Next.js reports three instrumentation Node/Edge runtime warnings. Builds passed despite these warnings.
- **Live data:** Only the persisted supported USD stock catalog is selectable. Provider search, live quotes, and ingestion belong to later milestones. News and the general assistant retain explicit demo behavior.
- **Authentication:** Demo authentication was verified. Real Microsoft Entra sign-in remains unverified without credentials. The browser expiry check simulated a 401; it was not a live Entra expiry test.
- **Retry scope:** A trade's retry key survives errors while its dialog stays open, not closing and reopening. Check history before re-entering a trade after an uncertain network result.
- **Pagination and rounding:** Concurrent or backdated trades can shift history pages. Rounded display rows and allocation weights may not add up exactly to rounded totals; accounting stays server-side.
- **Scope:** Long-only USD stocks are supported. Dividends, taxes, FX conversion, options, short selling, broker execution, and cash-flow-adjusted performance returns are outside this activity.
- **Environment:** No paid resources were provisioned and nothing was publicly deployed, committed, or pushed. This workspace was not a Git repository. No required local verification remained blocked at the end of milestone 10.

## Further Reading

- [Milestone 10 lesson notes](docs/lessons/10-portfolio-ui-and-watchlist.md)
- [Current project state and verification record](docs/project-state.md)
- [Project milestone plan](docs/project-plan.md)
- [Pinned versions](docs/versions.md)
- [Architecture decisions](docs/decisions/README.md)
- [Project rules](AGENTS.md)
