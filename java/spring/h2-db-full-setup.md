# 🗄️ What is H2?

* **H2 is a database engine written in Java.**
* It is **not part of Spring** or DBeaver — it’s its own project.
* It’s **lightweight, fast, embeddable**, and can run:

    * **In-memory** → everything stored in RAM, disappears when you stop your app.
    * **File-based** → saves data in a `.mv.db` file on disk.
    * **TCP server** → runs like a mini DB server that tools (like DBeaver) can connect to.

Think of H2 as a **mini MySQL/Postgres** but simpler, great for learning, demos, or development.

---

# 🖥️ What is DBeaver?

* **DBeaver is a GUI tool** — it is not a database itself.
* It lets you connect to databases (H2, MySQL, Postgres, SQLite, etc.), **browse tables, run SQL, and manage data**.
* DBeaver sometimes shows an “example DB” — that’s just a demo H2 database included so you can click around.

---

# 🔧 How to use H2 + DBeaver (step by step)

## 1. Install DBeaver

On Fedora (or most Linux):

```bash
flatpak install flathub io.dbeaver.DBeaverCommunity
```

Run it:

```bash
flatpak run io.dbeaver.DBeaverCommunity
```

---

## 2. Get H2

Download from: [https://www.h2database.com](https://www.h2database.com) → `h2-*.jar`.
This JAR contains the database engine.

Run H2 as a **TCP server**:

```bash
java -cp h2*.jar org.h2.tools.Server -tcp -tcpAllowOthers -tcpPort 9092 -ifNotExists
```

* `-tcp` → start in server mode
* `-tcpPort 9092` → listens on port 9092
* `-tcpAllowOthers` → lets other apps (like DBeaver) connect
* `-ifNotExists` → creates DB if it doesn’t exist

This keeps H2 running like a small DB server on your machine.

---

## 3. Connect DBeaver to H2

1. Open DBeaver → **New Connection** → choose **H2**.
2. Select **Server** mode (since we started TCP).
3. Fill in:

    * **Host:** `localhost`
    * **Port:** `9092`
    * **Database:** `~/testdb` (H2 will create `testdb.mv.db` in your home folder)
    * **User:** `sa`
    * **Password:** *(leave empty unless you set one)*

Now you can open SQL editor in DBeaver and start working with H2.

---

## 4. Try some SQL

In DBeaver SQL editor:

```sql
-- Create a table
CREATE TABLE users (
  id INT PRIMARY KEY AUTO_INCREMENT,
  name VARCHAR(100)
);

-- Insert rows
INSERT INTO users(name) VALUES ('Alice'), ('Bob');

-- Query rows
SELECT * FROM users;
```

You’ll see the results in DBeaver’s result grid.

---

# 🔍 Important concepts

* **H2 is the database engine** (like a lightweight MySQL).
* **DBeaver is just the UI client** that connects to it.
* **TCP server** mode lets external apps (like DBeaver) connect.
* **File DB** → persists data in `.mv.db` file.
* **In-memory DB** → temporary, vanishes when stopped.

---

# ✅ When to use

* Learning SQL without installing MySQL/Postgres.
* Quick development or testing.
* Exploring databases with a GUI (DBeaver).

⚠️ **Not recommended for production** — for real apps use MySQL, Postgres, etc.


---

# 🌱 What is Spring Boot?

* **Spring Boot** is a Java framework that makes it easy to build applications.
* It gives you:

    * Auto-configuration (sensible defaults).
    * Embedded servers (no need to deploy to Tomcat manually).
    * Starters (dependencies for common features).
* For databases, it integrates smoothly with **H2**.

---

# 🗄️ What is H2 in Spring Boot?

* H2 is just a database engine.
* Spring Boot can detect H2 on the classpath and automatically connect to it.
* You can run your app with:

    * **In-memory DB** → fast, temporary.
    * **File DB** → persists data to disk.
    * **TCP server** → allows external tools like DBeaver to connect.

---

# 🔧 How to set up

## 1. Create Spring Boot project

Use [Spring Initializr](https://start.spring.io) or CLI:

```bash
spring init --dependencies=web,data-jpa,h2 demo
cd demo
```

Dependencies:

* `spring-boot-starter-web` → REST API / web endpoints.
* `spring-boot-starter-data-jpa` → JPA/Hibernate for database.
* `com.h2database:h2` → the H2 DB engine.

---

## 2. Add an Entity

In `src/main/java/.../User.java`:

```java
import jakarta.persistence.*;

@Entity
public class User {
    @Id
    @GeneratedValue(strategy = GenerationType.IDENTITY)
    private Long id;

    private String name;

    // getters & setters
}
```

This defines a table `user` with columns `id` and `name`.

---

## 3. Create a Repository

In `UserRepository.java`:

```java
import org.springframework.data.jpa.repository.JpaRepository;

public interface UserRepository extends JpaRepository<User, Long> {}
```

Spring Data JPA generates CRUD methods (save, findAll, delete, etc.) automatically.

---

## 4. Configure H2

In `src/main/resources/application.properties`:

### In-memory DB (disappears when app stops)

```properties
spring.datasource.url=jdbc:h2:mem:testdb;MODE=MySQL
spring.datasource.driver-class-name=org.h2.Driver
spring.datasource.username=sa
spring.datasource.password=
spring.jpa.hibernate.ddl-auto=update
spring.h2.console.enabled=true
spring.h2.console.path=/h2-console
```

Visit: `http://localhost:8080/h2-console`

### File DB (persists to disk, easier with DBeaver)

```properties
spring.datasource.url=jdbc:h2:file:./data/devdb;MODE=MySQL;AUTO_SERVER=TRUE
spring.datasource.driver-class-name=org.h2.Driver
spring.datasource.username=sa
spring.datasource.password=
spring.jpa.hibernate.ddl-auto=update
spring.h2.console.enabled=true
```

---

## 5. Create a REST Controller

In `UserController.java`:

```java
import org.springframework.web.bind.annotation.*;
import java.util.List;

@RestController
@RequestMapping("/users")
public class UserController {
    private final UserRepository repo;

    public UserController(UserRepository repo) {
        this.repo = repo;
    }

    @GetMapping
    public List<User> getAll() {
        return repo.findAll();
    }

    @PostMapping
    public User add(@RequestBody User user) {
        return repo.save(user);
    }
}
```

---

## 6. Run the app

```bash
./mvnw spring-boot:run
```

or

```bash
./gradlew bootRun
```

Test it:

```bash
curl -X POST http://localhost:8080/users -H "Content-Type: application/json" -d '{"name":"Alice"}'
curl http://localhost:8080/users
```

---

# 🖥️ Inspect in DBeaver

If you used **file mode** with `AUTO_SERVER=TRUE`:

* Start your app (it creates `./data/devdb.mv.db`).
* In DBeaver → New Connection → H2 → Server mode.
* Host: `localhost`, Port: `9092` (if TCP server), or point to file (`./data/devdb`).
* User: `sa`, Password: *(blank)*.

You’ll see the `USER` table and the rows created from your REST API calls.

---

# ✅ Concepts Recap

* **Entity** = Java class → DB table.
* **Repository** = Interface → auto CRUD methods.
* **Datasource URL** = defines how Spring talks to DB (in-memory, file, or server).
* **H2 Console** = web UI at `/h2-console` for quick inspection.
* **DBeaver** = external UI if you want richer DB exploration.

---

# ⚠️ For Production

* Replace H2 with **real DB** (MySQL/Postgres).
* Change dependency to the right JDBC driver.
* Change `spring.datasource.url`, username, password.
* Keep entity + repository code the same.

---

✅ With this setup, you can build a Spring Boot app, test it against H2, and explore data with DBeaver — all without installing a heavy database server.

---
