# ShrutaBodha: Project Notes

Last updated: 2026-11-23

## 1. What this is
A learning project built from scratch. A video-to-knowledge app: upload a video, transcribe it, chunk and embed the transcript, then ask questions about it through a RAG chatbot.

The detailed original idea, data model and API ideas live in `ShrutaBodha_From_Scratch_Build_Blueprint.md` (written earlier in ChatGPT). This file records the decisions we made after reviewing it. Where the two differ, this file wins.

## 2. Learning goals
- Spring Boot topics that come up in interviews
- LLD: SOLID, design patterns implemented in real code, clean code
- HLD: architecture, queues, scaling, failure handling
- Unit testing (JUnit 5, Mockito, JaCoCo)
- AI integration: Python worker, RAG, possibly LangGraph later
- Deployment to a free cloud tier, then performance testing

## 3. About me
- Comfortable with Java, some Spring Boot exposure
- Windows 11 laptop: HP, Intel i5-1135G7 (4 cores / 8 threads), 8 GB RAM, no dedicated GPU, 512 GB SSD
- JDK 21 active (Microsoft Build of OpenJDK 21.0.13). Older JDKs (11, 17) still installed.
- Git and Docker Desktop already installed
- Study time: a few hours daily, more on weekends, possibly more later
- Tools: IntelliJ IDEA (free edition) for the Java backend, VS Code for Python, Docker Compose, config and frontend. Claude Pro planned once daily study starts.
- Terminal: prefers Command Prompt (cmd) over PowerShell. Give cmd commands.
- Learning style: wants the "why" explained simply, with analogies and concrete examples from this project.

## 4. Environment approach
- **Local first.** Start on the laptop. Move a piece to GitHub Codespaces only if the laptop struggles.
- Codespaces (free personal account): 120 core hours a month, about 60 hours on a 2-core machine. It is a fallback, so stop the codespace when done.
- Docker Desktop memory capped through `C:\Users\<name>\.wslconfig` (start with memory=2GB, processors=2, swap=2GB). Docker Desktop's built-in Kubernetes is disabled.
- Postgres runs in Docker on host port 5433 (D13), nothing installed natively. RabbitMQ is added later.
- Never run the whole stack and a Whisper job together.
- Avoid running IntelliJ and VS Code's Java extension at the same time (RAM).
- Everything is driven by Git, `.env` files and `docker-compose.yml`, so any piece can move to Codespaces without code changes.
- Office laptop: only through Codespaces in the browser, personal GitHub account, no company material, check company policy.

## 4a. Local tooling checklist
- IntelliJ IDEA (free edition), Project SDK set to JDK 21, Lombok annotation processing enabled
- VS Code with: Docker, Python (Java extensions not needed while IntelliJ is used)
- JDK 21, Git for Windows. Maven comes through the Maven Wrapper (`mvnw`).
- Spring Boot 4.2.0 (from start.spring.io)
- Repo location: `C:\projects\ShrutaBodha` (backend in `C:\projects\ShrutaBodha\backend`). Claude can access this folder through the desktop app when it is connected to the chat.
- Verify (cmd): `java -version`, `git --version`, `docker --version`, `mvnw -version`

## 5. Architecture (target)
- **Spring Boot backend**, modular monolith first: upload, validation, metadata, status tracking, chat orchestration
- **PostgreSQL + pgvector** for metadata, transcripts and embeddings. Schema through Flyway migrations.
- **RabbitMQ** for background jobs, added after the core works
- **Python worker + FastAPI service** for FFmpeg, transcription, chunking, embeddings and RAG
- **Storage** on local disk behind a `StorageService` interface first, later MinIO (S3-compatible)
- **Frontend** minimal and last. Swagger UI until then.
- No Kubernetes. Deployment is Docker Compose on one small VM or free-tier managed services.

## 6. AI components

| Part | Choice | Cost |
|---|---|---|
| Transcription | faster-whisper, small model, local, CPU | Free |
| Embeddings | MiniLM (Sentence Transformers), local | Free |
| LLM answers | Ollama, small quantized model, local | Free |
| Fallback | Hosted API behind the same interface | Free tier or small cost |

- Whisper on CPU is slow. Use small models and 2 to 5 minute test clips. Store each transcript once and reuse it.
- Every provider sits behind an interface (`TranscriptionProvider`, `EmbeddingProvider`, `LlmClient`) with model name and URL in config.
- LLM answer quality will be improved later (better model, prompts, reranking or a hosted API).
- Ollama is installed directly on Windows (not in Docker) to avoid Docker memory overhead.

## 7. Build order
1. Foundation: GitHub repo, README, `.gitignore`, `.env.example`, Spring Boot skeleton, `docker-compose.yml` with Postgres and pgvector
2. Core backend: `Video` entity, repository, upload API, validation, exception handling, Flyway, status endpoint
3. LLD in code: State (video status lifecycle), Strategy (storage, transcription, LLM), Factory/Builder (DTOs, messages), Template Method (processing pipeline), SOLID throughout
4. Unit tests: JUnit 5, Mockito, JaCoCo
5. Messaging: RabbitMQ producer, Python consumer, retries, idempotency
6. Python worker: audio extraction, Whisper, chunking with timestamps, embeddings
7. RAG: retrieval, then LLM, then FastAPI endpoint and Spring Boot chat endpoint
8. Hardening: Spring Security with JWT, failure handling, structured logging
9. Minimal frontend
10. Deployment: Docker Compose, GitHub Actions CI/CD, free cloud tier
11. Performance testing on the deployed app (k6 or JMeter), plus Testcontainers integration tests, with a short findings report

Phases 1 to 4 need only Spring Boot and Postgres.

## 8. How each session works
1. Design first: short LLD or HLD discussion with a diagram and trade-offs
2. I write the code (IntelliJ for Java, VS Code for the rest), then Claude reviews it
3. Claude explains the why behind each decision and the interview angle
4. End with a short interview-style quiz
5. Update the context files (PROGRESS.md, DECISIONS.md, PROJECT_NOTES.md, INTERVIEW_PREP.md) in **both** places (see 8b), then commit to Git

## 8a. Rule: interview prep file
- `INTERVIEW_PREP.md` is the single revision file for interview questions and answers.
- **Whenever a topic, decision, bug or concept with an interview angle comes up, add a Q&A to it** in the right section (Java, Spring Boot, JPA and Hibernate, Flyway, PostgreSQL, Docker, Git and configuration, Troubleshooting, and later: LLD and design patterns, Unit testing, Messaging, Security, System design, RAG and AI, Deployment and performance). Create a new section when a new topic starts.
- Keep answers short enough to say out loud. Add an "In this project" note where the answer maps to real code or a real problem we hit.
- Don't duplicate questions: refine an existing answer instead.
- If the repo folder is connected to the chat, Claude edits the file directly. If not, Claude gives the new Q&As in the reply for the user to paste in.
- The file is committed with the code.

## 8b. Rule: context files live in two places, kept in sync
- **Context files:** PROJECT_NOTES.md, DECISIONS.md, PROGRESS.md, INTERVIEW_PREP.md and the blueprint.
- **Place 1, the Claude Project "SrutaBodha":** every chat in the project can read these, so this copy is the **source of truth**.
- **Place 2, the repo folder `C:\projects\ShrutaBodha\docs\`:** a mirror, committed with the code, so the context travels with the project.
- **Whenever Claude changes a context file, it updates both places in the same turn**, without being asked. At the start of a session, Claude reads the Project copies.
- If the repo folder isn't connected to the chat, Claude updates the Project copy and says the repo mirror needs syncing next time the folder is connected.
- If the user edits a repo copy by hand, tell Claude so it can copy the change back to the Project.
- Note: the Project's doc list shows each file's *created* date, not when it was last edited, so an updated file can look old there.

## 9. Out of scope for now
- Load and performance testing (phase 11)
- Integration tests with Testcontainers (phase 11)
- Kubernetes
- Fancy frontend
