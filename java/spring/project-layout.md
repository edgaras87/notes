

# Project layout (dev-first)

---

## 1) Project tree (dev-first)

```text
my-service/
├─ build.gradle / pom.xml
├─ settings.gradle
├─ .gitignore
├─ README.md
├─ .env                           # local-only env vars for compose/scripts (never commit secrets)
│
├─ src/
│  ├─ main/
│  │  ├─ java/...                 # app code
│  │  ├─ resources/
│  │  │  ├─ application.yml       # shared defaults (safe to commit)
│  │  │  ├─ application-dev.yml   # dev profile overrides (safe to commit)
│  │  │  └─ logback-spring.xml    # logging config: console + file (to ./var/log)
│  └─ test/java/...               # tests
│
├─ config/                        # **externalized config** mounted/loaded in dev
│  ├─ application-local.yml       # machine-specific overrides; not committed
│  └─ secrets.example.yml         # example only (commit), copy to secrets.yml locally
│
├─ var/                           # dev runtime artifacts (git-ignored)
│  ├─ log/                        # app logs in dev
│  │  └─ app.log
│  ├─ data/                       # app writable data (e.g., file uploads)
│  └─ tmp/                        # ephemeral scratch files
│
├─ db/                            # dev database assets
│  ├─ migrations/                 # Flyway/Liquibase scripts (commit)
│  └─ data/                       # local DB files or mounted volume (git-ignored)
│
├─ docker/
│  ├─ Dockerfile
│  └─ compose.yml                 # local stack (app + postgres + pgadmin, etc.)
│
└─ scripts/
   ├─ run-dev.sh                  # runs app with dev profile + external config
   ├─ format.sh
   └─ wait-for-it.sh
```

### Why this layout?

* `src/main/resources/application.yml` holds **safe defaults** (commit).
* `application-dev.yml` contains **dev profile** overrides you’re happy to commit (e.g., use H2 or a local Postgres, verbose logging).
* `config/` is for **externalized, machine-specific** config (don’t commit secrets). Load via `SPRING_CONFIG_ADDITIONAL_LOCATION`.
* `var/log`, `var/data`, `var/tmp` keep runtime files **out of your code** and are **git-ignored**.
* `db/migrations` is versioned; `db/data` is not (local volumes only).

## 2) Minimal .gitignore

```gitignore
# Build
/target/
/build/
/out/

# IDE
.idea/
.project
.classpath
.settings/
*.iml

# Runtime (dev)
/var/
/config/application-local.yml
/config/secrets.yml
/db/data/
/*.log

# OS cruft
.DS_Store
Thumbs.db
```

## 3) Spring config: profiles + externalized location

### `src/main/resources/application.yml` (safe defaults)

```yaml
spring:
  application:
    name: my-service
  datasource:
    url: jdbc:postgresql://localhost:5432/my_service
    username: my_service
    password: changeme     # defaults for dev only; override via external config or env
  jpa:
    hibernate:
      ddl-auto: validate
  flyway:
    locations: classpath:db/migration
logging:
  file:
    name: var/log/app.log    # relative to project root when run from there
  level:
    root: INFO
server:
  port: 8080
```

### `src/main/resources/application-dev.yml`

```yaml
spring:
  config:
    activate:
      on-profile: dev
  jpa:
    show-sql: true
logging:
  level:
    org.springframework: DEBUG
```

### External (not committed): `config/application-local.yml`

```yaml
spring:
  datasource:
    url: jdbc:postgresql://localhost:5432/my_service
    username: my_service
    password: supersecret   # local only
```

Load it in dev with either:

* Env var:
  `SPRING_PROFILES_ACTIVE=dev`
  `SPRING_CONFIG_ADDITIONAL_LOCATION=./config/`
* Or via CLI:
  `./gradlew bootRun --args='--spring.profiles.active=dev --spring.config.additional-location=./config/'`

# 4) Logging to both console and file

### `src/main/resources/logback-spring.xml`

```xml
<configuration scan="true">
  <property name="LOG_FILE" value="var/log/app.log"/>

  <appender name="CONSOLE" class="ch.qos.logback.core.ConsoleAppender">
    <encoder>
      <pattern>%d{HH:mm:ss.SSS} [%thread] %-5level %logger{36} - %msg%n</pattern>
    </encoder>
  </appender>

  <appender name="FILE" class="ch.qos.logback.core.rolling.RollingFileAppender">
    <file>${LOG_FILE}</file>
    <rollingPolicy class="ch.qos.logback.core.rolling.TimeBasedRollingPolicy">
      <fileNamePattern>var/log/app.%d{yyyy-MM-dd}.log</fileNamePattern>
      <maxHistory>7</maxHistory>
    </rollingPolicy>
    <encoder>
      <pattern>%d{yyyy-MM-dd HH:mm:ss.SSS} %-5level %logger - %msg%n</pattern>
    </encoder>
  </appender>

  <root level="INFO">
    <appender-ref ref="CONSOLE"/>
    <appender-ref ref="FILE"/>
  </root>
</configuration>
```

# 5) Docker Compose for local stack

### `docker/compose.yml`

```yaml
version: "3.9"
services:
  db:
    image: postgres:16
    container_name: my_service_db
    environment:
      POSTGRES_DB: my_service
      POSTGRES_USER: my_service
      POSTGRES_PASSWORD: supersecret
    volumes:
      - ../db/data:/var/lib/postgresql/data
    ports:
      - "5432:5432"

  app:
    build:
      context: ..
      dockerfile: docker/Dockerfile
    environment:
      SPRING_PROFILES_ACTIVE: dev
      SPRING_CONFIG_ADDITIONAL_LOCATION: /workspace/config/
    volumes:
      - ..:/workspace           # mount code (hot reload with DevTools)
      - ../var/log:/workspace/var/log
      - ../config:/workspace/config
    ports:
      - "8080:8080"
    depends_on:
      - db
```

# 6) Run scripts

### `scripts/run-dev.sh`

```bash
#!/usr/bin/env bash
set -euo pipefail

export SPRING_PROFILES_ACTIVE=${SPRING_PROFILES_ACTIVE:-dev}
export SPRING_CONFIG_ADDITIONAL_LOCATION=${SPRING_CONFIG_ADDITIONAL_LOCATION:-./config/}

# Gradle
./gradlew bootRun \
  --args="--spring.config.additional-location=${SPRING_CONFIG_ADDITIONAL_LOCATION} --spring.profiles.active=${SPRING_PROFILES_ACTIVE}"
```

# 7) Gradle/Maven snippets

**Gradle (Kotlin DSL) — add Spring Boot DevTools for hot reload**

```kotlin
dependencies {
    developmentOnly("org.springframework.boot:spring-boot-devtools")
    runtimeOnly("org.postgresql:postgresql")
    implementation("org.flywaydb:flyway-core")
    testImplementation("org.springframework.boot:spring-boot-starter-test")
}
```

**Maven**

```xml
<dependencies>
  <dependency>
    <groupId>org.springframework.boot</groupId>
    <artifactId>spring-boot-devtools</artifactId>
    <scope>runtime</scope>
    <optional>true</optional>
  </dependency>
  <dependency>
    <groupId>org.postgresql</groupId>
    <artifactId>postgresql</artifactId>
    <scope>runtime</scope>
  </dependency>
  <dependency>
    <groupId>org.flywaydb</groupId>
    <artifactId>flyway-core</artifactId>
  </dependency>
</dependencies>
```

# 8) Conventions & tips

* **Never commit** anything in `var/`, `config/application-local.yml`, or `db/data/`.
* Keep **sane defaults** in `application.yml`; put **dev overrides** in `application-dev.yml`; put **machine secrets** in `config/application-local.yml`.
* Point file-based resources (uploads, temp exports) to `var/data` and `var/tmp`.
* If you need multiple services later, keep this shape per service and add a `/infra` repo or a root-level `compose.yml` that wires them together.
