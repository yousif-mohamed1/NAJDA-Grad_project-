# NAJDA — Backend

Spring Boot API powering every role in NAJDA: incident intake, dispatch, shift and mission lifecycle, hospital handoff, facility management, and the AI/geospatial/identity services that support them.

## Stack

- **Spring Boot** (Java) — REST API, WebSocket (STOMP over a simple in-memory broker), Spring Security, Spring Data JPA
- **PostgreSQL** — the single source of truth for everything except the auth credential itself
- **Firebase Admin SDK** — verifies ID tokens, creates and manages account credentials (employee accounts, and citizen accounts after email verification), reads live provider/verification state when needed, and backs the admin-facing Firebase↔database reconciliation tooling
- **Supabase Storage** — private bucket, accessed exclusively through short-lived signed URLs the backend mints on demand
- **OpenRouter** — free-tier LLM access for incident priority classification and duplicate-incident suggestion
- **Brevo** — transactional email over its HTTPS API (registration verification codes), behind an `EmailSender` interface so swapping providers means writing one adapter class
- **OpenStreetMap ecosystem** — Overpass API (facility seeding), Nominatim (reverse geocoding, rate-limit aware), OSRM (driving-route polylines for the live map)

## Domain model, briefly

- **Facility** — one entity, one `facilityType` (`HOSPITAL` / `AMBULANCE_STATION` / `FIRE_STATION` / `POLICE_STATION`), with a `registered` flag set automatically the moment real staff or a unit gets linked to it.
- **ResponseUnit** — a vehicle (or, for `FIRST_RESPONDER`, a person's personal unit). Ambulance and first-responder units may operate without a fixed facility; every other type requires one.
- **ShiftAssignment** — the record that a specific employee is on duty for a specific unit, as `LEAD` or `CREW`. A unit has no operational status without one. Every mission-action authorization check ultimately traces back to "is this caller the unit's active LEAD."
- **Incident → Mission → HospitalTransfer** — an incident can carry multiple concurrent missions (one per responding unit); an `AMBULANCE` mission's terminal step is a `HospitalTransfer`, which carries its own state (`SELECTED` → `EN_ROUTE` → `ARRIVED`) and, while active, a running vitals log. An incident also carries a `source` (citizen app vs. a paired device — the latter modeled, not yet wired to a real integration).
- **FirstResponderApplication** — a citizen's application to become a `FIRST_RESPONDER`, submitted as one atomic request with its required supporting documents attached (not added afterward), reviewed by an admin.
- **EmailVerification** — a short-lived record per email address going through registration: the hashed one-time code, attempt counter, resend cooldown, and — once verified — the hash of a one-time registration token. It's deleted in the same transaction that creates the account; expired rows are purged by a scheduled job.

## API surface, by domain

| Domain | Base path | Notes |
|---|---|---|
| Auth | `/api/auth` | Citizen email + password registration (`/register/otp/send` → `/register/otp/verify` → `/register/citizen/password`, all public), Google/phone registration bootstrap, admin-provisioned employee creation |
| Users | `/api/users` | Profile, role, facility assignment, phone/email verification, admin overrides |
| Firebase sync | `/api/admin/firebase-sync` | Detects and resolves drift between Firebase accounts and database rows (Super Admin only) |
| Facilities | `/api/facilities` | CRUD, OSM-backed seeding, address backfill |
| Units | `/api/units` | CRUD, location reporting, status reconciliation |
| Shifts | `/api/shifts` | Start/join/leave/end, admin force-end |
| Incidents | `/api/incidents` | Submission, queue/active views, history, injured-count edits, AI retry, duplicate marking/dismissal |
| Incident media | `/api/incident-media` | Signed upload flow, registration, signed download links, deletion |
| Missions | `/api/missions` | Offer, accept/reject, en-route/arrived/complete, withdraw, dispatcher cancel |
| Hospital transfers | `/api/hospital-transfers` | Recommendation, selection, status progression, vitals |
| Chat | `/api/incidents/{id}/chat` | Scoped to active participants for sending; readable by anyone who was ever a participant |
| First-responder applications | `/api/first-responder-applications` | Atomic submission with required documents, admin review |
| Support | `/api/support-messages` | Server-side rate limiting for the Support page |
| WebSocket | `/ws` | STOMP endpoint; `/topic/incidents`, `/topic/missions`, `/topic/units`, per-transfer vitals topics |

Every mutating endpoint enforces its real authorization in the service layer via `@PreAuthorize` and explicit ownership/role checks — never assume a client-side gate is the actual boundary. `/ws` is intentionally permitted at the Spring Security level; a STOMP handshake carries no Bearer token the way a normal REST call does.

## Environment variables

```
# Firebase Admin SDK
FIREBASE_SERVICE_ACCOUNT_JSON=

# Database (set directly when running outside Docker; docker-compose.yml wires these itself)
SPRING_DATASOURCE_URL=
SPRING_DATASOURCE_USERNAME=
SPRING_DATASOURCE_PASSWORD=

# Supabase Storage (private bucket)
SUPABASE_URL=
SUPABASE_SERVICE_ROLE_KEY=
SUPABASE_STORAGE_BUCKET=

# OpenRouter (AI priority + duplicate detection)
OPENROUTER_API_KEY=
OPENROUTER_MODEL=openai/gpt-oss-20b:free   # swappable without a redeploy if the free-tier lineup shifts

# Chat / incident text / vitals notes encryption at rest
# 256-bit AES key, base64-encoded — generate once with: openssl rand -base64 32
CHAT_ENCRYPTION_KEY=

# Registration email verification (OTP)
# HMAC key for hashing one-time codes, min 32 chars — generate with: openssl rand -base64 48
# Required in prod; the dev profile has a built-in default
OTP_HMAC_SECRET=

# Transactional email (Brevo) — required in every environment; the app won't start without them
BREVO_API_KEY=            # the API key, not the SMTP key
MAIL_FROM_EMAIL=          # must be a sender you've verified in Brevo
```

## Running locally

**Via Docker (recommended):**
```bash
docker compose up backend postgres
```

**Manually:**
```bash
./mvnw spring-boot:run
```
Requires a running Postgres instance and every variable above set in your environment.

## Schema management

Managed through Flyway, versioned under `src/main/resources/db/migration/`. `V1__baseline.sql` captures the schema as it stood when migrations were introduced, generated directly from a real `pg_dump` rather than reconstructed by hand. Hibernate's `ddl-auto` is set to `validate` — it checks that the entity mappings match the real schema and fails loudly on any mismatch, but never silently alters anything again.

Every schema change from here on is a new numbered file (`V2__...`, `V3__...`), applied automatically and identically across every environment the next time each one starts up — no more manually running the same `ALTER TABLE` twice and hoping neither environment was missed.

## Seeding data

- **Facilities** — from the admin Facilities tab, seed by type (Hospital / Fire / Police) directly from OpenStreetMap. Ambulance stations aren't reliably tagged on OSM and are added manually.
- **Addresses** — if a seeded facility comes back with no address (a transient upstream geocoding failure), the same tab's "Fill missing addresses" action retries it. Both the seed and the backfill stop themselves cleanly the moment Nominatim rate-limits the request stream, rather than grinding through the remaining list at a rate that's already being rejected.
- **Test accounts** — `SystemUserInitializer` bootstraps one account per role on startup, each backed by a real test facility, for exercising the full workflow without manually creating every role by hand.

## A few things worth knowing before extending this

- **Never trust `IncidentMedia.url` for display** — the bucket is private; always resolve a fresh signed download URL through `/api/incident-media/{id}/download-url` (or the equivalent facility-application endpoint) rather than caching a stored link.
- **`ProfileCompletionEvaluator` is the single source of truth** for whether an account's profile is complete, including whether it has a password provider attached (checked live against Firebase, since that's not something this database stores). Don't reintroduce a second copy of this logic elsewhere.
- **Phone uniqueness is verified-only, and it's an application-layer rule, not a database constraint.** Two accounts can hold the same unverified number simultaneously; a verified number can never be claimed by a second account. Any new code path that sets a phone number needs to check `existsByPhoneAndPhoneVerifiedTrue` itself — there's no `unique=true` column catching this for you. Email + password registration does exactly this and rejects with `PhoneAlreadyInUseException` (HTTP 422).
- **Deleting an incident is intentionally destructive and `SUPER_ADMIN`-only** — it cascades through missions, chat, and hospital-transfer records. For anything short of genuine cleanup, prefer cancelling or marking a duplicate instead.
- **The Firebase sync tool is for development convenience, not routine operation.** It exists because testing against one shared Firebase project from both a local machine and production can leave accounts on one side with no matching row on the other — it's not meant to run automatically or often.
- **Email registration creates nothing until the email is verified.** `POST /api/auth/register/otp/send` emails a 6-digit code (HMAC-SHA256 hashed at rest, 10-minute TTL, 60-second resend cooldown, 5 attempts); `/otp/verify` returns a one-time token; `/citizen/password` requires that token, then creates the Firebase user (already email-verified) and the database row in one transaction — deleting the Firebase user again if the database write fails. TTLs and limits live under `najda.auth.otp.*` in `application.yaml`.
- **Rate limiting is in-memory, per instance.** `InMemoryRateLimiter` backs the registration endpoints. In production `server.forward-headers-strategy: native` (set in `application-prod.yaml`) makes limits key on the real client IP instead of the proxy's. If the backend is ever scaled horizontally, move the limiter to a shared store.
- **Email delivery is required, not optional.** There is no log-only dev mode: the app refuses to start without `BREVO_API_KEY` and `MAIL_FROM_EMAIL`, and the sender must be verified in Brevo. Without an authenticated sending domain, some messages will land in spam — authenticate a domain in Brevo once you have one; no code change is needed.