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






