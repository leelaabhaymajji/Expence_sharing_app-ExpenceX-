# ExpenseX Corporate Showcase — Viva / Presentation Guide

## 30-second explanation
ExpenseX Corporate is a browser-based expense management system for a company with three roles. Employees submit
expense claims with receipts, their manager approves or rejects them and later marks them reimbursed, and the Admin
manages users and settings but only sees company-level totals, never individual expenses. It is built with only
HTML, CSS and vanilla JavaScript and stores everything in the browser's localStorage — there is no backend.

## 1-minute explanation
Companies need a controlled path from "I spent money for work" to "I was paid back". ExpenseX Corporate models that
path as a status workflow: *draft → pending → reimbursement pending → reimbursed*, with *rejected → resubmitted*
as the correction loop. Every step is recorded in an append-only history. Managers can set monthly spending limits
that warn at 80 % and flag at 100 % without blocking claims. Receipts are compressed in the browser before saving.
The Admin creates managers and employees, assigns employees to managers, and sees privacy-safe aggregates — team
totals appear only for teams of three or more. Authentication (salted SHA-256 via the Web Crypto API) and role-based
access control are simulated on the client, which makes it a realistic prototype but not a secure production system.
A one-click demo company, generated through the real workflow code, shows every feature.

## Problem statement
Paper or spreadsheet expense claims get lost, have no approval trail, mix up who is responsible for approving and
paying, give no early warning about overspending, and often expose everyone's spending to administrators who only
need totals.

## Objectives
1. A clear claim workflow with approval, rejection with reasons, resubmission and reimbursement.
2. Role-based access: each role sees and does only what it should.
3. A complete, unalterable history of every claim.
4. Spending-limit warnings for employees and managers.
5. Receipt attachment without a file server.
6. Management analytics — and privacy for individuals at the Admin level.
7. Work entirely in the browser with no installation, server or database.

## Need for the system
Small teams and students need to demonstrate an approval workflow, RBAC and analytics without paying for or
maintaining servers. A zero-backend prototype lets the design be tested and shown anywhere, and clarifies the rules
before investing in real infrastructure.

## Technologies
- **HTML5** — one static page per screen; semantic landmarks, `<dialog>` for modals.
- **CSS3** — design tokens (custom properties), grid/flex, a container query for responsive tables,
  reduced-motion support. No CSS framework.
- **Vanilla JavaScript** — classic `<script>` files sharing global functions; no modules, bundler or npm.
- **Web Crypto API** — `crypto.subtle.digest('SHA-256')` for salted password hashes; `crypto.getRandomValues` for
  IDs and salts.
- **Canvas API** — resizing and JPEG-compressing receipts.
- **localStorage** — persistence; **sessionStorage** only for one-time flash messages.

## Why no backend
The brief was a pure front-end showcase: no server, API or database. That keeps the project portable (a folder of
files), free to run, and focused on workflow, RBAC and UI design. The trade-off — no real security and no shared
data — is stated openly in the app, the README and below.

## Architecture
Layered, all in the browser:

```
Pages (HTML)  →  js/pages/*.js       page controllers: read the form, call a domain function, redraw
                 js/ui.js, layout.js, charts.js, expense-view.js    rendering helpers
                 js/guard.js         page-level role check (redirect with "Access denied")
Domain        →  permissions.js      can(actor, action, target), canViewExpense, visibleExpenses
                 auth, users, projects, expenses, reimbursements, limits, receipts,
                 notifications, activity, analytics, settings, demo-data
Core          →  core/storage.js     the ONLY code that reads/writes localStorage (atomic commit + rollback)
                 core/money.js, dates.js, validate.js, constants.js
```

Each action follows the same pattern: `db = loadDb()` → `can()` check → validate → change `db` → add history,
notifications and activity → `commit(db, [collections])`. If the commit fails (e.g. storage full), every collection
is restored and nothing is half-saved.

## User roles
- **Employee** — own claims only; drafts, submit, resubmit rejected claims; own dashboard and notifications.
- **Manager** — own claims (auto-approved) plus submitted claims of employees *currently* in their team; approve,
  reject, reimburse; set limits; manage own projects; team analytics.
- **Admin** — one per company (first-run setup); creates managers/employees, assigns and reassigns, activates and
  deactivates, settings, activity log, aggregates, demo data. No individual expense access.

## RBAC (role-based access control)
Enforced at three levels:
1. **Page guard** — `requireUser([roles])` on every signed-in page; the wrong role is redirected with
   "Access denied".
2. **Domain functions** — every action (`approveExpense`, `assignEmployee`, `markReimbursed` …) calls `can()` itself,
   so a button that is hidden in the UI would still be refused if called from the console.
3. **Data scoping** — lists are built only from `visibleExpenses(user)` / `visibleReimbursements(user)`, never from
   the raw collections.

`can()` re-reads the actor from storage each time, so a deactivated user or a changed role takes effect even in a
tab that was already open.

## Employee workflow
Sign up (or be created) → wait for a manager (drafts only) → create an expense (title, amount, date, category,
project, description, receipt) → save draft or submit → see status, history and notifications → if rejected, read
the comment, edit the same claim and resubmit → see reimbursement details when paid.

## Manager workflow
Log in (pending-approval digest) → **Approvals**: approve, or reject with a required comment → **Reimbursements**:
mark approved claims paid (optional reference) → **Team**: set monthly limits, see who is near/over, add an employee
to own team → **Projects**: create, edit, deactivate → **Analytics**: filtered charts and approval/payout stats →
own expenses are auto-approved as *Self-approved*.

## Admin workflow
First-run setup → **Users**: create managers and employees, assign unassigned signups, reassign, (de)activate →
**Dashboard**: company aggregates and privacy-safe team totals → **Activity** log → **Settings**: company name,
warning %, public signup on/off, storage usage, load/reset demo.

## Reassignment
`assignEmployee()` changes the employee's `managerId`; their **pending** claims are moved to the new manager's queue
and all three people are notified. Nothing historical is rewritten: approved/rejected claims keep the manager who
decided them, and history entries stay as they were. Because visibility is "employees *currently* in my team", the
new manager sees the employee's submitted history and the old manager no longer does.

## Reimbursement
Approval creates a reimbursement record (amount, approver = `managerId`, queued time) and sets the claim to
*Reimbursement Pending*. The payer is the employee's **current** manager (`reimbursementManagerId()`), or the
manager for their own claim. Marking paid records `reimbursedById`, `reimbursedAt` and an optional reference,
separately from the approver, so the detail page shows both *Approved by* and *Reimbursed by*. A claim cannot be
paid twice.

## Spending limits
A manager may set a monthly limit (₹) per team employee. Used = sum of that employee's pending, approved,
reimbursement-pending and reimbursed claims whose *expense date* falls in the month (drafts and rejected claims
don't count). ≥ warning % (default 80, Admin-configurable 50–99) → *Near limit*; ≥ 100 % → *Over limit*. Limits
never block submission — managers decide. The employee and manager are notified once per level per month (a
per-month key prevents repeats). The expense form previews the effect before submitting.

## Receipts
`prepareReceipt()` accepts only image files ≤ 5 MB, draws them onto a canvas scaled to at most 1280 px on the long
side, and exports JPEG at quality 0.72. The data URL is stored under its own ID in `exc:receipts`; the expense keeps
only the `receiptId`. Replacing or removing a receipt deletes the old image, so nothing is orphaned.

## localStorage architecture
Eleven keys prefixed `exc:` (schema, users, credentials, session, projects, expenses, reimbursements, receipts,
notifications, activity, settings). `storage.js` provides `loadDb()`, `commit(db, names)` (writes the named
collections; on any failure restores the previous values), `snapshotAppData()` / `restoreAppData()` (used by demo
load), `clearAppData()` (removes only `exc:` keys), and ID generation. No other file touches localStorage.

## Authentication
Signup/setup/admin-created accounts store a random 16-byte salt and `SHA-256(salt + ":" + password)` in
`exc:credentials`, separately from user profiles. Login re-hashes and compares, rejects deactivated accounts, and
writes the user ID to `exc:session`. Every page load re-reads the session and the user, so logout or deactivation
in another tab takes effect immediately.

## Password hashing
Salted so identical passwords produce different hashes and precomputed tables don't work. Web Crypto is native and
needs no library. Limitation: one fast SHA-256 round is not a password-hashing function; a real system would hash
on the server with Argon2/bcrypt/PBKDF2.

## Validation
`validate.js` + domain validators, run in the domain function (not just the form): required fields, trimmed text
lengths, email format and uniqueness, password rules, dates (valid, not in the future), categories from a fixed
list, project must be an active project of the right manager, amounts via `parseMoney` (see Q&A), limits,
rejection comment required. Errors appear next to each field, focus moves to the first invalid field, and all
user text is HTML-escaped before display (XSS prevention).

## Notifications
Stored per `recipientId` (max 200 per user, oldest dropped). Created by the domain functions for submit/resubmit,
approve, reject (with comment), reimburse, expense created for you, limit set/warning/exceeded, manager assigned,
team joined/left, new signup (Admin) and a pending digest refreshed at manager login. Badges show unread counts;
users can mark one or all read, and each links to the relevant page.

## Analytics
Pure functions in `analytics.js` over the user's *visible* data: monthly totals, category, project, status, approval
rate, reimbursement pending vs paid and average days to payout, per-employee spend. Charts are plain HTML/CSS bars
(`charts.js`), no charting library. The Admin uses `companyAggregates()`, which returns numbers only.

## Admin privacy
- Admin gets no expense records anywhere (permission functions return nothing; expense pages exclude the role).
- Dashboard: company totals and distributions only.
- Team totals only for teams with ≥ 3 employees, plus **complementary suppression**: if hiding a small team would
  let the Admin compute it as "company total − shown teams", another team is hidden too.
- Activity entries for expense actions contain only the expense ID.

## Demo data
`loadDemoCompany()` creates *Sync Systems Pvt Ltd* by calling the real functions (`setupCompany`,
`createManagedAccount`, `saveEmployeeExpense`, `approveExpense`, `assignEmployee` …) with a backdated clock, so the
six months of data obey every rule and produce real histories and notifications. Load requires typing `REPLACE`,
reset requires `RESET`; both are Admin-only (or the login page when the browser has no data), log everyone out, and
only touch `exc:` keys. A failed load restores the previous data exactly.

## Responsive design and accessibility
Mobile-first CSS: sidebar at ≥ 1024 px, bottom navigation plus a *More* sheet below. Tables become labelled cards
when their container is < 720 px (container query, so a narrow card on a wide screen also adapts). Verified at 1440,
1280, 1024, 768, 390 and 360 px with no horizontal scrolling. Accessibility: landmarks, one `h1` per page, labels
and error descriptions, native `<dialog>` with Escape and focus return, visible focus, table captions and scoped
headers, live-region toasts, descriptive badge text, contrast ≥ 6 : 1, reduced-motion rule.

## Limitations
Client-side only, so not secure; data tied to one browser profile; ~5 MB quota limits receipts; one session per
browser; INR and fixed categories only; no email, password reset, exports or real payments.

## Future scope
Real backend (REST API + relational DB) with server-side auth and RBAC; file/object storage for receipts; email
notifications; password reset and SSO; multi-currency with exchange rates; configurable categories and approval
chains (e.g. finance approval above an amount); CSV/PDF export and accounting integration; payroll/bank payout
integration; OCR to read receipts; audit-log export; PWA offline support.

---

## Questions and answers

**Why vanilla JavaScript?**
The brief required HTML/CSS/JS only. It also shows the fundamentals — DOM, events, state, rendering — without a
framework hiding them, needs no build step, and runs from any static host or folder.

**Why localStorage?**
It is the only persistent storage every browser has without a server, it is synchronous and simple, and it fires a
`storage` event for cross-tab updates. It is enough for a single-browser prototype.

**Why no backend?**
Project constraint: a front-end showcase with no server, API or database. It keeps the focus on workflow and UI
and makes the project free and portable. The costs (security, sharing) are documented.

**Is it production-secure?**
No. All code and data are in the user's browser; anyone with DevTools can edit localStorage, change their role, or
call functions directly. The checks show *how* RBAC should work; real enforcement must happen on a server.

**How is authentication implemented?**
Salted SHA-256 hashes in `exc:credentials` (separate from users). Login re-hashes the entered password and compares;
deactivated users are refused; the session is the user ID in `exc:session`, checked on every page load and by
`guard.js`.

**How is RBAC enforced?**
Three layers: the page guard (`requireUser(roles)`), `can(actor, action, target)` inside every domain function, and
scoped data (`visibleExpenses`). `can()` re-reads the stored user, so stale or deactivated sessions are refused.

**How does reassignment work?**
The Admin calls `assignEmployee()`: `managerId` changes, pending claims move to the new manager's queue, the
employee and both managers are notified and the action is logged. Completed decisions and history are untouched.

**Why preserve historical approvers?**
It is an audit trail: who approved what must never change afterwards, otherwise accountability is lost. Current
responsibility (who pays now) is modelled separately from historical fact (who approved).

**How are receipts stored?**
Compressed to JPEG (≤ 1280 px, quality 0.72) with the Canvas API, stored as a data URL in `exc:receipts` keyed by
receipt ID; the expense stores only the ID. Images only, ≤ 5 MB input.

**Why integer paise?**
Binary floating point cannot represent values like 0.1 exactly (`0.1 + 0.2 !== 0.3`). Storing whole paise keeps
all sums exact. `parseMoney` parses the text as a string (rupees + up to 2 decimals) instead of `parseFloat`, and
rejects negatives, zero, `NaN`, `Infinity`, exponents (`1e5`), more than two decimals and amounts over the cap.

**How are limits calculated?**
Sum of the employee's pending, approved, reimbursement-pending and reimbursed claims whose expense date is in the
month, compared with the monthly limit: ≥ warning % → near, ≥ 100 % → over. Warn only; one notification per level
per month.

**What happens when a manager is deactivated?**
It is refused while any employee (active or inactive) is still assigned — the Admin must reassign them first, so no
claim is left without an approver or payer. Once deactivated, the manager can't log in, open sessions are logged
out, and all their past decisions remain in history.

**How is the Admin prevented from seeing individual expenses?**
`canViewExpense`, `visibleExpenses` and `visibleReimbursements` return nothing for the Admin; the expense pages
exclude the Admin role; admin pages use only `companyAggregates()` (numbers) and `privacySafeTeams()` (≥ 3
employees + complementary suppression); activity entries carry only expense IDs.

**How does cross-tab sync work?**
Browsers fire a `storage` event in *other* tabs when localStorage changes. `startPage()` listens for `exc:` keys,
re-checks the session (redirecting if the user logged out, changed or was deactivated) and redraws the page with
fresh data.

**What are localStorage's limitations?**
About 5 MB per origin; strings only (JSON serialisation); synchronous (large writes block the page); per browser
profile and device; wiped by clearing site data; not available across users; readable and writable by any script on
the page or anyone with DevTools; no queries, indexes or transactions (the app simulates atomic saves itself).

**What would change with a real backend?**
Storage moves to a database (users, expenses, history, reimbursements tables with foreign keys); receipts to object
storage; passwords to server-side Argon2/bcrypt with HTTP-only session cookies or tokens; every `can()` check runs
on the server for each API request; notifications could be emailed or pushed; many users could work concurrently
from different devices. The domain rules (workflow, reassignment, payer, privacy) would stay the same — only where
they run would change.
