# Packages & Basic Modules
## What a package actually is
- A package is a namespace — a way of grouping related classes/interfaces/enums together and giving them a fully qualified identity. 
- `Account` in one package and `Account` in another package are genuinely different classes to the compiler, distinguished entirely by their package prefix (`com.bank.core.Account` vs. `com.bank.reporting.Account`).
- Packages are also a visibility boundary — classes in one package can access each other's package-private members, but code outside the package cannot. 
- This is the only visibility level that is not explicitly declared with a keyword; if you leave off `public`, `protected`, or `private`, the member is package-private by default.
- Packages are also a physical organization mechanism — the compiler and JVM expect the directory structure on disk to match the package structure. 
- For example, `com.bank.core.Account` must be in a file `Account.java` located in a directory `com/bank/core/` relative to the source root.
- `javac`/the JVM classloader locate classes by combining the classpath root with this directory path — `package com.brokerage.accounts;` absolutely requires the file to sit at `.../com/brokerage/accounts/Account.java`, or compilation/class-loading fails. 


### import — what it actually does (and doesn't do)
- The `import` statement is a compile-time convenience that allows you to refer to a class by its simple name rather than its fully qualified name.
- It has zero effect on bytecode or runtime behavior — the compiled `.class` file always references types by their fully qualified name regardless of whether you imported them.
- The wildcard form `import package.*;` imports only the types directly declared in that exact package — it is not recursive; it does not reach into sub-packages at all. Ex: `com.brokerage.*` does not import anything from `com.brokerage.accounts.*`.
- If two imported packages (via wildcard or otherwise) both have a class with the same simple name, and you reference that simple name unqualified, it's a compile error — ambiguous reference — you must fall back to the fully qualified name for at least one of them.
- java.lang is the one package auto-imported into every file, with no import statement needed at all — this is why String, Object, System, etc. are always available without qualification.
- It simply tells the compiler: "when you see `Account`, look in this package for it."
- You can always use the fully qualified name instead of importing, but imports make code more readable

### `.jar` files — brief, mechanical explanation
- A `.jar` (Java ARchive) file is just a zip file with a specific internal structure. It contains compiled `.class` files arranged in their package-derived directory layout.
- optionally a `META-INF/MANIFEST.MF` file with metadata about the archive. Most commonly, which class contains the main method to run, via a Main-Class: entry — this is what lets you do java -jar app.jar without naming the class on the command line.
- JARs are purely a packaging/distribution format — they don't change anything about packages, imports, or visibility; they're just a convenient single-file way to ship a whole compiled package tree (plus any bundled dependencies) instead of a loose folder of .class files. Further, The JVM can load classes directly from a `.jar` file on the classpath.
- The `jar` tool is just a convenient wrapper around the standard `zip` format, with some extra features for Java-specific metadata. 
- You can create a `.jar` file with ```jar cf mylib.jar com/brokerage/accounts/*.class```
- You can inspect the contents of a `.jar` file with ```jar tf mylib.jar```
- You can run a `.jar` file with a `main` method by specifying the `Main-Class` in the manifest, or by using the `-cp` option to include it on the classpath and invoking the main class directly.

### The classpath — how the JVM actually finds your classes
- The classpath is a list of directories and/or `.jar` files that the JVM searches to find compiled classes when loading them at runtime.
- You can set the classpath with the `-cp` or `-classpath` option when running `java` or `javac`, or by setting the `CLASSPATH` environment variable. If you don't specify a classpath, the JVM uses the current directory (`.`) by default.
- The JVM searches the classpath in order, looking for the first matching class file.
- If a class is not found in any of the specified locations, a `ClassNotFoundException` is thrown at runtime.

## Modules — the next level of organization
### JPMS — the Java Platform Module System
### The problem JPMS was built to solve
- Before module system support, Java had exactly two visibility tools: package-private (visible only within one package) and public (visible to literally anything that could reach the class on the classpath). 
- There was no tier in between. This created a real, structural problem for anyone building a library with more than a handful of packages:
  - You want to expose a public API for users of your library, but you also have internal implementation packages that you don't want to be used directly. 
  - Without modules, you have no way to enforce this — any public class in any package is accessible to anyone who can put it on the classpath. 
  - This leads to fragile APIs, where users can depend on internal classes that you might change or remove in future versions, breaking their code.
- JPMS's entire purpose is introducing a third tier, above the package: the module — with its own explicit, compiler-and-runtime-enforced boundary for what's visible outside it, independent of how many packages live inside.
- A module is a higher-level grouping of packages, introduced in Java 9 with the Java Platform Module System (JPMS). It allows you to define explicit dependencies between modules and control which packages are exposed to other modules.

### Declaring a module — `module-info.java`
- A module is defined in a `module-info.java` file at the root of the module's source directory. This file specifies the module's name, its dependencies on other modules, and which packages it exports for use by other modules.
- Example of a simple `module-info.java`:
```java
module com.brokerage.accounts {
    requires java.sql;
    requires transitive com.brokerage.common;
    requires static com.brokerage.devtools;

    exports com.brokerage.accounts.api;
    exports com.brokerage.accounts.api.dto to com.brokerage.reporting;

    opens com.brokerage.accounts.model to com.fasterxml.jackson.databind;

    uses com.brokerage.accounts.api.AccountValidator;
    provides com.brokerage.accounts.api.AccountValidator
        with com.brokerage.accounts.internal.DefaultAccountValidator;
}
```
- Modules provide strong encapsulation, allowing you to hide implementation details and only expose the necessary API to other modules. This helps to reduce coupling and improve maintainability of large applications.
- Every directive here does something genuinely distinct — worth going through each precisely.

**`requires` — declaring a dependency**
- `requires` specifies that this module depends on another module. The compiler and runtime will enforce that the required module is present when compiling or running code that uses this module.
- `requires transitive` means that any module that depends on this module will also implicitly depend on the transitive module. This is useful for modules that provide a public API that relies on another module.
- `requires static` means that the dependency is only needed at compile time, not at runtime. This is useful for optional dependencies, such as testing frameworks or development tools.

**`exports` — the real encapsulation enforcement**
- `exports` specifies which packages in the module are accessible to other modules. Only the exported packages can be used by code outside the module; all other packages are effectively private to the module.
- You can also specify `exports package to module;` to restrict access to specific modules, rather than making the package public to all modules. This allows for fine-grained control over which modules can access certain packages, enhancing encapsulation and reducing the risk of unintended dependencies.

**`opens` — a separate, narrower kind of visibility, for reflection**
- `opens` specifies which packages in the module are accessible for deep reflection by other modules. This is typically used for frameworks that rely on reflection, such as serialization libraries.
- You can also specify `opens package to module;` to restrict reflective access to specific modules, rather than making the package open to all modules.

**`uses` and `provides` — service loading**
- `uses` specifies that this module depends on a service interface, which is an interface that can have multiple implementations provided by different modules. This allows for a decoupled architecture where modules can provide different implementations of a service without knowing about each other.
- `provides` specifies that this module provides an implementation of a service interface. This allows other modules to discover and use the provided implementation without needing to know the specific class name or package. This is part of the service loader mechanism in Java, which allows for dynamic discovery and loading of service implementations at runtime.


---

# Miscellaneous
### Genuine new challenges JPMS introduced (didn't exist pre-9)

**1. Eager, whole-graph resolution at startup — a real behavioral difference from classpath**
On the classpath, a missing class is only discovered **lazily**, at the moment something actually tries to use it — `NoClassDefFoundError` can surface arbitrarily late in a running program, sometimes only under a rarely-exercised code path. On the module path, the JVM resolves the **entire** `requires` graph **upfront, at launch** — if any required module is missing, the program fails immediately at startup with a module resolution error, before `main` even runs. This is actually a genuine improvement (fail fast instead of a surprise three hours into a batch job) — but it's also a real behavioral change worth knowing, since it means module-path failures and classpath failures surface at completely different points in a program's life.

**2. No version resolution at all — a frequently-tested gap**
This is one of the most common things people get wrong in interviews: JPMS resolves dependencies purely by **module name**, with **no concept of version** built into the module system itself. If two different JARs on the module path declare the same module name at different versions, JPMS has no native mechanism to pick one, warn about the conflict, or let you request a specific version — that remains entirely the responsibility of your build tool (Maven/Gradle), sitting outside JPMS's boundary. This is a deliberate design contrast with **OSGi** (an older, more heavyweight Java module system) which *does* bake in versioned dependencies and dynamic install/uninstall/update of modules at runtime — a very common interview comparison point: *"how does JPMS differ from OSGi?"* — answer: JPMS gives strong encapsulation and a static, name-based dependency graph; OSGi additionally gives you versioning and true runtime dynamism (installing/removing bundles without restarting the JVM), which JPMS was never designed to provide.

**3. The module graph must be acyclic — unlike packages or classes**
Two classes (or two packages) can reference each other circularly all day — nothing stops it. But the **module** `requires` graph is checked and must be **acyclic**: module A cannot `requires` module B if B (directly or transitively) `requires` A. This is a real, compiler-enforced constraint that simply didn't exist as a concept before modules, since packages never had declared dependencies on each other at all. A common interview trap: assuming module-level circular dependencies are just "bad practice" the way circular class dependencies often are — they're not merely discouraged, they're a hard compile/link error.

**4. Migration pain from strong encapsulation breaking existing reflection-heavy code**
This was the single biggest real-world challenge when Java 9 shipped. Enormous amounts of existing code — especially frameworks doing dependency injection, serialization, and ORM (early Spring, Hibernate, Jackson versions at the time) — relied on reflection reaching into `private` fields of arbitrary classes, which pre-9 always worked unconditionally. JPMS's strong encapsulation initially broke this silently-tolerated behavior. The JDK's actual rollout was staged carefully because of this:
- Java 9–16: illegal reflective access attempts against JDK-internal modules produced a **runtime warning** ("An illegal reflective access operation has occurred") but were still **allowed** to succeed, specifically to avoid breaking the ecosystem outright.
- Later versions tightened this to the point of throwing `InaccessibleObjectException` by default for such access, requiring explicit opt-in flags like `--add-opens java.base/java.lang=ALL-UNNAMED` on the command line to restore the old behavior.
- This is exactly why you still sometimes see `--add-opens`/`--add-exports` flags littering real-world startup scripts for older applications — they're a manual patch for exactly the kind of access that `opens` was built to grant deliberately, applied to code that predates `opens` existing at all.

**5. Automatic module naming instability**
An automatic module's name is derived from its jar's filename if the jar has no `Automatic-Module-Name` manifest entry — which means **renaming a jar file can silently change its module name**, breaking any `requires` statement that referenced the old derived name. This is a genuinely fragile migration-era hazard that has no equivalent on the plain classpath, where filenames never had any semantic meaning to the classloader at all.

### Interview-shaped questions worth being ready for

| Question                                                                                                            | Sharp answer                                                                                                                                                                                                                                 |
|---------------------------------------------------------------------------------------------------------------------|----------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------|
| "How does JPMS differ from OSGi?"                                                                                   | JPMS: static graph, name-based, no versioning, no runtime install/uninstall. OSGi: versioned dependencies, true dynamic runtime module lifecycle.                                                                                            |
| "Can two modules have circular `requires`?"                                                                         | No — the module graph must be acyclic; this is enforced at compile/link time, unlike ordinary class/package circularity which is unrestricted.                                                                                               |
| "What's a split package, and why is it disallowed?"                                                                 | Same package declared in two different named modules on the module path — disallowed because it would make `exports`/encapsulation ambiguous about which module "owns" that package; it's a hard resolution error, not a silent first-match. |
| "Does JPMS handle dependency versioning?"                                                                           | No — purely name-based resolution; versioning is left entirely to the build tool, not the module system.                                                                                                                                     |
| "What's the practical difference between a missing class on the classpath vs. a missing module on the module path?" | Classpath: lazy, `NoClassDefFoundError` only when the class is actually used. Module path: eager, the whole `requires` graph is validated at JVM startup before `main` runs.                                                                 |
| "Why would `opens` ever be needed if a package is already `exports`ed?"                                             | It wouldn't — `exports` already covers ordinary type visibility. `opens` matters specifically for packages that are **not** exported but still need deep reflective access granted to specific (or all) other modules.                       |
| "Why did so much existing code break when upgrading to Java 9+?"                                                    | Reflection-dependent frameworks relied on unconditional deep access to `private` members, which strong encapsulation restricts by default — requiring explicit `opens` or `--add-opens` to restore.                                          |

### Why adoption is still partial, worth knowing as context

Given the migration cost described above (point 4 especially), a very large share of real-world Java applications — including many modern ones — still deliberately run entirely on the traditional classpath as the unnamed module, never writing a `module-info.java` at all, and get no benefit (or friction) from JPMS either way. This is a reasonable, common, and defensible real-world choice — not a sign of an outdated codebase — and is itself a fair thing to say plainly if asked "do you use JPMS in production" in an interview: most teams don't, and that's normal.
