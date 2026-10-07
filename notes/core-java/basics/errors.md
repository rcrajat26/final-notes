# Factual Errors Log

Status values: **Open** (to fix), **Won't fix** (author chose to leave the file untouched).

| # | File | Location | Error | Correct statement | Status |
|---|---|---|---|---|---|
| 1 | `01-jdk-jre-jvm-program-structure.md` | JVM section, line 11 | "JVM generates bytecode and converts it to native code" | `javac` generates bytecode. The JVM loads, verifies and executes it, interpreting it and JIT-compiling hot code to native code. | Open |
| 2 | `04-classes-objects.md` | Classes & Objects, line 4 | Class "contains... static fields" in Metaspace | Metaspace holds class metadata (bytecode, constant pool, method data). Static field storage sits in the `Class` mirror object on the **heap** (HotSpot, Java 7+). `26-class-loader.md` §12 states this correctly. | Open |
| 3 | `07-final-static-modifiers.md` | `static` table, Field row | "lives in Metaspace" | Same as #2: static fields live in the `Class` object on the heap. | Open |
| 4 | `15-packages-modules.md` | Interview Q&A, `opens` row | "Why would `opens` ever be needed if a package is already `exports`ed? It wouldn't — `exports` already covers ordinary type visibility." | `exports` gives access to public types and members only. Deep reflection on private members needs `opens`, even for an exported package. `21-reflection.md` states this correctly. | Open |
| 5 | `13-immutability-design.md` | Records section | "Java 14 introduced `record` classes" | Records were a preview in 14 and 15, and final in **Java 16**. | Open |
| 6 | `11-strings.md` | StringBuilder internal structure | "Holds a `char[] value`" | Since Java 9 `AbstractStringBuilder` uses `byte[]` plus a `coder`, the same compact-strings layout the file describes for `String`. | Open |
| 7 | `19-annotations.md` | Meta-annotations intro | "These four control how a custom annotation behaves" | The section covers three (`@Target`, `@Retention`, `@Repeatable`). Either fix the count or add `@Documented` / `@Inherited`. | Open |
| 8 | `21-reflection.md` | `setAccessible` section | Mentions the `--illegal-access` flag | The flag was removed in Java 17. Use `--add-opens` / `--add-exports`. | Won't fix (author: leave file as is) |
| 9 | `25-jvm-bytecode-essentials.md` | Initialization bullet | "Initialization... performed only once, when the class is first loaded" | Loading and initialization are separate. Initialization runs on first *active use* (see `26-class-loader.md` §8). | Won't fix (author: leave file as is) |
| 10 | `25-jvm-bytecode-essentials.md` | JIT Optimizations → Escape analysis | Non-escaping objects allocated "on the stack instead of the heap" | HotSpot does **scalar replacement** (the fields become locals or registers). It does not stack-allocate objects. | Won't fix |
| 11 | `25-jvm-bytecode-essentials.md` | Startup section | AOT via `jaotc` presented as available | `jaotc` was experimental in 9 and removed in JDK 17. | Won't fix |
| 12 | `25a-graalvm.md` | Closed-world section, line 83 | `--report-unsupported-elements-at-runtime` generates the config files | That flag only defers unsupported-feature errors to runtime. Config is generated with the tracing agent (`-agentlib:native-image-agent`), which the same file describes later. | Won't fix (author: leave file as is) |
