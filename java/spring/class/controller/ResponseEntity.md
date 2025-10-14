# Java `ResponseEntity` — Beginner-Friendly Cheatsheet

## What it is (and why it’s useful)

`ResponseEntity<T>` is Spring’s simple wrapper for an HTTP response: **status + headers + optional body**.
Use it when you need precise control over status codes (e.g., `201`, `204`, `404`), headers (`Location`, `ETag`, caching), or when your response body type varies.

---

## Quick glossary

* **Status**: HTTP code (e.g., `200 OK`, `404 Not Found`) describing the result.
* **Headers**: Key–value metadata (e.g., `Content-Type`, `Location`, `Cache-Control`).
* **Body**: The actual payload (`T`), often JSON.
* **Builder**: Fluent API to set status/headers/body (`ok()`, `status(...)`, `created(...)`).
* **Empty**: A response with no body (often `204 No Content`).

---

## Creating instances

### 1) Quick OK with body

```java
return ResponseEntity.ok(Map.of("message", "hello"));
```

**Output (typical)**
Status: `200 OK`
Headers: `Content-Type: application/json`
Body: `{"message":"hello"}`

### 2) Custom status

```java
return ResponseEntity.status(HttpStatus.ACCEPTED).body("queued");
```

**Output**: `202 Accepted`, Body: `"queued"`

### 3) Created with Location

```java
URI uri = URI.create("/api/items/42");
return ResponseEntity.created(uri).body(Map.of("id", 42));
```

**Output**: `201 Created`, `Location: /api/items/42`, Body: `{"id":42}`

### 4) No content

```java
return ResponseEntity.noContent().build();
```

**Output**: `204 No Content`, Body: *none*

### 5) From `Optional<T>` (not found vs ok)

```java
return ResponseEntity.of(repo.findById(id));  // OK(200) if present, else 404
```

**Output**: `200 OK` + body *or* `404 Not Found` (empty body)

### 6) With headers (ETag, caching)

```java
return ResponseEntity.ok()
        .eTag("\"v1\"")                    // must be quoted per RFC
        .cacheControl(CacheControl.maxAge(Duration.ofMinutes(5)))
        .body(Map.of("data", 123));
```

**Output**: `200 OK`, `ETag: "v1"`, `Cache-Control: max-age=300`, Body: `{"data":123}`

---

## Reading state / accessors (inside tests/services)

```java
ResponseEntity<String> resp = ResponseEntity.ok("hi");
HttpStatusCode sc = resp.getStatusCode(); // Spring 6+: HttpStatusCode
HttpHeaders headers = resp.getHeaders();
String body = resp.getBody();
boolean has = resp.hasBody(); // inherited from HttpEntity
```

**Typical values**: `sc=200`, `headers` contains `Content-Type`, `body="hi"`, `has=true`

---

## Checking properties

```java
ResponseEntity<Void> r = ResponseEntity.noContent().build();
boolean ok = r.getStatusCode().is2xxSuccessful();     // true
boolean redir = r.getStatusCode().is3xxRedirection(); // false
boolean err = r.getStatusCode().isError();            // false (Spring 6+)
```

**Output**: `ok=true`, `redir=false`, `err=false`

---

## Transformations (pure, create a new response)

> There’s no built-in `map()` on all versions. Just rebuild.

```java
ResponseEntity<User> r = service.getUser(id); // 200 or 404
if (!r.getStatusCode().is2xxSuccessful() || !r.hasBody()) {
    return ResponseEntity.status(r.getStatusCode()).headers(r.getHeaders()).build();
}
UserDto dto = toDto(r.getBody());
return ResponseEntity.status(r.getStatusCode()).headers(r.getHeaders()).body(dto);
```

**Effect**: Preserves status/headers, transforms body `User -> UserDto`.

---

## Conversions (to other types)

* **Controller return**: `ResponseEntity<T>` is directly returned by controller methods.
* **Body only**: When you don’t need headers/status control, return `T` and let Spring default to `200 OK`.
* **String view** (debugging)

  ```java
  ResponseEntity<String> r = ResponseEntity.ok("hi");
  r.toString(); // e.g., <200 OK OK,[],hi>
  ```

  **Output**: A readable summary (status, headers, body).

---

## Iteration & comparison

```java
ResponseEntity<String> r = ResponseEntity.ok().header("X-Trace","abc").body("hi");
r.getHeaders().forEach((k, v) -> System.out.println(k + "=" + v));

ResponseEntity<String> r2 = ResponseEntity.ok().header("X-Trace","abc").body("hi");
boolean eq = r.equals(r2);       // true (compares status+headers+body)
int h = r.hashCode();            // consistent with equals
```

**Output**:

```
X-Trace=[abc]
(eq=true)
```

---

## Common utilities you’ll often pair with `ResponseEntity`

* `HttpStatus` / `HttpStatusCode`: status constants & helpers.
* `HttpHeaders`: header names; build via `new HttpHeaders()` or fluent on builders.
* `MediaType`: set `Content-Type`.

  ```java
  return ResponseEntity.ok()
      .contentType(MediaType.APPLICATION_JSON)
      .body(Map.of("k", 1));
  ```
* `CacheControl`: caching builders (`noStore()`, `maxAge(...)`).
* `ResponseEntityExceptionHandler`: centralized exception → `ResponseEntity`.

---

## Gotchas / anti-patterns / version notes

* **200 with empty body vs 204**: If nothing to return, prefer `204 No Content` (not `200` with `null`).
* **Null body serialization**: Returning `200` with `null` body can produce `null` JSON or no content depending on converter—be explicit.
* **Headers mutability**: `HttpHeaders` is mutable. Prefer setting headers **via the builder** (`ok().header(...)`) rather than mutating `getHeaders()` after build.
* **ETag must be quoted**: use `"\"value\""`; otherwise conditional requests won’t work.
* **Location must be absolute in some clients**: Spring accepts relative URIs, but many HTTP clients expect absolute (`https://.../resource/42`).
* **Spring 6 / Boot 3 move**:

    * `getStatusCode()` returns `HttpStatusCode` (not `HttpStatus`).
    * Baseline Java 17; Jakarta namespace (`jakarta.*`), not `javax.*`.
* **Don’t overuse `ResponseEntity`**: If you always return `200` and no custom headers, just return the body type.

---

## Mini reference table

| Method / Builder         | What it does                      | Example (input → output)                                                  |
| ------------------------ | --------------------------------- | ------------------------------------------------------------------------- |
| `ok(body)`               | `200 OK` with body                | `"hi"` → `200 OK`, Body: `"hi"`                                           |
| `status(code).body(x)`   | Custom status with body           | `HttpStatus.ACCEPTED, "queued"` → `202 Accepted`, `"queued"`              |
| `noContent().build()`    | No body response                  | *none* → `204 No Content`                                                 |
| `created(uri).body(x)`   | `201 Created` + `Location` header | `"/api/items/42", {"id":42}` → `201`, `Location:/api/items/42`, body JSON |
| `of(Optional<T>)`        | `200` if present else `404`       | `Optional.empty()` → `404 Not Found`                                      |
| `.header(k,v)`           | Add a header                      | `("X-Id","a1")` → `X-Id:a1`                                               |
| `.contentType(mt)`       | Set `Content-Type`                | `MediaType.APPLICATION_JSON`                                              |
| `.eTag(tag)`             | Set `ETag` (quoted)               | `"\"v1\""` → `ETag:"v1"`                                                  |
| `.lastModified(Instant)` | Set `Last-Modified`               | `Instant.now()`                                                           |
| `.cacheControl(cc)`      | Set caching                       | `CacheControl.maxAge(300s)` → `Cache-Control:max-age=300`                 |

---

## End-to-end example (controller + typical outputs)

```java
@RestController
@RequestMapping("/api/items")
class ItemController {

    private final ItemService service;

    ItemController(ItemService service) { this.service = service; }

    // GET /api/items/{id}
    @GetMapping("/{id}")
    public ResponseEntity<ItemDto> get(@PathVariable long id) {
        return ResponseEntity.of(service.find(id)); // Optional<ItemDto>
    }

    // POST /api/items
    @PostMapping
    public ResponseEntity<ItemDto> create(@RequestBody NewItem newItem) {
        ItemDto saved = service.create(newItem);
        URI uri = URI.create("/api/items/" + saved.id());
        return ResponseEntity.created(uri)
                .eTag("\"" + saved.version() + "\"")
                .body(saved);
    }

    // PUT /api/items/{id}
    @PutMapping("/{id}")
    public ResponseEntity<Void> update(@PathVariable long id, @RequestBody UpdateItem u) {
        boolean existed = service.update(id, u);
        return existed ? ResponseEntity.noContent().build() : ResponseEntity.notFound().build();
    }
}
```

**Typical outputs**

* `GET /api/items/42` (found):
  `200 OK`
  Body: `{"id":42,"name":"Book","version":"v3"}`

* `GET /api/items/999` (missing):
  `404 Not Found`
  Body: *empty*

* `POST /api/items` with `{"name":"Pen"}`:
  `201 Created`
  `Location: /api/items/43`
  `ETag: "v1"`
  Body: `{"id":43,"name":"Pen","version":"v1"}`

* `PUT /api/items/43` with updates (exists):
  `204 No Content` (no body)

---

## Bottom line (best practices)

* **Use `ResponseEntity`** when you need non-200 statuses, custom headers, conditional/caching, or dynamic body/emptiness.
* **Prefer builders**: `ok()`, `created()`, `noContent()`, `status(...)`.
* **Use `of(Optional)`** to compact “found vs not found”.
* **Choose the right empty semantics**: `204` for successful “nothing to return”; `404` when resource doesn’t exist.
* **Be explicit with headers**: quote ETags, provide absolute `Location` when possible, set `Cache-Control` deliberately.
* **Keep it simple**: If you always return `200` + body and no headers, return the body type instead of wrapping in `ResponseEntity`.

---

### Copy-paste starter snippets

**Standard OK JSON**

```java
return ResponseEntity.ok(Map.of("status", "ok"));
```

**Created with Location + ETag**

```java
URI loc = URI.create("/api/items/" + id);
return ResponseEntity.created(loc).eTag("\"" + version + "\"").body(dto);
```

**Not found**

```java
return ResponseEntity.notFound().build();
```

**Conditional from Optional**

```java
return ResponseEntity.of(optionalDto);
```

**No content**

```java
return ResponseEntity.noContent().build();
```

**Custom headers**

```java
return ResponseEntity.ok()
    .header(HttpHeaders.X_REQUEST_ID, requestId)
    .cacheControl(CacheControl.noStore())
    .body(payload);
```
