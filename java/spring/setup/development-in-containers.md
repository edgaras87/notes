# 🚀 Spring Boot Development in Containers (Podman on Fedora)

---

## 1. Why develop inside containers?

* **Reproducible & clean** → Same JDK, Maven, OS libs as prod. No “works on my machine.”
* **No host pollution** → No global JDK/Maven installs, no messy `~/.m2`, no system packages.
* **Faster onboarding** → Teammates just clone repo, run `podman compose up`, and it works.
* **Easy reset** → `podman compose down --volumes` wipes everything clean.
* **Prod-like testing** → Multi-stage Dockerfile ensures runtime is identical to deploy image.

---

## 2. Core setup

### Dev Compose (`docker-compose.dev.yml`)

* Runs Spring app via `maven:temurin` image.
* Mounts your project source (`.:/workspace:Z`) → hot reload with Spring DevTools.
* Exposes:

    * `8080` → app HTTP port
    * `5005` → JVM remote debug
* Uses **named Maven repo volume** to cache dependencies without polluting host.

### Remote Debugging

* The app JVM runs inside container.
* IntelliJ attaches over `localhost:5005`.
* This is the standard way to debug any “remote” JVM (container, VM, Kubernetes, etc.).

---

## 3. Keeping host clean

* **Source code**: mounted into container with `:Z` (SELinux relabel).
* **Dependencies**: Maven repo isolated in named volume (`maven-repo`).
* **Configs/env**: use `environment:` in Compose or `.env` files → no leaking into host.
* **User IDs**: run container with `user: "${UID}:${GID}"` to avoid root-owned files on host.

Reset at any time:

```bash
podman compose -f docker-compose.dev.yml down --volumes
```

---

## 4. Workflow

1. **Start dev container**

   ```bash
   podman compose -f docker-compose.dev.yml up
   ```
2. **Open in IntelliJ** → attach **Remote JVM Debug** at `localhost:5005`.
3. **Edit code on host** → Spring DevTools reloads inside container.
4. **Test via browser** at [http://localhost:8080](http://localhost:8080).
5. **Stop/reset** with `podman compose down`.

Optional: wrap commands in a `Makefile` so you can do `make run`, `make test`, `make clean`.

---

## 5. Prod-like image

* **Multi-stage Dockerfile** builds `.jar` in Maven image, copies it into slim JRE image.
* Runtime container runs as **non-root user**.
* Useful for testing “real deployment” locally, CI/CD pipelines, or production itself.

---

## 6. Extensions

* **Databases**: Add Postgres or others as services in Compose (`depends_on: [db]`).
* **Multiple profiles**: `SPRING_PROFILES_ACTIVE=local | dev | prod` → separate configs.
* **Secrets**: For local dev, keep in `.env`; for production, use Podman secrets/K8s secrets.
* **Resource tuning**: JVM flags (`JAVA_TOOL_OPTIONS=-XX:MaxRAMPercentage=75.0`) if needed.

---

## 7. Fedora / Podman specifics

* Use `:Z` on **all bind mounts** → required for SELinux.
* **Rootless Podman** works fine for ports ≥1024 (8080/5005 safe).
* Shell access (when needed):

  ```bash
  podman compose -f docker-compose.dev.yml exec dev sh
  ```
* Increase inotify watches if hot reload doesn’t pick up changes:

  ```bash
  echo fs.inotify.max_user_watches=524288 | sudo tee -a /etc/sysctl.conf
  sudo sysctl -p
  ```

---

# ✅ TL;DR

* **Run dev in container**, edit locally → hot reload + remote debug.
* **Host stays clean** → Maven repo, JDK, dependencies live inside containers/volumes.
* **Debug like remote app** → attach IntelliJ to `localhost:5005`.
* **Reproducible prod builds** → Multi-stage Dockerfile.
* **Isolation + flexibility** → configs in env, volumes for data, no root pollution.

This gives you a **clean, repeatable, “as-it-runs-in-prod” dev setup**, without cluttering your Fedora host, while still keeping a smooth inner loop for coding + debugging.

---

```java




```

---

# 🐳 Spring Boot Development with Podman (Fedora-friendly Guide)

## 1. Why use Podman for dev?

* **Native on Fedora** → runs rootless, integrates with SELinux.
* **Clean host** → no global JDK/Maven installs, no clutter in `~/.m2`.
* **Same as prod** → your app runs inside the same Java/Maven image you’ll use for deployment.
* **Reproducible** → Dockerfile + Compose = self-documenting environment.
* **Fast feedback** → bind mounts + Spring DevTools = hot reload.
* **Debug-friendly** → remote JVM debug works seamlessly.

---

## 2. Key building blocks

### Dev container (`docker-compose.dev.yml`)

* Runs app using **maven + JDK image**.
* Mounts your project source so changes are seen instantly.
* Caches Maven dependencies in a **named volume**.
* Exposes **8080 (HTTP)** and **5005 (debug)**.
* Example:

```yaml
services:
  dev:
    image: maven:3.9-eclipse-temurin-21
    working_dir: /workspace
    command: mvn spring-boot:run
    ports:
      - "8080:8080"
      - "5005:5005"
    environment:
      - MAVEN_OPTS=-Xmx1g
      - JAVA_TOOL_OPTIONS=-agentlib:jdwp=transport=dt_socket,server=y,suspend=n,address=*:5005
    volumes:
      - .:/workspace:Z
      - maven-repo:/root/.m2:Z

volumes:
  maven-repo:
```

### Production-like image (`Dockerfile`)

* Multi-stage build: compile in Maven image → run in slim JRE image.
* App runs as **non-root user**.
* Example:

```dockerfile
FROM maven:3.9-eclipse-temurin-21 AS build
WORKDIR /app
COPY pom.xml .
RUN mvn -B -q -DskipTests dependency:go-offline
COPY src ./src
RUN mvn -B -DskipTests package

FROM eclipse-temurin:21-jre
WORKDIR /app
COPY --from=build /app/target/*.jar app.jar
RUN useradd --no-create-home spring && chown spring:spring /app/app.jar
USER spring
EXPOSE 8080
ENTRYPOINT ["java","-jar","/app/app.jar"]
```

---

## 3. Development workflow

1. **Start dev container**

   ```bash
   podman compose -f docker-compose.dev.yml up
   ```
2. **Open project in IntelliJ IDEA** (Community is fine).
3. **Attach debugger**:

    * Run → Edit Configurations → Remote JVM Debug.
    * Host `localhost`, Port `5005`.
    * Breakpoints will hit.
4. **Edit code locally** → DevTools reloads in container → refresh browser.
5. **Stop/reset**:

   ```bash
   podman compose -f docker-compose.dev.yml down
   ```

   Add `--volumes` if you want a totally fresh environment.

---

## 4. Keeping host clean & consistent

* **Bind mount code**: `.:/workspace:Z` → edit on host, run in container.
* **Named volume for Maven repo**: keeps dependencies out of host home.
* **Environment-as-code**: use `.env` or `environment:` in Compose.
* **User mapping**: run with your UID/GID to avoid root-owned files:

  ```yaml
  user: "${UID}:${GID}"
  ```

  and export:

  ```bash
  export UID=$(id -u) GID=$(id -g)
  ```

---

## 5. Extras

* **Database (e.g., Postgres)** → add as service in Compose:

  ```yaml
  db:
    image: postgres:16
    environment:
      POSTGRES_DB: appdb
      POSTGRES_USER: app
      POSTGRES_PASSWORD: secret
    volumes:
      - pgdata:/var/lib/postgresql/data:Z
  ```
* **Profiles** → use `SPRING_PROFILES_ACTIVE=local|dev|prod`.
* **Secrets** → in `.env` for dev, Podman/K8s secrets for production.
* **Automation** → wrap commands in `Makefile` for easy `make run/test/clean`.

---

## 6. Fedora / Podman specifics

* **SELinux**: always add `:Z` to bind mounts.
* **Rootless**: ports ≥1024 (8080/5005) work fine.
* **Debug**: remote JVM debug is the normal way.
* **File change detection**: if reload is flaky, bump inotify:

  ```bash
  echo fs.inotify.max_user_watches=524288 | sudo tee -a /etc/sysctl.conf
  sudo sysctl -p
  ```
* **Container shell** (when needed):

  ```bash
  podman compose -f docker-compose.dev.yml exec dev sh
  ```

---

# ✅ Final Takeaway

Using **Podman for Spring Boot development** gives you:

* **Reproducibility** → same environment as teammates & prod.
* **Isolation** → no clutter on host, easy reset with one command.
* **Fast inner loop** → local edits reload instantly in container.
* **Debug power** → attach IntelliJ via remote JVM debug.
* **Flexibility** → databases, profiles, secrets all defined in Compose.
* **Fedora-native** → SELinux-friendly, rootless, lightweight.

👉 The combination of `docker-compose.dev.yml` (for hot reload + debug) and a multi-stage `Dockerfile` (for prod-like builds) is the **golden setup** for containerized Spring Boot development on Fedora with Podman.

---


