
# Simple Guide to `systemctl` and `docker`

---

### What `systemctl` is:

* `systemctl` is a command-line tool used to **control the systemd system and service manager**.
* `systemd` is the init system used in most modern Linux distributions (Ubuntu, Debian, Fedora, CentOS, Arch, etc.). It’s responsible for booting the system, starting background services (daemons), and managing their lifecycle.

With `systemctl`, you can:

* **Start a service**: `sudo systemctl start docker`
* **Stop a service**: `sudo systemctl stop docker`
* **Restart a service**: `sudo systemctl restart docker`
* **Check status**: `sudo systemctl status docker`
* **Enable at boot**: `sudo systemctl enable docker`
* **Disable at boot**: `sudo systemctl disable docker`
* **List all services**: `systemctl list-units --type=service`

---

👉 In short:
`systemctl` is the tool, `systemd` is the service manager, and `docker` is the service you’re controlling.

---

# Simple Guide to Docker Concepts

---

### Core Docker concepts (refined)

* **Image**

    * A *blueprint* (template) for running software (e.g., `mysql:8.4`, `nginx:latest`, or your own built from a `Dockerfile`).
    * Immutable (read-only).

* **Container**

    * A *running instance* of an image.
    * One container runs **exactly one image** (but you can run multiple containers from the same image).
    * Containers are ephemeral: if they stop and you don’t use volumes, data inside is gone.

* **Volume**

    * Persistent storage that lives outside the container lifecycle.
    * Lets you keep data even if the container stops, restarts, or is deleted.
    * Example: MySQL database files in `/var/lib/mysql`.

* **Ports**

    * Containers live in their own isolated network.
    * You use **port mapping** (`-p hostPort:containerPort`) to let the host (and outside world) talk to the container.
    * Example: `-p 3306:3306` makes MySQL available at `localhost:3306` on your machine.

* **Dockerfile**

    * A script/blueprint for building an **image**.
    * Defines base image, environment, files to copy, commands to run, ports to expose, etc.
    * Example: your Spring Boot app Dockerfile builds a custom image from `openjdk:21`.

* **docker-compose.yml**

    * A declarative config for running **one or more containers** together.
    * Defines:

        * Which images to run
        * How they connect (networks)
        * How they persist data (volumes)
        * How they talk to the host (ports)
        * Dependencies/order (e.g., app waits for db)

---

✅ So your statement is spot-on:

* Images → run inside containers
* Containers → ephemeral, so use volumes for persistent data
* Ports → allow communication between container and host
* Dockerfile → defines how to build an image
* docker-compose.yml → defines how containers (services) run together and interact

---

## Simple Guide to Container Networking

---

### 1. Where are containers?

* Containers are **processes running on your host machine**, but with extra isolation provided by **Linux namespaces** (for PIDs, network, mounts, etc.) and **cgroups** (for resource limits).
* You can think of them as “lightweight virtual machines,” but technically they’re just regular Linux processes in a restricted view of the system.

---

### 2. Do containers see each other?

* By default:

    * Each container gets its **own isolated filesystem, process space, and network stack**.
    * So container A doesn’t automatically see container B’s files or processes.
    * But networking is a bit special: Docker usually puts containers into a **default bridge network**, so they **can talk to each other** if you connect them properly.

---

### 3. Communication between containers

There are a few options:

* **Same network (default: `bridge`)**

    * When you run containers, they often join the default `bridge` network.
    * On this network, they can reach each other by IP (assigned by Docker).
    * With Docker Compose, you can also give them **service names** (DNS).
      Example:

      ```yaml
      services:
        app:
          ...
        db:
          image: mysql:8.4
      ```

      → Here, `app` can connect to MySQL at hostname `db:3306`.

* **Custom user-defined network**

    * Best practice: create your own network (`docker network create mynet` or let Compose do it automatically).
    * Containers on the same custom network get **built-in DNS resolution** by service/container name.
    * Example: `spring.datasource.url=jdbc:mysql://db:3306/appdb`.

* **Through host ports**

    * If you `-p 3306:3306`, then MySQL is reachable on your host at `localhost:3306`.
    * Other containers **could** connect to it through the host’s IP/localhost, but this is less efficient than using a Docker network.

---

### 4. Mental model

Think of Docker like this:

* Each container is in its **own little room** (namespace).
* By default, they **don’t share rooms**.
* But Docker provides **hallways (networks)** where they can meet and talk, either:

    * Directly (container-to-container on the same network), or
    * Indirectly through the **host machine** (via published ports).

---

✅ **Best practice in dev projects**:

* Use **Docker networks** (automatic in Compose) so containers can talk by name.
* Only expose ports to the host when you want external tools (like your browser, Postman, or Workbench) to connect.

---

## 🕸️ Docker networks (the hallway between containers)

* Each container is isolated by default.
* To let containers talk to each other, you put them on the **same Docker network**.
* On a user-defined network:

    * Containers can **reach each other by name** (no need for IPs).
    * Example: `backend` can reach `mysql` at `mysql:3306`.

---

## Example setup

```yaml
services:
  frontend:
    image: my-frontend
    depends_on: [backend]
    ports:
      - "3000:3000"

  backend:
    image: my-backend
    depends_on: [mysql]
    ports:
      - "8080:8080"

  mysql:
    image: mysql:8.4
    environment:
      MYSQL_ROOT_PASSWORD: secret
      MYSQL_DATABASE: appdb
    volumes:
      - mysql_data:/var/lib/mysql
    ports:
      - "3306:3306"

  workbench:
    image: linuxserver/mysql-workbench
    environment:
      DISPLAY: :0  # (for GUI if you run with X11 or VNC)
    depends_on: [mysql]

  nginx:
    image: nginx:latest
    ports:
      - "80:80"
    depends_on: [frontend]

volumes:
  mysql_data:
```

---

## How they talk inside the network

* `backend` → talks to `mysql` using **`mysql:3306`**
* `frontend` → talks to `backend` using **`backend:8080`**
* `nginx` → can reverse-proxy to `frontend:3000`
* `workbench` → can connect to `mysql:3306`

⚡ Notice: they don’t need host IPs — just the **service names** (`mysql`, `backend`, `frontend`).

---

## Outside world access (via exposed ports)

* You (on host) open browser at `http://localhost:80` → hits **nginx**.
* You (on host) open `http://localhost:3000` → hits **frontend** directly.
* You (on host) use MySQL Workbench installed locally → connect to **`127.0.0.1:3306`**.

---

✅ **Rule of thumb**:

* Use **Docker networks** for container-to-container communication.
* Use **exposed ports** only when host/outside needs to talk in.

---

## Example

### **backend ↔ MySQL over a Docker network** using the service name as the hostname

# 1) `docker-compose.yml`

```yaml
services:
  mysql:
    image: mysql:8.4
    environment:
      MYSQL_ROOT_PASSWORD: change-me
      MYSQL_DATABASE: appdb
      MYSQL_USER: appuser
      MYSQL_PASSWORD: app-pass
    volumes:
      - mysql_data:/var/lib/mysql
    ports:
      - "3306:3306"        # so host tools (e.g., Workbench) can reach it
    healthcheck:
      test: ["CMD-SHELL", "mysqladmin ping -h 127.0.0.1 -p$${MYSQL_ROOT_PASSWORD} || exit 1"]
      interval: 10s
      timeout: 5s
      retries: 5

  backend:
    build: ./backend        # assumes a Dockerfile in ./backend (optional; see notes)
    depends_on:
      mysql:
        condition: service_healthy
    environment:
      # your app can also read these; optional convenience
      SPRING_DATASOURCE_URL: jdbc:mysql://mysql:3306/appdb?allowPublicKeyRetrieval=true&useSSL=false
      SPRING_DATASOURCE_USERNAME: appuser
      SPRING_DATASOURCE_PASSWORD: app-pass
    ports:
      - "8080:8080"

volumes:
  mysql_data:
```

> 🔑 Key bit: inside the Compose network, the backend reaches the DB at **`mysql:3306`** (the service name acts like DNS).

---

# 2) Spring Boot config (backend talks to `mysql:3306`)

If you prefer file-based config instead of env vars:

`src/main/resources/application.properties`

```properties
spring.datasource.url=jdbc:mysql://mysql:3306/appdb?allowPublicKeyRetrieval=true&useSSL=false
spring.datasource.username=appuser
spring.datasource.password=app-pass
spring.datasource.driver-class-name=com.mysql.cj.jdbc.Driver

# Optional while practicing:
spring.jpa.hibernate.ddl-auto=update
spring.jpa.show-sql=true
```

---

# 3) Minimal backend Dockerfile (optional)

`backend/Dockerfile`

```dockerfile
FROM eclipse-temurin:21-jre
WORKDIR /app
# copy your built jar (e.g., from mvn package or gradle build)
COPY target/app.jar app.jar
EXPOSE 8080
ENTRYPOINT ["java","-jar","/app/app.jar"]
```

(If you’re running Spring Boot on the **host** instead of in Docker, skip the backend service in Compose and just keep `mysql`. Your app still uses `jdbc:mysql://127.0.0.1:3306/appdb` from the host, but **inside Docker** the hostname is `mysql`.)

---

# 4) Run it

```bash
docker compose up -d
docker compose logs -f mysql   # watch DB health
```

---

# Notes / gotchas

* **Service name = hostname.** That’s why `mysql:3306` works between containers.
* **Healthcheck + depends_on** makes the backend start after MySQL is ready.
* If you keep MySQL only in Docker and run Spring Boot on **host**, use:

  ```
  spring.datasource.url=jdbc:mysql://127.0.0.1:3306/appdb
  ```
* Use **volumes** to persist DB data across container restarts.
* You can create multiple networks in Compose if you want to isolate groups of services (e.g., `frontend` on one, `backend+db` on another).

---
