# Java Cheatsheet: `ServletUriComponentsBuilder`

## What it is (and why it’s useful)

`ServletUriComponentsBuilder` (Spring MVC) builds **absolute URLs** from the **current HTTP request** (scheme, host, port, context path). It saves you from manual string concatenation, handles encoding, and stays correct across environments (localhost, prod, reverse proxies).

---

## Quick glossary

* **Context path** – The base path your app is mounted at (e.g., `/myapp`).
* **Request URI** – The path of the current request (e.g., `/api/users/42`).
* **Builder** – A fluent object you add parts to (path, query, fragment) to produce a URL.
* **UriComponents** – An immutable representation of a URI (scheme/host/port/path/query).
* **Forwarded headers** – Proxy headers (`X-Forwarded-*`) that tell the app the original external URL.
* **Encode** – Percent-encode unsafe characters (once—and only once).

---

## Creating instances

### From the current request/context (most common)

```java
// Assume current request: https://example.com/myapp/api/users/42?active=true
String a = ServletUriComponentsBuilder
        .fromCurrentContextPath()      // https://example.com/myapp
        .path("/files/123.png")
        .toUriString();

String b = ServletUriComponentsBuilder
        .fromCurrentRequestUri()       // https://example.com/myapp/api/users/42
        .replacePath("/health")        // path becomes /health
        .replaceQuery(null)            // drop query
        .toUriString();

System.out.println(a);
System.out.println(b);
```

**Output**

```
https://example.com/myapp/files/123.png
https://example.com/myapp/health
```

### From an explicit `HttpServletRequest`

```java
String url = ServletUriComponentsBuilder
        .fromRequest(request)          // uses scheme/host/port/context/path/query from request
        .replacePath("/docs")
        .replaceQuery("v=1")
        .toUriString();

System.out.println(url);
```

**Output**

```
https://example.com/myapp/docs?v=1
```

---

## Reading state / accessors (via `UriComponents`)

```java
var comps = ServletUriComponentsBuilder
        .fromCurrentRequestUri()
        .build();                      // -> UriComponents

System.out.println(comps.getScheme());
System.out.println(comps.getHost());
System.out.println(comps.getPort());
System.out.println(comps.getPath());
System.out.println(comps.getQuery());
```

**Typical output**

```
https
example.com
-1
/ myapp / api / users / 42   // actual output is "/myapp/api/users/42"
active=true
```

> Note: `-1` means “default port for scheme”.

---

## Checking properties

```java
var comps = ServletUriComponentsBuilder.fromCurrentRequest().build();
boolean hasQuery = comps.getQueryParams().containsKey("active");
boolean hasFragment = comps.getFragment() != null;

System.out.println(hasQuery);
System.out.println(hasFragment);
```

**Output**

```
true
false
```

---

## Transformations (pure operations, no side effects)

```java
String url = ServletUriComponentsBuilder
        .fromCurrentContextPath()
        .path("/api")
        .pathSegment("users", "{id}")  // safe segment-joining + encoding
        .queryParam("verbose", "{v}")  // adds ?verbose=1
        .buildAndExpand(42, 1)         // expand {id}=42, {v}=1
        .toUriString();

System.out.println(url);
```

**Output**

```
https://example.com/myapp/api/users/42?verbose=1
```

More transforms:

```java
String url = ServletUriComponentsBuilder
        .fromCurrentRequestUri()
        .replacePath("/search")
        .replaceQueryParam("q", "café") // encodes as caf%C3%A9
        .fragment("top")
        .toUriString();

System.out.println(url);
```

**Output**

```
https://example.com/myapp/search?q=caf%C3%A9#top
```

---

## Conversions (to other types)

```java
var comps = ServletUriComponentsBuilder.fromCurrentRequestUri().build();

java.net.URI uri = comps.toUri();            // java.net.URI
String asString = comps.toUriString();       // String
```

---

## Iteration & comparison

```java
var q = ServletUriComponentsBuilder.fromCurrentRequest().build().getQueryParams();
q.forEach((k, v) -> System.out.println(k + " -> " + v));

var a = ServletUriComponentsBuilder.fromCurrentContextPath().path("/a").toUriString();
var b = ServletUriComponentsBuilder.fromCurrentContextPath().path("/a/").toUriString();
System.out.println(a.equals(b));             // URL string comparison
```

**Typical output**

```
active -> [true]
false
```

---

## Common utilities (nearby helpers)

* **`UriComponentsBuilder`** – Generic (non-servlet) builder; set `scheme/host/port` manually.
* **`MvcUriComponentsBuilder`** – Builds URLs from controller methods/type-safe mappings.
* **`ForwardedHeaderFilter`** – Make builders respect `X-Forwarded-*` headers behind proxies.
* **`UriUtils` / `UriTemplate`** – Encoding and template processing helpers.

---

## Gotchas / anti-patterns / notes

* ❗ **Must have a current request.** Using `fromCurrent*()` off-request thread → `IllegalStateException`. For async jobs, construct with `UriComponentsBuilder` and configure base manually.
* ❗ **Reverse proxy correctness.** Without `ForwardedHeaderFilter`, URLs may come out `http://internal:8080`. Register the filter (or set server.forward-headers-strategy) to honor `X-Forwarded-*`.
* ❗ **Double encoding.** Don’t call `.encode()` and then add already-encoded values. Prefer `pathSegment()` (auto-encodes) and raw values in `queryParam()`.
* ⚠️ **Trailing slashes.** `path("/a").path("/b")` → `/a/b`; `path("/a/").path("/b")` → `/a//b`. Use `pathSegment()` to avoid slash gymnastics.
* ⚠️ **Null query values.** `queryParam("x", (Object) null)` yields `x` with no value (`?x`). If you want to **remove** a param, use `replaceQueryParam("x")`.
* 🔁 **Cloning.** Use `cloneBuilder()` if you need to branch variants from a base builder.
* 🧪 **Version notes.** API stable across Spring 5/6; Jakarta packages in Spring 6 move servlet APIs to `jakarta.servlet.*`. Import paths may change in your app.

---

## Mini reference table

| Method                           | What it does                                            | Example (input → output)      |
| -------------------------------- | ------------------------------------------------------- | ----------------------------- |
| `fromCurrentContextPath()`       | Base = scheme + host + port + context                   | `https://ex.com/app`          |
| `fromCurrentRequestUri()`        | Base = full current URI (no query if you later replace) | `https://ex.com/app/api/u/42` |
| `fromRequest(req)`               | Use given `HttpServletRequest`                          | same as request               |
| `path("/x")`                     | Appends raw path                                        | `/app` + `/x` → `/app/x`      |
| `pathSegment("a","b")`           | Appends encoded segments                                | `("a b")` → `/a%20b`          |
| `queryParam("k", v)`             | Adds query parameter(s)                                 | `k=1&k=2` for multiple values |
| `replacePath("/p")`              | Replaces path                                           | `/old` → `/p`                 |
| `replaceQuery("a=1")`            | Replaces the whole query                                | `?a=1`                        |
| `replaceQueryParam("k", v...)`   | Replace specific param                                  | `?a=1&b=2` → `?a=9&b=2`       |
| `fragment("top")`                | Sets `#fragment`                                        | `#top`                        |
| `build()`                        | → `UriComponents` (no encoding side-effect)             | usable for getters            |
| `buildAndExpand(vars…)`          | Expand `{var}` templates                                | `/u/{id}` + `42` → `/u/42`    |
| `toUriString()`                  | Final string                                            | `"https://ex.com/app/u/42"`   |
| `toUri()`                        | → `java.net.URI`                                        | `new URI(...)`                |
| `encode()` (on builder or comps) | Percent-encode                                          | `café` → `caf%C3%A9`          |

---

## End-to-end example (typical controller flow)

```java
// Incoming request: https://api.example.com/myapp/api/users/42?active=true

String profile = ServletUriComponentsBuilder
        .fromCurrentContextPath()              // https://api.example.com/myapp
        .path("/profiles/{id}")
        .buildAndExpand(42)
        .toUriString();

String avatar = ServletUriComponentsBuilder
        .fromCurrentContextPath()
        .path("/files/avatars/{id}.png")
        .buildAndExpand(42)
        .toUriString();

String search = ServletUriComponentsBuilder
        .fromCurrentRequestUri()               // https://api.example.com/myapp/api/users/42
        .replacePath("/search")
        .replaceQueryParam("q", "café")        // encoded
        .replaceQueryParam("page", 2)
        .toUriString();

System.out.println(profile);
System.out.println(avatar);
System.out.println(search);
```

**Output**

```
https://api.example.com/myapp/profiles/42
https://api.example.com/myapp/files/avatars/42.png
https://api.example.com/myapp/search?q=caf%C3%A9&page=2
```

---

## Bottom line summary

* **Use `fromCurrentContextPath()`** to build URLs for resources your app serves (stable base).
* **Use `fromCurrentRequestUri()`** to tweak the current endpoint into a related URL.
* Prefer **`pathSegment()`** and raw values in **`queryParam()`**; call **`encode()`** once if needed.
* In proxy setups, **enable `ForwardedHeaderFilter`** to get correct external URLs.
* Avoid using it outside a request thread—use `UriComponentsBuilder` with explicit base there.
