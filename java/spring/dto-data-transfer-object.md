
---

# Spring Boot – DTOs vs Entities Cheat Sheet

## 🔑 Core Idea

* **Entities = persistence model (DB schema)**
* **DTOs = API contract (request/response shapes)**
* **Jackson annotations ≠ DTOs** (good for formatting, not for API design)

---

## ✅ When You *Can* Skip DTOs

* Small, internal/prototype apps.
* Flat/simple entities, no/little relationships.
* You’re fine coupling DB schema ↔ API.
* Still need: `@JsonIgnore`, cycles handling (`@JsonManagedReference`/`@JsonBackReference`).

⚠️ Risks: API breaks if DB changes, over-posting vulnerabilities, lazy init exceptions.

---

## 🚀 Why Use DTOs

* **Security**: Don’t expose `id`, `isAdmin`, audit fields, **passwords**.
* **API stability/versioning**: Entities can evolve, DTOs stay stable.
* **Tailored views**: Different endpoints return different shapes.
* **Performance**: Avoid loading big graphs / cycles.
* **Validation**: `@NotBlank`, `@Size`, `@Email` in *request DTOs*.

---

## 🛠 Practical Patterns

### Entity Example

```java
@Entity
@Table(name = "contacts")
public class Contact {
  @Id @GeneratedValue(strategy = GenerationType.UUID)
  private String id;

  private String name;
  private String photoUrl;
  private String email;

  // ⚠️ NEVER expose this directly in responses!
  private String password;
}
```

### Request DTO

```java
public record CreateContactRequest(
    @NotBlank String name,
    @Email String email,
    String photoUrl,
    @NotBlank String password // required on create
) {}
```

### Response DTO

```java
public record ContactResponse(
    String id,
    String name,
    String email
    // ❌ password excluded for security
) {}
```

### Mapper (MapStruct)

```java
@Mapper(componentModel = "spring")
public interface ContactMapper {
  Contact toEntity(CreateContactRequest req);
  ContactResponse toResponse(Contact entity);
}
```

### Controller

```java
@PostMapping("/contacts")
public ContactResponse create(@Valid @RequestBody CreateContactRequest req) {
  Contact saved = repo.save(mapper.toEntity(req));
  return mapper.toResponse(saved);
}
```

---

## 🧰 Common Tools & Tips

### 1. Mapping Between DTOs & Entities ([Mapping Strategies Explained](#-mapping-strategies-explained))

* **[MapStruct](https://mapstruct.org/)** → recommended (fast, compile-time safety).
* **ModelMapper** → easier but reflection-based (slower, less explicit).
* **Manual mapping** → okay for very small apps (`new ContactResponse(e.getId(), e.getName())`).
* **Java 17 `record` DTOs** → concise, immutable, good for responses.

---

### 2. JSON Serialization with Jackson



**Formatting**:
 
* `@JsonIgnore` → hide field globally (e.g. password).
* `@JsonProperty(access = WRITE_ONLY)` → accept in requests but hide in responses.
* `@JsonInclude(Include.NON_NULL)` → omit `null` fields in JSON.
 
**Cycles handling ([Why cycles happen](#why-cycles-happen))**:

* `@JsonManagedReference` / `@JsonBackReference` (parent-child)
* `@JsonIdentityInfo(generator = ObjectIdGenerators.PropertyGenerator.class, property = "id")` (works with graphs)



---

### 3. Validation

* **Validation vs DB constraints**:

    * DTO: business/API validation (`@NotNull`, `@Pattern`)
    * Entity: persistence rules (`@Column(nullable=false)`)


* Benefit: request validation errors are API-level, not DB exceptions.

---

### 4. Tailored Views ([Data Interface Projections](#-spring-data-interface-projections--quick-guide))


 

* Different DTOs for different endpoints (don’t overload one DTO).
* **Spring Data projections** (good for read-only):

  ```java
  public interface ContactView {
    String getId();
    String getName();
  }
  ```

  → Return partial entity directly without mapping.

---

### 5. Security & Over-Posting

* Never expose internal fields (`password`, `isAdmin`, audit timestamps).
* DTOs give **whitelisting**: only fields you want are exposed.
* Entities with Jackson rely on **blacklisting** (`@JsonIgnore`) — easier to miss something.

---

### 6. Performance Considerations

* Lazy-loading traps: serializing entities often triggers `LazyInitializationException`.
* DTOs (or projections) avoid forcing huge object graphs.
* Use `fetch join` queries if you must load relations.

---

### 7. Versioning & Maintainability

* DTOs let you evolve entities freely.
* You can create **v2 DTOs** for breaking API changes, while keeping old contracts alive.
* Entities can rename fields / refactor DB schema without breaking clients.

---

## 📌 Bottom Line

* **Public/external APIs → always DTOs**.
* **Internal/simple apps → can return entities** (with guardrails).
* DTOs give: **security, stability, flexibility, validation, performance**.
* Entities are for persistence. DTOs are for communication.

---



```java









```








# 🌟 Spring Data Interface Projections – Quick Guide

**What it is:**
Instead of returning full JPA entities, you define a simple `interface` with getter methods. Spring Data will automatically generate a proxy that fetches **only those columns** and serializes them (e.g., as JSON).

---

### ✅ Example

**Entity**

```java
@Entity
public class Contact {
  @Id private String id;
  private String name;
  private String photoUrl;
}
```

**Projection Interface**

```java
public interface ContactView {
  String getId();
  String getName();
}
```

**Repository**

```java
public interface ContactRepository extends JpaRepository<Contact, String> {
  List<ContactView> findByNameContainingIgnoreCase(String q);
  <T> Optional<T> findById(String id, Class<T> type); // dynamic projection
}
```

**Controller**

```java
@GetMapping("/contacts")
public List<ContactView> list(@RequestParam String q) {
  return repo.findByNameContainingIgnoreCase(q);
}
```

**Response JSON**

```json
[
  { "id": "123", "name": "Alice" },
  { "id": "456", "name": "Bob" }
]
```

---

### 🔑 Key Points

* **Only fetch selected fields** → more efficient than loading full entity.
* **Read-only** → best for queries and responses.
* **No mapping needed** → Jackson serializes projection proxies directly.
* **Dynamic projection** → choose shape (`ContactView`, `ContactCardView`, DTO class with constructor) at runtime.
* **Writes (create/update)** → still need DTOs or mappers; projections are not for persisting.

---

👉 **In short**:
Use **interface-based projections** for **efficient, read-only slices** of your data. They cut down unnecessary fields/relationships and save you from manual DTO mapping — but they don’t replace DTOs for **write operations or API stability**.








```java









```






# 🔄 Mapping Strategies Explained

---

* The **entity** (baseline)
* **DTOs** (request/response)
* A **mapper** per strategy (how to go `DTO ↔ Entity`)
* A tiny **usage** sketch

> Security note: responses **never** include `password`, and on create we’d typically **encode** the password in a service layer (shown where relevant).

---

# 0) Baseline: Entity & DTOs

**Entity**

```java
@Entity
@Table(name = "contacts")
public class Contact {
  @Id @GeneratedValue(strategy = GenerationType.UUID)
  private String id;

  private String name;
  private String photoUrl;
  private String email;

  // Never return this in responses
  private String password;

  // getters/setters
}
```

**DTOs (Java 17 records)**

```java
public record CreateContactRequest(
    @NotBlank String name,
    @Email String email,
    String photoUrl,
    @NotBlank String password
) {}

public record UpdateContactRequest(
    String name,
    @Email String email,
    String photoUrl,
    String password // optional; if null/blank, do not change
) {}

public record ContactResponse(
    String id,
    String name,
    String email,
    String photoUrl
) {}
```

---

# 1) MapStruct (compile-time, fast, explicit)

* **What**: Code generator at compile time (writes mapping code for you).
* **Pros**:

    * Very fast (no reflection).
    * Explicit — you see what’s mapped, avoids surprises.
    * Industry standard in Spring projects.
* **When to use**: Medium/large apps, teams, production APIs.

---

```java
@Mapper(componentModel = "spring")
public interface ContactMapper {

  // Request -> Entity (for create). We usually encode password in service, not mapper.
  @Mapping(target = "id", ignore = true)
  Contact toEntity(CreateContactRequest req);

  // Partial update: ignore nulls
  @BeanMapping(nullValuePropertyMappingStrategy = NullValuePropertyMappingStrategy.IGNORE)
  void updateEntity(UpdateContactRequest req, @MappingTarget Contact target);

  // Entity -> Response (password excluded by DTO shape)
  ContactResponse toResponse(Contact entity);
}
```

**Service usage (encoding password)**

```java
@Service
@RequiredArgsConstructor
public class ContactService {
  private final ContactRepository repo;
  private final ContactMapper mapper;
  private final PasswordEncoder passwordEncoder;

  public ContactResponse create(CreateContactRequest req) {
    Contact c = mapper.toEntity(req);
    c.setPassword(passwordEncoder.encode(req.password()));
    Contact saved = repo.save(c);
    return mapper.toResponse(saved);
  }

  public ContactResponse update(String id, UpdateContactRequest req) {
    Contact c = repo.findById(id).orElseThrow();
    mapper.updateEntity(req, c);
    if (req.password() != null && !req.password().isBlank()) {
      c.setPassword(passwordEncoder.encode(req.password()));
    }
    return mapper.toResponse(repo.save(c));
  }
}
```

---

# 2) ModelMapper (reflection, quick to start)

* **What**: Libraries that use reflection to map fields with similar names automatically.
* 
* **Pros**:

    * Very little setup — works “out of the box.”
    * 
* **Cons**:
  
    * Slower (uses reflection).
    * Easy to miss edge cases.
  
* **When to use**: Small projects, prototypes, quick POCs.

---

**Config**

```java
@Configuration
public class ModelMapperConfig {
  @Bean
  public ModelMapper modelMapper() {
    ModelMapper mm = new ModelMapper();

    // Entity -> Response (auto by matching names)
    mm.createTypeMap(Contact.class, ContactResponse.class);

    // Create DTO -> Entity (skip id)
    mm.createTypeMap(CreateContactRequest.class, Contact.class)
      .addMappings(m -> m.skip(Contact::setId));

    // Update DTO -> Entity (skip nulls)
    mm.getConfiguration().setPropertyCondition(Conditions.isNotNull());

    return mm;
  }
}
```

**Service usage**

```java
@Service
@RequiredArgsConstructor
public class ContactService {
  private final ContactRepository repo;
  private final ModelMapper mm;
  private final PasswordEncoder encoder;

  public ContactResponse create(CreateContactRequest req) {
    Contact c = mm.map(req, Contact.class);
    c.setPassword(encoder.encode(req.password() ));
    return mm.map(repo.save(c), ContactResponse.class);
  }

  public ContactResponse update(String id, UpdateContactRequest req) {
    Contact c = repo.findById(id).orElseThrow();
    mm.map(req, c); // nulls ignored per config
    if (req.password() != null && !req.password().isBlank()) {
      c.setPassword(encoder.encode(req.password()));
    }
    return mm.map(repo.save(c), ContactResponse.class);
  }
}
```

---

# 3) Dozer (older reflection mapper; similar to ModelMapper)

* **What**: You write mapping code yourself (`new ContactResponse(e.getId(), e.getName())`).
* **Pros**:

    * Full control, no extra dependency.
    * Transparent.
* **Cons**:

    * Boilerplate grows fast if many entities.
* **When to use**: Small/simple apps, when you only have a few DTOs.

---

**Dependency & basic usage** (example with Dozer 6.x style)

```java
@Configuration
public class DozerConfig {
  @Bean
  public Mapper dozerBeanMapper() {
    return DozerBeanMapperBuilder.buildDefault();
  }
}
```

**Service usage**

```java
@Service
@RequiredArgsConstructor
public class ContactService {
  private final ContactRepository repo;
  private final Mapper dozer;
  private final PasswordEncoder encoder;

  public ContactResponse create(CreateContactRequest req) {
    Contact c = dozer.map(req, Contact.class); // matches by name
    c.setId(null); // ensure id not set from request
    c.setPassword(encoder.encode(req.password()));
    return dozer.map(repo.save(c), ContactResponse.class);
  }

  public ContactResponse update(String id, UpdateContactRequest req) {
    Contact c = repo.findById(id).orElseThrow();
    dozer.map(req, c); // nulls will overwrite; consider a custom mapper or checks
    if (req.password() != null && !req.password().isBlank()) {
      c.setPassword(encoder.encode(req.password()));
    }
    return dozer.map(repo.save(c), ContactResponse.class);
  }
}
```

> Tip: Dozer tends to be less common today; ModelMapper or MapStruct are preferred.

---

# 4) Spring `ConversionService` (lightweight converters)

* **What**: A built-in Spring mechanism for converting between types (`String ↔ LocalDate`, `Entity ↔ DTO`, etc.).
* **Pros**:

    * Already in Spring.
    * Can centralize conversions.
* **Cons**:

    * More often used for primitive type conversions, not large DTO mapping.
* **When to use**: When you want lightweight type conversions or already have ConversionService in play.

---

**Converters**

```java
@Component
public class CreateContactRequestToContact implements Converter<CreateContactRequest, Contact> {
  @Override
  public Contact convert(CreateContactRequest req) {
    Contact c = new Contact();
    c.setName(req.name());
    c.setEmail(req.email());
    c.setPhotoUrl(req.photoUrl());
    c.setPassword(req.password()); // encode later in service
    return c;
  }
}

@Component
public class ContactToContactResponse implements Converter<Contact, ContactResponse> {
  @Override
  public ContactResponse convert(Contact c) {
    return new ContactResponse(c.getId(), c.getName(), c.getEmail(), c.getPhotoUrl());
  }
}
```

**Service usage**

```java
@Service
@RequiredArgsConstructor
public class ContactService {
  private final ContactRepository repo;
  private final ConversionService conversionService;
  private final PasswordEncoder encoder;

  public ContactResponse create(CreateContactRequest req) {
    Contact c = conversionService.convert(req, Contact.class);
    c.setPassword(encoder.encode(c.getPassword()));
    return conversionService.convert(repo.save(c), ContactResponse.class);
  }

  public ContactResponse update(String id, UpdateContactRequest req) {
    Contact c = repo.findById(id).orElseThrow();
    // Manually apply only provided fields (ConversionService is not great for partial updates)
    if (req.name() != null) c.setName(req.name());
    if (req.email() != null) c.setEmail(req.email());
    if (req.photoUrl() != null) c.setPhotoUrl(req.photoUrl());
    if (req.password() != null && !req.password().isBlank()) {
      c.setPassword(encoder.encode(req.password()));
    }
    return conversionService.convert(repo.save(c), ContactResponse.class);
  }
}
```

---

# 5) Manual Mapping (full control, no deps)

* **What**: Java feature — immutable, concise classes (`public record ContactResponse(String id, String name) {}`).
* **Pros**:

    * Built-in immutability.
    * Less boilerplate.
* **Cons**:

    * Records are just DTOs; you still need some mapping strategy (manual, MapStruct, etc.).
* **When to use**: Modern Java apps, for clean DTO definitions.


---

**Mapper utility**

```java
@Component
public class ContactManualMapper {

  public Contact toEntity(CreateContactRequest req) {
    Contact c = new Contact();
    c.setName(req.name());
    c.setEmail(req.email());
    c.setPhotoUrl(req.photoUrl());
    c.setPassword(req.password());
    return c;
  }

  public void applyUpdate(UpdateContactRequest req, Contact c) {
    if (req.name() != null) c.setName(req.name());
    if (req.email() != null) c.setEmail(req.email());
    if (req.photoUrl() != null) c.setPhotoUrl(req.photoUrl());
    if (req.password() != null && !req.password().isBlank()) {
      c.setPassword(req.password()); // encode in service
    }
  }

  public ContactResponse toResponse(Contact c) {
    return new ContactResponse(c.getId(), c.getName(), c.getEmail(), c.getPhotoUrl());
  }
}
```

**Service usage**

```java
@Service
@RequiredArgsConstructor
public class ContactService {
  private final ContactRepository repo;
  private final ContactManualMapper mapper;
  private final PasswordEncoder encoder;

  public ContactResponse create(CreateContactRequest req) {
    Contact c = mapper.toEntity(req);
    c.setPassword(encoder.encode(c.getPassword()));
    return mapper.toResponse(repo.save(c));
  }

  public ContactResponse update(String id, UpdateContactRequest req) {
    Contact c = repo.findById(id).orElseThrow();
    mapper.applyUpdate(req, c);
    if (req.password() != null && !req.password().isBlank()) {
      c.setPassword(encoder.encode(req.password()));
    }
    return mapper.toResponse(repo.save(c));
  }
}
```

---

# 6) Using Java 17 `record` DTOs (with any strategy)

You’ve already seen records above for DTOs. They **pair well** with MapStruct or manual mapping (and are fine with ModelMapper/Dozer too). No extra code needed beyond what’s shown—just keep DTOs as records and map with your chosen approach.

---

### 📌 Key Takeaway

* These are **different approaches** to solve the *same problem*:
  *How do I turn a JPA entity into a DTO and vice versa?*

* You usually **pick one main strategy** (e.g., MapStruct or manual) and **optionally mix in records** to define DTOs.

### Which should you pick?

* **Production/teams** → **MapStruct + record DTOs** (explicit, fast, safe).
* **Tiny app/POC** → **ModelMapper** or **manual mapping**.
* **You already use ConversionService** → add simple converters for DTOs (still do manual logic for updates).









# Why cycles happen

---

Here’s what those Jackson annotations are for and how they differ—both are ways to prevent **infinite recursion** (stack overflows) when you serialize **bi-directional object graphs** (e.g., `User` ↔ `Order`) to JSON.

---

```java
class User { Order order; }
class Order { User user; }
```

Serializing `User` tries to serialize `order`, which tries to serialize its `user`, which tries to serialize its `order`… and so on forever.

---

# Option 1: `@JsonManagedReference` / `@JsonBackReference`

**Idea:** Mark one side as the “parent/managed” side and the other as the “back” side.
During **serialization**, Jackson **ignores the back reference**, breaking the loop.
During **deserialization**, Jackson uses the managed side to set the relationship.

**Example**

```java
class User {
  @JsonManagedReference
  private List<Order> orders;
}

class Order {
  @JsonBackReference
  private User user;
}
```

**Serialized JSON (from a `User`):**

```json
{
  "orders": [
    { /* no 'user' field here */ }
  ]
}
```

**When to use**

* Simple **parent → children** relationships.
* You’re OK **not** including the back link in JSON.

**Pros**

* Very simple; no IDs required.
* Great for tree-like structures.

**Cons**

* One-directional JSON: the back side disappears.
* Doesn’t handle **multiple references** to the same object well outside the parent/child tree pattern.
* Harder when graphs are not strict trees.

---

# Option 2: `@JsonIdentityInfo(generator = ObjectIdGenerators.PropertyGenerator.class, property = "id")`

**Idea:** Give each object an identity (usually its database `id`). When the same object appears again, Jackson writes **just the ID** instead of repeating the full object—this naturally breaks cycles.

**Example**

```java
@JsonIdentityInfo(
  generator = ObjectIdGenerators.PropertyGenerator.class,
  property = "id"
)
class User {
  private Long id;
  private List<Order> orders;
}

@JsonIdentityInfo(
  generator = ObjectIdGenerators.PropertyGenerator.class,
  property = "id"
)
class Order {
  private Long id;
  private User user;
}
```

**Serialized JSON (from a `User`):**

```json
{
  "id": 1,
  "orders": [
    {
      "id": 10,
      "user": 1     // references the User by id instead of nesting again
    }
  ]
}
```

**When to use**

* General **graphs** (not just parent/child trees).
* You want to **preserve both directions** in JSON using IDs.
* Objects may be **shared** in multiple places.

**Pros**

* Works for complex graphs and cycles.
* Keeps references explicit; both sides represented.

**Cons**

* Requires a stable identifier (the `property` must exist and be unique).
* JSON becomes **ID-heavy** and slightly harder to read if you wanted full expansions.
* Deserialization expects those IDs to be resolvable.

**Useful tweak**

```java
@JsonIdentityReference(alwaysAsId = true)
```

on a field makes that field **always** serialize as an ID, even on first occurrence (keeps payloads small).

---

# Quick decision guide

* **Simple parent→children, don’t need back link in JSON?**
  Use `@JsonManagedReference` / `@JsonBackReference`.
* **Bi-directional or complex graphs, want both sides represented?**
  Use `@JsonIdentityInfo` with a reliable `id`.

---

# Common pitfalls & tips

* With JPA/Hibernate, watch for **lazy loading**: if the back/child collection is lazy and not initialized, Jackson may serialize it as empty or throw exceptions. Tools: `@JsonIgnoreProperties({"hibernateLazyInitializer","handler"})`, DTOs, or forcing initialization.
* If your entities lack an `id` (e.g., transient objects), prefer `IntSequenceGenerator`:

  ```java
  @JsonIdentityInfo(generator = ObjectIdGenerators.IntSequenceGenerator.class, property = "@id")
  ```
* If you only need one direction in your API, the simplest and cleanest option is often to **use DTOs** and shape the JSON explicitly, or just use `@JsonIgnore` on the back field.

---

**TL;DR**

* `@JsonManagedReference/@JsonBackReference`: drop the back link during serialization; great for trees.
* `@JsonIdentityInfo(...PropertyGenerator.class, property = "id")`: keep both directions using IDs; great for arbitrary graphs and cycles.

```java








```

# Example 2

---

# 📁 Starter project layout (Spring Boot + MapStruct + record DTOs)

```
src/
  main/
    java/com/example/contacts/
      entity/Contact.java
      dto/CreateContactRequest.java
      dto/UpdateContactRequest.java
      dto/ContactResponse.java
      mapper/ContactMapper.java
      repo/ContactRepository.java
      service/ContactService.java
      web/ContactController.java
      config/SecurityConfig.java
    resources/
      application.yml
```

> Uses: Spring Boot, Spring Web, Spring Data JPA, Validation, Spring Security (for `PasswordEncoder`), MapStruct, Lombok (optional).

---

## 🧩 Maven `pom.xml` (key parts)

```xml
<properties>
  <java.version>17</java.version>
  <mapstruct.version>1.5.5.Final</mapstruct.version>
  <spring-boot.version>3.3.4</spring-boot.version>
</properties>

<dependencies>
  <!-- Spring -->
  <dependency>
    <groupId>org.springframework.boot</groupId>
    <artifactId>spring-boot-starter-web</artifactId>
  </dependency>
  <dependency>
    <groupId>org.springframework.boot</groupId>
    <artifactId>spring-boot-starter-data-jpa</artifactId>
  </dependency>
  <dependency>
    <groupId>org.springframework.boot</groupId>
    <artifactId>spring-boot-starter-validation</artifactId>
  </dependency>
  <dependency>
    <groupId>org.springframework.boot</groupId>
    <artifactId>spring-boot-starter-security</artifactId>
  </dependency>

  <!-- MapStruct -->
  <dependency>
    <groupId>org.mapstruct</groupId>
    <artifactId>mapstruct</artifactId>
    <version>${mapstruct.version}</version>
  </dependency>
  <dependency>
    <groupId>org.mapstruct</groupId>
    <artifactId>mapstruct-processor</artifactId>
    <version>${mapstruct.version}</version>
    <scope>provided</scope>
  </dependency>

  <!-- DB (choose one; H2 shown for dev) -->
  <dependency>
    <groupId>com.h2database</groupId>
    <artifactId>h2</artifactId>
    <scope>runtime</scope>
  </dependency>

  <!-- Lombok (optional but handy) -->
  <dependency>
    <groupId>org.projectlombok</groupId>
    <artifactId>lombok</artifactId>
    <optional>true</optional>
  </dependency>

  <!-- Test -->
  <dependency>
    <groupId>org.springframework.boot</groupId>
    <artifactId>spring-boot-starter-test</artifactId>
    <scope>test</scope>
  </dependency>
</dependencies>

<build>
  <plugins>
    <plugin>
      <groupId>org.apache.maven.plugins</groupId>
      <artifactId>maven-compiler-plugin</artifactId>
      <version>3.11.0</version>
      <configuration>
        <source>${java.version}</source>
        <target>${java.version}</target>
        <annotationProcessorPaths>
          <path>
            <groupId>org.mapstruct</groupId>
            <artifactId>mapstruct-processor</artifactId>
            <version>${mapstruct.version}</version>
          </path>
          <path>
            <groupId>org.projectlombok</groupId>
            <artifactId>lombok</artifactId>
            <version>1.18.34</version>
          </path>
        </annotationProcessorPaths>
      </configuration>
    </plugin>
  </plugins>
</build>
```

---

## ⚙️ `application.yml` (dev)

```yaml
spring:
  datasource:
    url: jdbc:h2:mem:contactsdb;DB_CLOSE_DELAY=-1
    driver-class-name: org.h2.Driver
    username: sa
    password:
  jpa:
    hibernate:
      ddl-auto: update
    show-sql: true
    properties:
      hibernate:
        format_sql: true
  h2:
    console:
      enabled: true
```

---

## 🧱 Entity

`entity/Contact.java`

```java
package com.example.contacts.entity;

import jakarta.persistence.*;
import lombok.Getter;
import lombok.Setter;

@Entity
@Table(name = "contacts")
@Getter @Setter
public class Contact {
  @Id
  @GeneratedValue(strategy = GenerationType.UUID)
  private String id;

  private String name;
  private String photoUrl;
  private String email;

  // Never expose in responses
  private String password;
}
```

---

## 📨 DTOs (Java 17 records)

`dto/CreateContactRequest.java`

```java
package com.example.contacts.dto;

import jakarta.validation.constraints.Email;
import jakarta.validation.constraints.NotBlank;

public record CreateContactRequest(
    @NotBlank String name,
    @Email String email,
    String photoUrl,
    @NotBlank String password
) {}
```

`dto/UpdateContactRequest.java`

```java
package com.example.contacts.dto;

import jakarta.validation.constraints.Email;

public record UpdateContactRequest(
    String name,
    @Email String email,
    String photoUrl,
    String password // optional; if provided, will be encoded & updated
) {}
```

`dto/ContactResponse.java`

```java
package com.example.contacts.dto;

public record ContactResponse(
    String id,
    String name,
    String email,
    String photoUrl
) {}
```

---

## 🔁 MapStruct mapper

`mapper/ContactMapper.java`

```java
package com.example.contacts.mapper;

import com.example.contacts.dto.*;
import com.example.contacts.entity.Contact;
import org.mapstruct.*;

@Mapper(componentModel = "spring")
public interface ContactMapper {

  @Mapping(target = "id", ignore = true)
  Contact toEntity(CreateContactRequest req);

  @BeanMapping(nullValuePropertyMappingStrategy = NullValuePropertyMappingStrategy.IGNORE)
  void updateEntity(UpdateContactRequest req, @MappingTarget Contact target);

  ContactResponse toResponse(Contact entity);
}
```

---

## 🗄️ Repository

`repo/ContactRepository.java`

```java
package com.example.contacts.repo;

import com.example.contacts.entity.Contact;
import org.springframework.data.jpa.repository.JpaRepository;

public interface ContactRepository extends JpaRepository<Contact, String> {}
```

---

## 🔐 Security config (for `PasswordEncoder` bean)

`config/SecurityConfig.java`

```java
package com.example.contacts.config;

import org.springframework.context.annotation.Bean;
import org.springframework.context.annotation.Configuration;
import org.springframework.security.crypto.bcrypt.BCryptPasswordEncoder;
import org.springframework.security.crypto.password.PasswordEncoder;

@Configuration
public class SecurityConfig {
  @Bean
  public PasswordEncoder passwordEncoder() {
    return new BCryptPasswordEncoder();
  }
}
```

> If you don’t need full security yet, this is enough to get a `PasswordEncoder` bean without locking down endpoints.

---

## 🧠 Service

`service/ContactService.java`

```java
package com.example.contacts.service;

import com.example.contacts.dto.*;
import com.example.contacts.entity.Contact;
import com.example.contacts.mapper.ContactMapper;
import com.example.contacts.repo.ContactRepository;
import lombok.RequiredArgsConstructor;
import org.springframework.security.crypto.password.PasswordEncoder;
import org.springframework.stereotype.Service;

@Service
@RequiredArgsConstructor
public class ContactService {
  private final ContactRepository repo;
  private final ContactMapper mapper;
  private final PasswordEncoder encoder;

  public ContactResponse create(CreateContactRequest req) {
    Contact entity = mapper.toEntity(req);
    entity.setPassword(encoder.encode(req.password()));
    return mapper.toResponse(repo.save(entity));
  }

  public ContactResponse update(String id, UpdateContactRequest req) {
    Contact entity = repo.findById(id).orElseThrow();
    mapper.updateEntity(req, entity);
    if (req.password() != null && !req.password().isBlank()) {
      entity.setPassword(encoder.encode(req.password()));
    }
    return mapper.toResponse(repo.save(entity));
  }

  public ContactResponse get(String id) {
    return mapper.toResponse(repo.findById(id).orElseThrow());
  }

  public void delete(String id) {
    repo.deleteById(id);
  }
}
```

---

## 🌐 Controller

`web/ContactController.java`

```java
package com.example.contacts.web;

import com.example.contacts.dto.*;
import com.example.contacts.service.ContactService;
import jakarta.validation.Valid;
import lombok.RequiredArgsConstructor;
import org.springframework.web.bind.annotation.*;

@RestController
@RequestMapping("/api/contacts")
@RequiredArgsConstructor
public class ContactController {
  private final ContactService service;

  @PostMapping
  public ContactResponse create(@Valid @RequestBody CreateContactRequest req) {
    return service.create(req);
  }

  @PatchMapping("/{id}")
  public ContactResponse update(@PathVariable String id,
                                @RequestBody UpdateContactRequest req) {
    return service.update(id, req);
  }

  @GetMapping("/{id}")
  public ContactResponse get(@PathVariable String id) {
    return service.get(id);
  }

  @DeleteMapping("/{id}")
  public void delete(@PathVariable String id) {
    service.delete(id);
  }
}
```

---

# 📝 README – DTOs vs Entities (Quick Cheat Sheet)

Save as `dto-vs-entity.md` in your repo:

```markdown
# Spring Boot – DTOs vs Entities (Cheat Sheet)

**Entities** = persistence model (DB).  
**DTOs** = API contract (request/response).  
**Jackson annotations ≠ DTOs** (formatting vs contract).

## When you can skip DTOs
Small/internal apps with simple entities — use @JsonIgnore, handle cycles.  
**Risks**: over-posting, API breaks on DB change, lazy init traps.

## Why use DTOs
- Security: exclude password, id, isAdmin, audit fields.
- Stability: entities evolve, DTOs stay stable.
- Tailored shapes per endpoint.
- Validation on requests (`@NotBlank`, `@Email`).
- Performance: avoid big graphs, cycles.

## Practical setup
- **MapStruct + record DTOs** (recommended).
- Request DTOs validate intent; Response DTOs hide sensitive fields.
- Partial updates: MapStruct `NullValuePropertyMappingStrategy.IGNORE`.
- Encode passwords in service (`PasswordEncoder`).
- For reads, consider Spring Data projections for lightweight payloads.

## Gotchas
- Don’t return entities with lazy relations directly.
- Avoid blacklisting fields; prefer DTO whitelisting.
- Versioning: add v2 DTOs instead of breaking existing ones.
```

---

## 🧪 Quick test (curl)

```bash
# Create
curl -X POST http://localhost:8080/api/contacts \
  -H "Content-Type: application/json" \
  -d '{"name":"Alice","email":"alice@example.com","photoUrl":null,"password":"s3cr3t"}'

# Get
curl http://localhost:8080/api/contacts/{id}

# Update (partial)
curl -X PATCH http://localhost:8080/api/contacts/{id} \
  -H "Content-Type: application/json" \
  -d '{"photoUrl":"https://cdn/img.png"}'
```







