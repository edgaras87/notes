# Java `Path` (and a quick note on `Paths`)

* **`Path`** is the main interface in `java.nio.file` that models a filesystem path (like `"images/cat.png"` or `"/usr/local/bin"`).
* **About `Paths`**: since **Java 11**, you can use `Path.of(...)` instead of `Paths.get(...)`. They do the same thing. Because of that, this guide **uses `Path.of` and ignores `Paths`**.

---

## Quick glossary (beginner-friendly)

* **Absolute path**: a full address from the filesystem root, e.g. `"/home/alex/report.txt"` (Unix) or `"C:\Users\Alex\report.txt"` (Windows). It doesn’t depend on the “current folder”.
* **Relative path**: a path **relative to a base** (usually the app’s current working directory), e.g. `"docs/report.txt"` or `"../images/logo.png"`.
* **Root**: the topmost starting point of the filesystem (`"/"` on Unix; a drive like `"C:\"` on Windows).
* **Normalize**: clean a path by removing `.` (current dir) and resolving `..` (go up one folder) **without touching the disk**.
* **Resolve**: join paths like URLs: base.resolve(child) → “append” child to base.
* **Relativize**: create a relative path that goes **from one path to another**.
* **Real path**: an absolute, normalized path with **symlinks resolved on disk**.

---

## Creating `Path` instances

### `Path.of(String first, String... more)`

Create a path from one or more parts.

```java
Path p1 = Path.of("images", "icons", "cat.png"); // "images/icons/cat.png"
Path p2 = Path.of("/usr", "local", "bin");       // "/usr/local/bin" (absolute on Unix)
```

**Output examples**

* `p1.toString()` → `"images/icons/cat.png"`
* `p2.isAbsolute()` → `true`

> Use this instead of `Paths.get(...)` on Java 11+.

---

## Reading path structure

### `getFileName()`

Last element of the path (the “leaf”).

```java
Path p = Path.of("/var/log/system.log");
p.getFileName().toString(); // "system.log"
```

### `getParent()`

Everything except the last element (or `null` if none).

```java
Path p = Path.of("/var/log/system.log");
p.getParent().toString();   // "/var/log"
```

### `getRoot()`

The root component (or `null` if relative).

```java
Path p1 = Path.of("/var/log/system.log");
Path p2 = Path.of("docs/readme.md");
p1.getRoot().toString();    // "/"
p2.getRoot();               // null (relative path)
```

### `getNameCount()` and `getName(int index)`

Count and access individual name elements (0-based, root excluded).

```java
Path p = Path.of("/usr/local/bin");
p.getNameCount();           // 3  ("usr","local","bin")
p.getName(1).toString();    // "local"
```

### `subpath(int beginIndex, int endIndex)`

Slice out a portion of the names (root not included).

```java
Path p = Path.of("/usr/local/share/docs");
p.subpath(1, 3).toString(); // "local/share"
```

---

## Checking path properties

### `isAbsolute()`

Is this path absolute?

```java
Path.of("/etc/hosts").isAbsolute(); // true
Path.of("etc/hosts").isAbsolute();  // false
```

### `startsWith(...)` / `endsWith(...)`

Compare by path elements (not plain string).

```java
Path p = Path.of("src/main/java/App.java");
p.startsWith("src");               // true
p.endsWith("App.java");            // true
p.endsWith(Path.of("java", "App.java")); // true
```

---

## Transforming paths (no filesystem access)

### `normalize()`

Remove `.` and fold `..` where possible.

```java
Path.of("a/./b/../c").normalize().toString(); // "a/c"
```

### `resolve(String|Path other)`

Append `other` to this path (unless `other` is **absolute**, then `other` is returned).

```java
Path base = Path.of("/home/alex");
base.resolve("docs/report.txt").toString(); // "/home/alex/docs/report.txt"
base.resolve("/etc/hosts").toString();      // "/etc/hosts" (absolute wins)
```

### `resolveSibling(String|Path other)`

Replace the last element with `other`.

```java
Path p = Path.of("/home/alex/docs/report.txt");
p.resolveSibling("notes.txt").toString();   // "/home/alex/docs/notes.txt"
```

### `relativize(Path other)`

Create a **relative path** from `this` to `other`. Both must be both absolute or both relative, and on the same root/drive.

```java
Path a = Path.of("/home/alex/docs");
Path b = Path.of("/home/alex/images/pic.png");
a.relativize(b).toString();                 // "../images/pic.png"
```

---

## Converting paths

### `toAbsolutePath()`

Turn a relative path into an absolute one using the **current working directory**; already-absolute paths are returned as-is. (No disk access.)

```java
Path.of("logs/app.log").toAbsolutePath().toString();
// e.g. "/Users/alex/project/logs/app.log"
```

### `toRealPath(LinkOption... options)`

Return the **real** absolute path: normalized and with symlinks resolved. **Touches the filesystem** and can throw if missing.

```java
Path.of("symlink/to/file.txt").toRealPath().toString();
// e.g. "/actual/location/file.txt"
```

> If you pass `LinkOption.NOFOLLOW_LINKS`, it won’t resolve symlinks (still checks existence).

### `toUri()`

Convert to a `file:` URI.

```java
Path.of("/tmp/test.txt").toUri().toString(); // "file:///tmp/test.txt"
```

### `toFile()`

Convert to the old `java.io.File` object.

```java
Path.of("docs/readme.md").toFile(); // java.io.File
```

---

## Interop & utilities

### `iterator()` (implements `Iterable<Path>`)

Iterate path elements.

```java
Path p = Path.of("src/main/java");
for (Path part : p) {
  // "src", then "main", then "java"
}
```

### `compareTo(Path other)` (and `equals`, `hashCode`)

Lexicographic comparison based on the filesystem’s rules.

```java
Path a = Path.of("a");
Path b = Path.of("b");
a.compareTo(b); // negative value (a < b)
```

### `getFileSystem()`

The `FileSystem` this `Path` belongs to (useful with custom/ZIP filesystems).

```java
Path.of("README.md").getFileSystem().provider().getScheme(); // "file"
```

---

## Common `Files` helpers you’ll often use **with** `Path` (not methods on `Path`, but handy)

```java
Files.exists(Path.of("notes.txt"));                 // true/false
Files.createDirectories(Path.of("out/logs"));       // create all missing dirs
Files.copy(srcPath, destPath, REPLACE_EXISTING);    // copy a file
Files.move(srcPath, destPath, ATOMIC_MOVE);         // move/rename
Files.delete(Path.of("old.txt"));                   // delete
Files.readString(Path.of("data.txt"));              // read whole file as String
Files.writeString(Path.of("out.txt"), "hello");     // write text
```

---

## Windows vs. Unix notes (gotchas)

* **Roots**: Unix root is `"/"`. Windows roots are drives like `"C:\"` and UNC roots like `"\\server\share"`.
* **Separators**: `Path` uses the **platform’s** separator under the hood. You can write either `'/'` or `'\'` in string literals, but prefer **`Path.of("a","b","c")`** to stay portable.
* **Relativize/resolve**: You can’t relativize/resolve across different roots/drives.

---

## Mini reference (method → what it does → example I/O)

| Method                 | What it does                                          | Example input → output                              |
| ---------------------- | ----------------------------------------------------- | --------------------------------------------------- |
| `Path.of("a","b","c")` | Make a path from parts                                | → `"a/b/c"`                                         |
| `getFileName()`        | Last path segment                                     | `"/x/y/z.txt"` → `"z.txt"`                          |
| `getParent()`          | Path without the last segment                         | `"/x/y/z.txt"` → `"/x/y"`                           |
| `getRoot()`            | The root component or `null`                          | `"/x/y"` → `"/"`; `"x/y"` → `null`                  |
| `getNameCount()`       | Number of segments (no root)                          | `"/a/b/c"` → `3`                                    |
| `getName(i)`           | Segment at index `i`                                  | `"/a/b/c"` & `i=1` → `"b"`                          |
| `subpath(i,j)`         | Slice of segments `[i,j)`                             | `"/a/b/c/d"`, `(1,3)` → `"b/c"`                     |
| `isAbsolute()`         | Is it a full path from the root?                      | `"/a/b"` → `true`; `"a/b"` → `false`                |
| `startsWith(x)`        | Starts with segment(s) `x`?                           | `"src/main/App.java"` & `"src"` → `true`            |
| `endsWith(x)`          | Ends with segment(s) `x`?                             | `"src/main/App.java"` & `"App.java"` → `true`       |
| `normalize()`          | Remove `.` and fold `..`                              | `"a/./b/../c"` → `"a/c"`                            |
| `resolve(x)`           | Append `x` (unless `x` is absolute)                   | `"/home/a" + "docs/r.txt"` → `"/home/a/docs/r.txt"` |
| `resolveSibling(x)`    | Replace the last segment                              | `"/a/b/c.txt" + "d.txt"` → `"/a/b/d.txt"`           |
| `relativize(other)`    | Path from `this` to `other`                           | `"/a/b"` → `"/a/c/d"` gives `"../c/d"`              |
| `toAbsolutePath()`     | Make absolute using CWD                               | `"logs/app.log"` → `"/…/logs/app.log"`              |
| `toRealPath()`         | Absolute, normalized, resolve symlinks (touches disk) | `"link/file"` → `"/actual/file"`                    |
| `toUri()`              | Convert to `file:` URI                                | `"/tmp/t.txt"` → `"file:///tmp/t.txt"`              |
| `toFile()`             | Convert to `java.io.File`                             | `"docs/readme.md"` → `File("docs/readme.md")`       |
| `iterator()`           | Iterate segments                                      | `"a/b/c"` → `"a"`, `"b"`, `"c"`                     |
| `compareTo()`          | Order paths lexicographically                         | `"a"` vs `"b"` → `< 0`                              |

---

## Tiny end-to-end example

```java
Path base = Path.of("/home/alex/projects");
Path rel  = Path.of("demo/../lib/utils.java"); // relative
Path norm = rel.normalize();                   // "lib/utils.java"
Path abs  = base.resolve(norm);                // "/home/alex/projects/lib/utils.java"
Path here = Path.of(".").toAbsolutePath();     // absolute CWD
Path linkFree = abs.toRealPath();              // resolves symlinks (if any; hits disk)
Path back = base.relativize(linkFree);         // relative from base to real file
```

**Typical outputs**

* `norm.toString()` → `"lib/utils.java"`
* `abs.toString()` → `"/home/alex/projects/lib/utils.java"`
* `back.toString()` → `"lib/utils.java"`



---

# Important behaviors & gotchas

1. **Relative vs Absolute**

    * `Path.of("docs/file.txt")` is **relative**: it depends on the program’s *current working directory*.
    * `Path.of("/home/user/docs/file.txt")` (Unix) or `Path.of("C:\\Users\\Alex\\file.txt")` (Windows) is **absolute**: it always points to the same place, no matter where the program is run.

2. **Normalize doesn’t touch the disk**

    * `normalize()` just cleans up the *string form* of the path (`.` and `..`).
    * It doesn’t check if the file exists.

3. **Real path touches the disk**

    * `toRealPath()` **does** check the filesystem.
    * It resolves symlinks, checks existence, and can throw exceptions if the file isn’t there.

4. **Resolve vs. Relativize confusion**

    * `resolve(child)` → go **deeper** into the tree.
    * `relativize(other)` → calculate the **path between** two paths.
    * You can’t `relativize` across different drives/roots (`C:\` vs `D:\` on Windows, or `/` vs a network share on Unix).

5. **StartsWith/EndsWith are by segments, not plain strings**

    * `"src/main/java/App.java".endsWith("java")` → **false** (last segment is `"App.java"`).
    * `"src/main/java/App.java".endsWith("App.java")` → **true**.

6. **Iterating excludes root**

    * `/usr/local/bin` will give elements `"usr"`, `"local"`, `"bin"` — the root `/` is **not** included.

7. **Cross-platform differences**

    * On Windows: roots are drives (`C:\`), case-insensitive by default.
    * On Unix: root is `/`, case-sensitive.

---

# Bottom line

* **Use `Path` instead of string concatenation**: It handles separators, OS differences, and path logic safely.
* **Prefer `Path.of(...)`** (Java 11+) over `Paths.get(...)`.
* **Use `normalize()`** when you want a clean path string.
* **Use `toRealPath()`** only if you need the true location on disk (and are ready to handle errors).
* **Combine with `Files` class** for actual file operations (`exists`, `read`, `write`, `delete`).
* **Remember**: `Path` itself is just a description of a location — it doesn’t create, delete, or read files.

---


