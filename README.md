# BloodLink: Blood Donor & Emergency Request Network

[![tests](../../actions/workflows/ci.yml/badge.svg)](../../actions)
![license](https://img.shields.io/badge/license-MIT-blue)
![python](https://img.shields.io/badge/python-3.12-blue)
![db](https://img.shields.io/badge/SQLite%20%7C%20PostgreSQL%2016-tested-green)

**BloodLink connects blood donors, patients, and hospitals, with the hospital in charge of every decision that matters.**

BloodLink is a Flask web application. A patient posts a blood request, the named hospital reviews it, compatible volunteer donors are shown the request and can offer to help, and the hospital coordinates the donors and **confirms each donation**. A request for several units is tracked until every unit is confirmed.

![BloodLink chat between a donor, a patient's family and the hospital](docs/screenshots/chat-thread-patient.png)

<p>
<img src="docs/screenshots/hospital-donor-coordination.png" width="49%" alt="Hospital coordinates donors and confirms units">
<img src="docs/screenshots/chat-thread-mobile.png" width="22%" alt="Chat on a phone">
</p>

<p>
<img src="docs/screenshots/donor-dashboard.png" width="49%" alt="Donor dashboard with availability, badges and next eligible date">
<img src="docs/screenshots/qr-checkin-donor.png" width="49%" alt="Donor's QR check-in code">
<img src="docs/screenshots/admin-analytics.png" width="49%" alt="Admin analytics: demand and supply by blood group">
<img src="docs/screenshots/public-stock.png" width="49%" alt="Public blood stock page">
</p>

> **Disclaimer:** BloodLink is a university project. It is **not** a medical service. It never decides who may donate or receive blood, and it makes no clinical claims. Hospitals decide eligibility, testing, and crossmatching.

---

## Table of Contents

1. [Team](#team)
2. [The Problem](#1-the-problem)
3. [Multi-Donor Allocation](#2-multi-donor-allocation)
4. [Donor Matching](#3-donor-matching-rule-based-not-machine-learning)
5. [Roles and Features](#4-roles-and-features)
6. [Advanced Features](#5-advanced-features)
7. [Configuration](#6-configuration)
8. [Privacy, Consent, and Policies](#7-privacy-consent-and-policies)
9. [Technology and Project Structure](#8-technology-and-project-structure)
10. [Installation and Testing](#9-installation-and-testing)
11. [Deployment](#10-deployment)
12. [Demo Accounts](#11-demo-accounts)
13. [Known Limitations](#12-known-limitations)
14. [Future Work](#13-future-work)
15. [Contributing and License](#14-contributing-and-license)

---

## Team

| Name | GitHub | Role |
|---|---|---|
| Ali Memon | [@alimemonnn](https://github.com/alimemonnn) | Developer |

> Add your other team members to this table.

---

## 1. The Problem

*A patient needs three units of blood, and three compatible donors are available. How does the system coordinate them, know how many units have been provided, and know when the request is fulfilled?*

**Answer:** Totals are derived from hospital-verified donation records, never from donor claims.

```
required = 3
Donor A offers 1          -> pledged 1, confirmed 0    status Open
Hospital confirms A       -> confirmed 1, remaining 2  status Partially Fulfilled (33%)
Hospital confirms B       -> confirmed 2, remaining 1  status Partially Fulfilled (66%)
Hospital confirms C       -> confirmed 3, remaining 0  status Fulfilled           (100%)
```

`remaining = max(0, required - confirmed)` and `progress = min(100, confirmed / required * 100)`. Zero, negative, or missing `required` values are handled safely.

---

## 2. Multi-Donor Allocation

Implemented in `utils/allocation.py`.

| Concept | Where it lives |
|---|---|
| Units **required** | `blood_requests.units_required` (CHECK ≥ 1) |
| One donor's offer or contribution | One row in `donation_responses` per donor per request |
| Units **pledged** (not yet verified) | Sum of `units_pledged` for *active* responses (Interested, Contacted, Appointment Scheduled) |
| Units **confirmed** | Sum of `units_collected` for responses with status *Donation Completed*. **Only a hospital can set this.** |
| Units **remaining** and **progress %** | Computed properties on `BloodRequest`. There is no stored counter that can drift. |

### Rules enforced on the server

Browser checks are a convenience only. The server enforces every rule below.

1. Only the hospital that owns a request can confirm a donation (object-level authorization).
2. A donor's "I can donate" is *willingness only*. It never counts as blood and is not medical approval.
3. A confirmation can never push the total above `required`. By default, one unit per donor per event (`MAX_UNITS_PER_DONOR_DONATION`).
4. A response can be confirmed once. A retried or double-clicked submit is rejected, not double-counted.
5. The request row is **locked** during confirmation (PostgreSQL `SELECT … FOR UPDATE`; on SQLite, a no-op write takes the single write lock), so two hospital users cannot both claim the last unit. A two-thread test verifies this.
6. Each operation runs in one transaction. Any failure rolls back, and this is also tested.
7. On fulfillment, leftover offers are cancelled and those donors are notified. On cancellation or expiry, outstanding offers are cancelled, but verified donations stay on record.
8. `flask check-consistency` verifies that request statuses agree with the donation records.

### State machines

**Request states:** Pending Hospital Approval → Open (stored as `Approved`) → Partially Fulfilled → Fulfilled. A request can also end as Rejected, Cancelled, or Expired.

**Donor-response states:** Interested → Contacted → Appointment Scheduled → Donation Completed (hospital only). A response can also end as Cancelled or Ineligible.

```mermaid
flowchart LR
  P[Patient posts request] --> H{Hospital reviews}
  H -- reject --> R[Rejected]
  H -- approve --> O[Open: donors notified]
  O --> D[Donors offer to help<br/>willingness only]
  D --> C[Hospital contacts / schedules / screens]
  C -- ineligible --> I[Ineligible, optional deferral]
  C -- donation verified --> V[Units confirmed +1]
  V --> Q{confirmed >= required?}
  Q -- no --> PF[Partially Fulfilled]
  PF --> D
  Q -- yes --> F[Fulfilled<br/>leftover offers cancelled]
  O --> X[Cancelled or Expired<br/>offers cancelled, history kept]
```

---

## 3. Donor Matching (Rule-Based, Not Machine Learning)

Implemented in `utils/matching.py`. Hard filters run first, followed by an explainable score that only *orders* the donors who pass.

- **Hard filters (never overridden):** ABO/Rh red-cell compatibility, active account, marked as available, a configurable rest interval since the last donation, not deferred by a hospital, and has not already responded.
- **Score (weights in `config.MATCH_WEIGHTS`):** Same city, exact blood group, and no donation on record. Each match includes human-readable reasons shown to the hospital. Without GPS data, proximity is city-level.
- Compatibility applies to **red cells and whole blood**. Plasma and platelets differ and are not modeled.
- The output is a *shortlist for professional verification*. The hospital screens, tests, and crossmatches.

| Component | Type |
|---|---|
| Compatibility table, filters, ranking score | Rule-based automation (transparent and tested) |
| Dashboard counts and charts | Analytics (plain SQL aggregates of real records) |
| Machine learning | **None.** No model is claimed. Demand forecasting would need real historical data, a baseline, and evaluation. |

---

## 4. Roles and Features

| Role | Capabilities |
|---|---|
| **Patient / Family** | Register (18+), post a request (blood group, units, hospital, urgency, deadline), track approval status, view per-request progress (required, pledged, confirmed, remaining), see donor offers and updates. Cancel with history kept. Export or delete their data. |
| **Donor** | Register with explicit consent, set availability, see compatible open requests (no patient details), offer to help or cancel an offer, view history and verified donations. Choose whether contact details are shared and whether to receive email. Export or delete their data. |
| **Hospital** | Dashboard. Approve or reject requests. Contact, schedule, or screen donors. Mark donors ineligible (with optional deferral). **Confirm donations.** Close a request with a reason. See who verified what and when. |
| **Admin** | Verify hospitals. Manage accounts (erasure keeps counts anonymous). Database-driven metrics, charts, and activity log. **Privacy inbox** for data requests and contact messages. |

### Chat (Donor, Patient, and Hospital)

Once a donor offers to help, a private conversation opens for that one offer. The donor, the patient's family, and the hospital can write in it. Nobody else can read it. An administrator can read it only after someone **reports** it, and can then lock the thread.

Chat is for coordination only. It never changes a request or counts blood, and messages never appear in notifications. Delivery uses short polling (about every 4 seconds), which needs no WebSocket server and works on any host.

- **Live timeline:** Hospital and donor actions (offer made, contacted, appointment scheduled, donation confirmed, cancelled, request closed) appear in the thread automatically. Screening notes are never copied into chat.
- **Donation plan card:** Hospital name and address, appointment time (updates live), a *Directions* link, and an *Add to calendar* (.ics) button.
- **Quick replies:** One tap for messages such as "I'm on my way", "I've arrived", "Running late", and "Thank you".
- **Share my location (donor and patient only):** Asks for confirmation and sends **one** position. It never tracks. The hospital can see it; administrators never see coordinates. It is hidden after 6 hours and erased after 24 hours (configurable). It opens in Google Maps or OpenStreetMap. No map tiles or third-party scripts are loaded.
- **Urdu and Sindhi:** Messages display right-to-left automatically.
- **Privacy:** Patients appear to donors by initials only. The thread becomes read-only when the offer is cancelled, the request closes, or the hospital confirms the donation. Messages are rate limited, HTML-escaped, included in data export, erased with the account, and deleted 90 days after the request closes.

**Using VS Code?** Follow [docs/VSCODE.md](docs/VSCODE.md): open the folder, select the `venv` interpreter, and press **F5**.

---

## 5. Advanced Features

| Area | What it provides |
|---|---|
| **Matching** | Distance-aware ranking (built-in city centers, or an exact hospital position and donor area rounded to about 1 km). An "available now" boost and a small penalty for recorded no-shows. Every match shows its reasons. |
| **Emergencies** | Urgent and Critical requests that nobody answers are **escalated automatically**. After 30 minutes, donors within 50 km receive SMS, WhatsApp, push, or email (if they opted in). After 2 hours, every compatible donor is contacted. The hospital and patient are notified. |
| **Trust** | **QR check-in** at the hospital (arrival only). No-show record. **Donation certificate PDF** with a verification code and a public verification page (`/verify`). Only hospital-confirmed donations count. |
| **Reaching people** | SMS and WhatsApp (Twilio), email, an **installable app (PWA) with push notifications**, and appointment reminders 24 hours and 2 hours before. All are opt-in and off until configured. |
| **Accounts** | Forgot-password and email-confirmation links. **Two-factor login** (authenticator app and recovery codes). Optional mandatory 2FA by role via `REQUIRE_2FA_ROLES`. |
| **AI features** | *Paste a message to fill the request form* (rule-based, works offline). Automatic money and scam flagging in chat (rule-based, works offline). An optional **Translate** button (English, Urdu, Sindhi) that uses the Claude API (requires `ANTHROPIC_API_KEY`). |
| **Hospitals** | Blood stock page (public at `/stock`). Donation camps with sign-ups (`/camps`). Analytics and CSV export. |
| **Donors** | Badges, donation streak, next-eligible countdown, camp sign-ups, and certificates. |
| **Interface** | English, **Urdu, and Sindhi** (right-to-left). Navigation, sign-in, chat, and quick replies are translated. Other content falls back to English. *The Urdu and Sindhi wording is a draft and should be reviewed by a native speaker.* |
| **Platform** | Public read-only **REST API** with OpenAPI documentation (`/api/v1/docs`), Alembic migrations, optional Redis rate limiting, JSON logs with request IDs, and optional Sentry error tracking. See [docs/ARCHITECTURE.md](docs/ARCHITECTURE.md). |

---

## 6. Configuration

Only `SECRET_KEY` and `DATABASE_URL` are required. Everything else is optional.

| Variable | Enables |
|---|---|
| `NOTIFY_PROVIDER=twilio` with `TWILIO_ACCOUNT_SID`, `TWILIO_AUTH_TOKEN`, `TWILIO_SMS_FROM`, `TWILIO_WHATSAPP_FROM` | SMS and WhatsApp (`log` only prints messages, for testing) |
| `VAPID_PUBLIC_KEY`, `VAPID_PRIVATE_KEY`, `VAPID_SUBJECT` | Push notifications. Generate keys with `python scripts/gen_vapid_keys.py`. |
| `ANTHROPIC_API_KEY` (with `AI_MODEL`, `AI_DAILY_LIMIT`) | The Translate button |
| `SMTP_HOST`, `SMTP_PORT`, `SMTP_USER`, `SMTP_PASSWORD`, `SMTP_FROM` (and `PUBLIC_BASE_URL`) | Email for alerts, password reset, and confirmation. Put these in a `.env` file. See [docs/EMAIL_SETUP.md](docs/EMAIL_SETUP.md). Without SMTP, links are printed in the terminal. |
| `REDIS_URL` | Shared rate limits across workers |
| `SENTRY_DSN` | Error tracking (personal data off) |
| `LOG_FORMAT=json` | One JSON log line per event |
| `REQUIRE_2FA_ROLES=admin,hospital` | Mandatory two-factor login for those roles |
| `USE_ALEMBIC=1` | Schema managed by `flask db upgrade` |

---

## 7. Privacy, Consent, and Policies

Policy pages: `/privacy`, `/terms`, `/refund-policy`, `/cookies`, `/accessibility`, `/licenses`, `/data-request`.

See **[docs/COMPLIANCE_CHECKLIST.md](docs/COMPLIANCE_CHECKLIST.md)** for the full 20-point mapping and the tasks that remain for the project owner (business email, legal review).

Highlights:
- Cookie notice. Only two first-party necessary cookies are used. No trackers.
- Unticked consent boxes, recorded with the policy version.
- 18+ confirmation, and guardian confirmation for requests involving minors.
- Public pages show patient initials only. Donor phone and email are shown only with the donor's consent.
- Self-service **data export and deletion**, a public deletion request form, and a retention job.
- No third-party scripts or fonts. Content Security Policy (CSP) is enforced.
- Third-party license texts are included.

---

## 8. Technology and Project Structure

**Stack:** Python 3.12 · Flask 3 · Flask-SQLAlchemy / SQLAlchemy 2 · SQLite (development) / PostgreSQL (production) · Jinja2 · Bootstrap 5 · Chart.js · Font Awesome Free. All assets are self-hosted.

```
app.py            Application factory, CSP/CSRF, safe additive migration, CLI commands
config.py         Environment-driven settings; TestingConfig
models/           Donor, Patient, Hospital, Admin, BloodRequest, DonationResponse,
                  Notification, compliance tables
routes/           public, auth, donor, patient, hospital, admin, legal, account
utils/            allocation (core), matching, notifications, mailer, privacy,
                  maintenance, validators, auth, ratelimit
templates/        UI templates
static/           Assets; static/vendor/licenses holds third-party license texts
tests/            pytest suite (fake data, temporary databases)
docs/             Audit report, compliance checklist, presentation guide
```

### Database

Schema changes are additive. New columns were added to `donors`, `patients`, `hospitals`, `blood_requests`, `donation_responses`, and `notifications`. New tables: `consent_records`, `contact_messages`, `data_requests`, `account_notifications`, and `audit_logs`. Each donor has one response row per request. A donor who cancels re-opens the *same* row.

---

## 9. Installation and Testing

```bash
python -m venv venv
venv\Scripts\activate                # Windows (macOS/Linux: source venv/bin/activate)
pip install -r requirements-dev.txt
python seed.py                       # Load fake demo data into an EMPTY database
python app.py                        # Open http://127.0.0.1:5000
python -m pytest                     # Run automated tests (SQLite)
TEST_DATABASE_URL=postgresql://user:pass@host/db python -m pytest   # Same suite on PostgreSQL
python test_workflows.py && python test_new_features.py             # Original scripts
flask --app app check-consistency    # Verify totals agree with donation records
flask --app app retention-cleanup    # Run daily in production
```

`seed.py` refuses to touch a database that already contains data. Use `python seed.py --reset` and type `YES` to wipe a *local* database. It refuses to run in production.

### Background jobs

```bash
flask --app app jobs                  # Run all jobs once; schedule every 5 minutes (render.yaml does this)
flask --app app jobs --only deliver   # Run one job: expire, reminders, escalate, deliver, or cleanup
```

### Database migrations

```bash
flask --app app db upgrade            # New database
flask --app app db stamp head         # Database created by an older version: adopt it once, then use upgrade
flask --app app db migrate -m "..."   # After changing a model
```

Notifications are stored in the database, deduplicated by key, and never contain patient names or medical details. Optional email goes only to donors who opted in, and every email includes a one-click unsubscribe link.

### Upgrading an existing database

1. **Back up first.** Copy `database.db` (SQLite) or take a snapshot in Neon or Supabase.
2. Replace the code and start the app. `db.create_all()` creates new tables. `_migrate_schema()` only **adds** columns (nullable or with defaults) and backfills them. It never drops or rewrites a table.
3. On SQLite, a `database.db.pre-migration-<timestamp>.bak` file is written before any change.
4. Backfills: completed donations get `units_collected = 1`. Older "Open" requests that already had verified donations become *Partially Fulfilled*. Requests closed by the old one-click flow are flagged `closed_reason='legacy'`, since their unit records never existed, so they are flagged rather than rewritten.
5. Run `flask --app app check-consistency`. Re-running the migration is harmless.

### Environment variables

See `.env.example`. Required in production: `SECRET_KEY`, `DATABASE_URL`, `ADMIN_EMAIL`, `ADMIN_PASSWORD`. Recommended: `BUSINESS_EMAIL`, `BUSINESS_PHONE`, `BUSINESS_REGISTRATION`, `PUBLIC_BASE_URL`.

---

## 10. Deployment

Deployed on **Render with Neon PostgreSQL**.

- Set `DATABASE_URL`, `SECRET_KEY`, `ADMIN_EMAIL`, `ADMIN_PASSWORD`, `BUSINESS_EMAIL`, and `PUBLIC_BASE_URL`.
- `render.yaml` uses `/healthz` as the health check.
- Production mode (`RENDER=true` or `BLOODLINK_ENV=production`) enables secure cookies, HSTS, and proxy headers, and refuses to start without a secret key.
- Run `flask --app app retention-cleanup` on a daily schedule.
- Free hosting services sleep when idle. Open the site a few minutes before a demo.
- For demo data on a live database, run `python seed.py` against an **empty** database **before** setting `ADMIN_EMAIL`.

---

## 11. Demo Accounts

*Local fake data only.*

| Role | Email | Password |
|---|---|---|
| Admin | admin@bloodlink.com | Admin123 |
| Hospital | civil@sukkurhospital.com | Hospital123 |
| Donor | ahmed.ali@example.com | Donor123 |
| Patient | rizwan@example.com | Patient123 |

**Change or delete these accounts before any real use.** The seed data includes a 3-unit O+ request with 2 units already verified, so you can confirm the final unit live.

---

## 12. Known Limitations

- **Not a medical device.** Eligibility rules (such as the 90-day interval) are configurable placeholders that a blood bank should confirm.
- **External services are untested against live accounts.** SMS and WhatsApp (Twilio), web push, the Claude translation call, Redis, and Sentry are implemented, covered by tests using fakes, and disabled by default. They have not been run against the live services. Test each one with your own keys before relying on it.
- **Location.** Proximity uses built-in city centers, or an exact hospital position with a donor area rounded to about 1 km. There is no routing or live traffic data. "Directions" and shared locations open Google Maps or OpenStreetMap.
- **Urdu and Sindhi** texts are a first draft covering navigation, sign-in, chat, and quick replies. They have not been reviewed by a native speaker. Right-to-left layout has not been checked on a device with Arabic-script fonts.
- **QR check-in** uses the browser's built-in scanner (Chrome, Edge, and Android). On other browsers, type the short code.
- **Blood stock** is self-reported by hospitals and may be out of date. The pages say so.
- **Policies** are drafted for this project and are **not legal advice**. They should be reviewed before real use.
- **Chat** uses polling, not WebSockets. Messages can take up to about 4 seconds to appear.
- **Shared location and push notifications** require HTTPS (Render provides it) and the user's browser permission.
- **Accessibility.** The main pages for every role pass automated axe-core checks (WCAG 2.1 A/AA). This is not a substitute for testing with real assistive-technology users.
- **Schema constraints.** New CHECK constraints exist only in freshly created databases, because SQLite cannot add them to an existing table. The same rules are enforced in application code.

---

## 13. Future Work

- Live-provider testing of all optional services
- Native-speaker review and full interface translation
- WebSocket or Server-Sent Events chat
- Hospital staff accounts with roles
- An official donor-eligibility questionnaire reviewed by a blood bank
- Volunteer driver coordination (no payments)
- Mobile app wrapper

---

## 14. Contributing and License

BloodLink is licensed under the [MIT License](LICENSE).

Contributions are welcome. Please read [CONTRIBUTING.md](CONTRIBUTING.md) and [SECURITY.md](SECURITY.md) first.
