# ShrutaBodha: Progress

Last updated: 2026-11-23

## Current phase
Phase 0 → Phase 1 foundation: Postgres + pgvector in Docker, app connected

## Done
- [x] Reviewed the ChatGPT blueprint and agreed the revised plan (see PROJECT_NOTES.md)
- [x] Checked laptop specs (i5-1135G7, 8 GB RAM, no dedicated GPU)
- [x] Git installed
- [x] Docker Desktop installed
- [x] Java 17 installed
- [x] JDK 21 installed (Microsoft Build of OpenJDK 21.0.13) and active, JAVA_HOME set. Considered Java 25, stayed on 21 (D9 unchanged).
- [x] Chose IntelliJ IDEA for the Java backend, VS Code for the rest (D12)
- [x] IntelliJ IDEA installed, Project SDK set to 21
- [x] `.wslconfig` created (memory=2GB, processors=2, swap=2GB)
- [x] Docker Desktop: built-in Kubernetes disabled, old Kubernetes images cleaned up, engine running
- [x] Private GitHub repo `ShrutaBodha` created and cloned to `C:\projects\ShrutaBodha`
- [x] Spring Boot 4.2.0 project generated into `backend/` (Maven, Java 21, Web MVC, JPA, PostgreSQL, Validation, Flyway, Lombok)
- [x] Project opened in IntelliJ
- [x] `docs/INTERVIEW_PREP.md` created in the repo (D15)
- [x] Context files mirrored into repo `docs/` and INTERVIEW_PREP.md added to the Claude Project; both kept in sync (PROJECT_NOTES 8b)

## To do now
- [ ] Build Project in IntelliJ (☰ → Build → Build Project) succeeds
- [x] Root `.gitignore` (`.env`, `.idea/`, `*.iml`)
- [x] `docker-compose.yml` (repo root) with Postgres + pgvector (`pgvector/pgvector:pg17`), named volume, healthcheck
- [x] `.env` (git-ignored, POSTGRES_PORT=5433, user `shrutauser`) and `.env.example` (committed, placeholder password)
- [x] `application.properties` replaced with `application.yml` reading DB settings from env vars, `ddl-auto: validate`
- [x] First Flyway migration `V1__enable_pgvector.sql`
- [ ] IntelliJ run configuration: working directory = `C:\projects\ShrutaBodha\backend`
- [x] `docker compose up -d` works: `shrutabodha-postgres` healthy on 5433 (2026-11-23)
- [ ] **Next session starts here:** open Docker Desktop → `docker compose up -d` → run `BackendApplication` → confirm Flyway applied V1 and `vector` extension exists (`\dx`)
- [ ] First commit and push (including `docs/`)

## Next
- [ ] Phase 1 lesson: Spring Boot skeleton and project structure
- [ ] Phase 2: `Video` entity, repository, upload API

## Open items
- Claude subscription: set up the Claude Project on the free plan first, buy Pro when daily study begins
- LLM: Ollama locally, revisit hosted API for deployment
- Optional cleanup: several old JDKs installed (Microsoft 11, Zulu 17, Oracle 17). Uninstall unused ones later.
- Optional: VS Code Docker and Python extensions (needed from Phase 5/6)

## Session log
| Date | Topic | Notes |
|---|---|---|
| 2026-11-21 | Planning | Plan, stack and environment decided. Project files drafted. |
| 2026-11-21 | Phase 0 setup | JDK 21 installed and active. Prefers Command Prompt (cmd) over PowerShell: give cmd commands. Switched to IntelliJ for Java (D12). |
| 2026-11-22 | Phase 0 setup | Docker memory capped, Docker Kubernetes disabled and leftover images removed (lesson: images can't be deleted while containers, even hidden or stopped ones, reference them). Repo cloned, Spring Boot skeleton opened in IntelliJ. |
| 2026-11-22/23 | Flyway and config concepts | Covered Flyway, DDL vs DML, ddl-auto, Compose `$$` escaping, superuser vs app user. Postgres on port 5433 (D13), migration editing policy (D14), interview prep file (D15). |
| 2026-11-23 | Postgres up | Config files reviewed (user `shrutauser`). Postgres + pgvector container healthy on 5433. First commit before running the app; app run + Flyway check pending. Lesson: Docker CLI is a client, needs Docker Desktop's engine running. |
