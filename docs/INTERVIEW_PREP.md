# ShrutaBodha: Interview Prep

Questions and answers that came up while building ShrutaBodha, grouped by topic.
Each answer is short enough to say out loud in an interview. "In this project" notes tie the answer to real code you can talk about.

**How to revise:** cover the answer, say it out loud, then check. Add new questions to the right section as the project grows.

## Contents
1. [Java](#1-java)
2. [Spring Boot](#2-spring-boot)
3. [JPA and Hibernate](#3-jpa-and-hibernate)
4. [Flyway and database migrations](#4-flyway-and-database-migrations)
5. [PostgreSQL](#5-postgresql)
6. [Docker and Docker Compose](#6-docker-and-docker-compose)
7. [Git and configuration](#7-git-and-configuration)
8. [Troubleshooting habits](#8-troubleshooting-habits)
9. Later sections (added as we reach them): LLD and design patterns, Unit testing, Messaging (RabbitMQ), Security (Spring Security + JWT), System design (HLD), RAG and AI integration, Deployment and performance

---

## 1. Java

**Q: Which Java versions are LTS, and which did you use?**
A: Recent LTS releases are 17, 21 and 25. The project uses Java 21: it's LTS, fully supported by Spring Boot 4, and has records, pattern matching, sealed classes and virtual threads. Java 25 adds scoped values, flexible constructor bodies and a fix for virtual threads pinning inside `synchronized` blocks (from Java 24).

**Q: What are virtual threads and why do they matter for a web backend?**
A: Lightweight threads managed by the JVM instead of the OS. You can have millions of them, so blocking I/O (DB calls, HTTP calls) no longer ties up a scarce platform thread. A thread-per-request server can scale much further without switching to reactive code.

---

## 2. Spring Boot

**Q: How does Spring Boot handle configuration across environments?**
A: Externalized configuration. Values come from `application.yml`, environment variables, command-line args and imported files, with a defined precedence order. Code reads them through placeholders like `${POSTGRES_PORT:5432}`, meaning "use the environment variable, or this default".
*In this project:* `application.yml` imports `../.env` with `spring.config.import: optional:file:../.env[.properties]`, so Docker Compose and Spring Boot share one settings file.

**Q: What does `spring.jpa.open-in-view: false` do and why turn it off?**
A: Open Session in View keeps the Hibernate session (and a DB connection) open for the whole web request, including view rendering, so lazy loading "just works" in controllers. It hides N+1 query problems and holds connections longer than needed. Turning it off forces data loading to happen in the service layer, inside transactions.

**Q: In what order do things start when the app boots with Flyway and JPA?**
A: The datasource is created, Flyway runs pending migrations, then Hibernate initialises and (with `ddl-auto: validate`) checks the schema matches the entities. Then the web server starts.

---

## 3. JPA and Hibernate

**Q: What's the difference between JPA and Hibernate?**
A: JPA is the specification (annotations like `@Entity`, the `EntityManager` API). Hibernate is the most common implementation of it. Spring Data JPA sits on top and generates repositories.

**Q: If JPA writes the queries, why write any SQL at all?**
A: JPA generates DML: `SELECT`, `INSERT`, `UPDATE`, `DELETE` on rows. The schema itself is DDL: `CREATE TABLE`, `ALTER TABLE`, indexes, extensions. Something must create the tables first, and some things Hibernate can't generate at all (e.g. `CREATE EXTENSION vector`, an HNSW vector index, data backfills). That DDL lives in Flyway migrations.

**Q: Doesn't Hibernate create tables automatically?**
A: Only if you set `spring.jpa.hibernate.ddl-auto` to `create`, `create-drop` or `update`. Spring Boot's default is `create-drop` for embedded databases like H2, and `none` otherwise or when Flyway/Liquibase is present.

**Q: What are the `ddl-auto` options and which do you use where?**
A:
- `none`: do nothing.
- `validate`: check entities match tables, fail at startup if not. **Use with Flyway, in every environment.**
- `update`: add missing tables/columns. Never renames or drops, leaves no history. Prototyping only.
- `create`: drop and recreate at startup. Destroys data.
- `create-drop`: like `create`, plus drop at shutdown. Tests with in-memory DBs.

**Q: Why is `ddl-auto: update` risky in production?**
A: It guesses changes from entity classes, can't rename (it adds a new column and leaves the old one), never drops, can't migrate data, and leaves no reviewable or repeatable history. Different environments drift apart.

---

## 4. Flyway and database migrations

**Q: What problem does Flyway solve?**
A: Code is versioned in Git but the database schema changes too. Without a tool, people run SQL by hand and environments drift. Flyway versions schema changes as ordered SQL files in source control and applies them automatically, so every environment ends up identical.

**Q: Git already stores the SQL files. Why do you need Flyway?**
A: Git only stores them. Flyway runs them against each database, in order, exactly once, and records which ones each database has already applied.

**Q: How does Flyway know what to run?**
A: It keeps a `flyway_schema_history` table in each database recording every applied migration (version, description, checksum, time, success). At startup it compares files on disk with that table and runs only the new ones, in version order.

**Q: What's the naming convention?**
A: `V<version>__<description>.sql` with **two** underscores, e.g. `V2__create_videos_table.sql`. Files with the wrong format are ignored. `R__name.sql` files are repeatable migrations, re-run whenever their content changes (useful for views).

**Q: What happens if someone edits a migration that has already run?**
A: On every startup Flyway validates applied migrations by recomputing their checksums. A changed file causes a checksum mismatch and the app refuses to start. Changes are never silently skipped. The fix is to undo the edit and put the change in a new migration.

**Q: Why doesn't Flyway just re-run the edited migration?**
A: The old version has already changed the database and may have data on top. Re-running `CREATE TABLE` would fail or lose data. Only the developer knows how to get from the old state to the new one, so that goes in a new migration (usually `ALTER TABLE`).

**Q: What if a migration fails halfway?**
A: In PostgreSQL, DDL is transactional, so the failed migration is rolled back. Flyway marks it failed and the app won't start until it's fixed. Failing fast beats running on a half-migrated schema.

**Q: Two app instances start at the same time. Do both run migrations?**
A: No. Flyway takes a database lock, so one instance migrates while the other waits, then sees nothing left to do.

**Q: What about adding an older version after newer ones ran, or deleting an applied file?**
A: Both fail validation by default. Out-of-order versions need `outOfOrder=true`; a missing applied file is reported as missing.

**Q: Isn't writing an `ALTER` for every small change during early development tedious?**
A: The immutability rule matters once a migration has reached a database you can't wipe (shared, staging, production). Before that, it's normal to edit the latest migrations and reset the local database. After the first shared or deployed database, migrations are frozen and changes go in new files.
*In this project:* until the first deployment, edit migrations freely and reset with `docker compose down -v` (decision D14).

**Q: What is `flyway repair`?**
A: It removes failed entries from the history table and realigns stored checksums with the files on disk. Use it only when a change was harmless (e.g. a comment), never to hide a real schema change.

**One-line summary:** "Flyway versions schema changes as ordered SQL migrations in source control. At startup it applies only the new ones and checksums applied files to prevent drift, so every environment has an identical schema. Hibernate only validates."

---

## 5. PostgreSQL

**Q: Should the application connect as the `postgres` superuser?**
A: Not in production. Use an owner/migration user for Flyway (can create and alter tables) and a separate least-privilege runtime user for the app (can read and write data, can't drop tables). If the app is compromised, the damage is limited.
*In this project:* locally, `shrutauser` is the superuser created by the Docker image, which is fine for development. The split is planned for deployment (D13).

**Q: What is a Postgres extension, and why pgvector?**
A: Extensions add features to Postgres (`CREATE EXTENSION ...`). pgvector adds a `vector` column type and similarity operators (`<=>` for cosine distance), so embeddings and relational data live in one database instead of adding a separate vector DB.

**Q: How do you avoid clashing with another Postgres on the same machine?**
A: Publish the container on a different host port, e.g. `5433:5432`. Inside the container Postgres still listens on 5432.

---

## 6. Docker and Docker Compose

**Q: What's the difference between an image and a container?**
A: An image is a read-only template (like a class). A container is a running or stopped instance created from it (like an object). Many containers can come from one image.

**Q: Why can't you delete an image that's "in use"?**
A: Every container, even a stopped one, keeps a reference to its image, like a foreign key. Docker won't delete the image while anything references it. Remove the containers first (`docker ps -a` shows all of them, including stopped ones), or force it with `docker rmi -f`.
*In this project:* Docker Desktop's hidden Kubernetes containers held the `registry.k8s.io/*` images. Disabling Kubernetes and force-removing fixed it.

**Q: Why use a named volume for a database container?**
A: Containers are disposable, data shouldn't be. A volume stores data outside the container's lifecycle, so recreating or upgrading the container keeps the data.

**Q: `docker compose down` vs `docker compose down -v`?**
A: `down` removes containers and networks but keeps named volumes, so the data survives. `down -v` also deletes the volumes, wiping the data.

**Q: Inside a container, what does `localhost` mean?**
A: The container itself, not your laptop and not other containers. Containers on the same Compose network reach each other by service name, e.g. `postgres:5432`.

**Q: What does a healthcheck add?**
A: It tells Docker when the service is actually ready (e.g. `pg_isready`), not just started. Other services can wait for `service_healthy` before starting.

**Q: What does `$$` mean in a Compose file?**
A: Compose substitutes `${VAR}` from `.env` when it reads the file. `$$` escapes that and passes a literal `$` through, so the variable is expanded later by the shell inside the container (e.g. in a healthcheck command). `docker compose config` shows the file after substitution.

**Q: Why cap Docker Desktop's memory on Windows?**
A: Docker runs inside a WSL2 VM that can take up to half the RAM by default and is slow to release it. On an 8 GB machine that starves the IDE and browser. `.wslconfig` sets limits (`memory=2GB` here).

---

## 7. Git and configuration

**Q: How do you keep secrets out of Git but still document required settings?**
A: Put real values in `.env`, listed in `.gitignore`. Commit a `.env.example` with the same keys and placeholder values, so a fresh clone shows exactly what to configure.

---

## 8. Troubleshooting habits

**Q: Walk me through how you debug an error you haven't seen before.**
A:
1. Read the error literally and find the key phrase ("in use", "port already allocated", "checksum mismatch").
2. Identify the dependency: what is holding or conflicting with the thing that failed?
3. Don't trust the GUI alone. Use the CLI for the full picture (`docker ps -a`, logs).
4. Remove the cause, then retry.
5. Use force or reset only once you know what you're throwing away.
