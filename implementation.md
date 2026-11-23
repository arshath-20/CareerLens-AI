# CareerLens-AI — Implementation Log

## Project Overview
- Project: CareerLens-AI
- Migration: Render/Vercel to AWS
- Target Region: ap-southeast-2
- Documentation: Continuous implementation history

---

## Implementation 01 — Repository audit

### Status
Completed

### Objective
Inspect the repository and establish a verified baseline and migration sequence before implementation.

### Files Created
- `implementation.md` — this chronological migration log.

### Files Modified
- None in the application. This entry documents the audit only.

### Files Removed
- None.

### Changes Implemented
- Audited backend routes and supporting modules, frontend API calls and upload/evaluation flows, deployment workflows, dependencies, tests, environment-file tracking, and current working-tree changes.
- Confirmed the backend is FastAPI with PyMongo and local disk resume uploads; the frontend is React/Vite and uses a centralized Axios client.
- Confirmed existing code still contains Render keep-alive behavior and URLs, wildcard Vercel CORS matching, in-memory application/session fallbacks, and a global `latest` evaluation fallback.
- Confirmed evaluation submission currently runs synchronously. No S3 resume service, presigned upload flow, persistent job/worker mechanism, or Mangum handler was found in the audited files.
- Confirmed `backend/database.py:get_user_evaluation_detail` includes a query branch matching evaluation ID without `user_id`.
- Found an existing frontend S3 deployment workflow on the `production` branch using GitHub OIDC, alongside Render/Vercel deployment and keep-alive workflows. AWS deploy values are sourced from GitHub variables; no AWS deployment was performed or verified in this phase.
- Recorded the migration sequence below. Implementation phases 02–10 remain pending.

### Dependencies
- No dependencies added, removed, or updated in this phase.

### Tests Performed
- None. This phase was a read-only repository audit; no test commands were run.

### Test Results
- Not run.

### Configuration Required
- Future backend deployment configuration will require the AWS and application values specified in the migration brief, including region `ap-southeast-2`, private resume bucket configuration, MongoDB URI, Groq settings, and GitHub token as applicable.
- The existing frontend deployment workflow expects GitHub repository variables `VITE_API_BASE_URL`, `AWS_ROLE_TO_ASSUME`, `AWS_REGION`, and `AWS_S3_BUCKET`. These were identified from workflow configuration; their presence or correctness in GitHub was not checked.
- AWS resource existence, account IDs, IAM policies, and live service configuration were not checked.

### Pending Work
- Backend Lambda compatibility and configuration.
- MongoDB reliability, persistent session data, and ownership enforcement.
- Private S3 resume storage and direct upload integration.
- Asynchronous evaluation jobs and status polling.
- Frontend AWS API and upload/evaluation integration.
- Docker/ECR readiness.
- AWS GitHub Actions CI/CD migration.
- Security and production validation.

### Important Notes
- At audit time, the working tree already contained user changes in `.gitignore`, `backend/services/resume_parser.py`, `backend/test_atlas_integration.py`, `backend/test_mongo.py`, `backend/test_pdf_parser.py`, and `frontend/.gitignore`. These changes were not created or modified by this audit and have not been validated here.
- The audit found tracked frontend environment files (`frontend/.env.development` and `frontend/.env.production`) and untracked/local environment files in the working tree. No secret values are recorded in this log.
- `backend/requirements.txt` did not list Mangum or Boto3 at audit time.
- No AWS deployment or live AWS resource verification was performed.

---

## Implementation Tracking

| Number | Implementation | Status |
| ------ | -------------- | ------ |
| 01 | Repository audit | Completed |
| 02 | Backend Lambda compatibility | Partially Completed |
| 03 | MongoDB reliability and persistence | Partially Completed — Unit Tests Passed; Atlas Integration Pending |
| 04 | S3 resume storage | Pending |
| 05 | Resume upload integration | Pending |
| 06 | Asynchronous evaluation workflow | Pending |
| 07 | Frontend AWS migration | Pending |
| 08 | Docker and ECR readiness | Pending |
| 09 | GitHub Actions CI/CD | Pending |
| 10 | Security and production validation | Pending |


---

## Implementation 02 — Backend Lambda compatibility

### Status
Partially Completed

### Objective
Adapt the FastAPI backend for AWS Lambda through Mangum, retain local Uvicorn operation, remove Render keep-alive behavior, and expose AWS deployment configuration through environment variables.

### Files Created
- None.

### Files Modified
- `backend/main.py` — removed the Render keep-alive worker and startup hook; initialize the existing database connection once during module import and expose `handler = Mangum(app)`; retained the Uvicorn entry point.
- `backend/config.py` — added environment-based `AWS_REGION` and `S3_RESUME_BUCKET`, with defaults `ap-southeast-2` and `careerlens-ai-resumes-unique-suffix`.
- `backend/requirements.txt` — added `mangum>=0.19.0,<1.0.0`.
- `implementation.md` — recorded this phase and updated the tracking table.

### Files Removed
- None.

### Changes Implemented
- Mangum adapts the existing FastAPI application as the Lambda entry point (`main.handler`). Existing routes were not intentionally changed.
- Removed `_keep_alive_ping_worker`, its Render URL/request loop, and the startup hook that created a background thread.
- Moved database initialization to module import. The process-level `database.client` and `database.db` globals are consequently retained across warm invocations in the same Lambda execution environment; this phase does not change database fallback or persistence behavior.
- Kept the `if __name__ == "__main__"` Uvicorn development entry point and configured port.
- Added environment-backed AWS region and resume bucket configuration without adding AWS resource IDs or deployment URLs.

### Dependencies
- Added `mangum>=0.19.0,<1.0.0`.
- No other dependency changes.

### Tests Performed
- python -m compileall -q . from backend.
- python -c "import main; print(callable(main.handler))" from backend.
- FastAPI application import and Mangum handler availability check.
- MongoDB connection during module import.
- Static search for remaining Render keep-alive references and for the Mangum handler/configuration declarations.

### Test Results
- python -m compileall -q . — PASSED.
- python -c "import main; print(callable(main.handler))" — PASSED; the Mangum handler returned True.
- FastAPI application imported successfully.
- MongoDB connection succeeded during module import.
- Static search confirmed no Render keep-alive code remains in backend/main.py.
- A representative API Gateway/Lambda event compatibility test is still outstanding. No real AWS Lambda deployment was tested. Implementation 02 remains Partially Completed for this reason.
### Configuration Required
- Set `AWS_REGION` in the Lambda environment; the code default is `ap-southeast-2`.
- Set `S3_RESUME_BUCKET` when the later S3 phase is implemented; the current code default is `careerlens-ai-resumes-unique-suffix`.
- Continue configuring `GROQ_API_KEY`, optional `GROQ_MODEL`, `MONGO_URI`, and optional `GITHUB_TOKEN` through environment variables in deployment. Do not store actual secret values in this log.
- No additional dependency setup is pending for the reported runtime checks. No AWS deployment was performed.

### Pending Work
- Validate a representative API Gateway/Lambda event against the Mangum handler.
- MongoDB reliability, persistence, ownership, and fallback behavior remain for Implementation 03.
- S3 storage and asynchronous evaluation remain in their later phases.

### Important Notes
- `backend/database.py` was not modified. Its current `connect_db()` behavior, including in-memory fallback, remains unchanged and is explicitly out of scope for this phase; module-level initialization only ensures the existing process globals are reused when available.
- The existing `.env` loader remains for local development. Environment variables supplied by Lambda remain supported because `load_dotenv` does not override already-set process variables by default.
- No S3 storage, asynchronous evaluation, frontend migration, or AWS deployment was performed.




---

## Implementation 03 — MongoDB Reliability & Persistence

### Status
Partially Completed — Unit Tests Passed; Atlas Integration Pending

### Objective
Make MongoDB connections lazy and reusable, fail safely when durable production storage is unavailable, secure owner-scoped evaluation retrieval, and persist resume sessions and evaluation context beyond a Lambda execution environment.

### Files Created
- backend/test_database_persistence.py — mocked unit tests for connection reuse, production fail-closed behavior, persistence, and ownership isolation.

### Files Modified
- backend/database.py — added lazy locked connection initialization, timeouts, connection reuse and retry backoff; production failure handling; scoped MongoDB persistence and indexes for resumes, evaluations, users, and conversations; removed in-memory fallbacks after operation failures; fixed evaluation ownership queries.
- backend/main.py — removed import-time database initialization and active_sessions; persisted resume sessions and evaluations; loads evaluation context from MongoDB using the authenticated owner for JD matching, roadmaps, chat, history, and evaluation detail; removed global latest context; added controlled 503 database errors.
- backend/config.py — added APP_ENV configuration for strict production behavior.
- backend/.env.example — documented local development memory fallback and production behavior.
- implementation.md — recorded the actual phase status and results.

### Files Removed
- None.

### Changes Implemented
- MongoDB initialization occurs on first database use rather than during application import. A single process-scoped MongoClient is reused across requests and warm Lambda invocations; it is not closed per request.
- Set serverSelectionTimeoutMS=3000, connectTimeoutMS=3000, and socketTimeoutMS=5000. Failed production connection attempts are retried no more often than once per five seconds.
- Production mode is enabled by APP_ENV=production/prod or Lambda's AWS_LAMBDA_FUNCTION_NAME. Missing or unavailable MongoDB raises DatabaseUnavailable, mapped to a generic HTTP 503 response. Configured MongoDB operation failures also raise instead of silently writing to memory.
- Local development retains the prior in-memory behavior when MongoDB is not configured or its initial connection fails, with an explicit warning. An operation failure after a successful MongoDB connection is surfaced rather than falling back.
- Evaluation detail queries always include the owning user_id; missing identity and invalid IDs return no record. The latest identifier resolves to the latest evaluation for that authenticated user only.
- Added MongoDB persistence for resume session ID, owner, extracted text, metadata, filename, optional S3 object key, optional analysis, and UTC creation/update timestamps. PDF bytes are not stored.
- Evaluation records retain the complete analysis context. JD match and roadmap updates are persisted; chat history is queried and written with both authenticated user and evaluation IDs.
- Removed all active_sessions use in backend/main.py, including the global latest cache. The legacy evaluation-history route now returns the authenticated user's records.
- Added indexes for unique user email/token, user/date evaluation listing, user/evaluation ownership, user/evaluation/timestamp conversation access, and owner-scoped resume lookup/update.

### Dependencies
- No dependencies added, removed, or updated.

### Tests Performed
- python -m compileall -q . from backend, using Python 3.12.10.
- python -m unittest -v test_database_persistence from backend, using Python 3.12.10.
- rg source inspection for active_sessions and RENDER_EXTERNAL_URL in backend/main.py.
- git diff --check for the phase application files.

### Test Results
- Compilation completed successfully.
- All 11 database persistence unit tests passed with no failures or errors.
- Expected MongoDB OSError messages appeared in tests simulating connection and write failures.
- Static source search confirmed active_sessions and RENDER_EXTERNAL_URL are absent from backend/main.py.
- git diff --check passed.
- Atlas integration tests were not run and remain pending. End-to-end validation and local filesystem resume migration to private S3 remain pending. No AWS deployment was performed.

### Configuration Required
- Set APP_ENV=production for production deployments outside Lambda. Lambda is identified through its runtime environment and automatically uses strict database behavior.
- Configure MONGO_URI in the production runtime. Never record its value in this log.
- Local development can keep APP_ENV=development; if MongoDB is not configured or unavailable during initial connection, the historical in-memory mode remains enabled and is logged.
- No AWS deployment was performed.

### Pending Work
- Run controlled MongoDB Atlas integration checks when a safe test database is available.
- Validate the existing application flows end to end.
- Review remaining local filesystem resume PDF handling; the PDF is still written under backend/uploads. Move it to private S3 in Implementations 04/05.
- Implementations 04–10 remain pending.

### Important Notes
- memory_store remains in backend/database.py only for local development when initial MongoDB setup is absent/unavailable; production never selects that fallback. This preserves the documented local behavior in backend/.env.example.
- The new unit tests use mocked collections and a mocked MongoClient; they do not require or print production MongoDB credentials.
- The resume parser and frontend/workflow files were not modified in Implementation 03. No S3 storage, asynchronous evaluation, frontend migration, or AWS deployment was performed.
- Implementation 02 remains documented as Partially Completed; its Lambda event compatibility test is still outstanding.

