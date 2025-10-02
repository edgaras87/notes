# MySQL in Docker

---

```yaml
# File: compose.yaml
# Purpose: Run a reusable local MySQL 8 container that your Spring Boot app
# (running on host) and DBeaver can connect to via localhost.

services:
  mysql:
    image: mysql:8.0
    # Official MySQL 8 image; pin exact patch (e.g., mysql:8.0.43) for reproducibility.

    container_name: mysql8
    # Stable name so `docker exec -it mysql8 ...` and logs are predictable.

    restart: unless-stopped
    # Auto-start on reboot; won’t restart if you stop it manually.

    environment:
      # ---- Initial credentials (applied only on first run; persisted in volume) ----
      # !! Change these in real use or move to a .env file.
      MYSQL_ROOT_PASSWORD: rootpass
      MYSQL_DATABASE: appdb
      MYSQL_USER: appuser
      MYSQL_PASSWORD: apppass

      # Keep DB time aligned with host (nice for logs / JDBC serverTimezone).
      TZ: Europe/Vilnius

    ports:
      - "3306:3306"
      # If host 3306 is in use, remap: "3307:3306" (then JDBC port=3307).

    # mysqld startup flags (tune server behavior)
    # IMPORTANT: Put comments on their own lines (not after the flag) so MySQL doesn’t see '#' as an argument.
    command:
      # JDBC/DBeaver-friendly auth; remove if all clients support caching_sha2_password (default).
      - --default-authentication-plugin=mysql_native_password
      # Full Unicode (incl. emoji)
      - --character-set-server=utf8mb4
      # Modern, case-insensitive, accent-insensitive collation
      - --collation-server=utf8mb4_0900_ai_ci
      # Avoid DNS lookups for faster, predictable logins
      - --skip-host-cache
      - --skip-name-resolve

    volumes:
      - mysql_data:/var/lib/mysql
      # Named volume for persistent data (recommended default on Linux).

      - ./initdb:/docker-entrypoint-initdb.d:ro
      # Optional: .sql/.sh here run once on first boot (great for schema/seed).

    # Healthcheck helps you see when DB is ready and lets other services depend on it.
    healthcheck:
      test: ["CMD-SHELL", "mysqladmin ping -h localhost -p$${MYSQL_ROOT_PASSWORD} --silent"]
      interval: 10s
      timeout: 5s
      retries: 10

volumes:
  mysql_data:
  # Declares the named volume used above.
```

---

## Run & inspect

```bash
docker compose up -d
docker compose logs -f mysql   # wait for: "ready for connections"
docker ps                      # see container
```

---

## Connect (DBeaver)

* Host: `localhost`
* Port: `3306` (or your remap, e.g., `3307`)
* DB: `appdb`
* User/Pass: `appuser / apppass` (standard) or `root / rootpass` (admin)

---

## Add a new DB user (later)

```bash
docker exec -it mysql8 mysql -u root -p
# enter rootpass
```

```sql
-- Create user (remote; use 'localhost' instead of '%' to restrict to inside container/local)
CREATE USER 'newuser'@'%' IDENTIFIED BY 'newpass';

-- Pick one grant:
GRANT ALL PRIVILEGES ON appdb.* TO 'newuser'@'%';                 -- full access to appdb
-- or least-privilege:
GRANT SELECT, INSERT, UPDATE, DELETE ON appdb.* TO 'newuser'@'%'; -- CRUD only

FLUSH PRIVILEGES;
```

Use `newuser/newpass` in DBeaver or Spring Boot.

---

## SQL schema & seed via `./initdb`

Anything you put in `./initdb` (next to `compose.yaml`) is executed **once** on the **first** container start (when the `mysql_data` volume is empty). Files run **alphabetically**, so use numeric prefixes to control order:

```
./initdb/
 ├─ 01-schema.sql   # create tables, indexes, constraints
 ├─ 02-seed.sql     # insert baseline/lookup data
 └─ 03-users.sql    # (optional) create app-level users/roles
```

### 1) Example schema (`01-schema.sql`)

```sql
-- Use your default DB created via MYSQL_DATABASE (appdb)
USE appdb;

-- Idempotent table creation
CREATE TABLE IF NOT EXISTS customers (
  id           BIGINT AUTO_INCREMENT PRIMARY KEY,
  email        VARCHAR(255) NOT NULL UNIQUE,
  first_name   VARCHAR(100) NOT NULL,
  last_name    VARCHAR(100) NOT NULL,
  status       ENUM('ACTIVE','INACTIVE') NOT NULL DEFAULT 'ACTIVE',
  created_at   TIMESTAMP NOT NULL DEFAULT CURRENT_TIMESTAMP,
  updated_at   TIMESTAMP NOT NULL DEFAULT CURRENT_TIMESTAMP ON UPDATE CURRENT_TIMESTAMP
) ENGINE=InnoDB DEFAULT CHARSET=utf8mb4 COLLATE=utf8mb4_0900_ai_ci;

CREATE TABLE IF NOT EXISTS orders (
  id           BIGINT AUTO_INCREMENT PRIMARY KEY,
  customer_id  BIGINT NOT NULL,
  total_cents  INT NOT NULL,
  currency     CHAR(3) NOT NULL DEFAULT 'EUR',
  placed_at    TIMESTAMP NOT NULL DEFAULT CURRENT_TIMESTAMP,
  CONSTRAINT fk_orders_customer
    FOREIGN KEY (customer_id) REFERENCES customers(id)
    ON DELETE CASCADE
) ENGINE=InnoDB DEFAULT CHARSET=utf8mb4 COLLATE=utf8mb4_0900_ai_ci;

-- Helpful index examples
CREATE INDEX IF NOT EXISTS idx_orders_customer ON orders(customer_id);
CREATE INDEX IF NOT EXISTS idx_customers_email ON customers(email);
```

### 2) Example seed data (`02-seed.sql`)

```sql
USE appdb;

-- Seed customers (idempotent with ON DUPLICATE KEY)
INSERT INTO customers (email, first_name, last_name, status)
VALUES
  ('ada@example.com',  'Ada',  'Lovelace', 'ACTIVE'),
  ('grace@example.com','Grace','Hopper',   'ACTIVE')
ON DUPLICATE KEY UPDATE
  first_name = VALUES(first_name),
  last_name  = VALUES(last_name),
  status     = VALUES(status);

-- Seed orders (guard with EXISTS to avoid duplicates)
INSERT INTO orders (customer_id, total_cents, currency)
SELECT c.id, 1999, 'EUR' FROM customers c WHERE c.email = 'ada@example.com'
AND NOT EXISTS (SELECT 1 FROM orders o WHERE o.customer_id = c.id AND o.total_cents = 1999);

INSERT INTO orders (customer_id, total_cents, currency)
SELECT c.id, 3499, 'EUR' FROM customers c WHERE c.email = 'grace@example.com'
AND NOT EXISTS (SELECT 1 FROM orders o WHERE o.customer_id = c.id AND o.total_cents = 3499);
```

### 3) (Optional) App-level users/privileges (`03-users.sql`)

> You already set `MYSQL_USER`/`MYSQL_PASSWORD` in compose, but if you want **more users** created automatically:

```sql
-- Create a read-only user for analytics dashboards, etc.
CREATE USER IF NOT EXISTS 'reporter'@'%' IDENTIFIED BY 'reporterpass';

-- Read-only on appdb
GRANT SELECT ON appdb.* TO 'reporter'@'%';

FLUSH PRIVILEGES;
```

### Re-running init scripts

* They only run when the data volume is **fresh**. To re-run from scratch:

  ```bash
  docker compose down -v     # removes mysql_data volume
  docker compose up -d
  ```
* If you don’t want to wipe data, make your seeds **idempotent** using:

    * `CREATE TABLE IF NOT EXISTS`
    * `INSERT ... ON DUPLICATE KEY UPDATE`
    * `INSERT ... WHERE NOT EXISTS (...)`
    * `CREATE INDEX IF NOT EXISTS` (MySQL 8.0.13+)

### Using migrations later (recommended)

When your schema evolves, prefer **Flyway** or **Liquibase** in Spring Boot. They run versioned scripts (`V1__init.sql`, `V2__add_indexes.sql`, …) at app startup, which is safer and traceable than ad-hoc SQL.

---

## Spring Boot quick reminder (JPA strategy)

```properties
# pick ONE:
spring.jpa.hibernate.ddl-auto=update      # dev-friendly; tries to evolve schema non-destructively
# spring.jpa.hibernate.ddl-auto=validate  # safest; checks schema matches entities; fails if not
# spring.jpa.hibernate.ddl-auto=create    # drop & create schema at startup (destructive)
# spring.jpa.hibernate.ddl-auto=create-drop # create on start, drop on shutdown (tests only)
```

> If you use `./initdb` schema or Flyway/Liquibase, set `ddl-auto=validate` to catch mismatches without altering the DB.

---


