# ShrutaBodha: Decisions Log

Format: date, decision, why, alternatives, revisit when.

## D1. Start from scratch with a revised stack (2026-11-21)
- **Decision:** Rebuild ShrutaBodha from scratch, keeping the blueprint's idea but adjusting the stack and build order.
- **Why:** The blueprint has many moving parts (Spring Boot, Python worker, FastAPI, RabbitMQ, Postgres + pgvector, LocalStack, Ollama, React). Starting with all of them would mean weeks of infrastructure before real Spring Boot practice.
- **Revisit when:** a phase feels too slow or too fast.

## D2. Local first, Codespaces as fallback (2026-11-21)
- **Decision:** Develop on the laptop. Move a piece to GitHub Codespaces only if the laptop struggles.
- **Why:** The laptop (i5-1135G7, 8 GB RAM) can handle one thing at a time. Codespaces free hours are limited (about 60 hours a month on 2 cores).
- **Alternatives:** Codespaces only; a rented cloud VM.
- **Revisit when:** Docker or Spring Boot freezes the laptop, or Whisper jobs block everything else.

## D3. Docker memory cap (2026-11-21)
- **Decision:** Cap WSL2 memory at 2 GB through `.wslconfig`.
- **Why:** Docker Desktop's VM can otherwise take most of the 8 GB.

## D4. Modular monolith first (2026-11-21)
- **Decision:** One Spring Boot app with clear modules. Split into services only if there is a reason.
- **Why:** Splitting too early is a common beginner mistake and costs memory and time.
- **Revisit when:** the domain boundaries are clear and messaging is in place.

## D5. Local, free AI components behind interfaces (2026-11-21)
- **Decision:** faster-whisper (small model), MiniLM embeddings and an Ollama small model, all local. Each behind an interface (`TranscriptionProvider`, `EmbeddingProvider`, `LlmClient`).
- **Why:** Whisper and MiniLM are free to run. Ollama avoids API costs. Interfaces give Strategy and Dependency Inversion practice and make swapping easy.
- **Trade-off:** Without a GPU, transcription is slow and the small LLM gives modest answers.
- **Revisit when:** answer quality needs improving, or deployment to a free cloud tier needs a hosted LLM.

## D6. Storage behind an interface (2026-11-21)
- **Decision:** Local disk first behind `StorageService`, MinIO (S3-compatible) later.
- **Why:** Simpler than LocalStack, and a good Strategy example.

## D7. RabbitMQ after the core (2026-11-21)
- **Decision:** Build upload, status and a stub worker first, add RabbitMQ in phase 5.
- **Why:** Learn the flow first, then messaging as a separate interview topic.

## D8. Testing and performance testing later (2026-11-21)
- **Decision:** Unit tests (JUnit 5, Mockito, JaCoCo) from the start. Integration tests (Testcontainers) and load testing come in phase 11, after deployment to a free cloud tier.
- **Why:** Keeps early phases light on the laptop.
- **Note:** Free tiers are small, so results show relative bottlenecks, not production numbers.

## D9. Java 21 (2026-11-21)
- **Decision:** Use JDK 21 for the project. JDK 17 can stay installed.
- **Why:** LTS, and useful for interviews (records, virtual threads, pattern matching, sealed classes).
- **Note:** Java 25 (newer LTS, recommended by Spring Boot 4) was considered on 2026-11-21. Stayed on 21 since it was already installed and fully supported. Installed build: Microsoft Build of OpenJDK 21.0.13.
- **Revisit when:** a feature or library needs 25.

## D10. Spring Security and JWT added (2026-11-21)
- **Decision:** Add Spring Security with JWT in phase 8, even though the blueprint marked auth out of scope.
- **Why:** It is a common interview topic.

## D11. Claude workflow (2026-11-21)
- **Decision:** One Claude Project ("ShrutaBodha") holding these three files. One chat per topic.
- **Why:** Chats inside a project do not see each other. Only the instructions and files carry across, so these files are the project's memory.

## D12. IntelliJ IDEA for the Java backend (2026-11-21)
- **Decision:** Use IntelliJ IDEA (free edition) for `backend/`. Keep VS Code for Python, Docker Compose, config files and the frontend.
- **Why:** IntelliJ is the stronger Java IDE (refactoring, inspections, debugging) and the industry standard for Java teams.
- **Trade-off:** Uses more RAM than VS Code. Deep Spring tooling is in the paid Ultimate edition, which this project does not need.
- **Alternatives:** VS Code with the Java and Spring extensions for everything.
- **Rule:** Avoid running IntelliJ and VS Code with the Java extension at the same time on the 8 GB laptop.
- **Revisit when:** the laptop struggles for memory.

## D13. Postgres on host port 5434, app user `shrutauser` (2026-11-22, port changed 2026-11-24)
- **Decision:** Publish the Docker Postgres on host port 5434 (container stays 5432). Database `shrutadb`, user `shrutauser`.
- **Host:** the app connects to `127.0.0.1` (`POSTGRES_HOST=127.0.0.1`). Explicit IPv4 avoids any `localhost` → IPv6 `::1` ambiguity on Windows, where `wslrelay.exe` also listens on `[::1]:<port>` for Docker's ports.
- **History (2026-11-24):** First app run failed with "password authentication failed". We first suspected `wslrelay.exe` on `[::1]` (seen in `netstat`), moved 5433 → 5434 and switched to `127.0.0.1`, but it still failed. The Postgres container log proved the login *was* reaching our container, so Spring was sending the wrong password. **Real root cause:** IntelliJ's temporary run config used the repo root as working directory, so `optional:file:../.env` pointed to `C:\projects\.env` (missing), was silently skipped, and the unresolved `${POSTGRES_PASSWORD}` was sent literally. **Fix:** `spring.config.import: optional:file:.env[.properties],optional:file:../.env[.properties]` so `.env` is found from either folder. Port 5434 and `127.0.0.1` kept (harmless, explicit).
- **Why:** Avoids clashing with a natively installed Postgres on 5432. A project-specific user name makes it obvious which server you're connected to.
- **Note:** In the official Postgres image, `POSTGRES_USER` is created as a superuser, so `shrutauser` can run `CREATE EXTENSION vector`. Fine for local dev.
- **Revisit when:** deploying. Then split into a migration/owner user (used by Flyway) and a least-privilege runtime user for the app.

## D14. Flyway owns the schema; migrations editable until first deployment (2026-11-23)
- **Decision:** All schema changes go through Flyway migrations. Hibernate runs with `ddl-auto: validate`. Until the first deployment, migrations may be edited and the local DB reset with `docker compose down -v`. From the first deployed or shared database on, applied migrations are frozen and every change goes in a new file.
- **Why:** Avoids piling up tiny `ALTER` migrations while the schema is still being designed, while keeping the production-grade habit (immutable migrations) once it matters.
- **Alternatives:** `ddl-auto: update` during development (rejected: hides what changed and builds bad habits); strict immutability from day one (rejected: needless churn while learning).
- **Revisit when:** the first deployment happens (switch to strict mode).

## D15. Interview prep file (2026-11-23)
- **Decision:** Keep one revision file, `docs/INTERVIEW_PREP.md`, in the repo, organised by topic. Add Q&As whenever a topic with an interview angle comes up (rule in PROJECT_NOTES.md section 8a).
- **Why:** Turns everything learned while building into revision material in one place.
