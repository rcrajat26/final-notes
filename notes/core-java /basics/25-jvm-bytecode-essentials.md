# JVM & Bytecode Essentials
## The big picture 
```
Account.java ──javac──► Account.class ──► ClassLoader ──► Runtime data areas ──► Execution engine
 (source)   (compile)   (bytecode)        (load, link)     (heap, stacks,         (interpreter + JIT,
                                                             Metaspace)             GC runs alongside)
```

### Source code 
- Java source code is written in `.java` files.
- The source code is human-readable and contains the logic of the program.
- The Java compiler (`javac`) translates the source code into bytecode, a compact instruction set for an abstract machine, which is stored in `.class` files.
- It does very little optimization, and most speed-ups happen later at runtime.
- The JVM is the program that loads, verifies and executes that bytecode. Bytecode is the portable layer ("write once, run anywhere"), and the JVM is the platform-specific layer.
- The JVM specification defines behavior, and implementations build it. 
- HotSpot (OpenJDK, Oracle, Temurin and others) is the one nearly everyone runs. Others include Eclipse OpenJ9 and GraalVM.
- The JVM runs bytecode, not Java. Kotlin, Scala and Groovy compile to the same bytecode, so the JVM is a language-neutral platform.

### The `.class` file
- The `.class` file is a binary file that contains the bytecode and metadata for a Java class or interface.
- It contains:
  - The constant pool: a table of constants (strings, numbers, class references, method references, etc.) used by the class.
  - The class declaration: the name of the class, its superclass, and its interfaces.
  - The fields: the instance and static variables of the class.
  - The methods: the instance and static methods of the class, including their bytecode.
  - The attributes: additional information about the class, such as source file name, line number tables, and annotations.

**Java Class File Structure**

| Part | Contents |
|---|---|
| Magic number | `0xCAFEBABE`, so the JVM can reject non-class files immediately |
| Version | Minor and major version. The JVM refuses to load a class whose major version is newer than it supports (`UnsupportedClassVersionError`). |
| Constant pool | A table of every literal and symbolic reference the class uses: strings, numbers, class names, field and method names with their descriptors |
| Access flags, this class, super class, interfaces | The class's identity and its supertypes |
| Fields and methods | Name, descriptor, flags, attributes. A method's bytecode sits in its `Code` attribute. |
| Other attributes | Line numbers, local variable names (only with `-g`), annotations (per retention), `InnerClasses`, `NestHost`/`NestMembers`, `Record`, `PermittedSubclasses`, `BootstrapMethods` |

**Major version numbers**: Java 8 = 52, 11 = 55, 17 = 61, 21 = 65. Code compiled with a newer JDK can't run on an older JVM unless you compile with --release N. Using --release rather than -source/-target also checks against the real API of that version.

**Descriptors**
- Descriptors are strings that describe the types of fields and methods in a class.
- Field descriptors specify the type of a field.
- Method descriptors specify the parameter types and return type of a method.
- Descriptors use a compact notation, for example:
  - `I` for `int`
  - `J` for `long`
  - `F` for `float`
  - `D` for `double`
  - `Ljava/lang/String;` for `String`
  - `[I` for `int[]`
  - `(I)I` for a method that takes an `int` and returns an `int`
  - `(Ljava/lang/String;I)V` for a method taking a `String` and an `int`, returning `void`

**Inspecting with javap**
```
javap -c -p Account.class      # disassemble methods (-p includes private)
javap -v Account.class         # everything: constant pool, flags, attributes
```

### Runtime data areas
**JVM Runtime Memory Areas**

| Area | Scope | Holds |
|---|---|---|
| Heap | Shared by all threads | All objects and arrays. Managed by the garbage collector. |
| Metaspace ("method area") | Shared | Class metadata: field and method layouts, bytecode, runtime constant pool. It lives in native memory (Java 8+, replacing PermGen) and grows on demand. |
| JVM stack | One per thread | A stack of frames, one per active method call |
| PC register | One per thread | Address of the bytecode instruction currently executing |
| Native method stack | One per thread | Used when the thread calls native (JNI) code |
| Code cache | Shared | Native machine code produced by the JIT (section 6) |

- The JVM divides memory into several runtime data areas, each with a specific purpose:
    - **Heap**: Stores all objects and their instance variables. Managed by the garbage collector.
    - **JVM Stack**: Each thread has its own stack, which stores frames for method calls, including local variables and operand stacks.
    - **Metaspace**: Stores class metadata, including class definitions, method data, and constant pool information. Replaces the old PermGen space in Java 8 and later.
    - **Program Counter (PC) Register**: Keeps track of the address of the currently executing instruction for each thread.
    - **Native Method Stack**: Used for native methods written in languages like C or C++.

Errors map to areas: OutOfMemoryError: Java heap space (heap), OutOfMemoryError: Metaspace (too many or too large loaded classes, often a class-loader leak), StackOverflowError (a thread's stack is full, typically from runaway recursion; size set by -Xss, commonly 512 KB to 1 MB by default), and OutOfMemoryError: unable to create native thread (the OS refused another thread's stack).

**The frame**:
- Each method call creates a new frame on the thread's stack. A frame contains:
  - Local variables: space for method parameters and local variables.
  - Operand stack: used for intermediate calculations and passing arguments to methods.
  - Frame data: references to the constant pool, return address, and other bookkeeping information.
- When a method is invoked, a new frame is pushed onto the stack. When the method returns, the frame is popped off the stack.
- The size of the stack is limited, and if a thread exceeds its stack size (e.g., due to deep recursion), a `StackOverflowError` is thrown.
- The JVM stack is thread-local, meaning each thread has its own stack and does not share it with other threads. This isolation helps prevent data races and ensures that method calls are independent across threads.
execution state, including the current instruction being executed and the values of local variables and operands.

### Bytecode: a stack machine
- Bytecode instructions operate on the operand stack rather than on named registers. 
- Each instruction is one byte of opcode plus operands, which is where "bytecode" comes from.
- The JVM is a stack-based virtual machine, meaning it uses an operand stack to perform operations rather than registers.
- Each method has its own operand stack, which is used to pass parameters, store intermediate results, and return values.
- Bytecode instructions operate on the operand stack, pushing and popping values as needed. For example, to add two integers, the bytecode would push both integers onto the stack, perform the addition, and then pop the result off the stack.
- The stack-based architecture simplifies the instruction set and allows for a more compact representation of code, but it can also lead to performance overhead due to frequent stack operations. The JIT compiler can optimize these operations at runtime to improve performance.

**Example 1: arithmetic**
```java
int total(int a, int b) {       // slot 0 = this, 1 = a, 2 = b, 3 = c
    int c = a + b;
    return c * 2;
}
```

```bytecode
0: iload_1     // push a
1: iload_2     // push b
2: iadd        // pop both, push a+b
3: istore_3    // pop into c
4: iload_3     // push c
5: iconst_2    // push constant 2
6: imul        // pop both, push c*2
7: ireturn     // pop and return it
```
The `i` prefix means int. There are parallel `l`, `f`, `d` and `a` (reference) families. Types are encoded in the opcode, which is part of what makes the code verifiable.

**Example 2: creating an object**
```java
Account a = new Account("A-1");      // in a static method with no parameters
```

```bytecode
0: new           #7    // class Account        → allocate, push uninitialized reference
3: dup                 //                       → duplicate the reference
4: ldc           #9    // String "A-1"         → push constant-pool string
6: invokespecial #11   // Account."<init>":(Ljava/lang/String;)V
9: astore_1            // store reference in local 1
```

Key points:
- `new` only allocates and leaves a reference to an object whose fields hold defaults. It does not run the constructor.
- The constructor is a method named `<init>`, called with `invokespecial`. That call consumes the reference as its `this` argument, which is why `dup` makes a second copy first, so one remains to be stored.
- Constructor code runs after allocation, which is why a superclass constructor calling an overridable method can observe a half-initialized subclass.
- Static initializers compile to a method named `<clinit>`.

**The five invoke instructions**
| Instruction | Used for | How the target is chosen |
|---|---|---|
| `invokestatic` | Static methods | Fixed at link time |
| `invokespecial` | Constructors (`<init>`) and `super.method()` calls | Fixed, no virtual lookup |
| `invokevirtual` | Ordinary instance methods on a class type (including private ones since Java 11) | Runtime class of the receiver, via the vtable |
| `invokeinterface` | Methods called through an interface type | Runtime class of the receiver, via an itable-style lookup |
| `invokedynamic` | Call sites linked at runtime by a bootstrap method | Decided once, on first execution, then cached |


Other things worth knowing
- Control flow is jumps: `if_icmplt`, `goto` and so on. Loops are conditional jumps backward.
- `switch` compiles to `tableswitch` (dense int cases, O(1) jump) or `lookupswitch` (sparse, binary search).
- `try/catch` has no instructions. The Code attribute carries an exception table: ranges of bytecode indexes mapped to handler addresses and catch types. When an exception is thrown the JVM searches the table. The cost on the non-throwing path is zero. `finally` is implemented by javac copying the finally code into each exit path.
- `synchronized` blocks compile to `monitorenter` and `monitorexit`. A synchronized method instead has a method flag.
- Generics are erased before bytecode, so the JVM sees raw types plus casts (flagged only, the generics topic is excluded).
- Nested-class private access: before Java 11 javac generated synthetic `access$000` bridge methods. Java 11 added nestmates (JEP 181), letting nested classes access each other's private members directly, with the relationship recorded in the `NestHost`/`NestMembers` attributes.
- Constant folding is done at compile time: `3600 * 24` becomes `86400` in the bytecode.

### From class file to running: loading, linking, initialization
- **Loading**: 
  - The class loader reads the `.class` file and creates the class's metadata in Metaspace and its Class object on the heap. 
  - The class loader does not run any code in the class, including static initializers. It just reads the bytes and sets up the data structures.
- **Linking**: The JVM verifies the bytecode, allocates memory for static fields, and resolves references to other classes, fields, and methods. Linking consists of three steps:
  - **Verification**: The JVM checks the bytecode for correctness and security. It ensures that the bytecode adheres to the JVM specification and does not violate access control or type safety.
  - **Preparation**: The JVM allocates memory for static fields and initializes them to their default values.
  - **Resolution**: The JVM resolves symbolic references to other classes, fields, and methods. This step may involve loading additional classes if they are not already loaded.
- **Initialization**: The JVM executes the static initializers and static blocks of the class. This step is performed only once, when the class is first loaded. The static initializers and static blocks are executed in the order they appear in the source code.

### The execution engine: interpreter and JIT
- The execution engine is responsible for executing the bytecode instructions. The JVM runs bytecode in two ways, and uses both together.:
  - **Interpreter**: The interpreter reads and executes the bytecode instructions one at a time. Executes bytecode instruction by instruction. It is simple and portable but can be slow for frequently executed code. It also counts how often each method is invoked and how often each loop jumps backward.
  - **Just-In-Time (JIT) Compiler**: Once code is "hot", The JIT compiler translates bytecode into native machine code at runtime and stores the result in the code cache. It optimizes the code based on runtime profiling information, such as which methods are called most frequently. The JIT compiler can significantly improve performance by eliminating the overhead of interpretation and applying various optimizations, such as inlining, loop unrolling, and dead code elimination.
- The JIT compiler works in conjunction with the interpreter. Initially, the interpreter executes the bytecode, and when a method is called frequently, the JIT compiler compiles it into native code. The native code is then cached and executed directly for subsequent calls, resulting in improved performance.
- The JIT compiler can also perform adaptive optimizations, where it monitors the execution of the code and applies further optimizations based on the observed behavior. This allows the JVM to adapt to changing workloads and improve performance over time.
- The execution engine also manages the garbage collection process, which automatically reclaims memory occupied by objects that are no longer reachable. The garbage collector runs in the background and can be triggered by various events, such as low memory conditions or explicit calls to `System.gc()`. The JIT compiler and garbage collector work together to ensure efficient memory management and optimal performance of Java applications.

HotSpot has two compilers:

| Compiler | Purpose | Characteristics |
|---|---|---|
| C1 (client) | Quick startup, low memory footprint | Optimizes for fast compilation and low memory usage, suitable for desktop applications and small devices. |
| C2 (server) | High performance, long-running applications | Optimizes for maximum performance, suitable for server applications and long-running processes. It performs more aggressive optimizations, such as inlining, loop unrolling, and escape analysis, to improve the performance of frequently executed code. |

## JIT Optimizations
- Inlining: 
  - The JIT compiler replaces a method call with the body of the called method. This eliminates the overhead of the method call and allows for further optimizations, such as constant folding and dead code elimination.
  - Inlining is particularly effective for small methods, such as getters and setters, which can be completely eliminated from the bytecode.
- Devirtualization: 
  - The JIT compiler can convert a virtual method call (which requires a runtime lookup) into a direct method call when it can determine that only one implementation of the method is possible. This allows for further optimizations, such as inlining.
- Escape analysis: 
  - The JIT compiler can analyze the code to determine if an object is only used within a single method and does not escape to other methods or threads. If an object is determined to be non-escaping, the JIT compiler can allocate it on the stack instead of the heap, which reduces the overhead of garbage collection. Additionally, if an object is non-escaping, the JIT compiler can eliminate locks on the object, as it is guaranteed to be thread-confined.
- Loop optimizations: 
  - The JIT compiler can perform various optimizations on loops, such as unrolling (expanding the loop body to reduce the number of iterations), hoisting invariant computations (moving computations that do not change within the loop outside of the loop), and removing redundant bounds and null checks (eliminating unnecessary checks that are guaranteed to be true).
- Constant folding and dead-code elimination: 
  - The JIT compiler can evaluate constant expressions at compile time and replace them with their computed values. Additionally, it can remove code that is never executed or whose results are never used, reducing the overall size of the bytecode and improving performance.
- Lock coarsening and elision: 
  - The JIT compiler can merge adjacent synchronized blocks into a single block, reducing the overhead of acquiring and releasing locks. Additionally, if an object is determined to be thread-confined, the JIT compiler can eliminate locks on that object, as it is guaranteed to be accessed by only one thread.


### Startup: reducing warm-up cost
- The JVM has a warm-up period where it collects profiling information and compiles hot code. This can lead to slower startup times for applications, especially for short-lived programs.
- To reduce warm-up cost, the JVM can use tiered compilation, where it starts with the interpreter and gradually compiles hot code using the C1 compiler, and then further optimizes it using the C2 compiler. This allows the JVM to balance startup time and long-term performance.
- The JVM can also use ahead-of-time (AOT) compilation, where the bytecode is compiled into native code before the application is run. This can reduce startup time, but may result in less optimized code compared to JIT compilation, as the AOT compiler does not have access to runtime profiling information. AOT compilation is available in Java 9 and later, and can be used with the `jaotc` tool to compile Java classes into native code. However, AOT compilation is not as widely used as JIT compilation, as it requires additional setup and may not provide the same level of performance optimizations as JIT compilation.

Because the JVM interprets, loads, verifies and profiles before reaching peak speed, short-lived and bursty workloads pay for it. The standard mitigations:
- **Tiered compilation**: start with the interpreter, then C1, then C2. This is the default since Java 8.
- **Ahead-of-time (AOT) compilation**: compile to native code before running.
- **Class data sharing (CDS)**: The JVM can share class metadata across multiple JVM instances, reducing memory usage and improving startup time. This is available in Java 9 and later, and can be enabled with the `-Xshare:dump` option to create a shared class data archive, which can then be used by multiple JVM instances with the `-Xshare:on` option.
- **Snapshot/restore**: Start once, snapshot the initialized process, restore for later invocations (AWS Lambda SnapStart, CRaC).

### Performance by Runtime Scenario
| Scenario             | What dominates                                                                      |
|----------------------|-------------------------------------------------------------------------------------|
| First request        | Class loading, interpretation, early compilation                                    |
| Steady-state service | C2-compiled hot paths. GC and I/O dominate.                                         |
| Cold Lambda (JVM)    | Whole startup chain:JVM boot, class load, framework init, first-call interpretation |
| Cold Lambda (native) | Process start and runtime init only                                                 |

### Trap summary
- A class file records name + descriptor for members, so a changed signature is a different member at the JVM level, even if it looks similar in source. 
- Class file version must be ≤ what the running JVM supports, or you get UnsupportedClassVersionError. Use --release when targeting older runtimes. 
- new only allocates. The constructor (<init>) runs afterward via invokespecial, which is why dup appears. 
- long and double use two local-variable and operand-stack slots. 
- try/catch costs nothing on the normal path (exception table). finally is duplicated into each exit path by the compiler. 
- invokedynamic is not just for lambdas. It has been used for string concatenation since Java 9, and its bootstrap runs once per call site. 
- Verification happens at link time, so a bytecode problem appears as VerifyError when the class is first linked, not at compile time. 
- The JIT optimizes by speculation. Loading a new implementation class later can trigger deoptimization and recompilation. 
- Monomorphic and bimorphic call sites get inlined. Megamorphic ones don't. 
- A full code cache silently stops further compilation. 
- StackOverflowError = stack of one thread exhausted. OutOfMemoryError: Metaspace = too many or too large loaded classes. They are different areas with different fixes. 
- Metaspace is native memory (Java 8+), not part of the heap, and it is unbounded by default unless -XX:MaxMetaspaceSize is set. 
- Native Image uses a closed-world assumption. Reflection, proxies, resources and serialization need explicit metadata, and runtime class loading is not available.
- "Faster startup" in Native Image trades away peak throughput and dynamic flexibility. It is a trade-off, not a free upgrade. 
- javap -c -p -v is the ground truth for what the compiler generated. Use it instead of guessing.

## Miscellaneous 
# Follow-ups on G10

## 1. Interpreter vs JIT: is anything converted to machine code before "hot"?

Correct: before code is hot, **nothing about your method is converted to machine code**. But "interpreter" doesn't mean "no machine code runs". The interpreter itself is a native program, and your bytecode is its **input data**.

Conceptually, the interpreter is a loop:

```
while (true) {
    opcode = bytecode[pc++];
    switch (opcode) {
        case iload_1: push(locals[1]);                 break;
        case iadd:    b = pop(); a = pop(); push(a+b); break;
        case ireturn: ...
    }
}
```

Each bytecode instruction is **looked up and dispatched at runtime, every time it executes**. Run a loop body a million times and the dispatch is repeated a million times. (HotSpot's "template interpreter" is a refinement: at JVM startup it generates a small native snippet per opcode, and dispatch jumps between snippets. It is still instruction-by-instruction, and nothing is specific to your method.)

| | Interpreter | JIT-compiled |
|---|---|---|
| Your method's bytecode | Read and dispatched each execution | Translated once into native instructions specific to that method |
| Machine code that exists for your method | None. Only the generic interpreter code runs. | Yes, stored in the code cache |
| Per-execution overhead | Decode and dispatch every instruction | None, the CPU runs the instructions directly |
| Optimization across instructions | None | Inlining, escape analysis, and so on |

Why not compile everything up front? Compilation costs time and memory, and most methods run only a few times. Compiling them wastes more than it saves. Counting first and compiling only the hot ones spends effort where it pays off, and the counters double as the **profile data** the optimizing tier later uses for speculation.

Also note that the bytecode **stays**. The compiled version is an additional copy. The original bytecode is kept so the JVM can fall back to the interpreter on deoptimization.

---

## 2. What is inlining replacing, and why does the method "disappear"?

I need to correct the wording in the module's table: the method is **not eliminated from the bytecode**. Your `.class` file and the method's bytecode are untouched. Inlining happens **inside the compiler, in the generated machine code**, for one specific compiled method.

**What it replaces: a call site, in the compiled code of the caller.**

```java
int fee(Account a) {
    return a.getBalance() * 2;
}
// Account:  int getBalance() { return balance; }
```

Without inlining, the compiled `fee` contains: set up arguments, **jump to** `getBalance`'s code, run it, **jump back**, take the return value. With inlining, the compiler copies the callee's body into the caller at that point:

```
before:   result = CALL getBalance(a)  * 2
after:    result = a.balance           * 2      // the call is gone; the body is pasted in
```

So what "disappears" is the **call instruction and its overhead** (argument passing, jump, stack frame creation, return), not the method. `getBalance` still exists in the class and can be called from anywhere else. If it is called from 20 places and all are inlined, there are 20 pasted copies in compiled code, plus the original still available for non-inlined callers and the interpreter.

**Why this matters more than just saving the call overhead:** once the callee's body sits in the caller, the optimizer sees one larger piece of code instead of two opaque ones, and it can optimize across the old boundary:

- `a.balance` can be loaded once and reused if used twice.
- A null check on `a` that would have happened inside the callee may be proven redundant.
- If an argument is a constant, the callee's logic can fold into a constant.
- It can reveal that an object never leaves the method, which feeds escape analysis (section 4).

That is why inlining is called the enabling optimization.

Limits: the callee must be small enough (HotSpot has bytecode-size thresholds, roughly 35 bytes for ordinary callees and larger for very hot ones), the call target must be known (see the next point), and recursion depth is capped. `-XX:+PrintInlining` shows each decision.

---

## 3. Virtual call vs direct call

**Direct call:** the target method is known when the call is linked, so the instruction jumps to a fixed address. `static` methods, constructors, `super.x()` and `private` methods fit here (`invokestatic`, `invokespecial`). `final` methods and methods in `final` classes can also be resolved to a single target.

**Virtual call:** the target depends on the **runtime class** of the object, which the compiler can't know.

```java
Account a = getAccount();     // declared type Account, but might be MT4Account at runtime
a.describe();                 // which describe() runs: Account's or MT4Account's?
```

The JVM decides at execution time. Each class has a **vtable**, an array of method pointers, with one slot per overridable method. The mechanism is:

```
1. Read the object's header → find its class.
2. Look up that class's vtable.
3. Take the pointer in the slot for describe().
4. Jump to it.
```

So a virtual call costs a couple of extra memory loads and an **indirect jump** (the address isn't known until the loads finish), and, more importantly, the CPU and the JIT can't see through it, so it **blocks inlining**. `invokeinterface` is the same idea through an interface, with a slightly more involved lookup.

| | Direct | Virtual |
|---|---|---|
| Target known | Link or compile time | Only at runtime, from the receiver's class |
| Cost | Plain jump | Loads + indirect jump |
| Inlinable | Yes | Not unless the JIT can narrow it to one target |

**Devirtualization** is the JIT's answer. If it can prove, or at least observe, that only one class ever arrives at a call site (class hierarchy analysis: "only one loaded class overrides this method", or profile data), it replaces the virtual call with a direct call guarded by a cheap check, and then inlines it. The deoptimization example in the module is the failure case: a second implementation is loaded and the guard's assumption breaks.

---

## 4. "Escaping" and "non-escaping" objects

An object **escapes** a method if a reference to it can become reachable from **outside that method's own execution**: it is stored in a field or static, returned, passed to a method the compiler can't see into, or handed to another thread. **Non-escaping** means the reference is created, used and dropped entirely inside the method, so nothing else could ever observe the object.

```java
// Does NOT escape
BigDecimal fee(int qty) {
    Point p = new Point(qty, 2);          // only used locally
    return BigDecimal.valueOf(p.x * p.y); // p itself never leaves
}

// ESCAPES
Point current;                            // field
void remember(int qty) {
    Point p = new Point(qty, 2);
    this.current = p;                     // stored in a field: others can reach it
}

Point make(int qty) { return new Point(qty, 2); }   // returned: escapes
```

**Why the compiler cares:** if no one else can ever see the object, it doesn't have to exist as an object at all. The JIT can apply **scalar replacement**: drop the allocation and keep the object's fields as plain local values (registers or stack slots).

```
before:  p = new Point(qty, 2);  result = p.x * p.y;
after:   x = qty;  y = 2;        result = x * y;       // no heap allocation, no header, no GC work
```

It can also remove locks on such an object (nobody else can contend for them).

Why inlining and escape analysis go together: the analysis is **per compiled method**. If `p` were passed to a method that wasn't inlined, the compiler wouldn't know what that method does with it, so it must assume `p` escapes. Once the callee is inlined, the compiler can see every use and prove it doesn't.

Caveats:

- It is an optimization, not a guarantee. Large objects, objects in loops with complex control flow, and objects involved in non-inlined calls often don't qualify.
- It is invisible semantically. The program behaves identically either way.
- It is why short-lived small objects (iterators, small value holders, builders used within one method) can cost nothing after JIT, and why hand-pooling them is counterproductive, as noted in the performance module.

---

## Tying the four together

Interpreter counts executions → hot method gets compiled → the compiler **inlines** small callees (turning virtual calls into direct ones where it can **devirtualize** them) → with the bodies visible, **escape analysis** finds objects that never leave and removes their allocation. Each step enables the next, which is why the optimized tier can run dramatically faster than the same bytecode interpreted.

I'll also suggest correcting the section 6 table row on inlining to read: "Replaces a call with a copy of the callee's body in the compiled code. The callee's bytecode and the original method are unchanged." If you'd like, I can send that as a single corrected row.

When you're ready, G11 (Class Loading & Initialization Internals) is next.



