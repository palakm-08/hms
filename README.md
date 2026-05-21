# HMS — Hospital Management System

A locally-runnable hospital management web application built with Django, PostgreSQL, and a separate Python serverless email notification service.

---

## Setup and Run

### Prerequisites

Install these before starting:

| Requirement | Notes |
|---|---|
| Python 3.11+ | `python3 --version` |
| Node.js 18+ | `node --version` — needed for serverless-offline |
| PostgreSQL 14+ | Running locally |
| npm | Bundled with Node.js |

---

### Step 1 — Create the PostgreSQL database

Connect as your PostgreSQL superuser and run:

```bash
psql -U postgres -f setup_db.sql
```

This creates:
- Database: `hms_db`
- User: `hms_user` / Password: `hms_password`

Or manually:
```sql
CREATE USER hms_user WITH PASSWORD 'hms_password';
CREATE DATABASE hms_db OWNER hms_user;
GRANT ALL PRIVILEGES ON DATABASE hms_db TO hms_user;
```

---

### Step 2 — Configure the Django environment

```bash
cp hms/.env.example hms/.env
```

Edit `hms/.env`:

```env
DB_NAME=hms_db
DB_USER=hms_user
DB_PASSWORD=hms_password
DB_HOST=localhost
DB_PORT=5432

EMAIL_SERVICE_URL=http://localhost:3000/dev/send-email

# Optional — for Google Calendar integration
GOOGLE_CLIENT_ID=your-google-client-id
GOOGLE_CLIENT_SECRET=your-google-client-secret
GOOGLE_REDIRECT_URI=http://localhost:8000/appointments/google/callback/
```

---

### Step 3 — Install Python dependencies and run migrations

```bash
python3 -m venv .venv
source .venv/bin/activate        # Windows: .venv\Scripts\activate

pip install -r requirements.txt

cd hms
python manage.py migrate
python manage.py createsuperuser  # optional, for admin panel
```

---

### Step 4 — Start the Django server

```bash
# From the hms/ directory, with virtualenv active
python manage.py runserver
```

App available at: **http://localhost:8000**

---

### Step 5 — Configure the email service

```bash
cp email-service/.env.example email-service/.env
```

Edit `email-service/.env`:

```env
SMTP_HOST=smtp.gmail.com
SMTP_PORT=587
SMTP_USER=your-gmail@gmail.com
SMTP_PASSWORD=your-app-password   # Generate at myaccount.google.com/apppasswords
FROM_EMAIL=your-gmail@gmail.com
FROM_NAME=HMS Hospital System
```

> **Gmail App Password:** Go to Google Account → Security → 2-Step Verification → App passwords. Generate one for "Mail". Use that, not your Gmail password.
>
> **No SMTP configured?** Emails print to the serverless-offline console — no email setup required for local demo.

---

### Step 6 — Start the serverless email service

In a **second terminal**:

```bash
cd email-service
npm install
npx serverless offline
```

Service runs at: **http://localhost:3000/dev/send-email**

---

### Verify everything works

```
Terminal 1:  Django running on http://localhost:8000
Terminal 2:  serverless-offline running on http://localhost:3000
```

Test the email service directly:
```bash
curl -X POST http://localhost:3000/dev/send-email \
  -H "Content-Type: application/json" \
  -d '{"event_type":"SIGNUP_WELCOME","email":"test@example.com","name":"Test User","role":"patient"}'
```

---

## System Architecture

### Overview

```
Browser
  │
  ▼
Django App (port 8000)
  ├── accounts/          User auth (signup, login, logout)
  ├── appointments/      Slots, bookings, dashboards
  │     ├── services/email_service.py   → HTTP POST → Serverless Email
  │     └── services/calendar_service.py → Google Calendar API
  └── templates/         HTML templates (no JS framework)

Serverless Email Service (port 3000)
  └── handler.py         Receives POST, sends SMTP email
```

### Data Models

**User** (extends AbstractUser)
- `role`: `'doctor'` | `'patient'`
- `phone`: optional contact number
- `google_access_token`, `google_refresh_token`, `google_token_expiry`: OAuth2 tokens stored directly on the user row for simplicity

**AvailabilitySlot**
- Belongs to one doctor
- `date`, `start_time`, `end_time`
- `is_booked`: boolean flag — set to `True` atomically on booking
- Unique constraint on `(doctor, date, start_time)` — no duplicate slots

**Booking**
- OneToOne with AvailabilitySlot (one booking per slot)
- Belongs to one patient
- `doctor_calendar_event_id`, `patient_calendar_event_id`: stored after Google Calendar creation

### Role-Based Access

Enforced via two custom decorators in `appointments/decorators.py`:

```python
@doctor_required   # wraps @login_required, checks user.role == 'doctor'
@patient_required  # wraps @login_required, checks user.role == 'patient'
```

Applied directly to views. Any attempt by a patient to hit a doctor URL (or vice versa) is caught and redirected with an error message. The `dashboard` view also routes by role, so `/appointments/dashboard/` shows the correct UI without exposing the other role's URL.

### Google Calendar Integration

Flow:
1. User clicks "Connect Google Calendar" → redirected to Google OAuth2 consent screen
2. Google redirects back to `/appointments/google/callback/?code=...`
3. Django exchanges the code for `access_token` + `refresh_token` and stores them on the User model
4. On each booking confirmation, `calendar_service.create_booking_calendar_events(booking)` is called:
   - Creates event on doctor's primary calendar: "Appointment with [PatientName]"
   - Creates event on patient's primary calendar: "Appointment with Dr. [DoctorName]"
   - Stores returned event IDs on the Booking record
5. Token refresh is handled automatically — if `access_token` is expired, `refresh_access_token()` is called using the stored `refresh_token`

Calendar failure is non-fatal: if the user hasn't connected Google or the API call fails, the booking proceeds normally and an error is logged.

### Serverless Email Service

Django's `email_service.py` calls the serverless function via a plain `requests.post()`:

```python
requests.post(settings.EMAIL_SERVICE_URL, json={
    'event_type': 'BOOKING_CONFIRMATION',
    'email': user.email,
    ...
}, timeout=10)
```

The serverless function (`handler.py`) receives this, selects the appropriate email template, and sends via SMTP. In development with no SMTP configured, it prints the email to stdout — so you can demo without any email account.

Both `SIGNUP_WELCOME` and `BOOKING_CONFIRMATION` are handled. Email failure is non-fatal — a `try/except` catches all errors so bookings and signups are never blocked by email issues.

---

## The Design Decision

### Race Condition in Slot Booking

**The Problem**

Two patients can view the same available slot simultaneously. Without a concurrency guard, both could submit a booking request at the same instant — both pass the "is this slot free?" check and both create a booking. This is a classic lost-update race condition.

**Two approaches I considered**

**Option A — Application-level flag check with re-check after write**
Check `is_booked` before creating the booking, then check again after. Use an optimistic approach: let both proceed and rely on the database's unique constraint on `Booking.slot` (OneToOneField) to reject the second. The second request gets a database integrity error, which you catch and turn into a user-facing message.

This works but has a window: between the read and the write, another transaction can sneak in. The unique constraint on OneToOneField does prevent a double-booking from being committed, but the error handling is messier (catching IntegrityError and deciding whether it's a race condition or a different bug). It also doesn't handle the case where a slot is marked booked via the `is_booked` flag but the Booking row creation fails.

**Option B — SELECT FOR UPDATE inside transaction.atomic()**
Lock the slot row pessimistically at the start of the transaction:
```python
with transaction.atomic():
    slot = AvailabilitySlot.objects.select_for_update().get(pk=slot_id)
    if slot.is_booked:
        # reject
    slot.is_booked = True
    slot.save()
    Booking.objects.create(slot=slot, patient=request.user)
```
The database-level row lock ensures the second request blocks until the first transaction commits. By the time the second request reads the slot, `is_booked` is already `True` and it gets cleanly rejected.

**Why I chose Option B**

I chose `SELECT FOR UPDATE` because it makes the race condition elimination explicit, readable, and guaranteed by the database — not by application logic. The intent is visible in the code: anyone reading it immediately understands concurrency is being handled here. Option A's reliance on catching IntegrityError mixes two concerns (concurrency and data validation) and makes it harder to understand why the error handling is there.

The performance cost is real — row-level locking adds latency for concurrent bookings on the same slot — but for a booking system, slot contention is rare, wait times are milliseconds, and the correctness guarantee is worth it. A double-booked appointment is a much worse user experience than a 5ms wait.

---

## Limitations

**What would break in production**

1. **Google OAuth2 tokens stored in plain DB columns.** In production, `google_access_token` should be encrypted at rest. A DB dump would expose all tokens. Fix: encrypt with Django's encrypted fields or a secrets manager.

2. **No token revocation on logout.** When a user logs out, their Google token remains valid. Fix: call the Google revoke endpoint on logout or account deletion.

3. **Email service has no authentication.** The serverless endpoint accepts requests from anyone — no API key, no signature. Fix: add a shared secret header (`X-HMS-Secret`) that Django sends and the function verifies.

4. **`SELECT FOR UPDATE` serializes concurrent bookings.** For very high traffic, this creates a lock bottleneck. Fix: for production scale, consider optimistic locking with a version counter and retry logic, or a queue-based booking system.

5. **No HTTPS.** Session cookies are sent in plaintext over HTTP. Fix: enforce HTTPS in production with `SECURE_SSL_REDIRECT = True` and proper TLS termination.

6. **Secret key is hardcoded in settings.** `SECRET_KEY` must come from environment variables in production. Fix: already using `.env` pattern — just ensure the example key is never used.

7. **Static files served by Django's dev server.** In production, serve statics via Nginx or a CDN. Fix: configure `whitenoise` or S3 + CloudFront.

8. **No email rate limiting.** A signup flood could exhaust SMTP quotas. Fix: add rate limiting on the serverless endpoint and the Django signup view.

**What I'd fix first:** The token encryption (#1) and the email service authentication (#3) — both are low-effort, high-security-impact changes that matter before any real user data goes into the system.
