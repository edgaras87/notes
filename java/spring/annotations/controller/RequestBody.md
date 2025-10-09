# What `@RequestBody` Does

* **Purpose:** Bind the **HTTP request body** to a controller method parameter.
* **How:** Delegates to **HttpMessageConverters** (Jackson for JSON by default) to **deserialize** bytes → Java object.
* **Content-type aware:** Chooses a converter based on `Content-Type` (e.g., `application/json`, `application/xml`, `text/plain`, etc.).

```java
@PostMapping("/contacts")
public Contact create(@RequestBody Contact contact) { ... }
```

# Lifecycle & Flow

1. Client sends body + `Content-Type`.
2. Spring picks the **first compatible HttpMessageConverter**.
3. Converter **reads** the body and **converts** into the parameter type.
4. (Optional) **Validation** runs if you add `@Valid` / `@Validated`.
5. Controller logic executes with the hydrated object.

# Key Options & Defaults

* **Required:** `@RequestBody` is **required by default** → missing body ⇒ 400.

    * Make optional: `@RequestBody(required = false)` (parameter may be `null`).
* **Validation:** Add `@Valid` and bean validation annotations (`jakarta.validation`).

    * Failures throw `MethodArgumentNotValidException` → typically 400 (customize with `@ControllerAdvice`).
* **Unknown fields:** Jackson behavior controlled by `ObjectMapper` (e.g., `FAIL_ON_UNKNOWN_PROPERTIES`).

# Supported Parameter Shapes

* **Domain types / DTOs:** Typical case.
* **Collections / arrays:** e.g., `List<Contact>`.
* **Maps:** `Map<String,Object>` for ad-hoc JSON.
* **Raw types:** `String`, `byte[]`, `InputStream`, `Resource` (for streaming).
* **Records / Lombok / Kotlin data classes:** Fully supported with Jackson.

# Common Converters (MVC)

* **MappingJackson2HttpMessageConverter** (JSON ↔ POJO)
* **StringHttpMessageConverter** (text)
* **ByteArrayResource/ResourceHttpMessageConverter** (binary/streams)
* **Jaxb2RootElementHttpMessageConverter** (XML, if on classpath)

(You can register custom ones via `WebMvcConfigurer#extendMessageConverters`.)

# Error Handling You’ll See

* **400 Bad Request** for:

    * Missing body (when required),
    * Malformed JSON (`HttpMessageNotReadableException`),
    * Validation failures (`MethodArgumentNotValidException`).
* **415 Unsupported Media Type:** `Content-Type` not supported.
* **406 Not Acceptable** (response side) if you also do content negotiation for the return type.

Customize responses with:

```java
@RestControllerAdvice
class GlobalErrors {
  @ExceptionHandler(MethodArgumentNotValidException.class)
  @ResponseStatus(HttpStatus.BAD_REQUEST)
  ApiError handleValidation(MethodArgumentNotValidException ex) { ... }
}
```

# Best Practices

* **Use DTOs, not entities** for request bodies (prevents over-posting, stabilizes API).
* **Validate aggressively:** `@Valid` + constraints (`@NotBlank`, `@Email`, etc.).
* **Document required fields** and payload shape (OpenAPI/Swagger).
* **Return saved instance/DTO** and set `Location` for creates (201 Created).
* **Harden ObjectMapper:** decide on unknown properties, null handling, date formats, etc.
* **Use `@JsonView` or separate DTOs** when you need different read/write projections.

# `@RequestBody` vs. Others

* **`@PathVariable`**: pulls from the URL path, not the body.
* **`@RequestParam`**: query string/form fields, not the body JSON.
* **`@ModelAttribute`**: binds from form data/params; not raw JSON (unless combined differently).
* **`@RequestPart`**: for **multipart/form-data** parts (e.g., JSON + file upload).

# Multipart & Files

* For uploads like “JSON + file” use:

```java
@PostMapping(consumes = MediaType.MULTIPART_FORM_DATA_VALUE)
public void upload(@RequestPart("meta") MyDto meta,
                   @RequestPart("file") MultipartFile file) { ... }
```

`@RequestBody` alone is **not** for multipart.

# Response Side Symmetry

* **`@ResponseBody`** (or `@RestController`) serializes return values to the response using the **same** converter family (e.g., Jackson for JSON).
* Content negotiation uses the request `Accept` header to select format.

# Advanced Customization

* **Custom converters:** implement `HttpMessageConverter<T>` for exotic media types.
* **Per-endpoint ObjectMapper tweaks:** `@JsonView`, mix-ins, `@JsonDeserialize`, `@JsonFormat`.
* **Streaming:** accept `InputStream` / `Reader` for huge payloads; parse manually.
* **Generics:** for generic containers, Spring/Jackson can preserve type info with `@RequestBody List<MyType>` or `HttpEntity<List<MyType>>`.

# Typical Patterns

**Create with validation + Location header**

```java
@PostMapping
public ResponseEntity<ContactDto> create(@Valid @RequestBody ContactCreateDto in) {
  Contact saved = service.create(map(in));
  URI location = ServletUriComponentsBuilder.fromCurrentRequest()
                    .path("/{id}").buildAndExpand(saved.getId()).toUri();
  return ResponseEntity.created(location).body(map(saved));
}
```

**Bulk create**

```java
@PostMapping("/bulk")
public List<ContactDto> bulk(@Valid @RequestBody List<ContactCreateDto> items) { ... }
```

**Optional body**

```java
@PostMapping("/maybe")
public ResponseEntity<Void> maybe(@RequestBody(required = false) Foo body) {
  if (body == null) return ResponseEntity.noContent().build();
  ...
}
```

# Gotchas & Pitfalls

* **Double-creating** by calling your service twice (once to save, once again in `body(...)`).
* **Wrong `Content-Type`**: client sends JSON but forgets `application/json`.
* **Silently ignored fields** if your mapper allows unknown properties and you expect strictness.
* **Binding to entities** may expose writeable fields you didn’t intend (prefer DTOs).
* **Time/Zone formats**: standardize ISO-8601; configure Jackson `JavaTimeModule`.

# WebFlux Note

* In reactive controllers, use `Mono<T>` / `Flux<T>` with `@RequestBody`:

```java
@PostMapping
public Mono<Contact> create(@RequestBody Mono<Contact> contactMono) { ... }
```

---

**TL;DR:** `@RequestBody` takes the raw request body and, using **HttpMessageConverters** (Jackson for JSON), **deserializes it into your parameter type**, optionally **validates** it, and surfaces errors as **400/415**. Use DTOs, add `@Valid`, set the correct `Content-Type`, and craft clear error responses.
