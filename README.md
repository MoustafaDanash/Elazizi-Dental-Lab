# Al-Azizi Dental Lab — Web System

> **Note for Claude Code sessions (any PC):** this README is the cross-machine context bridge.
> Read it fully before touching anything. Monthly-accounting context lives separately in
> Google Drive → "Dental Lab Accounting" → `ACCOUNTING_CONTEXT.md`.

Management system for Al-Azizi dental lab (Damanhour, Egypt): doctors, orders/cases,
payments & balances, expenses/income, salaries, scan scheduling, printable doctor bills.
Single-file app — all HTML/CSS/JS lives in one `index.html` (~1.2 MB), no build step.

## Live deployments (Cloudflare Workers)

| App | URL | Source |
|-----|-----|--------|
| Main app | https://elazizi-dental-lab.moustafadanash.workers.dev | `lab-deploy/index.html` |
| Catalogue | (catalogue worker) | `catalogue-deploy/` |

## Repo layout vs. working folders (IMPORTANT)

The git repo root is `D:\` and tracks only mirrors: `index.html`, `catalogue-index.html`, `manifest.json`.
The **actually deployed** files live in untracked folders:

- `D:\lab-deploy\index.html` — **edit THIS one**, deploy with `npx wrangler deploy` from `D:\lab-deploy`
- After deploying, mirror it into the repo: `Copy-Item D:\lab-deploy\index.html D:\index.html -Force`, then commit & push.
- Never edit `D:\index.html` via PowerShell `Get-Content`/`Set-Content` — it corrupts emoji/Arabic/em-dashes.
  Use byte-level copy (`Copy-Item`) or an editor that reads/writes UTF-8 bytes.

## Backend — Supabase

- Project: `tkatwjsqsrsexlpucclg` (URL and anon key are hardcoded in `index.html`, search `SB_URL`).
- Whole app state syncs as **one JSON blob**: table `app_state`, row id `alazizi-main`,
  column `data` is jsonb containing a JSON **string** — query with `(data #>> '{}')::jsonb`.
- Manual backup snapshots are extra `app_state` rows named `backup-YYYY-MM-DD-<reason>`.
  Create one before risky data surgery.
- Legacy per-entity tables (`orders`, `doctors`, `expenses`, …) exist but are mostly empty; the blob is authoritative.

### Sync architecture & the 2026-07 data-loss lesson

Blob sync is last-write-wins. In July 2026 that wiped team-entered expenses for July 1–11
(a device refreshed before its entries synced; `loadFromSupabase` used to overwrite localStorage).
**Fix (2026-07-25):** expenses now merge by id (`mergeExpensesFromCloud`) on load AND before every
push; deletions use tombstones (`alazizi_expenses_deleted` locally, `expensesDeleted` in the blob,
capped 500); same-id conflicts keep the newer `mtime`. `syncToSupabase` also blocks pushes from
devices holding drastically fewer orders/doctors/scans than the cloud.
⚠️ `advances`, `deductions`, `salaryHistory` in the blob are still overwrite-style — apply the same
merge pattern if they ever show missing entries.

## Business rules

- **Financial month runs the 5th → 4th** of the next month ("July" = 5 Jul–4 Aug).
  Code: `FIN_MONTH_START_DAY`, `finMonthRange(offset)`, `orderDateISO(o)` (uses `o.due || o.createdAt`).
  The doctor bill modal has a `bill-period-mode` selector: current/last financial month
  (previous months carried as line 1 "رصيد سابق — Previous balance due") or full statement.
- Divisions: 🦷 Porcelain (Haitham) and ⬜ Zirconia (Mostafa); expenses/income tagged per division.
- Doctor payments settle the doctor's TOTAL balance via allocations; excess becomes prepaid credit
  (`doctorFin()` returns invoiced/paid/due/prepaid — due already nets prepaid).
- Custom per-doctor pricing lives on the doctor record (`customPricing`/`positionPricing`/`caseBonus`).
- Folder/case rules from the offline workflow: "remake" = not billed (lab cost); "pfm" = PFM material.

## Development gotchas

- ES5-style code in places; app must run directly in browser — no transpiling, no modules.
- Verify inline JS after edits: extract `<script>` blocks and `new Function(src)` them in Node.
- UI verification rule: never claim a page renders from DOM checks alone — screenshot it
  (headless Edge + puppeteer-core); check div nesting if something "renders but is invisible".
- Deploy: `cd D:\lab-deploy && npx wrangler deploy` (worker name `elazizi-dental-lab`).
