<p align="center">
  <h1 align="center">🚑 Sevatra</h1>
  <p align="center">
    <strong>Hospital Management System × Ambulance Booking × Ambulance Management</strong>
    <br />
  </p>
</p>

---

## The Problem

Every minute counts in a medical emergency. But when an ambulance arrives at a hospital, the staff is often unprepared: wrong doctor, no bed, no context. Precious time is lost at the door.

Paramedics call ahead — if they can. Reception scrambles to find a bed. The right doctor may not even be on the floor. The patient's condition deteriorates while logistics catch up.

## Where We Did Things Differently

| Step | What Happens |
|------|-------------|
| **1. Instant SOS** | In critical moments, anyone can book an ambulance with our SOS mode — no login required. Logged-in users get instant auto-dispatch; others verify via phone OTP. |
| **2. Nearest Ambulance** | The system calculates haversine distance to all available ambulances and assigns the closest one automatically. |
| **3. Vitals Record** | Once the patient is in the ambulance, the paramedic or helper records vitals (heart rate, SpO₂, respiratory rate, temperature, blood pressure) into the platform and sends them directly to the chosen hospital. |
| **4. Smart Triage** | A severity scoring algorithm (0–10) calculates the patient's condition from those vitals and classifies them as Recovering, Stable, Serious, or Critical. |
| **5. Pre-Arrival Allocation** | Based on that score, the system automatically checks bed availability and assigns the right ward (General / HDU / ICU) and the best available doctor — before the patient even reaches the hospital. |

**So when the patient reaches the hospital, there's no rush at the reception. No confusion about who should take the case. The hospital is already informed and prepared.**

---

## Table of Contents

- [Overview](#overview)
- [Architecture](#architecture)
- [Tech Stack](#tech-stack)
- [Applications](#applications)
  - [AmbiSevatra — Ambulance Booking App](#ambisevatra--ambulance-booking-app)
  - [LifeSevatra — Hospital Management System](#lifesevatra--hospital-management-system)
  - [OperatoSevatra — Ambulance Management System](#operatosevatra--ambulance-management-system)
- [Backend API](#backend-api)
  - [Authentication](#authentication)
  - [API Endpoints](#api-endpoints)
- [Severity Score Algorithm](#severity-score-algorithm)
- [Getting Started](#getting-started)
  - [Prerequisites](#prerequisites)
  - [Backend Setup](#backend-setup)
  - [Frontend Setup](#frontend-setup)
- [Environment Variables](#environment-variables)
- [Data Seeding](#data-seeding)
- [Deployment](#deployment)
- [Project Structure](#project-structure)

---

## Overview

**Sevatra** is a unified healthcare platform comprising three interconnected applications and a single backend API:

| Application | Purpose | Users |
|---|---|---|
| **AmbiSevatra** | Ambulance booking & emergency SOS | Patients / Public |
| **LifeSevatra** | Hospital management, admissions, triage | Hospital admins & Doctors |
| **OperatoSevatra** | Ambulance fleet & operator management | Ambulance operators / providers |

All three frontends share a single **FastAPI** backend (the **master-backend**) which manages authentication, data storage, real-time tracking, and cross-platform ambulance assignment.

---

## Architecture

```
┌──────────────────┐  ┌──────────────────┐  ┌──────────────────────┐
│  AmbiSevatra     │  │  LifeSevatra     │  │  OperatoSevatra      │
│  (React + Vite)  │  │  (React + Vite)  │  │  (React + Vite)      │
│  :5173           │  │  :5175           │  │  :5174               │
└────────┬─────────┘  └────────┬─────────┘  └──────────┬───────────┘
         │                     │                       │
         └─────────────────────┼───────────────────────┘
                               │ REST + WebSocket
                    ┌──────────▼──────────┐
                    │   Master Backend    │
                    │   FastAPI (Python)  │
                    │   /api/v1           │
                    └──┬──────┬───────┬───┘
                       │      │       │
              ┌─────────▼┐ ┌──▼────┐ ┌▼─────────┐
              │Firebase  │ │Redis  │ │ Dropbox  │
              │Auth +    │ │(OTP   │ │ (File    │
              │Firestore │ │Store) │ │ Storage) │
              └──────────┘ └───────┘ └──────────┘
```

---

## Tech Stack

### Backend
| Technology | Purpose |
|---|---|
| **Python 3.12+** | Runtime |
| **FastAPI** | Web framework (async, OpenAPI docs) |
| **Pydantic v2** | Data validation & settings management |
| **Firebase Admin SDK** | Authentication (email/password, Google OAuth) + Firestore NoSQL database |
| **Redis (Upstash)** | OTP storage with auto-expiry |
| **Twilio** | SMS OTP delivery for SOS & phone verification |
| **Dropbox API** | File storage (patient ID photos, documents) |
| **SMTP** | Email OTP delivery for registration |
| **Uvicorn** | ASGI server |
| **HTTPX** | Async HTTP client (Firebase REST APIs) |

### Frontend (all three apps)
| Technology | Purpose |
|---|---|
| **React 19** | UI framework |
| **TypeScript** | Type safety |
| **Vite 7** | Build tool & dev server |
| **Tailwind CSS 4** | Utility-first styling |
| **React Router v7** | Client-side routing |
| **Framer Motion** | Animations (AmbiSevatra & OperatoSevatra) |
| **Leaflet / React-Leaflet** | Map display (AmbiSevatra) |
| **Lucide React** | Icon library (AmbiSevatra) |

---

## Applications

### AmbiSevatra — Ambulance Booking App

The **patient-facing** mobile-first app for ambulance booking and emergencies.

**Key Features:**
- **Emergency SOS** — One-tap activation with 5-second countdown, GPS auto-detection, and ambulance auto-dispatch for logged-in users
- **SOS OTP Verification** — Unauthenticated users verify via phone OTP (Twilio SMS) before dispatch
- **Scheduled Ambulance Booking** — Book transport with custom date/time picker, patient details, and special needs
- **Real-Time Tracking** — Live ambulance location via WebSocket with ETA, driver details, and map visualization
- **User Profiles** — Manage personal info, blood group, saved addresses, emergency contacts, and medical conditions
- **Booking History** — View past and current bookings with status tracking
- **Google Sign-In** — Quick registration/login via Google OAuth
- **Email Verification** — 6-digit OTP-based email verification on signup

**Routes:** `/`, `/login`, `/book-ambulance`, `/sos-activation`, `/profile`, `/booking-history`, `/ambulance-confirmed`

---

### LifeSevatra — Hospital Management System

The **hospital admin and doctor** portal for managing admissions, staff, beds, and clinical workflows.

**Key Features:**
- **Dashboard** — Overview with total patients, critical count, today's admissions/discharges, bed occupancy by ward type (ICU/HDU/General)
- **Patient Admissions** — Full admission form with patient demographics, vitals, clinical notes, lab results, government ID upload, and guardian information
- **Automatic Severity Scoring** — Vitals-based triage scoring (0–10 scale) with automatic ward recommendation (ICU/HDU/General)
- **Automatic Bed Assignment** — Assigns beds based on severity-recommended ward type on admission
- **Automatic Doctor Assignment** — Assigns the on-duty doctor with the least patient load
- **Staff Management** — CRUD for doctors, surgeons, specialists, and nurses with shift management and duty toggles
- **Bed Management** — View all beds, occupancy stats, manual assign/release
- **File Uploads** — Patient photos and ID documents stored in Dropbox
- **Doctor Portal** — Separate role-based views for doctors including:
  - **My Patients** — View only assigned patients
  - **Patient Updates** — Update vitals (auto-recalculates severity) and clinical info
  - **Schedule** — Create and manage daily schedule slots
  - **Clinical Notes** — Add observations, prescriptions, discharge summaries, progress notes
  - **Doctor Profile** — Manage personal profile, bio, languages, consultation fee
- **Role-Based Authentication** — Admin (hospital) and Staff (doctor/nurse) roles with separate login flows
- **Registration with Email OTP** — Hospital registers with bed configuration, verifies email

**Admin Routes:** `/overview`, `/newadmission`, `/staff`, `/new-staff`  
**Doctor Routes:** `/doctor`, `/doctor/patients`, `/doctor/patients/:id`, `/doctor/schedule`, `/doctor/notes`, `/doctor/profile`

---

### OperatoSevatra — Ambulance Management System

The **ambulance operator** portal for managing fleets and viewing assignments.

**Key Features:**
- **Operator Registration** — Register as individual or provider (fleet) operator
- **Ambulance Fleet CRUD** — Register ambulances with full details: vehicle info, equipment (oxygen, defibrillator, stretcher, ventilator), driver details (name, phone, license, experience, photo), base location (GPS), service radius, pricing
- **Ambulance Types** — Basic, Advanced, Patient Transport, Neonatal, Air
- **Status Management** — Toggle ambulance status: Available, On Trip, Maintenance, Off Duty
- **Dashboard** — Fleet statistics: total ambulances, availability breakdown, trips completed
- **Operator Profiles** — Facility details, license number, verification status
- **Provider-Only Routes** — Fleet management restricted to provider-type operators

**Routes:** `/login`, `/dashboard`, `/ambulances`, `/ambulances/add`, `/profile`

---

## Backend API

### Authentication

Sevatra uses a **platform-isolated Firebase Auth** strategy. Each platform (ambi, operato, life) prefixes emails with the platform name to maintain separate user pools in a single Firebase project:

```
ambi.user@example.com    → AmbiSevatra user
operato.user@example.com → OperatoSevatra user
life.user@example.com    → LifeSevatra user (hospital or staff)
```

**Auth flows:**

| Flow | Description |
|---|---|
| Email/Password Signup | Create account → Email OTP → Verify → Tokens issued |
| Google Sign-In | Google JWT → Firebase user created → Tokens issued (AmbiSevatra) |
| Hospital Registration | Register with bed config → Email OTP → Verify → Tokens issued |
| Staff Login | Credentials created by hospital admin → Login → Staff token issued |
| Token Refresh | Exchange refresh token for new access token |

All authenticated endpoints expect a Firebase ID token in the `Authorization: Bearer <token>` header.

### API Endpoints

All routes are prefixed with `/api/v1`.

#### AmbiSevatra Endpoints (`/auth`, `/users`, `/bookings`, `/sos`, `/tracking`)

| Method | Endpoint | Auth | Description |
|---|---|---|---|
| POST | `/auth/signup` | No | Create account (email verification required) |
| POST | `/auth/signup/google` | No | Google Sign-In register/login |
| POST | `/auth/verify-email` | No | Verify email with 6-digit OTP |
| POST | `/auth/resend-email` | No | Resend verification OTP |
| POST | `/auth/login` | No | Email/password login |
| POST | `/auth/refresh` | No | Refresh access token |
| POST | `/auth/logout` | Yes | Sign out |
| POST | `/auth/otp/send` | No | Send SMS OTP (general purpose) |
| POST | `/auth/otp/verify` | No | Verify SMS OTP |
| POST | `/auth/internal/cleanup` | Cron | Delete unverified users past TTL |
| GET | `/users/me` | Yes | Get current user profile |
| PUT | `/users/me` | Yes | Update profile |
| GET/POST/DELETE | `/users/me/addresses` | Yes | Manage saved addresses |
| GET/POST/DELETE | `/users/me/emergency-contacts` | Yes | Manage emergency contacts |
| GET/POST/DELETE | `/users/me/medical-conditions` | Yes | Manage medical conditions |
| POST | `/bookings/` | Yes | Create scheduled booking |
| GET | `/bookings/` | Yes | List user's bookings |
| GET | `/bookings/{id}` | Yes | Get booking details |
| PATCH | `/bookings/{id}` | Yes | Update a booking |
| DELETE | `/bookings/{id}` | Yes | Cancel a booking |
| POST | `/sos/activate` | Optional | Initiate emergency SOS |
| POST | `/sos/{id}/send-otp` | No | Send SOS verification OTP |
| POST | `/sos/{id}/verify` | No | Verify OTP & dispatch ambulance |
| GET | `/sos/{id}/status` | No | Get SOS event status |
| POST | `/sos/{id}/cancel` | No | Cancel SOS event |
| POST | `/tracking/{id}/location` | No | Push GPS location (driver) |
| GET | `/tracking/{id}/location` | No | Get latest location (REST) |
| WS | `/tracking/{id}/ws` | No | Live tracking WebSocket |

#### LifeSevatra Endpoints (`/life/auth`, `/life/admissions`, `/life/beds`, `/life/staff`, `/life/dashboard`, `/life/vitals`, `/life/files`, `/life/doctor`)

| Method | Endpoint | Auth | Description |
|---|---|---|---|
| POST | `/life/auth/register` | No | Register hospital account |
| POST | `/life/auth/login` | No | Hospital/staff login |
| POST | `/life/auth/verify-email` | No | Verify registration OTP |
| POST | `/life/auth/resend-email` | No | Resend OTP |
| POST | `/life/auth/refresh-token` | No | Refresh token |
| POST | `/life/admissions/` | Hospital | Admit patient (auto-severity, bed, doctor) |
| GET | `/life/admissions/` | Hospital | List patients (filterable) |
| GET | `/life/admissions/{id}` | Hospital | Get patient details |
| PUT | `/life/admissions/{id}/vitals` | Hospital | Update vitals (recalculates severity) |
| PUT | `/life/admissions/{id}/clinical` | Hospital | Update clinical info |
| POST | `/life/admissions/{id}/discharge` | Hospital | Discharge patient (releases bed & doctor) |
| DELETE | `/life/admissions/{id}` | Hospital | Delete admission record |
| GET | `/life/beds/` | Hospital | List all beds |
| GET | `/life/beds/stats` | Hospital | Bed statistics by ward type |
| GET | `/life/beds/availability` | Hospital | Bed availability summary |
| PUT | `/life/beds/{id}/assign` | Hospital | Manually assign bed |
| PUT | `/life/beds/{id}/release` | Hospital | Release bed |
| POST | `/life/staff/` | Hospital | Add staff member |
| GET | `/life/staff/` | Hospital | List all staff |
| GET | `/life/staff/stats` | Hospital | Staff count statistics |
| GET | `/life/staff/available-doctors` | Hospital | On-duty doctors with capacity |
| GET/PUT/DELETE | `/life/staff/{id}` | Hospital | Staff CRUD |
| PATCH | `/life/staff/{id}/duty` | Hospital | Toggle duty status |
| GET | `/life/dashboard/stats` | Hospital | Dashboard KPIs |
| POST | `/life/vitals/calculate-severity` | No | Calculate severity from vitals |
| POST | `/life/files/upload` | Hospital | Upload file to Dropbox |
| GET | `/life/files/download` | Hospital | Download file |
| GET | `/life/files/list` | Hospital | List uploaded files |
| DELETE | `/life/files/delete` | Hospital | Delete file |
| GET | `/life/doctor/patients` | Staff | Doctor's assigned patients |
| GET | `/life/doctor/patients/{id}` | Staff | Patient details |
| GET/POST | `/life/doctor/schedule` | Staff | Schedule management |
| PUT | `/life/doctor/schedule/{id}` | Staff | Update slot status |
| GET/POST | `/life/doctor/notes` | Staff | Clinical notes |
| GET/PUT | `/life/doctor/profile` | Staff | Doctor profile |

#### OperatoSevatra Endpoints (`/operator`)

| Method | Endpoint | Auth | Description |
|---|---|---|---|
| POST | `/operator/register` | Yes | Register as operator |
| GET | `/operator/profile` | Yes | Get operator profile |
| PUT | `/operator/profile` | Yes | Update operator profile |
| GET | `/operator/check` | Yes | Check operator status |
| GET | `/operator/dashboard` | Yes | Dashboard statistics |
| POST | `/operator/ambulances` | Yes | Register ambulance |
| GET | `/operator/ambulances` | Yes | List fleet |
| GET | `/operator/ambulances/{id}` | Yes | Get ambulance details |
| PATCH | `/operator/ambulances/{id}` | Yes | Update ambulance |
| DELETE | `/operator/ambulances/{id}` | Yes | Delete ambulance |
| PATCH | `/operator/ambulances/{id}/status` | Yes | Toggle ambulance status |

---

## Severity Score Algorithm

The platform features a **clinical severity scoring system** used for patient triage, automated ward allocation, and decision support. The algorithm runs both server-side (Python) and client-side (TypeScript) with identical logic.

### Scoring Breakdown

The total score is the **sum of 5 sub-scores** (each 0–3), **capped at 10**:

| Vital Sign | Normal Range | Score 1 | Score 2 | Score 3 |
|---|---|---|---|---|
| **Heart Rate** | 60–100 bpm | 50–60 or 100–110 | 40–50 or 110–130 | <40 or >130 |
| **SpO₂** | 95–100% | 90–94% | 85–90% | <85% |
| **Resp. Rate** | 12–20 br/min | 8–12 or 20–24 | 24–30 | <8 or >30 |
| **Temperature** | 36.5–37.5°C | 36–36.5 or 37.5–38 | 35–36 or 38–40 | <35 or >40 |
| **Blood Pressure** | SBP 90–140, DBP 60–90 | Mild deviation | Moderate deviation | Critical (SBP <70, >180) |

### Classification

| Score | Condition | Ward | Urgency |
|---|---|---|---|
| 0–2 | Recovering | General | Low |
| 3–4 | Stable | General | Routine |
| 5–7 | Serious | HDU | Urgent |
| 8–10 | Critical | ICU | Immediate |

### Ambulance Assignment

When an SOS is activated or a booking is created with GPS coordinates, the system:

1. Queries all ambulances with `status = "available"` from Firestore
2. Calculates **haversine distance** from the request GPS to each ambulance's base location
3. Assigns the **nearest ambulance** and sets its status to `on_trip`
4. Returns ambulance details (vehicle, driver, equipment, distance) embedded in the SOS/booking response

---

## Getting Started

### Prerequisites

- **Python 3.12+**
- **Node.js 20+** and **npm**
- **Firebase project** with Firestore and Authentication enabled
- **Redis instance** (e.g., Upstash Redis)
- (Optional) Twilio account, Dropbox app, SMTP server

### Backend Setup

```bash
cd master-backend

# Create virtual environment
python -m venv .venv
.venv\Scripts\activate     # Windows
# source .venv/bin/activate  # macOS/Linux

# Install dependencies
pip install -r requirements.txt

# Configure environment
cp .env.example .env
# Edit .env with your Firebase credentials, Redis URL, etc.

# Place Firebase credentials
# Option A: firebase-credentials.json in master-backend/
# Option B: Set FIREBASE_CREDENTIALS_BASE64 in .env

# Run development server
uvicorn app.main:app --reload --port 8000
```

The API docs will be available at `http://localhost:8000/docs` (Swagger UI) and `http://localhost:8000/redoc` (ReDoc).

### Frontend Setup

Each frontend runs independently:

```bash
# AmbiSevatra (port 5173)
cd ambisevatra-frontend
npm install
npm run dev

# OperatoSevatra (port 5174)
cd operatosevatra-frontend
npm install
npm run dev

# LifeSevatra (port 5175)
cd lifesavatra-frontend
npm install
npm run dev
```

---

## Environment Variables

Create a `.env` file in `master-backend/`:

```env
# ── Firebase ──
FIREBASE_CREDENTIALS_PATH=firebase-credentials.json
FIREBASE_CREDENTIALS_BASE64=            # Alternative: base64-encoded credentials JSON
FIREBASE_API_KEY=                       # Firebase Web API key (for REST auth)

# ── Google OAuth ──
GOOGLE_CLIENT_ID=                       # Google OAuth client ID

# ── Redis (Upstash) ──
REDIS_URL=rediss://default:...@....upstash.io:6379

# ── Twilio (SMS OTP) ──
TWILIO_ENABLED=true
TWILIO_ACCOUNT_SID=
TWILIO_AUTH_TOKEN=
TWILIO_PHONE_NUMBER=

# ── Dropbox (File Storage) ──
DROPBOX_APP_KEY=
DROPBOX_APP_SECRET=
DROPBOX_REFRESH_TOKEN=

# ── SMTP (Email OTP) ──
SMTP_HOST=smtp.gmail.com
SMTP_PORT=587
SMTP_EMAIL=
SMTP_PASSWORD=

# ── App Settings ──
APP_NAME=Sevatra API
APP_ENV=development
SECRET_KEY=change-this-to-a-random-secret-key
OTP_EXPIRY_SECONDS=300
OTP_LENGTH=6

# ── Cron Cleanup ──
CRON_SECRET=
UNVERIFIED_USER_TTL_HOURS=1
```

Each frontend uses a `VITE_API_URL` environment variable (defaults to `https://api-sevatra.vercel.app`).

---

## Data Seeding

Two utility scripts are provided at the repository root for populating test data:

### `add_hospitals.py`
Seeds 3 hospitals directly via Firebase Admin SDK (bypasses email verification):
```bash
python add_hospitals.py
```
Creates hospitals with pre-configured bed layouts (ICU/HDU/General) and generates bed records.

### `add_data.py`
Seeds 20 random patients and 15 staff members into a hospital via the API:
```bash
python add_data.py
```
Generates patients with varying severity tiers (critical/serious/moderate/normal) and staff across roles (doctor, surgeon, specialist, nurse).

---

## Deployment

The backend is configured for **Vercel** serverless deployment:

```json
{
  "builds": [{ "src": "app/main.py", "use": "@vercel/python" }],
  "routes": [{ "src": "/(.*)", "dest": "app/main.py" }]
}
```

Frontend apps can be deployed to any static hosting (Vercel, Netlify, etc.) with `npm run build`.

**Production API:** `https://api-sevatra.vercel.app`

---

## Project Structure

```
sevatra-sbr/
├── master-backend/                 # Unified FastAPI backend
│   ├── app/
│   │   ├── main.py                 # App entry, router registration, middleware
│   │   ├── config.py               # Pydantic Settings (env vars)
│   │   ├── firebase_client.py      # Firebase Admin SDK init, Firestore helpers
│   │   ├── redis_client.py         # Upstash Redis connection, OTP helpers
│   │   ├── core/
│   │   │   ├── dependencies.py     # get_current_user (Firebase token validation)
│   │   │   ├── middleware.py       # Request logging middleware
│   │   │   └── email.py           # Email OTP sending via SMTP
│   │   ├── ambi/                   # AmbiSevatra domain
│   │   │   ├── models.py          # Auth, User, Booking, SOS models
│   │   │   ├── routers/           # auth, bookings, sos, tracking, users
│   │   │   └── services/          # auth, booking, sos, twilio, user, ambulance assignment
│   │   ├── life/                   # LifeSevatra domain
│   │   │   ├── models.py          # Hospital, Admission, Staff, Doctor, Severity models
│   │   │   ├── dependencies.py    # get_current_hospital, get_current_staff
│   │   │   ├── routers/           # auth, admissions, beds, staff, dashboard, vitals, files, doctor
│   │   │   ├── services/          # auth, admission, bed, staff, dashboard, doctor, dropbox
│   │   │   └── utils/severity.py  # Severity score calculator
│   │   └── operato/                # OperatoSevatra domain
│   │       ├── models.py          # Operator, Ambulance, Dashboard models
│   │       ├── routers/operators.py
│   │       └── services/operator_service.py
│   ├── requirements.txt
│   └── vercel.json
├── ambisevatra-frontend/           # Patient ambulance booking app
│   └── src/
│       ├── pages/                  # Home, Login, BookAmbulance, SosActivation, Profile, etc.
│       ├── services/api.ts         # Full API client with token refresh
│       └── context/UserContext.tsx  # Auth state & profile management
├── lifesavatra-frontend/           # Hospital management portal
│   └── src/
│       ├── pages/
│       │   ├── auth/               # Login, Register
│       │   ├── dashboard/          # Overview, NewAdmission, Staff, NewStaff, SeverityScore
│       │   └── doctor/             # DoctorDashboard, MyPatients, PatientUpdate, Schedule, ClinicalNotes, DoctorProfile
│       ├── services/               # admissionService, authService, doctorService, staffService
│       ├── utils/severityCalculator.ts  # Client-side severity algorithm
│       └── context/AuthContext.tsx  # Role-based auth (admin vs doctor)
├── operatosevatra-frontend/        # Ambulance operator portal
│   └── src/
│       ├── pages/                  # Login, Dashboard, Ambulances, AddAmbulance, OperatorProfile
│       ├── services/api.ts         # Operator API client
│       └── context/OperatorContext.tsx
├── add_data.py                     # Seed patients & staff
└── add_hospitals.py                # Seed hospitals
```

---

<p align="center">
  Built with ❤️ using FastAPI, React, Firebase, and Redis
  <br />
  <em>Because no patient should wait at the door while logistics catch up.</em>
</p>
