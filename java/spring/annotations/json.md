# jackson in one sentence

Jackson turns **Java objects ⇄ JSON**. It works out of the box, and you use **annotations on your DTOs** when the default JSON isn’t what you want.

---

# 1) `@JsonProperty` — rename or control access

**Why**: your Java field names don’t match the API JSON, or you want a field to be read-only/write-only in JSON.

**How it affects direction**

* Serialization (object → JSON): field **appears** using the name you give.
* Deserialization (JSON → object): field **is read** from that JSON name.

```java
public class UserDto {
  @JsonProperty("user_id")
  private Long id;

  @JsonProperty(value = "email", access = JsonProperty.Access.WRITE_ONLY)
  private String email; // accepted on input but hidden in output
}
```

**Input → Object**

```json
{ "user_id": 1, "email": "a@b.com" }
```

**Object → Output**

```json
{ "user_id": 1 }
```

**Handy flags (max 5)**

* `value = "name"` → JSON field name.
* `access = READ_ONLY` → show in output only.
* `access = WRITE_ONLY` → accept on input only.
* `required = true` → fail if missing on input.
* `defaultValue = "..."` → used if absent on input.

---

# 2) `@JsonInclude` — drop noise (nulls, empties, defaults)

**Why**: make responses smaller/cleaner.

```java
@JsonInclude(JsonInclude.Include.NON_NULL)
public class ProfileDto {
  private String name;
  private String bio; // null → omitted
}
```

**Object → Output**

```json
{ "name": "Lin" }
```

**Useful modes (pick what you need)**

1. `ALWAYS` — include everything (default).
2. `NON_NULL` — drop only `null`.
3. `NON_EMPTY` — drop `null`, `""`, `[]`, `{}`.
4. `NON_DEFAULT` — drop values equal to field defaults (e.g., `0`, `false`, empty).
5. `NON_ABSENT` — drop `Optional.empty()` (keeps non-empty Optionals).

**Per-field override**: you can also put `@JsonInclude(...)` directly on a field.

---

# 3) `@JsonIgnore` / `@JsonIgnoreProperties` — hide or tolerate extras

**Why**: hide sensitive/internal fields, or ignore unknown JSON fields so clients don’t break you.

```java
@JsonIgnoreProperties(ignoreUnknown = true) // ignore extra JSON input
public class AccountDto {
  private String email;

  @JsonIgnore // never show in JSON
  private String passwordHash;
}
```

**Input with extras → Object**

```json
{ "email": "a@b.com", "unexpected": 123 }
```

**Object → Output**

```json
{ "email": "a@b.com" }
```

**Useful options (max 5)**

* `ignoreUnknown = true` — skip unexpected input fields.
* `value = {"field1","field2"}` — ignore these by name.
* `allowGetters = true` — still serialize (getters allowed).
* `allowSetters = true` — still deserialize (setters allowed).
* `@JsonIgnore` (field or getter) — hard hide that one property.

> Tip: prefer `@JsonIgnore` for single fields; use `@JsonIgnoreProperties` for class-level rules.

---

# 4) `@JsonFormat` — make dates predictable

**Why**: you control the exact text format instead of gambling on defaults.

```java
public class EventDto {
  @JsonFormat(shape = JsonFormat.Shape.STRING,
              pattern = "yyyy-MM-dd'T'HH:mm:ssXXX")
  private ZonedDateTime startsAt;
}
```

**Object → Output**

```json
{ "startsAt": "2025-10-04T09:30:00+03:00" }
```

**Useful options (max 5)**

* `shape = STRING` — format as text (common for dates).
* `shape = NUMBER` — epoch millis/seconds (with `@JsonFormat` + config).
* `pattern = "..."` — custom date pattern.
* `timezone = "UTC"` — force a TZ (or `Europe/Vilnius`).
* `locale = "en"` — locale for month/day names, etc.

---

# 5) `@JsonAlias` — accept many input names, output one

**Why**: smooth migrations: multiple incoming names map to one field.

```java
public class ProductDto {
  @JsonProperty("price")                // output as "price"
  @JsonAlias({"cost", "amount"})        // accept these on input
  private BigDecimal price;
}
```

**Valid Inputs**

```json
{ "price": 10.99 }    { "cost": 10.99 }    { "amount": 10.99 }
```

**Object → Output**

```json
{ "price": 10.99 }
```

---

# 6) `@JsonCreator` — build immutable objects from JSON

**Why**: records/immutables or no no-arg constructor? Tell Jackson which ctor/factory to use.

```java
public class PointDto {
  private final int x;
  private final int y;

  @JsonCreator(mode = JsonCreator.Mode.PROPERTIES)
  public PointDto(@JsonProperty("x") int x,
                  @JsonProperty("y") int y) {
    this.x = x; this.y = y;
  }
}
```

**Input → Object**

```json
{ "x": 3, "y": 4 }
```

**Useful modes (max 5)**

* `Mode.PROPERTIES` — match by property names (most common).
* `Mode.DELEGATING` — pass the whole input into one param (single-value wrapper).
* Works with **static factory** + `@JsonCreator` too.
* Combine with `@JsonProperty(required = true)` on params.
* Pair with `@JsonValue` on the *other* side to serialize as a single value.

---

# 7) `@JsonValue` — serialize as a single value (enums/value objects)

**Why**: make an enum or tiny object show up as a simple string/number.

```java
public enum Role {
  ADMIN("admin"), USER("user");
  private final String label;
  Role(String l) { this.label = l; }

  @JsonValue
  public String toJson() { return label; }
}
```

**Enum → Output**

```json
"admin"
```

*For deserializing back*, add a `@JsonCreator` factory that turns the string into the enum/value object.

---

# 8) Bidirectional relations — stop infinite loops

**Problem**: parent → child → parent → … during serialization.

**Option A: “managed/back” pair**

```java
public class OrderDto {
  @JsonManagedReference
  private List<OrderItemDto> items;
}
public class OrderItemDto {
  @JsonBackReference
  private OrderDto order;
}
```

**Object → Output (simplified)**

```json
{ "items": [ { /* no "order" back-ref here */ } ] }
```

**Option B: identity by id**

```java
@JsonIdentityInfo(generator = ObjectIdGenerators.PropertyGenerator.class, property = "id")
public class CategoryDto {
  private Long id;
  private String name;
  private List<CategoryDto> children;
  private CategoryDto parent;
}
```

**Object → Output**

```json
{
  "id": 1,
  "name": "Root",
  "children": [ { "id": 2, "name": "Child", "parent": 1 } ],
  "parent": null
}
```

**When to choose**

* Use **A** for simple parent↔child where you just want to drop the back link.
* Use **B** when the graph is more complex or you want references by id.

---

# 9) Polymorphism — `@JsonTypeInfo` + `@JsonSubTypes`

**Why**: a field/collection holds different concrete types; you need a type hint.

```java
@JsonTypeInfo(use = JsonTypeInfo.Id.NAME,
              include = JsonTypeInfo.As.PROPERTY,
              property = "type")
@JsonSubTypes({
  @JsonSubTypes.Type(value = CardPaymentDto.class, name = "card"),
  @JsonSubTypes.Type(value = BankTransferDto.class, name = "bank")
})
public abstract class PaymentDto { }

public class CardPaymentDto extends PaymentDto { public String cardLast4; }
public class BankTransferDto extends PaymentDto { public String iban; }
```

**Object → Output**

```json
{ "type": "card", "cardLast4": "4242" }
```

**Input → Object**

```json
{ "type": "bank", "iban": "LT12 3456 7890 1234" }
```

**Useful options for `@JsonTypeInfo` (max 5)**

* `use = Id.NAME` — symbolic names (most common with `@JsonSubTypes`).
* `use = Id.CLASS` — fully qualified class name (tightly couples API to Java).
* `include = As.PROPERTY` — add `"type": "..."` field (most common).
* `include = As.EXISTING_PROPERTY` — reuse an existing field as the type key.
* `property = "type"` — name of that discriminator field.

---

## quick “when do i use what?” map

* Names don’t match? → `@JsonProperty` (maybe with `access`).
* Hide or tolerate extras? → `@JsonIgnore`, `@JsonIgnoreProperties`.
* Too many nulls/empties? → `@JsonInclude` with `NON_NULL`/`NON_EMPTY`.
* Dates weird? → `@JsonFormat` (pattern + timezone).
* Legacy input names? → `@JsonAlias`.
* Immutable DTOs? → `@JsonCreator` (+ `@JsonProperty` params).
* Graph cycles? → managed/back **or** identity ids.
* Mixed subtypes? → `@JsonTypeInfo` + `@JsonSubTypes`.

---

## tiny end-to-end example (mixing a few)

```java
@JsonInclude(JsonInclude.Include.NON_EMPTY)
public class UserDto {
  @JsonProperty("id")               private Long id;
  @JsonProperty("name")             private String fullName;

  @JsonIgnore                       private String internalNote;

  @JsonProperty(value = "email", access = JsonProperty.Access.WRITE_ONLY)
  private String email; // input only

  @JsonFormat(shape = JsonFormat.Shape.STRING, pattern = "yyyy-MM-dd")
  private LocalDate birthday;

  @JsonProperty("role")
  private Role role; // enum with @JsonValue as shown earlier
}
```

**Input**

```json
{
  "id": 10,
  "name": "Sam",
  "email": "sam@ex.com",
  "birthday": "1990-05-01",
  "role": "admin",
  "extra": "ignored client junk"
}
```

**Output**

```json
{
  "id": 10,
  "name": "Sam",
  "birthday": "1990-05-01",
  "role": "admin"
}
```

---


```java






```



# jackson advanced annotations & patterns

Jackson has many more annotations and patterns for advanced use cases. Here are some of the most useful ones, explained with examples.

---

# 1) global vs per-class: when to annotate vs configure

**Rule of thumb**

* If it’s **about a DTO’s contract**, prefer annotations (portable, self-documenting).
* If it’s **a cross-cutting policy**, prefer mapper config (Spring Boot auto-config or your `ObjectMapper` bean).

*(No snippet here because you asked to include code only where annotations appear. Below, everything is annotation-first.)*

---

# 2) `@JsonInclude` — advanced tricks (field-level + value filters)

**Why**: fine-tune which values are omitted.

```java
// class-level: drop nulls by default
@JsonInclude(JsonInclude.Include.NON_NULL)
public class ProductDto {
  private String name;

  // field override: drop empty strings/collections too
  @JsonInclude(JsonInclude.Include.NON_EMPTY)
  private String description;

  // drop defaults (e.g., 0, false, empty) — great for PATCH-like responses
  @JsonInclude(JsonInclude.Include.NON_DEFAULT)
  private int stock;

  // drop Optional.empty but keep present values
  @JsonInclude(JsonInclude.Include.NON_ABSENT)
  private Optional<String> ean;

  // contentInclude: drop nulls inside collections/maps (keeps container)
  @JsonInclude(content = JsonInclude.Include.NON_NULL)
  private List<String> tags;
}
```

**Object → Output**

```json
{
  "name": "Desk",
  "stock": 0,              // NOTE: kept unless default value is actually 0 at field init
  "tags": []               // container kept; null elements dropped
}
```

**Useful modes (max 5)**

* `ALWAYS`, `NON_NULL`, `NON_EMPTY`, `NON_DEFAULT`, `NON_ABSENT`
* `content = ...` — apply rule to elements of collections/maps

> tip: `NON_DEFAULT` compares to the field’s **initialized default**. If `int stock = 0;`, zero will be dropped; if uninitialized, zero may be kept.

---

# 3) `@JsonProperty` — access, required, defaults, ordering partner

**Why**: total control over naming and read/write visibility.

```java
@JsonPropertyOrder({"id","name","email"})
public class UserDto {
  @JsonProperty("id")
  private Long id;

  // write-only: accepted on input, hidden on output
  @JsonProperty(value = "email", access = JsonProperty.Access.WRITE_ONLY, required = true, defaultValue = "unknown@local")
  private String email;

  // read-only: shown in output, ignored on input
  @JsonProperty(value = "name", access = JsonProperty.Access.READ_ONLY)
  private String fullName;
}
```

**Input → Object**

```json
{ "id": 7, "email": "a@b.com", "name": "ignored on input" }
```

**Object → Output**

```json
{ "id": 7, "name": "Ada Lovelace" }
```

**Useful options (max 5)**

* `value = "..."` (json name)
* `access = READ_ONLY | WRITE_ONLY | READ_WRITE`
* `required = true`
* `defaultValue = "..."` (used if missing on input)
* pair with `@JsonPropertyOrder` (stable field order)

---

# 4) `@JsonSetter` / `@JsonDeserialize` — input rules & null handling

**Why**: control *how* JSON becomes fields.

```java
public class SettingsDto {
  // treat empty string as null on input
  @JsonSetter(nulls = Nulls.SKIP, contentNulls = Nulls.SKIP)
  private String theme;

  // custom deserializer for tricky field (e.g., "yes"/"no" → boolean)
  @JsonDeserialize(using = YesNoBooleanDeserializer.class)
  private boolean marketingConsent;
}
```

**Input → Object**

```json
{ "theme": "", "marketingConsent": "yes" }
```

→ `theme` stays `null` (skipped), `marketingConsent` becomes `true`.

**Useful options (max 5)**

* `@JsonSetter(nulls = Nulls.SKIP | Nulls.SET | Nulls.FAIL)`
* `contentNulls = ...` (for collection/map elements)
* `@JsonDeserialize(using = YourDeserializer.class)`
* `@JsonDeserialize(contentUsing = ... , keyUsing = ...)` (elements/keys)
* `@JsonDeserialize(builder = ...)` (for builder patterns)

---

# 5) `@JsonSerialize` — output shaping (single field or keys/elements)

**Why**: teach Jackson how to write a field.

```java
public class ReportDto {
  // custom serializer: mask last 6 of IBAN
  @JsonSerialize(using = MaskingSerializer.class)
  private String iban;

  // map with non-string keys — serialize keys in a custom way
  @JsonSerialize(keyUsing = ComplexKeySerializer.class)
  private Map<UUID, Integer> balancesByAccount;
}
```

**Object → Output (conceptual)**

```json
{ "iban": "LT****************90", "balancesByAccount": { "acc:550e8400-e29b-41d4-a716-446655440000": 120 } }
```

**Useful options (max 5)**

* `using = ...` (value serializer)
* `keyUsing = ...` (map key serializer)
* `contentUsing = ...` (collection/map element serializer)
* works with `@JsonFormat` (dates/numbers) — format first, then serialize
* combine with `@JsonInclude` to omit post-serialization empties

---

# 6) `@JsonAnySetter` / `@JsonAnyGetter` — “the rest of the fields”

**Why**: capture unknown/variable properties into a map and optionally emit them back.

```java
public class FlexibleDto {
  private Map<String, Object> other = new HashMap<>();

  @JsonAnySetter
  public void put(String name, Object value) { other.put(name, value); }

  @JsonAnyGetter
  public Map<String, Object> getOther() { return other; }
}
```

**Input → Object**

```json
{ "x": 1, "y": 2, "anythingElse": 3 }
```

**Object → Output**

```json
{ "x": 1, "y": 2, "anythingElse": 3 }
```

**Useful notes (max 5)**

* great with evolving schemas
* coexists with `@JsonIgnoreProperties(ignoreUnknown = true)` (but you usually pick one)
* values can be typed (`Map<String, JsonNode>` or specific DTO)
* pair with validation after deserialization
* beware of over-accepting junk (log or validate)

---

# 7) `@JsonUnwrapped` — flatten nested objects

**Why**: emit a child object’s fields at the parent level (and read them back).

```java
public class OrderDto {
  private String id;

  @JsonUnwrapped(prefix = "ship_", suffix = "")
  private AddressDto shipping;
}
```

**Object → Output**

```json
{
  "id": "o1",
  "ship_street": "Baker St",
  "ship_city": "Vilnius"
}
```

**Useful options (max 5)**

* `prefix = "..."`, `suffix = "..."` (avoid collisions)
* works both ways (serialize + deserialize)
* avoid name clashes with parent fields
* can nest multiple unwrapped sub-objects with careful prefixes
* combine with `@JsonInclude` on the nested type

---

# 8) `@JsonNaming` — automatic casing conventions

**Why**: avoid sprinkling `@JsonProperty` everywhere for simple case mappings.

```java
@JsonNaming(PropertyNamingStrategies.SnakeCaseStrategy.class)
public class MetricsDto {
  private long requestCount;
  private double errorRate;
}
```

**Object → Output**

```json
{ "request_count": 120, "error_rate": 0.01 }
```

**Useful strategies (max 5)**

* `SnakeCaseStrategy`
* `SNAKE_CASE` (enum alias)
* `LowerCamelCaseStrategy` (default)
* `KebabCaseStrategy`
* `UpperCamelCaseStrategy`

> tip: prefer `@JsonProperty` for **exceptions**, `@JsonNaming` for **bulk** rules.

---

# 9) `@JsonView` — role-based field visibility

**Why**: one DTO, different audiences (public vs admin).

```java
public class Views { public static class Public{} public static class Admin extends Public{} }

public class UserDto {
  @JsonView(Views.Public.class)
  public Long id;

  @JsonView(Views.Public.class)
  public String displayName;

  @JsonView(Views.Admin.class)
  public String email;
}
```

**Serialize with a view (Spring controller example)**

```java
// @JsonView(Views.Public.class) on method or return type
```

**Public Output**

```json
{ "id": 1, "displayName": "Ada" }
```

**Admin Output**

```json
{ "id": 1, "displayName": "Ada", "email": "ada@b.com" }
```

**Useful notes (max 5)**

* views compose via inheritance
* annotate fields and/or getters
* controller-level `@JsonView` applies to response
* tests: use `mapper.writerWithView(Views.Public.class)`
* don’t overuse; it increases complexity

---

# 10) polymorphism deeper — names, existing/external type ids, wrappers

**Why**: control where the type info lives.

```java
// common base
@JsonTypeInfo(use = JsonTypeInfo.Id.NAME,
              include = JsonTypeInfo.As.PROPERTY,
              property = "type")
@JsonSubTypes({
  @JsonSubTypes.Type(value = CardPaymentDto.class, name = "card"),
  @JsonSubTypes.Type(value = BankTransferDto.class, name = "bank")
})
public abstract class PaymentDto {}

// optional: name at subtype (instead of listing in @JsonSubTypes)
@JsonTypeName("card")
public class CardPaymentDto extends PaymentDto {
  public String cardLast4;
}
```

**Variants (most used)**

* `include = As.EXISTING_PROPERTY` — reuse a real field as the discriminator.
* `include = As.EXTERNAL_PROPERTY` — type sits next to the property holding the object (used inside containers).
* `include = As.WRAPPER_OBJECT` — `{"card": {...}}` style.

**I/O (wrapper example)**

```java
// With As.WRAPPER_OBJECT
{ "card": { "cardLast4": "4242" } }
```

**Useful options (max 5)**

* `use = Id.NAME | Id.CLASS | Id.MINIMAL_CLASS`
* `include = As.PROPERTY | As.EXISTING_PROPERTY | As.EXTERNAL_PROPERTY | As.WRAPPER_OBJECT`
* `property = "type"`
* `visible = true` (keep the type property in POJO if also a real field)
* `defaultImpl = ...` (fallback subtype)

---

# 11) identity & cycles advanced — scopes and resolvers

**Why**: control how object identity is generated and resolved.

```java
@JsonIdentityInfo(
  generator = ObjectIdGenerators.PropertyGenerator.class,
  property = "id",
  resolver = MyIdResolver.class,  // optional custom lookups
  scope = ProjectDto.class        // id uniqueness per scope
)
public class ProjectDto {
  public Long id;
  public String name;
  public List<ProjectDto> children;
}
```

**Output (with refs)**

```json
{ "id": 1, "name": "Root", "children": [ { "id": 2, "name": "Child", "children": [1] } ] }
```

**Useful knobs (max 5)**

* `generator = PropertyGenerator | IntSequenceGenerator | UUIDGenerator`
* `property = "..."` for property-based ids
* `scope = ...` limit id space
* `resolver = ...` custom id resolution
* pair with `@JsonIdentityReference(alwaysAsId = true)` to always emit just the id

---

# 12) mixins — annotate without touching the class

**Why**: 3rd-party classes or you want “annotation profiles”.

```java
// mixin type: only annotations, same shape as target
public abstract class UserMixin {
  @JsonProperty("id") abstract Long getId();
  @JsonIgnore abstract String getInternalNote();
}

// register mixin with mapper (Spring bean config, not shown per your ask)
```

**Effect**: your DTO JSON changes as if you had annotated the original class.

**Useful notes (max 5)**

* great for libraries/records you can’t modify
* test different API contracts in tests
* combine with views or filters
* keep mixins close to config for discoverability
* annotate getters/setters/fields in the mixin

---

# 13) dynamic filters — `@JsonFilter`

**Why**: choose at runtime which fields to include.

```java
@JsonFilter("userFilter")
public class UserDto {
  public Long id;
  public String name;
  public String email;
}
```

*(You apply a `FilterProvider` at write time — code lives outside the DTO; included here just to show the annotation anchor.)*

**Useful notes (max 5)**

* per-write control (good for ad-hoc projections)
* combine with `SimpleBeanPropertyFilter`
* can whitelist or blacklist fields
* filters stack with other annotations
* more flexible but harder to reason about than `@JsonView`

---

# 14) records & immutables — minimal ceremony

**Why**: concise DTOs with stable contracts.

```java
public record CustomerDto(
  @JsonProperty("id") Long id,
  @JsonProperty("name") String name,
  @JsonFormat(pattern = "yyyy-MM-dd") LocalDate since
) {}
```

**Input**

```json
{ "id": 5, "name": "Mila", "since": "2021-01-01" }
```

**Output**

```json
{ "id": 5, "name": "Mila", "since": "2021-01-01" }
```

---

# quick chooser (deep cut)

* **Mass renaming** → `@JsonNaming`; exceptions → `@JsonProperty`.
* **Selective omission** → `@JsonInclude` (field or content-level).
* **Input cleanup** → `@JsonSetter(nulls=...)` or custom `@JsonDeserialize`.
* **Dynamic & role-based views** → `@JsonView` or `@JsonFilter` (runtime).
* **Graphy models** → `@JsonManagedReference/@JsonBackReference` (simple) or `@JsonIdentityInfo` (complex).
* **Flat API shape** → `@JsonUnwrapped` (use prefixes).
* **Polymorphism** → `@JsonTypeInfo` + `@JsonSubTypes` (pick `include` mode carefully).

