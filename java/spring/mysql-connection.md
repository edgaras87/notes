# Spring Boot + MySQL (running in Docker)

👉 Prereq: MySQL container is up (see [Docker setup link]).

---

## 1. Add MySQL driver

**Maven (pom.xml):**

```xml
<dependency>
  <groupId>com.mysql</groupId>
  <artifactId>mysql-connector-j</artifactId>
</dependency>
```

**Gradle (build.gradle):**

```groovy
implementation 'com.mysql:mysql-connector-j'
```

---

## 2. Configure datasource

**`src/main/resources/application.properties`**

```properties
spring.datasource.url=jdbc:mysql://localhost:3306/appdb
spring.datasource.username=appuser
spring.datasource.password=apppass
spring.datasource.driver-class-name=com.mysql.cj.jdbc.Driver

# Choose ONE ddl-auto option (see below)
spring.jpa.hibernate.ddl-auto=update

# Tells Hibernate which SQL dialect to use so it can generate the correct SQL syntax
# (MySQL 8 has features and keywords not in older versions, so use MySQL8Dialect).
spring.jpa.properties.hibernate.dialect=org.hibernate.dialect.MySQL8Dialect

spring.jpa.show-sql=true   # optional, for debugging
```

**Alternative (YAML):**

```yaml
spring:
  datasource:
    url: jdbc:mysql://localhost:3306/appdb
    username: appuser
    password: apppass
    driver-class-name: com.mysql.cj.jdbc.Driver
  jpa:
    hibernate:
      ddl-auto: update   # or validate/create-drop
    show-sql: true
    properties:
      hibernate:
        dialect: org.hibernate.dialect.MySQL8Dialect
```

---

## 3. Schema management options

Spring Boot can handle schema in several ways. Pick **one strategy**:

### A) JPA auto-DDL (fast dev)

```properties
spring.jpa.hibernate.ddl-auto=update
```

* **update** → tries to evolve schema without dropping data (dev-friendly).
* **create-drop** → creates fresh schema on start, drops on shutdown (tests only).
* **create** → drop + create on each startup.
* **validate** → just checks schema matches entities; fails if not.

---

### B) SQL scripts (deterministic, app-driven)

Put files in `src/main/resources/`:

* `schema.sql` → runs first
* `data.sql` → runs after schema

**Example** `schema.sql`:

```sql
CREATE TABLE users (
  id BIGINT PRIMARY KEY AUTO_INCREMENT,
  email VARCHAR(255) NOT NULL UNIQUE
);
```

**Example** `data.sql`:

```sql
INSERT INTO users(email) VALUES ('admin@example.com');
```

Enable init:

```properties
spring.jpa.hibernate.ddl-auto=none
spring.sql.init.mode=always
```

---

### C) Database migrations (best for real projects)

Use **Flyway** or **Liquibase** for versioned migrations.

**Flyway example (recommended):**

```properties
spring.jpa.hibernate.ddl-auto=none
spring.flyway.enabled=true
spring.flyway.locations=classpath:db/migration
```

Place migrations in:

```
src/main/resources/db/migration/V1__init.sql
src/main/resources/db/migration/V2__add_orders.sql
```

---

## 4. Using DB users

* Use `appuser/apppass` (created by Docker setup) in `application.properties`.
* Don’t use `root` for apps; keep root only for DB admin.
* If you add new users (e.g., `reporter`), just switch the credentials.

---

## 5. Testing the connection

* Run `docker compose ps` → MySQL container must show as running.
* In Spring Boot logs you should see:

  ```
  HHH000400: Using dialect: org.hibernate.dialect.MySQL8Dialect
  ...
  HikariPool-1 - Start completed.
  ```
* Try a simple JPA repository or JDBC query to confirm.

---

✅ With this config:

* Spring Boot talks to the MySQL container via `localhost:3306`.
* You can decide whether JPA builds schema, you run SQL scripts, or Flyway manages migrations.
* Everything stays local, consistent, and reproducible.








# Spring Boot × MySQL schema strategy cheatsheet

## TL;DR recommendation

* **Dev (local):** JPA **`update`** or **Flyway**
* **Test (integration):** **`create-drop`** (throwaway DB) or Flyway clean+migrate
* **Prod:** **Flyway/Liquibase** (versioned migrations). Avoid auto-DDL.

---

## Option A — JPA auto-DDL

**What:** Hibernate creates/changes tables from your entities.

**Use when**

* Fast local prototyping.
* Early-stage projects.

**Pros**

* Zero SQL to write.
* Very quick iteration.

**Cons**

* Not deterministic across environments.
* Risky in prod (unexpected alters/drops, subtle diffs).

**Key settings**

```properties
# application-dev.properties
spring.jpa.hibernate.ddl-auto=update     # dev-friendly
spring.jpa.show-sql=true
spring.jpa.properties.hibernate.dialect=org.hibernate.dialect.MySQL8Dialect
```

```properties
# application-test.properties
spring.jpa.hibernate.ddl-auto=create-drop  # ephemeral DB for tests
```

```properties
# application-prod.properties
spring.jpa.hibernate.ddl-auto=validate     # guardrails only
```

**Guardrails**

* Never use `create`/`create-drop` in prod.
* Switch to `validate` before deploying.

---

## Option B — Spring SQL scripts (`schema.sql`, `data.sql`)

**What:** Spring runs `schema.sql` then `data.sql` at startup.

**Use when**

* You want deterministic startup without a migration tool.
* Simple apps or demos.

**Pros**

* Clear order: schema → data.
* Easy to reason about.

**Cons**

* No versioning/history out of the box.
* Harder to evolve schema over time.

**Key settings**

```properties
# disable Hibernate DDL; let SQL files run
spring.jpa.hibernate.ddl-auto=none
spring.sql.init.mode=always     # or "embedded" (defaults for H2)
spring.sql.init.encoding=UTF-8
```

**File layout**

```
src/main/resources/schema.sql
src/main/resources/data.sql
```

**Tip**

* Make inserts idempotent when possible (`ON DUPLICATE KEY UPDATE`) so app restarts don’t explode.

---

## Option C — Migrations (Flyway / Liquibase) ✅

**What:** Versioned, incremental SQL (or YAML/XML) migrations, applied automatically.

**Use when**

* Teamwork, CI/CD, multiple environments.
* Long-lived apps with auditable schema changes.

**Pros**

* Deterministic & repeatable across envs.
* Easy rollback strategies (by version).
* Plays great with CI/CD.

**Cons**

* Slightly more setup.
* Requires discipline (one migration per change).

**Flyway example**

```properties
spring.jpa.hibernate.ddl-auto=none
spring.flyway.enabled=true
spring.flyway.locations=classpath:db/migration
# Optional but handy:
# spring.flyway.clean-disabled=true      # protect prod
# spring.flyway.baseline-on-migrate=true # adopt existing DBs
```

```
src/main/resources/db/migration/
 ├─ V1__init.sql
 ├─ V2__add_orders.sql
 └─ V3__seed_lookup_data.sql
```

**Liquibase (alt)**

```properties
spring.jpa.hibernate.ddl-auto=none
spring.liquibase.change-log=classpath:db/changelog/db.changelog-master.yaml
```

---

## Which to pick by environment

| Environment            | Recommended                                | Why                                                     |
| ---------------------- | ------------------------------------------ | ------------------------------------------------------- |
| **Dev (local)**        | JPA `update` **or** Flyway                 | Speed vs. discipline. Flyway mirrors prod behavior.     |
| **Test (integration)** | `create-drop` **or** Flyway clean+migrate  | Fresh DB per run; predictable tests.                    |
| **Prod**               | **Flyway/Liquibase** + `ddl-auto=validate` | Versioned, controlled changes; fail fast on mismatches. |

---

## Typical profile files (quick copy)

**`application-dev.properties`**

```properties
spring.datasource.url=jdbc:mysql://localhost:3306/appdb
spring.datasource.username=appuser
spring.datasource.password=apppass
spring.jpa.hibernate.ddl-auto=update
spring.jpa.show-sql=true
spring.jpa.properties.hibernate.dialect=org.hibernate.dialect.MySQL8Dialect
```

**`application-test.properties`**

```properties
spring.datasource.url=jdbc:mysql://localhost:3306/appdb
spring.datasource.username=appuser
spring.datasource.password=apppass
spring.jpa.hibernate.ddl-auto=create-drop
spring.jpa.properties.hibernate.dialect=org.hibernate.dialect.MySQL8Dialect
```

**`application-prod.properties`**

```properties
spring.datasource.url=jdbc:mysql://localhost:3306/appdb
spring.datasource.username=${DB_USER}
spring.datasource.password=${DB_PASS}
spring.jpa.hibernate.ddl-auto=validate
spring.jpa.properties.hibernate.dialect=org.hibernate.dialect.MySQL8Dialect
spring.flyway.enabled=true
spring.flyway.locations=classpath:db/migration
```

---

## Common workflows

### Starting from scratch (solo dev)

1. Begin with **JPA `update`**.
2. When entities stabilize, **generate SQL** (from DB or JPA tools) and move to **Flyway V1**.
3. Set `ddl-auto=validate`, build new features with **V2+, V3+** migrations.

### Team/CI from day 1

* Start with **Flyway**.
* Every schema change = new `Vx__description.sql`.
* CI runs app → Flyway migrates → integration tests run.

---

## Troubleshooting tips

* **Connection refused?** Ensure Docker MySQL is up and mapped (`3306:3306`), and your URL matches the port.
* **Timezone warnings?** Add `serverTimezone=UTC` only if you use older drivers; with modern MySQL 8 + `TZ` set in the container, you typically don’t need it.
* **Access denied?** Use app-level user (not root). Confirm grants and password.
* **Migrations failing in prod?** Enable `baseline-on-migrate` to adopt an existing DB; disable `clean` in prod.

---

## Quick decision helper

* Need **speed** today? → JPA `update`.
* Need **consistency** tomorrow? → Flyway/Liquibase.
* Need **repeatable seeds** for demos? → `schema.sql` + `data.sql` or Flyway with a `V2__seed.sql`.
