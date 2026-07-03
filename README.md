# Clinic Care — Healthcare Appointment & Follow-up Manager

A full-stack appointment platform with separate portals for **patients**, **doctors**, and an
**admin**. Patients book appointments and describe symptoms in advance; doctors get an AI-generated
pre-visit summary and produce a patient-friendly post-visit summary; both sides get email + Google
Calendar notifications.

**Stack:** React (frontend) · Node.js/Express (backend API) · MongoDB (database) · Anthropic Claude
(LLM summaries) · Nodemailer (email) · Google Calendar API (OAuth 2.0)

---

## 1. Project structure

```
healthcare-app/
├── backend/            Express API, MongoDB models, LLM/email/calendar services, cron jobs
│   ├── config/db.js
│   ├── models/         User, DoctorProfile, Appointment, Notification
│   ├── middleware/auth.js
│   ├── routes/         auth, admin, patient, doctor, appointment, googleAuth
│   ├── services/       llmService, emailService, calendarService, slotService
│   ├── jobs/           reminderJob, emailRetryJob (node-cron)
│   ├── seed/seed.js     creates a demo admin + doctor
│   └── server.js
├── frontend/            React app (patient / doctor / admin portals)
│   └── src/
│       ├── pages/patient  |  pages/doctor  |  pages/admin
│       ├── context/AuthContext.js
│       └── api/client.js
├── docs/
│   └── SYSTEM_DESIGN.md
└── README.md            (this file)
```

---

## 2. Prerequisites

- Node.js 18+
- MongoDB (local install, or a free Atlas cluster — [atlas.mongodb.com](https://atlas.mongodb.com))
- (Optional, for live mode) an Anthropic API key, SMTP/SendGrid credentials, and a Google Cloud
  OAuth 2.0 client. **None of these are required to run and demo the app** — see §4, "Mock mode."

---

## 3. Setup

### 3.1 Backend

```bash
cd backend
cp .env.example .env      # then edit .env — at minimum set MONGO_URI and JWT_SECRET
npm install
npm run seed               # creates a demo admin + demo doctor account
npm run dev                 # starts on http://localhost:5000
```

Seeded accounts:
| Role   | Email                        | Password    |
|--------|------------------------------|-------------|
| Admin  | admin@clinic.example.com     | Admin@123   |
| Doctor | dr.smith@clinic.example.com  | Doctor@123  |

### 3.2 Frontend

```bash
cd frontend
cp .env.example .env       # REACT_APP_API_URL=http://localhost:5000/api
npm install
npm start                   # starts on http://localhost:3000
```

### 3.3 Try it out

1. Log in as **admin** → create a doctor (or use the seeded one) → optionally mark a leave day.
2. Register as a **patient** → search doctors → pick a slot → fill the symptom form → confirm.
3. Log in as the **doctor** → open the appointment → read the AI pre-visit summary → after the
   visit, submit clinical notes + prescription → the patient gets an AI-generated summary and
   scheduled medication reminders.

---

## 4. Mock mode vs. live mode (LLM / Email / Google Calendar)

This sandbox/demo environment can't reach `api.anthropic.com`, SMTP servers, or Google's OAuth
endpoints directly, so **all three integrations ship with a working mock mode that's on by
default**, driven entirely by `.env` flags. The application logic, API contracts, and UI behave
identically in both modes — only the "does a real network call happen" part changes.

| Integration      | `.env` flag                | Mock behaviour                                              | Live behaviour |
|-------------------|----------------------------|---------------------------------------------------------------|----------------|
| LLM summaries     | `LLM_MODE=mock` (default)  | Deterministic canned pre/post-visit summaries                 | Calls Claude via `@anthropic-ai/sdk` |
| Email              | `EMAIL_MODE=mock` (default)| Logs the email to the console + saves it in the `Notification` collection | Sends via Nodemailer (SMTP or SendGrid) |
| Google Calendar   | `GOOGLE_CALENDAR_MODE=mock`(default) | Returns a fake event id/link, stored on the appointment | Creates/updates/deletes real events via `googleapis` |

To go live, set the relevant `*_MODE=live` and fill in the corresponding credentials in `.env`:

```bash
# LLM
LLM_MODE=live
ANTHROPIC_API_KEY=sk-ant-...

# Email (SMTP example)
EMAIL_MODE=live
EMAIL_PROVIDER=smtp
SMTP_HOST=smtp.gmail.com
SMTP_PORT=587
SMTP_USER=you@gmail.com
SMTP_PASS=your_app_password
EMAIL_FROM="Clinic Care <no-reply@yourclinic.com>"

# Google Calendar
GOOGLE_CALENDAR_MODE=live
GOOGLE_CLIENT_ID=...
GOOGLE_CLIENT_SECRET=...
GOOGLE_REDIRECT_URI=http://localhost:5000/api/auth/google/callback
```

All three services (`services/llmService.js`, `services/emailService.js`,
`services/calendarService.js`) catch their own errors and fall back gracefully — an LLM outage or
a bad SMTP password never breaks booking, cancellation, or any other user-facing flow.

---

## 5. Google Calendar setup (for live mode)

1. In [Google Cloud Console](https://console.cloud.google.com), create a project and enable the
   **Google Calendar API**.
2. Configure the OAuth consent screen (External, add your test users while in testing mode).
3. Create an **OAuth 2.0 Client ID** (type: Web application).
   - Authorized redirect URI: `http://localhost:5000/api/auth/google/callback` (or your deployed
     backend URL + `/api/auth/google/callback`).
4. Copy the Client ID and Client Secret into `backend/.env`.
5. In the app, a logged-in patient or doctor calls `GET /api/patient/google/connect` (or the
   equivalent doctor route can be added the same way) to get a consent URL, visits it, and Google
   redirects back to `/api/auth/google/callback`, which stores their refresh/access tokens on the
   `User` document. From then on, `calendarService.createEvent/updateEvent/deleteEvent` will use
   their real calendar.

---

## 6. Database schema (MongoDB / Mongoose)

**User** — `name, email, password(hashed), role(patient|doctor|admin), phone, dateOfBirth, gender,
googleCalendar{connected, refreshToken, accessToken, tokenExpiry, calendarId}`

**DoctorProfile** — `user(ref User), specialisation, qualifications, bio, slotDurationMinutes,
workingHours[{day, isWorking, startTime, endTime}], leaveDays[{date, reason}], consultationFee,
isAcceptingPatients`

**Appointment** — `doctor(ref User), patient(ref User), date, startTime, endTime,
status(hold|confirmed|cancelled|completed|no_show|doctor_leave_cancelled), holdExpiresAt(TTL),
symptomForm{...}, preVisitSummary{urgencyLevel, chiefComplaint, suggestedQuestions[], failed},
postVisit{clinicalNotes, prescription[]}, postVisitSummary{summaryText, medicationSchedule[],
followUpSteps[], failed}, calendarEvents{patientEventId, doctorEventId, ...}, cancellationReason,
cancelledBy`

- **Unique partial index** on `(doctor, date, startTime)` for `status in [hold, confirmed]` — the
  core double-booking guard (see `SYSTEM_DESIGN.md`).
- **TTL index** on `holdExpiresAt` — auto-expires abandoned holds.

**Notification** — `type, channel, recipient(ref User), recipientEmail, appointment(ref
Appointment), subject, body, status(pending|sent|failed|abandoned), attempts, lastError,
scheduledFor, sentAt` — doubles as an email queue/log for the reminder and retry cron jobs.

---

## 7. API reference

All routes are prefixed with `/api`. Authenticated routes require `Authorization: Bearer <JWT>`
(returned from login/register).

### Auth
| Method | Route | Access | Description |
|---|---|---|---|
| POST | `/auth/register` | Public | Register a patient |
| POST | `/auth/login` | Public | Log in, returns `{ user, token }` |
| GET  | `/auth/me` | Any authenticated | Current user profile |
| GET  | `/auth/google/callback` | Public (OAuth redirect) | Completes Google Calendar linking |

### Admin (`role: admin`)
| Method | Route | Description |
|---|---|---|
| POST | `/admin/doctors` | Create a doctor account + profile |
| GET  | `/admin/doctors` | List all doctors |
| PUT  | `/admin/doctors/:profileId` | Update a doctor profile (hours, fee, specialisation, ...) |
| POST | `/admin/doctors/:profileId/leave` | Mark a doctor on leave for a date — cancels affected appointments and emails patients |
| GET  | `/admin/appointments` | List/filter all appointments (`?date=&doctorId=&status=`) |

### Patient (`role: patient`)
| Method | Route | Description |
|---|---|---|
| GET  | `/patient/doctors?specialisation=` | Search doctors |
| GET  | `/patient/doctors/:profileId/slots?date=` | Available slots for a date |
| GET  | `/patient/appointments` | My appointments |
| GET  | `/patient/appointments/:id` | Appointment detail (incl. post-visit summary) |
| GET  | `/patient/google/connect` | Get Google OAuth consent URL |

### Appointment booking flow (any authenticated role, scoped by ownership)
| Method | Route | Description |
|---|---|---|
| POST | `/appointments/hold` | `{doctorId, date, startTime}` → atomically holds a slot for 5 min |
| POST | `/appointments/:id/symptoms` | Submit symptom form → triggers LLM pre-visit summary |
| POST | `/appointments/:id/confirm` | Confirms the hold → emails + calendar events created |
| POST | `/appointments/:id/cancel` | `{reason}` → cancels, emails patient, removes calendar events |

### Doctor (`role: doctor`)
| Method | Route | Description |
|---|---|---|
| GET  | `/doctor/profile` / `PUT /doctor/profile` | View/update own profile |
| GET  | `/doctor/appointments?date=` | Appointment list |
| GET  | `/doctor/appointments/:id` | Appointment detail incl. AI pre-visit summary |
| POST | `/doctor/appointments/:id/complete` | `{clinicalNotes, prescription[]}` → generates patient summary + schedules medication reminders |

---

## 8. LLM prompts used

**Pre-visit summary** (`services/llmService.js :: generatePreVisitSummary`):
> "Analyse these symptoms and return: urgency level (Low / Medium / High), chief complaint, and
> three suggested questions for the doctor. Symptoms: `<symptoms>`"
> — plus a strict instruction to respond with raw JSON only, in a fixed shape, so the backend can
> parse it deterministically.

**Post-visit summary** (`services/llmService.js :: generatePostVisitSummary`):
> "Convert these clinical notes into a patient-friendly summary with medication schedule and
> follow-up steps: `<notes>`" (with the prescription details appended)
> — same JSON-only response contract.

Both calls use `model: "claude-sonnet-4-6"`. On any parse or network failure, the service logs the
error, marks `failed: true` on the summary, and substitutes a deterministic fallback summary so
the booking/visit-completion flow **never breaks** because of an LLM issue (see §9 in
`SYSTEM_DESIGN.md`).

---

## 9. Deployment

- **Backend**: deploy to Render/Railway/Fly.io. Set all `.env` vars in the platform's dashboard.
  Point `MONGO_URI` at an Atlas cluster. Update `GOOGLE_REDIRECT_URI` and `CLIENT_URL` to your
  real domains.
- **Frontend**: deploy to Vercel/Netlify. Set `REACT_APP_API_URL` to your deployed backend's
  `/api` URL.
- Run `npm run seed` once against the production database to bootstrap an admin account (then
  change its password via the admin's own login, or edit directly in the DB).

---

## 10. System design write-up

See [`docs/SYSTEM_DESIGN.md`](./docs/SYSTEM_DESIGN.md) for the required write-up on double-booking
prevention, doctor leave conflict handling, the slot hold mechanism, and notification failure
handling.

---

## Healthcare-Appointment-app
