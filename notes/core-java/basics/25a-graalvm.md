# GraalVM in Detail
## What "GraalVM" actually is
- GraalVM is a high-performance runtime that provides support for multiple programming languages and execution modes. It is designed to run applications written in Java and other languages, as well as to compile these languages into native executables.
- It includes a Just-In-Time (JIT) compiler, an Ahead-Of-Time (AOT) compiler, and a polyglot runtime that allows interoperability between different languages.
- GraalVM is particularly known for its ability to compile Java applications into native images, which can significantly reduce startup time and memory footprint, making it suitable for cloud-native applications and microservices.

GraalVM is not one thing. It is a project umbrella with three technologies and several ways to get them:

| Piece | What it is |
|---|---|
| Graal compiler | An optimizing compiler written in Java. It can act as a JIT compiler inside a normal JVM, or as the code generator for ahead-of-time (AOT) builds. |
| Native Image | A build tool that compiles a Java application ahead of time into a standalone native executable, with no JVM needed on the target. |
| Truffle and polyglot | A framework for implementing other languages (JavaScript, Python and others) on top of Graal, and an API for embedding them in Java. |

Two terms you need:
- JIT (just-in-time) compiler: compiles bytecode to machine code while the program runs, once a method proves hot.
- AOT (ahead-of-time) compiler: compiles to machine code before the program ever starts.

## Graal as a JIT inside a regular JVM
- Normal HotSpot has two JIT compilers: C1 (fast to compile, light optimization) and C2 (slow to compile, heavy optimization). Both are written in C++. 
- Graal replaces C2 with a compiler written in Java. It plugs in through JVMCI (JVM Compiler Interface, available since Java 9), an interface that lets a Java-written compiler receive bytecode and hand machine code back to the JVM. 
- Why write a compiler in Java?
  - Maintainability and safety. A compiler bug in managed code is an exception, not a crash of the whole JVM process, and it is far easier to test and evolve than C++.
  - Optimization quality. Graal has strong partial escape analysis: it can avoid allocating an object on the paths where it doesn't escape, even if it escapes on other paths. It also inlines aggressively. Its gains over C2 vary a lot by workload, and are most visible on allocation-heavy, abstraction-heavy code (the kind frameworks produce).
  - One compiler, two uses. The same Graal codebase compiles hot methods at runtime in the JVM and compiles the whole application ahead of time in Native Image.

## Native Image: how it works
- Native Image is a tool that compiles Java applications into standalone executables. It performs an ahead-of-time (AOT) compilation of the application, including all its dependencies, into a native binary.
- The resulting native image has a significantly reduced startup time and lower memory footprint compared to running the application on a traditional JVM. This makes it particularly suitable for cloud-native applications, microservices, and serverless environments where fast startup and low resource usage are critical.
- Native Image performs aggressive optimizations, including dead code elimination, which removes unused classes and methods from the final binary. It also performs whole-program analysis to determine the reachable code and data, allowing for further optimizations.
- However, Native Image has some limitations compared to running on a traditional JVM. It does not support dynamic class loading, reflection, or certain Java features that rely on runtime introspection. Developers need to provide configuration files to specify which classes and methods should be included in the native image if they are accessed dynamically.

### The four build phases
- Points-to analysis (the closed-world step). Starting from main (or a Lambda handler), the tool computes every class, method and field that could be reached. Anything unreachable is not compiled and does not exist in the binary. This is why the executable is small and starts fast, and also why anything the analysis cannot see statically is a problem (section 5).
- Build-time initialization. Static initializers of selected classes run during the build, and the resulting objects are saved in the image heap: a pre-populated snapshot of the heap that is memory-mapped into the process at startup, so no initialization work happens at runtime.
- AOT compilation. The Graal compiler compiles all reachable methods to machine code for one specific OS and CPU architecture.
- Linking with Substrate VM. The machine code is linked with Substrate VM, a small runtime embedded in the executable.

## What Substrate VM contains, and what is absent
| Present | Absent |
|---|---|
| Garbage collector (Serial by default; Epsilon, a no-op collector, for tests; G1 in Oracle GraalVM only) | The JIT compiler and bytecode interpreter |
| Thread support, locks, safepoints | Class loading from `.class` files at runtime |
| Exception handling, stack walking | Bytecode verification (it already happened at build time) |
| JNI, JFR recording and heap dumps (supported in recent releases) | Metaspace and the code cache, because everything is already compiled |
| Basic signal and shutdown hooks | Java agents and JVMTI-based tooling |

Because there is no JIT, there is no warm-up and no profiling: the code that runs on the first request is the same as on the millionth. It is also why peak throughput is generally lower than a warmed-up HotSpot.

### Heap and memory behaviour
- A native image holds no class metadata, JIT data structures or code cache, so its baseline footprint is much lower. A freshly started JVM service often uses several times the resident memory of the same service as a native image. 
- Recent releases enable compressed 32-bit object references by default, which limits the heap to 32 GB. Disable it with -H:-UseCompressedReferences if you need more.
- The heap is still garbage-collected, and -Xmx and related flags still apply at runtime, but the collector options are fewer than HotSpot's.

### Build-time versus run-time initialization
This is the trickiest configuration decision, and it is a frequent source of subtle bugs.

| Initialized at build time | Initialized at run time |
|---|---|
| Faster startup, because the result is baked into the image heap | The class's static initializer runs when the process starts (or on first use) |
| The state it captured is frozen into the binary | Reads the real environment of the running process |

- Anything that must differ per process, if captured at build time, becomes a constant for every instance: environment variables, system properties, secrets, a SecureRandom seed, timestamps, a hostname, thread objects, open sockets. 
- The classic bug is a static final field that reads an environment variable or generates a random seed, silently frozen into the executable at build time. 
- Native Image provides --initialize-at-build-time=<package> and --initialize-at-run-time=<class> to choose. 
- For application classes the default is run-time initialization, and the JDK and many frameworks are configured for build-time. Frameworks like Spring AOT or Quarkus set most of this for you.

### Profile-guided optimization (PGO)
Since no JIT exists to observe real behaviour, Oracle GraalVM lets you:
- Build an instrumented executable.
- Run it against a representative workload, collecting a profile (.iprof file).
- Rebuild using that profile.

### Build characteristics
- Slow and memory-hungry builds: typically minutes, and several GB of RAM. Quick-build modes (-Ob) trade runtime speed for faster builds during development.
- One binary per OS and architecture. You cannot build a Linux executable on macOS. The usual solution is a CI step or a container build that targets the deployment platform.
- Static or mostly-static linking (including musl) allows very small container images, even scratch or distroless ones.

## The closed-world cost: what breaks and how to fix it
- Native Image's closed-world analysis is a static analysis that determines which classes, methods, and fields are reachable from the application's entry point. 
- Anything not reachable is excluded from the final binary. This can lead to issues if your application relies on dynamic features like reflection, dynamic class loading, or certain frameworks that use runtime introspection.
- To address these issues, you can provide configuration files to specify which classes and methods should be included in the native image. 
- These configuration files can be generated automatically by running your application with the `--report-unsupported-elements-at-runtime` option, which will log any unsupported elements that are encountered during execution. 
- You can then use this information to create the necessary configuration files to include those elements in the native image.
- Static analysis cannot see anything decided by a string or a runtime condition. These need reachability metadata, which is declared up front.

| Feature | Why analysis cannot see it | What you provide |
|---|---|---|
| Reflection (`Class.forName`, `getDeclaredMethod`, frameworks that instantiate by name) | The class name is a string | `reflect-config` entries, annotations, or framework hints |
| Resources (`getResourceAsStream`, config files, templates) | The path is a string | `resource-config` patterns |
| Dynamic proxies | The interface list is built at runtime | Proxy configuration |
| JNI | Native code calls back by name | JNI configuration |
| Java serialization | Classes are chosen by the stream | Serialization configuration |
| Runtime class loading, custom class loaders, runtime bytecode generation | The code doesn't exist at build time | Not supported (frameworks generate it at build time instead) |

### Getting the metadata
- Library-provided. Many libraries ship their own metadata inside the JAR, and a shared GraalVM Reachability Metadata repository covers popular libraries that don't.
- Framework AOT. Spring Boot 3 runs an AOT processing step that generates reflection and proxy hints from your bean definitions at build time. You add your own where needed through RuntimeHints or annotations such as @RegisterReflectionForBinding (relevant for JSON model classes that Jackson fills reflectively).
- The tracing agent. Run the app on a regular JVM with -agentlib:native-image-agent, exercise it thoroughly, and it records every reflective access, resource lookup and so on into config files. It captures only what your test run exercised, so coverage matters.
- Manual JSON config. The last resort.

### The typical failure mode
- The app works perfectly on the JVM and then fails after switching to native, only on a specific code path, with ClassNotFoundException, NoSuchMethodException, a missing resource, or a library that quietly falls back to a different behaviour. 
- Anything that is looked up by name is a prime suspect. Security and authentication plug-ins are a typical case: a class named in a config string (a Kafka SASL login module, a JDBC driver, a crypto provider) is only ever reached reflectively, so the analysis never sees it. Check the metadata first. 
- A few other differences to expect:
  - invokedynamic and method handles are resolved at build time. Runtime-spun lambdas and dynamic bytecode generation aren't available.
  - Some JDK behaviours differ: for instance, default time zone data and locale data can be trimmed, and security providers must be registered.
  - Debugging uses gdb and native tooling instead of JVM debuggers. JFR works, but jcmd-style live JVM inspection is much more limited.
  - Library compatibility is not guaranteed. Libraries that rely heavily on runtime bytecode generation or classpath scanning may need a native-aware alternative or version.

### HotSpot JVM vs GraalVM JVM vs Native Image
| Aspect | HotSpot JVM (any distribution) | GraalVM as a JVM (Graal JIT) | Native Image |
|---|---|---|---|
| How code becomes machine code | Interpreter, then C1, then C2 at runtime | Same, with Graal replacing C2 | AOT compiled at build time, no runtime compiler |
| Startup | Hundreds of ms to seconds | Same as HotSpot | Milliseconds |
| Warm-up | Yes, a first-requests penalty | Yes | None |
| Peak throughput | Excellent after warm-up | Excellent, sometimes better than C2 on some workloads | Usually lower, unless PGO is used (Oracle GraalVM) |
| Baseline memory | Higher: metadata, JIT, code cache | Higher | Much lower |
| Dynamic features (reflection, class loading, agents) | Fully supported | Fully supported | Restricted: metadata required, no runtime class loading |
| Speculation and deoptimization | Yes | Yes | None: no runtime profile to speculate on |
| GC options | Several (G1, ZGC, Parallel, Serial, Shenandoah) | Same | Fewer (Serial; G1 in Oracle GraalVM) |
| Build time | Seconds | Seconds | Minutes, with heavy memory use |
| Artifact portability | One JAR runs everywhere | Same | One binary per OS and architecture |
| Deployment | Needs a JRE or JDK | Same | Self-contained executable |
| Observability and debugging | Full JVM tooling | Full | Reduced (JFR yes, many JVM tools no) |
| Compatibility risk | Lowest | Low | Highest: tied to library support and metadata |
| Security surface | Large (whole JDK, runtime class loading) | Large | Smaller: only reachable code is included, no dynamic loading |


### When to Choose What

| Situation | Better fit |
|---|---|
| Serverless functions, scale-to-zero services, CLIs, short batch jobs | Native Image: cold start and memory dominate |
| Long-running, high-throughput services, steady load | HotSpot: JIT-optimized peak speed, full tooling |
| Heavy dynamic framework use, plug-in systems, runtime class loading | HotSpot |
| Need startup help but can't take native's constraints | JVM-level startup techniques below |
| Memory-constrained containers with many small instances | Native Image |


### Alternatives That Improve Startup Without Leaving the JVM

| Technique | Idea |
|---|---|
| Application Class Data Sharing (AppCDS) | Pre-parsed class metadata stored in an archive and memory-mapped |
| Snapshot and restore | Initialize once, snapshot the process, restore later (AWS Lambda SnapStart, CRaC) |
| JDK AOT cache (Project Leyden, Java 24 and later) | Stores loaded and linked classes and, in Java 25, method profiles from a training run, so the JVM starts warmer. It is still a normal JVM with full dynamic features. |
| Tiered compilation tuning | Limit to C1 for short-lived processes |


### Practical checklist for a native migration
- Confirm that your frameworks and libraries support native (Spring Boot 3+, Quarkus and Micronaut do, via AOT).
- Build on the target OS and architecture, usually in CI.
- Run the full test suite against the native binary, not just the JVM build. Many failures appear only on specific code paths.
- Exercise rarely-used paths (auth failures, error handlers, less-common endpoints) with the tracing agent to capture their metadata.
- Audit static initializers for values that must differ per process, and make sure they run at run time.
- Compare startup, resident memory and throughput against the JVM version under realistic load, since "native is always faster" is false for throughput.
- Plan for slower builds and different debugging.

### Trap summary
- "GraalVM" is an umbrella (compiler, Native Image, Truffle). Running on a GraalVM JDK does not mean you are using native executables.
- Graal-as-JIT replaces only the top tier (C2). Everything else in the JVM is unchanged.
- Stock OpenJDK does not include the Graal compiler. You need a GraalVM distribution or a separately added Graal.
- Native Image has no JIT, no interpreter, no runtime class loading. All code must be reachable at build time.
- Closed world means that anything found via a string (reflection, resources, proxies, JNI, serialization) needs declared metadata.
- "Works on the JVM, fails on native, only on one path" almost always means missing reachability metadata for a rarely-run branch.
- The tracing agent finds only what your test run exercised.
- Static state initialized at build time is frozen into every instance of the binary. Env vars, secrets, seeds and timestamps must be read at run time.
- A native binary is specific to one OS and CPU architecture and cannot be cross-built from another platform.
- Peak throughput is usually lower than a warmed-up JVM. PGO (Oracle GraalVM) narrows the gap.
- G1 and PGO in Native Image are Oracle GraalVM features and not in the Community Edition.
- Faster startup and lower memory are traded against build time, debugging convenience, library compatibility and some GC and tooling choices.
- For cold-start improvements without losing JVM flexibility, consider snapshot/restore or the JDK's AOT cache before committing to native.
- Verify the version and licence details of any specific release before standardizing, because this project changes quickly.

## Miscellaneous 
# Follow-ups on 10a

## 1. Why is native peak throughput lower than a warmed-up HotSpot?

The short answer: a JIT compiles with **evidence from the running program**, and an ahead-of-time compiler only has **what it can prove from the code**. Here is a small example.

```java
interface FeeRule { long fee(Account a); }
final class FlatFee    implements FeeRule { public long fee(Account a) { return 25; } }
final class PercentFee implements FeeRule { public long fee(Account a) { return a.balance() / 1000; } }

long totalFees(Account[] accounts, FeeRule rule) {
    long sum = 0;
    for (Account a : accounts) {
        if (a.status() == Status.SUSPENDED) {      // true for about 0.01% of accounts
            log.warn("skipping {}", a.id());
            continue;
        }
        sum += rule.fee(a);                         // interface call
    }
    return sum;
}
```

Suppose production runs this over millions of accounts, always with `FlatFee`, though `PercentFee` also exists in the codebase.

### What HotSpot does after warm-up

While the method runs in the interpreter and the lightly-compiled tier, the JVM records **profile data**: which branches were taken and how often, and which concrete classes arrived at each call site. When the top-tier compiler finally compiles `totalFees`, it uses that profile:

| Observation | What the compiler does |
|---|---|
| `rule.fee(a)` has only ever seen `FlatFee` (100%) | Replaces the interface call with a cheap type check plus the inlined body. The loop body becomes `sum += 25`. With the call gone, it can unroll and even vectorize the loop. |
| The `SUSPENDED` branch was never (or almost never) taken | Compiles **no code** for the logging path. It plants an "uncommon trap" there, and the hot path is straight-line code. |
| `a.status()` and `a.id()` are tiny getters | Inlines them into plain field reads, and hoists repeated null and bounds checks out of the loop. |
| The CPU supports AVX2 or AVX-512 | Emits instructions for the **actual** CPU it is running on. |

All of this is speculation, and it is safe because of **deoptimization**. If a `PercentFee` ever reaches that call site, or a suspended account turns up, the compiled code hits the guard or the trap, the JVM discards it, drops back to the interpreter at the exact bytecode position, and recompiles with the new facts.

### What a native image can do

An AOT compiler works on the code alone.

- **No profile, so no speculation.** It cannot assume "the suspended branch never runs" or "only `FlatFee` shows up". It has no way to back out if the assumption is wrong, because there is no interpreter or deoptimization in the executable. It must compile **both** paths and generate code valid for **every** possibility.
- **The closed world helps in one case.** If the whole-program analysis finds that only **one** implementation of `FeeRule` is reachable, it can devirtualize and inline with certainty. Here `PercentFee` is reachable, so the call stays an indirect call, or at best a small type switch, with no knowledge of which side is hot.
- **Baseline CPU target.** The binary is built for a fixed architecture level (often a conservative x86-64 baseline), not for the CPU in production. A JIT picks the best instructions at run time.
- **No adaptation.** Whatever decisions were made at build time are permanent. If traffic changes shape, nothing re-optimizes.

With **PGO** (Oracle GraalVM), you run an instrumented build on training traffic and feed the profile back. The compiler can then inline `FlatFee` behind a guard and put the cold branch out of line. That recovers a lot of the gap, but the profile is a snapshot from one training run, whereas HotSpot keeps adapting for the life of the process.

### Caveats

- "Lower peak" is a tendency, not a law. For small applications, short runs, or code that is mostly I/O, native often matches or beats the JVM, because the JVM spends its first minutes interpreting and compiling on background threads that compete with your requests.
- The gap shows up in long-running, CPU-bound, polymorphism- or allocation-heavy code, which is what big services with heavy frameworks tend to contain.
- The crossover depends on how long the process lives. A 200 ms Lambda never reaches the JIT's peak, while a service running for weeks does.

---

## 2. Build-time vs run-time initialization: consequences

A class's `static` initializer normally runs when the class is first used, in every process. Native Image offers a second option: **run the initializer once, during the image build**, and save the resulting objects into the executable's pre-populated heap.

```java
public final class FeeConfig {
    static final String REGION  = System.getenv("APP_REGION");            // (a) environment
    static final Instant LOADED = Instant.now();                          // (b) time
    static final Map<String, BigDecimal> RATES = loadRates();             // (c) resource file
    static final String API_KEY = System.getenv("PRICING_API_KEY");       // (d) secret

    private static Map<String, BigDecimal> loadRates() { /* reads /rates.json from the classpath */ }
}
```

| Field | Initialized at **build time** | Initialized at **run time** |
|---|---|---|
| `REGION` | The value of `APP_REGION` **on the build machine** (probably `null` in CI), frozen forever. Every deployment, in every region, sees that value. | Reads the real environment of each running process. Correct. |
| `LOADED` | The **build timestamp**. The "loaded at" value is weeks old when the app starts. | Process start time. Correct. |
| `RATES` | Loaded once during the build, and already in memory at startup with zero parsing cost. A benefit, and it needs no resource entry at run time. | Parsed on first use, in each process. Slower startup, but the resource must be included in the image. |
| `API_KEY` | **The secret is baked into the binary.** It lives in every copy of the artifact, in your registry, in build logs and caches, and rotating it requires a rebuild. | Read from the environment of the running process. Correct. |

### Rules of thumb

| Safe at build time | Must be at run time |
|---|---|
| Pure computation producing constants (lookup tables, parsed immutable config shipped inside the artifact) | Anything reading environment variables, system properties, hostnames, time, or secrets |
| Classes with no dependence on the process environment | Anything holding **threads, sockets, open files, or connections** (the build detects some of these and fails) |
| Immutable data structures | Random seeds and `SecureRandom` state (a seeded `java.util.Random` frozen into the image would produce the same sequence in every process, and the build rejects some such cases) |

### What is shared and what isn't

At startup the image heap is mapped into the process. Each process starts from the same snapshot, then diverges as it mutates its own copy. So build-time-initialized **mutable** state is a template, not shared memory between processes. But anything that was computed from the build environment is the same in every one.

### How you control it

```
--initialize-at-build-time=com.ig.pricing.RateTables
--initialize-at-run-time=com.ig.pricing.FeeConfig
```

The default for **your** application classes is run-time initialization. JDK classes and many libraries are build-time initialized by design, and frameworks such as Spring AOT or Quarkus configure the rest. The trap is a library or class that was configured for build-time and reads the environment. Initialization of a class also forces initialization decisions on what it depends on, so a build-time class that touches a run-time-only class produces a build error.

---

## 3. What exactly are we losing with native builds?

| Lost entirely | Partly lost or reduced | Kept |
|---|---|---|
| **Runtime class loading** and custom class loaders (plug-in systems, hot reload) | **Profile-based optimization**, with peak throughput generally lower (PGO partly compensates) | Garbage collection (fewer collector choices) |
| **Runtime bytecode generation** (CGLIB-style proxies, dynamic subclassing, ASM at runtime) | **Reflection**, which is available only for what you declared | Java language semantics and the JDK libraries you reach |
| **Java agents and JVMTI** (`-javaagent`) | **Live JVM diagnostics** (`jcmd`, attach-based tooling, many JMX features) | JFR recording and heap dumps (supported in recent releases) |
| **The "one JAR runs anywhere" property** | **Build speed and developer feedback loop** (minutes, heavy memory) | Threads, locks, memory model, exceptions |
| **JIT warm-up adaptation** and deoptimization | **Debugging** (native tooling like `gdb`, not a JVM debugger) | Libraries that are native-compatible |
| **Library freedom**: anything depending on the above | **Behavior parity**: the same code can behave differently, so you must test the native binary itself | A much smaller startup time and memory footprint |

The loss that most affects backend teams in practice is **agents**. Commercial APM products and the OpenTelemetry Java agent work by instrumenting bytecode as classes load, and none of that exists in a native image. Observability has to come from library-level instrumentation instead (for example Micrometer and Micrometer Tracing wired into the framework), which is a design change, not a flag.

You also lose some **peace of mind**. On a JVM, a missing class or rarely-used feature fails loudly in a way you can inspect. In native, a missing-metadata failure may appear only on one path in production, so test coverage of the binary matters far more.

---

## 4. Why can't the analysis "see" a string?

Think of the analysis as a **call-graph tracer**. It starts at the entry point and follows the **instructions** in the bytecode: this method calls that method, which creates that class, which reads that field. Every edge is explicit in the code.

Now look at reflection:

```java
String name = config.get("driver.class");        // value comes from a file, env var, DB, user input
Class<?> c = Class.forName(name);                 // which class? Only known when the program runs
Object driver = c.getDeclaredConstructor().newInstance();
```

At build time the analysis sees "call `Class.forName` with some string". The string's value does not exist yet. It is read from a config file at run time, and may differ between environments. To know which classes to keep, the analysis would have to know **every possible runtime value**, which in general means running the program on all inputs. That is impossible (undecidable in general), so the tool keeps nothing it cannot prove reachable. The class is stripped, and at run time `Class.forName` throws `ClassNotFoundException`.

Some simple cases do work. If the arguments are **compile-time constants**, such as `Class.forName("com.ig.Foo")` or `Foo.class.getMethod("bar")` with literal names, Native Image can resolve them during analysis and register them for you. Anything computed, concatenated, read from config or passed in from elsewhere is invisible.

Reflection has a second problem. Even when the class is kept, the executable normally **drops the names and metadata** of its methods and fields to save space. A reflective `getDeclaredMethod("getName")` needs both compiled code **and** a metadata entry saying "this class has such a method", and that entry exists only if it was requested.

### Why resources are even less visible

```java
InputStream in = getClass().getResourceAsStream("/rates/jp.json");
```

On a JVM, this reads a file out of a JAR on the classpath. The native executable has **no JARs and no classpath**. It is a single binary. A resource is not code, so the call-graph analysis has no reason to consider it. Unless you declare "include `rates/*.json`", the build leaves the file out, and the lookup at run time returns `null`, usually followed by a `NullPointerException` somewhere downstream.

---

## 5. What does metadata look like, and what does the repository cover?

### Your own metadata

Metadata lives in JSON files under `META-INF/native-image/<group>/<artifact>/` inside your project. In recent releases the format is a single unified file (older releases used separate `reflect-config.json`, `resource-config.json` and so on, and the format has been evolving, so check the documentation for your exact version).

```json
{
  "reflection": [
    {
      "type": "com.ig.model.Client",
      "allDeclaredConstructors": true,
      "allDeclaredFields": true,
      "methods": [
        { "name": "getName",  "parameterTypes": [] },
        { "name": "setEmail", "parameterTypes": ["java.lang.String"] }
      ]
    },
    {
      "type": "org.apache.kafka.common.security.scram.ScramLoginModule",
      "allDeclaredConstructors": true
    }
  ],
  "resources": [
    { "glob": "rates/*.json" },
    { "glob": "messages*.properties" }
  ]
}
```

It reads as "keep these classes with these members, reachable by reflection, and embed these files". The second `reflection` entry is the kind of thing a SASL/JAAS setup needs, because its login module is named in a **config string**, so no code ever references it.

Entries can be made conditional, so they only apply if some other class is reachable:

```json
{ "condition": { "typeReached": "com.ig.web.ClientController" },
  "type": "com.ig.model.Client", "allDeclaredFields": true }
```

In a Spring Boot project you normally don't write the JSON by hand. You declare hints in code and the AOT step generates it:

```java
@RegisterReflectionForBinding(Client.class)        // Jackson will reflectively populate Client
@ImportRuntimeHints(AppHints.class)
@Configuration
class NativeConfig { }

class AppHints implements RuntimeHintsRegistrar {
    public void registerHints(RuntimeHints hints, ClassLoader cl) {
        hints.resources().registerPattern("rates/*.json");
    }
}
```

### The reachability metadata repository

The **GraalVM Reachability Metadata repository** is a community-maintained collection of metadata for **third-party libraries that don't ship their own**. It lives on GitHub under the `oracle` organization (`graalvm-reachability-metadata`), and the Gradle and Maven native build plugins download and apply the matching entries automatically. The layout is, roughly:

```
metadata/
  org.postgresql/postgresql/
    index.json                       // which versions are covered
    42.7.3/
      reachability-metadata.json     // reflection, resources, etc. for that library version
```

with an index that looks like:

```json
[ { "metadata-version": "42.7.3",
    "tested-versions": ["42.7.2", "42.7.3"],
    "latest": true } ]
```

(Illustrative: the exact field names evolve.) Each entry is backed by tests that run the library in a native image.

What it covers: **the library's own internals**, such as which of its classes it instantiates reflectively and which files it reads. What it does **not** cover:

- **Your own classes.** Your JSON models, your SASL login configuration and your resources are yours to declare.
- **Library versions nobody has tested.** Coverage is per version, and it is not complete for every library.
- **Application-specific use of a library.** If you tell Jackson to bind your `Client`, the library's metadata can't know that.

Many libraries now ship metadata inside their own JAR, which is the best outcome. The repository fills the gap for those that don't.

---

## 6. "Lambda": it's AWS Lambda, not Java lambda expressions

When I said "runtimes on Lambdas" and "Lambda cold starts", I meant **AWS Lambda**, the serverless compute service where you upload a function and AWS runs it on demand. That has nothing to do with Java's `x -> x + 1` lambda expressions, an unrelated feature. The name clash is just unlucky.

### What happens on AWS Lambda

```java
public class DepositHandler implements RequestHandler<SQSEvent, Void> {

    private static final ObjectMapper MAPPER = new ObjectMapper();       // runs once per environment
    private static final DynamoDbClient DDB   = DynamoDbClient.create();  // runs once per environment

    @Override
    public Void handleRequest(SQSEvent event, Context ctx) {              // runs on every invocation
        for (var msg : event.getRecords()) { /* parse and write */ }
        return null;
    }
}
```

AWS creates an **execution environment** (a lightweight micro-VM) to run your code, and its lifecycle has two phases:

| Phase | What happens |
|---|---|
| **Init (the cold start)** | 1. AWS fetches your code package or container image. 2. It starts the micro-VM. 3. It starts the **runtime**: for Java, a JVM. 4. The JVM loads and verifies your classes. 5. Your **static initializers run** (`MAPPER`, `DDB`), and in a Spring app the whole application context is built. |
| **Invoke** | `handleRequest` runs for the event. After it returns, the environment is **frozen**, not destroyed. |

If another event arrives soon, AWS reuses the frozen environment (a **warm start**) and skips Init. If many events arrive at once, or the environment has been idle for a while, AWS creates **new** environments, and each of those pays the full cold-start cost. That is why static fields for expensive clients (`MAPPER`, `DDB`) are the standard idiom: they are paid for once per environment, not per request.

A JVM cold start is expensive because everything described earlier happens within that Init window: JVM boot, class loading and verification, framework startup, and then the first invocation running in the **interpreter**, since nothing has been JIT-compiled yet. Lambda also allocates **CPU in proportion to memory**, so a small memory setting means a slow JVM warming up.

### Where native fits in

For Java there are two ways to run:

| | Managed Java runtime | Native image on a custom runtime |
|---|---|---|
| What AWS provides | A JVM, with your JAR | An OS-only runtime (`provided.al2023` or similar) |
| What you upload | JAR or ZIP | A single executable, conventionally named `bootstrap` |
| How it starts | JVM boots, loads classes, runs init | The executable starts, maps its image heap, and polls AWS's **Runtime API** in a loop (get next event, run handler, post result) |
| Cold start | Seconds with a framework, or hundreds of ms for plain code | Typically tens to low hundreds of ms |

Libraries such as the AWS Lambda Java runtime interface client, or Spring Cloud Function's adapter, implement that Runtime API loop for you.

---

## 7. AppCDS vs snapshot/restore: different data, different mechanism

They are **not** the same kind of thing, and they save different amounts of work.

### AppCDS (Application Class Data Sharing)

| | |
|---|---|
| **What is stored** | A **file of class metadata**: your (and the JDK's) classes already parsed into the JVM's internal format, with details like the constant pool and linking info. It can also hold some archived JDK heap objects. |
| **What it is not** | Not your running application's state: no heap contents of your objects, no results of your static initializers, no live threads. |
| **How it's used** | The JVM **boots normally** and memory-maps the archive file instead of finding, parsing and verifying each class from the JAR. |
| **What still happens each start** | JVM boot, **every static initializer**, framework startup (Spring still builds its context), and the interpreter and JIT warm-up. |
| **What it saves** | Class loading, parsing and verification time. Typically a modest but real startup improvement. |
| **Mechanism** | An archive created by a training run (`-XX:ArchiveClassesAtExit=app.jsa`, or `-XX:+AutoCreateSharedArchive`), used with `-XX:SharedArchiveFile=app.jsa`. It is validated against the classpath, so it breaks if the JARs change. |

### Snapshot/restore (Lambda SnapStart, CRaC)

| | |
|---|---|
| **What is stored** | A **snapshot of the whole initialized execution environment**: the process memory, including the heap with your objects, initialized statics, loaded classes and the state of the framework after startup. |
| **How it's used** | The snapshot is taken **after the Init phase**. Later cold starts **restore** that memory image instead of re-running Init. |
| **What it saves** | The **entire initialization**: JVM boot, class loading, static initializers, and the framework's startup. That is the biggest slice of a Java cold start. |
| **What it does not do** | Make the code fast: the JIT's profile and compiled code are not what the mechanism is about. Throughput after restore depends on warm-up as usual. |
| **Mechanism** | On Lambda, SnapStart snapshots the micro-VM's memory and disk state after init, and restores it on demand. CRaC (a JVM feature available in some distributions) checkpoints a process, in the style of CRIU. |

### The cost: state that must not survive the snapshot

Because your **running state** is frozen and then cloned into many environments, you must handle what should **not** be identical or still valid:

- Open sockets, database connections, HTTP clients (they are stale after restore).
- Random number generator state (every restored copy would produce the same "random" values).
- Cached credentials or tokens, and DNS or time-based caches.
- Anything that captured a unique ID or timestamp during init.

Hooks exist for this (CRaC's `beforeCheckpoint` and `afterRestore`, and Lambda's runtime hooks). It is the same class of bug as build-time initialization in a native image: "I computed something in a different time and place than where it's used".

### Side by side

| | AppCDS | Snapshot/restore | Native Image |
|---|---|---|---|
| **What is precomputed** | Parsed class metadata | The whole initialized process memory | The compiled code **and** an initialized heap, at build time |
| **Process still boots a JVM?** | Yes | Restores one that already booted | No JVM |
| **Your static initializers rerun?** | Yes | No (already run, state is captured) | Depends on build-time versus run-time choice |
| **JIT still warms up?** | Yes | Yes | No JIT at all |
| **Compatibility risk** | Very low | Medium (state hygiene) | Highest (closed world) |

They can be **combined** (snapshot a JVM that also uses an AppCDS archive). The JDK's newer AOT cache work (Project Leyden) extends the AppCDS idea, caching not just parsed classes but also linked state and profile information from a training run, so the JVM starts warmer. It is still a JVM cache, not a process snapshot.

---

If it helps, the next step could be a worked native build of the `DepositHandler` above: the same code run as a managed-runtime JAR, as a SnapStart function and as a native `bootstrap`, with the build-time-versus-run-time and metadata failures reproduced and fixed. Otherwise, G11 (Class Loading & Initialization Internals) is next when you're ready.




