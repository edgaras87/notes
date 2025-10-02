# 📝 Minimal Java/Spring Muscle-Memory Checklist (with Examples)

### ✅ REST basics

```java
@RestController
@RequestMapping("/api/hello")
public class HelloController {
    @GetMapping
    public String sayHello() {
        return "Hello, World!";
    }
}
```

---

### ✅ DTOs (use instead of Entities for I/O)

```java
// Input DTO with validation
public record UserDto(
    @NotBlank String name,
    @Email String email
) {}

// In controller
@PostMapping("/users")
public UserDto createUser(@Valid @RequestBody UserDto dto) {
    return dto; // later convert to Entity, save, return response DTO
}
```

👉 **Note:** Use DTOs for **input validation** and **API responses**.
Don’t expose Entities directly — keep Entities for persistence logic only.

👉 **Why DTOs instead of Entities for input/return?**

* **Entities** = your database model (tables, persistence rules).
* **DTOs (Data Transfer Objects)** = what you expose to the outside world (API requests & responses).

### Why not use entities directly?

* **Security** → prevents exposing internal DB fields (e.g. passwords, IDs).
* **Flexibility** → API contract can evolve without breaking your DB schema.
* **Validation** → DTOs are designed for request/response shape, so they fit `@Valid` checks.
* **Separation of concerns** → Entities = persistence, DTOs = communication. Keeps your code cleaner.

### Typical pattern

* **Incoming request**: DTO with `@Valid` (only the fields you expect from client).
* **Service layer**: Map DTO → Entity, persist.
* **Outgoing response**: DTO again (so you don’t return raw Entity).

---

### ✅ Service + Dependency Injection

```java
@Service
public class UserService {
    public String welcomeUser(String name) {
        return "Welcome, " + name;
    }
}
```

```java
@RestController
@RequiredArgsConstructor
@RequestMapping("/api/users")
public class UserController {
    private final UserService userService;

    @GetMapping("/{name}")
    public String welcome(@PathVariable String name) {
        return userService.welcomeUser(name);
    }
}
```

---

### ✅ Entity + Repository

```java
@Entity
public class User {
    @Id
    @GeneratedValue
    private Long id;

    @Column(nullable = false)
    private String name;

    private String email;
}

public interface UserRepository extends JpaRepository<User, Long> {}
```

---

### ✅ `application.yml`

```yaml
spring:
  datasource:
    url: jdbc:h2:mem:testdb
    driver-class-name: org.h2.Driver
    username: sa
    password:
  jpa:
    hibernate:
      ddl-auto: update
    show-sql: true
```

---

### ✅ JUnit Test

```java
@SpringBootTest
class UserServiceTest {

    @Autowired
    private UserService service;

    @Test
    void testWelcomeUser() {
        assertEquals("Welcome, Alice", service.welcomeUser("Alice"));
    }
}
```

---

### ✅ Running

```bash
./gradlew bootRun
./gradlew test
```

---

👉 **Muscle-memory loop:**

* Controller → DTO (`@Valid`)
* Service → business logic
* Entity + Repo → persistence
* Return response DTO
* Config in `application.yml`
* One test + run

---







# 📦 Product Inventory – Example (Java + Spring Boot)

here’s a **broader, realistic example** that shows:

* an **Entity** with more fields,
* **DTOs** for input/output with validation,
* a **Service** layer with business rules,
* a **Repository** and **Controller**, plus a tiny **mapper**.

Domain: simple **Product** inventory.

---

## 1) Entity (persistence model)

```java
// src/main/java/com/example/product/Product.java
package com.example.product;

import jakarta.persistence.*;
import java.time.Instant;

@Entity
@Table(name = "products", indexes = {
    @Index(name = "idx_products_sku_unique", columnList = "sku", unique = true)
})
public class Product {

    public enum Status { DRAFT, ACTIVE, DISCONTINUED }

    @Id
    @GeneratedValue(strategy = GenerationType.IDENTITY)
    private Long id;

    @Column(nullable = false, length = 64, unique = true)
    private String sku; // business key, unique

    @Column(nullable = false, length = 160)
    private String name;

    @Column(length = 2000)
    private String description;

    @Column(nullable = false)
    private Long priceCents; // store money as integer

    @Column(nullable = false, length = 3)
    private String currency; // "EUR", "USD"

    @Column(nullable = false)
    private Integer stock; // >= 0

    @Enumerated(EnumType.STRING)
    @Column(nullable = false, length = 20)
    private Status status = Status.DRAFT;

    @Column(nullable = false, updatable = false)
    private Instant createdAt;

    @Column(nullable = false)
    private Instant updatedAt;

    @PrePersist
    void prePersist() {
        var now = Instant.now();
        createdAt = now;
        updatedAt = now;
    }

    @PreUpdate
    void preUpdate() {
        updatedAt = Instant.now();
    }

    // getters/setters omitted for brevity
    // ...
}
```

> Note: we keep entity constraints mostly DB-related (nullability, lengths, uniqueness). **Validation annotations for requests live on DTOs**.

---

## 2) DTOs (request/response) + validation

```java
// src/main/java/com/example/product/dto/ProductCreateDto.java
package com.example.product.dto;

import jakarta.validation.constraints.*;

public record ProductCreateDto(
    @NotBlank @Size(max = 64)
    String sku,

    @NotBlank @Size(max = 160)
    String name,

    @Size(max = 2000)
    String description,

    @NotNull @Positive
    Long priceCents,

    @NotBlank @Pattern(regexp = "^[A-Z]{3}$", message = "Use ISO currency like EUR/USD")
    String currency,

    @NotNull @PositiveOrZero
    Integer stock
) {}
```

```java
// src/main/java/com/example/product/dto/ProductUpdateDto.java
package com.example.product.dto;

import jakarta.validation.constraints.*;
import com.example.product.Product.Status;

public record ProductUpdateDto(
    @Size(max = 160)
    String name,

    @Size(max = 2000)
    String description,

    @Positive
    Long priceCents,

    @Pattern(regexp = "^[A-Z]{3}$")
    String currency,

    @PositiveOrZero
    Integer stock,

    Status status
) {}
```

```java
// src/main/java/com/example/product/dto/ProductResponseDto.java
package com.example.product.dto;

import java.time.Instant;
import com.example.product.Product.Status;

public record ProductResponseDto(
    Long id,
    String sku,
    String name,
    String description,
    Long priceCents,
    String currency,
    Integer stock,
    Status status,
    Instant createdAt,
    Instant updatedAt
) {}
```

---

## 3) Mapper (Entity ↔ DTO)

```java
// src/main/java/com/example/product/ProductMapper.java
package com.example.product;

import com.example.product.dto.*;

public class ProductMapper {

    public static Product toEntity(ProductCreateDto dto) {
        var p = new Product();
        p.setSku(dto.sku().trim());
        p.setName(dto.name().trim());
        p.setDescription(dto.description());
        p.setPriceCents(dto.priceCents());
        p.setCurrency(dto.currency());
        p.setStock(dto.stock());
        p.setStatus(Product.Status.DRAFT);
        return p;
    }

    public static void applyUpdate(Product p, ProductUpdateDto dto) {
        if (dto.name() != null) p.setName(dto.name().trim());
        if (dto.description() != null) p.setDescription(dto.description());
        if (dto.priceCents() != null) p.setPriceCents(dto.priceCents());
        if (dto.currency() != null) p.setCurrency(dto.currency());
        if (dto.stock() != null) p.setStock(dto.stock());
        if (dto.status() != null) p.setStatus(dto.status());
    }

    public static ProductResponseDto toResponse(Product p) {
        return new ProductResponseDto(
            p.getId(), p.getSku(), p.getName(), p.getDescription(),
            p.getPriceCents(), p.getCurrency(), p.getStock(),
            p.getStatus(), p.getCreatedAt(), p.getUpdatedAt()
        );
    }
}
```

---

## 4) Repository

```java
// src/main/java/com/example/product/ProductRepository.java
package com.example.product;

import org.springframework.data.jpa.repository.JpaRepository;
import java.util.Optional;

public interface ProductRepository extends JpaRepository<Product, Long> {
    Optional<Product> findBySku(String sku);
    boolean existsBySku(String sku);
}
```

---

## 5) Service (business rules)

```java
// src/main/java/com/example/product/ProductService.java
package com.example.product;

import com.example.product.dto.*;
import jakarta.transaction.Transactional;
import lombok.RequiredArgsConstructor;
import org.springframework.stereotype.Service;

@Service
@RequiredArgsConstructor
public class ProductService {

    private final ProductRepository repo;

    @Transactional
    public ProductResponseDto create(ProductCreateDto dto) {
        // business rule: SKU must be unique
        if (repo.existsBySku(dto.sku())) {
            throw new IllegalArgumentException("SKU already exists: " + dto.sku());
        }

        // business rule: enforce EUR only (example)
        if (!"EUR".equals(dto.currency())) {
            throw new IllegalArgumentException("Only EUR supported for now.");
        }

        var entity = ProductMapper.toEntity(dto);

        // business rule: newly created ACTIVE products must have stock > 0
        // (we start as DRAFT; activate via update when ready)

        var saved = repo.save(entity);
        return ProductMapper.toResponse(saved);
    }

    @Transactional
    public ProductResponseDto update(Long id, ProductUpdateDto dto) {
        var product = repo.findById(id).orElseThrow(() -> new IllegalArgumentException("Product not found"));

        // business rule: stock cannot be negative
        if (dto.stock() != null && dto.stock() < 0) {
            throw new IllegalArgumentException("Stock cannot be negative");
        }

        // business rule: if switching to ACTIVE, ensure stock > 0 and has price
        if (dto.status() == Product.Status.ACTIVE) {
            var newStock = dto.stock() != null ? dto.stock() : product.getStock();
            var newPrice = dto.priceCents() != null ? dto.priceCents() : product.getPriceCents();
            if (newStock == null || newStock <= 0) throw new IllegalArgumentException("ACTIVE requires stock > 0");
            if (newPrice == null || newPrice <= 0) throw new IllegalArgumentException("ACTIVE requires price > 0");
        }

        ProductMapper.applyUpdate(product, dto);
        var saved = repo.save(product);
        return ProductMapper.toResponse(saved);
    }

    public ProductResponseDto getById(Long id) {
        return repo.findById(id)
                   .map(ProductMapper::toResponse)
                   .orElseThrow(() -> new IllegalArgumentException("Product not found"));
    }
}
```

---

## 6) Controller

```java
// src/main/java/com/example/product/ProductController.java
package com.example.product;

import com.example.product.dto.*;
import jakarta.validation.Valid;
import lombok.RequiredArgsConstructor;
import org.springframework.http.ResponseEntity;
import org.springframework.web.bind.annotation.*;

@RestController
@RequestMapping("/api/products")
@RequiredArgsConstructor
public class ProductController {

    private final ProductService service;

    @PostMapping
    public ResponseEntity<ProductResponseDto> create(@Valid @RequestBody ProductCreateDto dto) {
        return ResponseEntity.ok(service.create(dto));
    }

    @PatchMapping("/{id}")
    public ResponseEntity<ProductResponseDto> update(
            @PathVariable Long id,
            @Valid @RequestBody ProductUpdateDto dto
    ) {
        return ResponseEntity.ok(service.update(id, dto));
    }

    @GetMapping("/{id}")
    public ResponseEntity<ProductResponseDto> get(@PathVariable Long id) {
        return ResponseEntity.ok(service.getById(id));
    }
}
```

---

## 7) Basic config for quick runs

```yaml
# src/main/resources/application.yml
spring:
  datasource:
    url: jdbc:h2:mem:demo;MODE=PostgreSQL;DATABASE_TO_LOWER=TRUE;DEFAULT_NULL_ORDERING=HIGH
    driver-class-name: org.h2.Driver
    username: sa
    password:
  jpa:
    hibernate:
      ddl-auto: update
    show-sql: true
  jackson:
    serialization:
      WRITE_DATES_AS_TIMESTAMPS: false
```

---

## 8) Minimal test (service)

```java
// src/test/java/com/example/product/ProductServiceTest.java
package com.example.product;

import com.example.product.dto.ProductCreateDto;
import org.junit.jupiter.api.Test;
import org.springframework.beans.factory.annotation.Autowired;
import org.springframework.boot.test.context.SpringBootTest;

import static org.junit.jupiter.api.Assertions.*;

@SpringBootTest
class ProductServiceTest {

    @Autowired
    ProductService service;

    @Test
    void create_ok() {
        var dto = new ProductCreateDto("SKU-123", "Mouse", "Ergonomic", 2999L, "EUR", 10);
        var res = service.create(dto);
        assertNotNull(res.id());
        assertEquals("SKU-123", res.sku());
        assertEquals(10, res.stock());
    }
}
```

---

## 🔑 Key takeaways (tie back to your note)

* **Use DTOs** for input (`@Valid`) and output; **keep Entities internal** to persistence.
* **Service layer** = place for business rules (unique SKU, currency support, state transitions).
* **Entity** holds DB-focused constraints; **DTO** holds request/response validation.
* If you can wire: `Controller → DTO → Service → Repo/Entity → Response DTO`, plus `application.yml` and one **test**, you’ve hit the muscle-memory loop.
