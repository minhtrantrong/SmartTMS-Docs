# SmartTMS Multi-Repo Development Guide

# Introduction - Operation Procedure

![Operation Procedure](operation_procedure.png)

## 1. Repository Map

| Repository | Purpose | Default Local Port |
| --- | --- | --- |
| `SmartTMS-BE` | Go + Gin backend API, auth, RBAC, PostgreSQL access, bill upload flow | `8080` |
| `SmartTMS-FE` | Next.js admin web application | `3000` |
| `SmartTMS-AI` | FastAPI OCR microservice used for bill scanning and OCR extraction | `8000` |
| `SmartTMS-Mobile` | Flutter mobile client for operational flows and OCR-assisted bill scan | device/emulator runtime |

## 2. Service Relationships

```text
SmartTMS-FE ----------------------> SmartTMS-BE ----------------------> PostgreSQL
     http://localhost:3000              http://localhost:8080

SmartTMS-Mobile ------------------> SmartTMS-BE
Android emulator uses 10.0.2.2:8080 instead of localhost

SmartTMS-BE ----------------------> SmartTMS-AI
Bill OCR endpoint uses AI_OCR_EXTRACT_URL

SmartTMS-Mobile ------------------> SmartTMS-AI
Mobile OCR screen currently calls the AI service directly
```

## 3. Verified Local Defaults

These values are the current verified development defaults across the codebase.

| Setting | Value |
| --- | --- |
| Backend base URL | `http://localhost:8080` |
| Backend API root | `http://localhost:8080/api` |
| Backend health check | `http://localhost:8080/api/health` |
| Backend Swagger UI | `http://localhost:8080/swagger/index.html` |
| Frontend URL | `http://localhost:3000` |
| AI service URL | `http://localhost:8000` |
| AI health check | `http://localhost:8000/status` |
| AI OCR endpoint | `http://localhost:8000/apis/ocr/extract` |
| Shared backend API key default | `SmartTMS-Dev-API-Key-2025` |
| Seeded development password | `smartTMS@2026` |

Important:

- Some older backend documents still mention `admin123`, but the current seed migration `000002_seed_data.up.sql` states the seeded password is `smartTMS@2026`.
- The mobile app is not fully environment-driven yet. It currently hardcodes backend and AI URLs in source files.

## 4. Prerequisites

Install these tools before starting local development:

- Git
- Go `1.21+`
- PostgreSQL `14+`
- Node.js `20.9+`
- npm
- Python `3.10+`
- Flutter SDK `3.11+`
- Android Studio and an emulator if you are developing the Android mobile app

Recommended Windows tools:

- PowerShell 5.1 or PowerShell 7+
- PostgreSQL command-line tools available in `PATH`

## 5. One-Time Local Setup

### 5.1 SmartTMS-AI

```powershell
cd D:\Tony_learn_to_code\SmartTMS\SmartTMS-AI
py -3.10 -m venv .venv
.\.venv\Scripts\Activate.ps1
pip install -r requirements.txt
Copy-Item .env.example .env
```

Edit `.env` and set:

```env
AI_API_SECRET_KEY=your-local-ai-secret
APP_MODE=development
```

Run the AI service:

```powershell
uvicorn main:app --reload --host 0.0.0.0 --port 8000
```

### 5.2 SmartTMS-BE

```powershell
cd D:\Tony_learn_to_code\SmartTMS\SmartTMS-BE
Copy-Item .env.example .env
go mod download
go mod tidy
go install github.com/swaggo/swag/cmd/swag@latest
go install -tags 'postgres' github.com/golang-migrate/migrate/v4/cmd/migrate@latest
```

Edit `.env` and confirm these fields at minimum:

```env
APP_PORT=8080
DB_HOST=localhost
DB_PORT=5432
DB_USER=smarttms_user
DB_PASSWORD=smarttms_dev_pass
DB_NAME=smarttms_dev
API_KEY=SmartTMS-Dev-API-Key-2025
AI_OCR_EXTRACT_URL=http://localhost:8000/apis/ocr/extract
AI_API_SECRET_KEY=your-local-ai-secret
```

Create the database and apply migrations:

```powershell
psql -U postgres -f reference\setup-database.sql
migrate -path migrations -database "postgres://smarttms_user:smarttms_dev_pass@localhost:5432/smarttms_dev?sslmode=disable" up
swag init -g cmd/api/main.go -o docs
```

Run the backend:

```powershell
go run cmd/api/main.go
```

Notes:

- `make` targets exist in `SmartTMS-BE\Makefile`, but the raw PowerShell commands above are the most reliable Windows-first path.
- Backend bill scanning depends on the AI service being available and the AI secret matching the value configured in `SmartTMS-AI\.env`.

### 5.3 SmartTMS-FE

```powershell
cd D:\Tony_learn_to_code\SmartTMS\SmartTMS-FE
Copy-Item .env.example .env.local
npm install
```

Edit `.env.local` and confirm:

```env
NEXT_PUBLIC_API_BASE_URL=http://localhost:8080/api
NEXT_PUBLIC_API_KEY=SmartTMS-Dev-API-Key-2025
```

Run the frontend:

```powershell
npm run dev
```

Notes:

- The frontend sends `X-API-Key` from `NEXT_PUBLIC_API_KEY` on all API requests.
- The frontend also sends the JWT token from local storage after login.

### 5.4 SmartTMS-Mobile

```powershell
cd D:\Tony_learn_to_code\SmartTMS\SmartTMS-Mobile
flutter pub get
flutter analyze
```

Before running the mobile app, update the current hardcoded local configuration.

Backend config is currently in:

- `lib/core/network/api_client.dart`

AI OCR config is currently in:

- `lib/services/ocr_service.dart`

For Android emulator development, keep these values aligned with your local services:

```dart
// lib/core/network/api_client.dart
baseUrl = "http://10.0.2.2:8080/api"
apiKey = "SmartTMS-Dev-API-Key-2025"

// lib/services/ocr_service.dart
_kOcrBaseUrl = "http://10.0.2.2:8000"
_kApiKey = "your-local-ai-secret"
```

Run the mobile app:

```powershell
flutter run
```

Notes:

- `10.0.2.2` is correct for Android emulator access to your host machine.
- `localhost` will not work from the Android emulator.
- For a physical device, replace `10.0.2.2` with your machine's LAN IP.

## 6. Recommended Startup Order

Use this order when starting the full local stack:

1. Start PostgreSQL.
2. Start `SmartTMS-AI` on port `8000`.
3. Start `SmartTMS-BE` on port `8080`.
4. Start `SmartTMS-FE` on port `3000`.
5. Start `SmartTMS-Mobile` after backend and AI are already reachable.

If you are only working on standard CRUD flows in backend or frontend, the AI service can be skipped.

If you are working on any of these flows, the AI service must be running:

- Bill upload from backend endpoints
- Shipment bill scanning from frontend flows that proxy backend bill upload
- Direct mobile OCR scanning

## 7. Recommended Terminal Layout

Use separate terminals to avoid mixing logs:

1. `SmartTMS-AI`: `uvicorn main:app --reload --host 0.0.0.0 --port 8000`
2. `SmartTMS-BE`: `go run cmd/api/main.go`
3. `SmartTMS-FE`: `npm run dev`
4. `SmartTMS-Mobile`: `flutter run`

## 8. Quick Health Checks

### 8.1 AI service

```powershell
Invoke-RestMethod -Uri "http://localhost:8000/status" -Method Get
```

Expected response:

```json
{
  "status": "ok",
  "message": "SmartTMS AI Engine is running."
}
```

### 8.2 Backend service

```powershell
Invoke-RestMethod -Uri "http://localhost:8080/api/health" -Method Get
Invoke-RestMethod -Uri "http://localhost:8080/api" -Method Get
```

### 8.3 Backend login test

```powershell
$headers = @{
  "Content-Type" = "application/json"
  "X-API-Key" = "SmartTMS-Dev-API-Key-2025"
}

$body = @{
  username = "admin"
  password = "smartTMS@2026"
} | ConvertTo-Json

Invoke-RestMethod -Uri "http://localhost:8080/api/auth/login" -Method Post -Headers $headers -Body $body
```

If login fails with `401` or `403`, verify the API key and password before debugging anything else.

## 9. Shared Development Rules

### 9.1 Keep shared credentials synchronized

The backend API key must stay aligned across:

- `SmartTMS-BE\.env` as `API_KEY`
- `SmartTMS-FE\.env.local` as `NEXT_PUBLIC_API_KEY`
- `SmartTMS-Mobile\lib\core\network\api_client.dart`

The AI OCR secret must stay aligned across:

- `SmartTMS-AI\.env` as `AI_API_SECRET_KEY`
- `SmartTMS-BE\.env` as `AI_API_SECRET_KEY`
- `SmartTMS-Mobile\lib\services\ocr_service.dart`

### 9.2 Treat backend contracts as the source of truth

When changing DTOs, routes, auth behavior, or enum values:

1. Update `SmartTMS-BE` first.
2. Regenerate Swagger docs if route or schema changes are involved.
3. Update `SmartTMS-FE` API calls and types.
4. Update `SmartTMS-Mobile` request and response handling.

### 9.3 Treat database migrations as the source of truth for seed data

If a README conflicts with the migration files, trust the migration files.

Current example:

- Seeded users are created in `SmartTMS-BE\migrations\000002_seed_data.up.sql`
- That migration says the seeded password is `smartTMS@2026`

### 9.4 OCR-specific workflow rules

- Backend endpoint `POST /api/shipments/{id}/bills` only works for shipments in `DELIVERED` status.
- Generic OCR upload is also available through `POST /api/bills/upload`.
- Mobile OCR currently calls the AI service directly instead of going through the backend.

## 10. Daily Commands by Repo

| Repo | Install | Run | Typical Check |
| --- | --- | --- | --- |
| `SmartTMS-AI` | `pip install -r requirements.txt` | `uvicorn main:app --reload --host 0.0.0.0 --port 8000` | `GET /status` |
| `SmartTMS-BE` | `go mod download` | `go run cmd/api/main.go` | `GET /api/health` |
| `SmartTMS-FE` | `npm install` | `npm run dev` | open `http://localhost:3000` |
| `SmartTMS-Mobile` | `flutter pub get` | `flutter run` | login from emulator/device |

Useful repo-specific checks:

```powershell
# Backend tests
cd D:\Tony_learn_to_code\SmartTMS\SmartTMS-BE
go test ./...

# Backend Task 19 RBAC regression
go test ./internal/middleware -run "^TestRequireRole_" -count=1

# Backend Task 19 order-request workflow regression
go test .\internal\service\order_request_service.go .\internal\service\shipment_service.go .\internal\service\shipment_workflow.go .\internal\service\shipment_journal.go .\internal\service\operation_journal_service.go .\internal\service\order_request_service_test.go -run "^TestOrderRequestService_" -count=1

# Frontend lint
cd D:\Tony_learn_to_code\SmartTMS\SmartTMS-FE
npm run lint

# Frontend Task 19 route/workflow regression
npm run test -- src/lib/routePermissions.test.ts src/lib/workflows.test.ts

# Mobile tests
cd D:\Tony_learn_to_code\SmartTMS\SmartTMS-Mobile
flutter test

# Mobile Task 19 role-routing regression
flutter test test/task19_navigation_access_test.dart
```

Task 19 notes:

- Backend migration `000017_task19_hardening_backfills` seeds default allocation rules and reconstructs missing approval-request links for historical trip cost workflows.
- Package-wide `go test ./internal/service` is currently blocked by the existing compile mismatch in `SmartTMS-BE\internal\service\user_service_test.go`.
- `flutter` must be on `PATH` to execute the focused mobile regression command.

## 11. Troubleshooting

### 11.1 Login fails even though the backend is running

Check these in order:

1. Confirm the request includes `X-API-Key`.
2. Confirm the API key matches backend `.env`.
3. Use password `smartTMS@2026` for seeded users.
4. If the database was seeded earlier with older data, rerun the seed or use `reference\update_passwords.sql`.

### 11.2 Frontend shows network or CORS errors

Check these in order:

1. Backend is reachable at `http://localhost:8080/api/health`.
2. `SmartTMS-FE\.env.local` points to `http://localhost:8080/api`.
3. `NEXT_PUBLIC_API_KEY` matches backend `API_KEY`.

### 11.3 Mobile cannot reach backend or AI

Common causes:

- Using `localhost` instead of `10.0.2.2` on Android emulator
- Backend or AI not actually running on the host machine
- Emulator/device and host machine are not on the same reachable network

### 11.4 OCR requests fail

Check these in order:

1. AI service is up at `http://localhost:8000/status`.
2. `AI_API_SECRET_KEY` matches between AI and backend.
3. Mobile OCR hardcoded `_kApiKey` matches the AI secret.
4. Backend `AI_OCR_EXTRACT_URL` points to the active AI instance.

### 11.5 A local port is already in use

```powershell
Get-NetTCPConnection -LocalPort 3000,8080,8000 -ErrorAction SilentlyContinue
```

If needed, identify the owning process and stop it before restarting the service.

## 12. Suggested Next Improvement

The highest-value cleanup for this workspace is to centralize mobile environment configuration so that backend URL, AI URL, backend API key, and AI secret are not hardcoded in source.

That would reduce setup errors and make local, staging, and device testing much easier to manage.