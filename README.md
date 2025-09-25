HospitalLink — HIV Care Coordination Platform

Short description:
HospitalLink is a lightweight hospital management system that securely interconnects hospitals with HIV patient records to support consistent HIV management and treatment. It provides simple authentication for patients and hospital admins, role-based access to records, and APIs for uploading, querying, and syncing care data across participating hospitals.

Note: this README outlines a practical, privacy-first design and a simple deployment / dev setup. If you plan to deploy in production or across real hospitals, follow local health-data regulations (e.g., HIPAA/GDPR equivalents) and consult legal/compliance experts.

Features

Patient accounts and simple authentication (email/phone + OTP or password + JWT).

Hospital admin accounts with role-based access and JWT authentication.

Patient profiles containing anonymized/pseudonymized HIV clinical data (CD4, viral load, ART regimen, adherence notes).

Hospital registry and inter-hospital linking for sharing patient consented records.

Audit logs for access and modifications (who accessed what and when).

RESTful JSON API endpoints for integration with EHRs, mobile apps, or web dashboards.

Basic notifications (email/SMS) hooks for appointment reminders and medication alerts.

Configurable privacy and retention settings.

Quick start (developer)

Below are minimal steps to get the app running locally using Docker. This example assumes a typical Node/Express + MySQL/Postgres stack, but structure is framework-agnostic.

Clone the repo:

git clone https://github.com/your-org/hospitallink.git
cd hospitallink


Copy env example and edit:

cp .env.example .env
# then edit .env to set DB credentials, JWT secret, mail provider, SMS provider, etc.


Run with Docker Compose:

docker-compose up --build


Visit:

API: http://localhost:4000/api

Admin UI (if included): http://localhost:4000/admin

Example .env.example
# App
APP_ENV=development
APP_PORT=4000
APP_URL=http://localhost:4000

# Database (MySQL or Postgres)
DB_DRIVER=mysql
DB_HOST=db
DB_PORT=3306
DB_USER=root
DB_PASSWORD=secret
DB_NAME=hospitallink

# Auth
JWT_SECRET=replace_with_a_strong_secret
JWT_EXPIRES_IN=7d

# Email / SMS (for OTPs and notifications)
MAILER_HOST=smtp.example.com
MAILER_USER=mailer@example.com
MAILER_PASS=supersecret
SMS_PROVIDER_API_KEY=your_sms_key

# Security
ENCRYPTION_KEY=32_byte_hex_or_base64  # use for sensitive fields encryption at rest

# Other
AUDIT_LOG_RETENTION_DAYS=365

Architecture overview

API Layer — RESTful endpoints (Express, FastAPI, Django REST, etc.)

Auth — JWT for hospital admins; simple session/JWT or passwordless OTP (email/SMS) for patients.

Database — Relational DB (MySQL/Postgres). Sensitive fields optionally encrypted.

Storage — Attachments (scans, consent forms) stored in S3-compatible storage (with signed URLs).

Audit & Logging — Write-ahead audit logs table + structured application logs.

Optional — Background worker for email/SMS, data sync tasks and periodic reports.

Suggested database schema (high-level)

users (hospital staff/admins): id, name, email, password_hash, hospital_id, roles, created_at

patients: id, first_name, last_name, dob (optional), identifier_hash, phone, email, consent_flags, created_at

patient_records / hiv_data: id, patient_id, hospital_id, record_type (cd4, viral_load, art_regimen, appointment), payload (JSON), recorded_at, created_by, created_at

hospitals: id, name, address, contact_email, active

patient_hospital_links: id, patient_id, hospital_id, linked_at, consent_given (boolean)

audit_logs: id, entity, entity_id, action, performed_by, details (JSON), created_at

otps (if using passwordless): id, user_id OR patient_id, code_hash, expires_at, used

Notes:

Use identifier_hash for any unique patient identifier (hashed/pseudonymized) and store the clear identifier only if necessary and encrypted.

Record clinical data in structured JSON columns when flexibility is required; also provide separate columns for commonly queried values (e.g., viral_load_value, vl_date) for indexing.

Authentication approach (simple + secure)

Hospital admins

Sign-up / invite by admin.

Authenticate with email + password.

Session via JWT (short-lived access token + optional refresh token).

RBAC: role field (e.g., admin, clinician, viewer) to control permissions.

Patients
Two recommended options (pick one based on UX needs):

Passwordless / OTP (recommended for simplicity)

Patient provides email or phone.

System sends one-time code (OTP) via email/SMS.

OTP verified, then issue short-lived JWT for the patient session.

Password-based

Patient creates password (store hashed with Argon2/Bcrypt).

Use JWT for sessions.

Security best practices

Securely store JWT secret, rotate regularly.

Hash passwords with Argon2 or bcrypt.

Rate-limit and throttle OTP requests.

Use TLS everywhere.

Encrypt PII in DB-at-rest where possible.

Implement role-based authorization checks on every endpoint.

Core API (example endpoints)

All endpoints assume Authorization: Bearer <JWT> for protected routes.

Auth

POST /api/auth/patient/request-otp — request OTP (body: { phone/email })

POST /api/auth/patient/verify-otp — verify OTP → returns JWT

POST /api/auth/admin/login — admin login with email + password → JWT

Patients

GET /api/patients/:id — get patient profile (consented hospitals only)

PUT /api/patients/:id — update contact / consent

POST /api/patients/:id/consent — record consent for sharing with a hospital

Clinical records

POST /api/patients/:id/records — add HIV-related record (hospital or clinician)

GET /api/patients/:id/records?from=...&type=... — query patient records

PUT /api/records/:recordId — edit a non-critical note (audit logged)

DELETE /api/records/:recordId — soft-delete (audit logged) — require proper role & justification

Hospital management

POST /api/hospitals — register hospital (admin-only)

GET /api/hospitals/:id/patients — list linked patients (with authorization + consent)

Audit

GET /api/audit?entity=patients&id=... — admins can view access logs (auditing)

Data privacy & compliance (recommended)

Consent-first model: store patient consent flags and enforce them before sharing.

Pseudonymization: use hashed identifiers for cross-hospital linking; keep direct identifiers encrypted.

Least privilege: staff get minimum permissions; require explicit roles for data exports or patient deletion.

Retention policies: implement configurable retention/archival for sensitive data.

Audit trails: every access/update must be logged with user id, timestamp, and IP.

Security audits & pen tests: mandatory before production.

Testing

Unit tests for business logic (auth flows, consent enforcement, audit logging).

Integration tests for API endpoints (use test DB).

End-to-end tests for primary flows: patient login (OTP), consent grant, clinician add record, cross-hospital sync.

Deployment considerations

Use managed DB (RDS, Cloud SQL) with encryption at rest.

Use HTTPS with a valid certificate.

Run background workers for OTP expiration, notifications, and syncing tasks.

Use IAM / secrets manager for storing JWT secrets and encryption keys.

Set up monitoring & alerting (logs, latency, error rates).

Backup DB regularly and test restorations.

Contributing

Fork the repo.

Create a feature branch: git checkout -b feat/awesome-feature.

Write tests for new features.

Open a PR with a clear description and link to any tickets.

Roadmap / Next steps

Add multi-factor auth for clinicians.

Integrate with national health APIs / EHR connectors (FHIR).

Implement role-based consent workflows (guardian consent, proxy).

Build mobile patient app for offline support and sync.

License

This project is provided with a permissive license (e.g., MIT) by default. Replace with your preferred license.

Contact

If you want, I can scaffold a starter repo (Node/Express or Django), include the authentication flows (OTP + JWT), and create a sample database migration and seed data tailored to this README. Would you like me to generate that now?

ChatGPT can make mistakes. Check important info.
