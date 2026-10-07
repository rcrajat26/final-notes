# JDK, JRE, JVM, and Program Structure
## Why is Java this way? 
Core goals of Java:
- *Portability:* Write once, run anywhere (WORA) — the same Java program should run on any platform without modification.
- *Memory safety:* no manual pointer arithmetic, GC handles memory
- **Single inheritance of implementation** — a class extends only one class (avoids C++'s diamond problem)
- **Multithreading support** — Java should provide built-in support for concurrent programming and synchronization.
- **Dynamic linking** — classes are loaded and linked at runtime, not all bundled into one static binary

## JVM (Java Virtual Machine) 
- It generates bytecode and converts it to native code.
- Even though Java is platform-independent, JVM is actually platform-dependent. For each OS type there is one JVM. Example: Linux, Windows. 
- JVM is a specification, and each vendor implements it differently. For example, Oracle's HotSpot JVM, OpenJDK's JVM, IBM's J9 JVM, etc.

## JRE (Java Runtime Environment)
- It is a package that provides the libraries, Java Virtual Machine (JVM), and other components to run applications written in Java. 
- It does not contain development tools such as compilers
- JRE is platform-dependent. For example, Windows JRE, Linux JRE, etc.

## JDK (Java Development Kit)
- It is a software development kit used to develop Java applications.
- It contains JRE, an interpreter/loader (Java), a compiler (javac), an archiver (jar), a documentation generator (Javadoc), and other tools needed for Java development
- JDK is platform-dependent. For example, Windows JDK, Linux JDK, etc.

```
JDK = JRE + development tools (javac, etc.)
JRE = JVM + standard libraries
JVM = the thing that actually executes bytecode
```

### Compile → Run flow
```
HelloWorld.java  --(javac)-->  HelloWorld.class  --(java)-->  Output
   (source code)                  (bytecode)                (JVM executes it)
```

- You write `HelloWorld.java`
- `javac HelloWorld.java` → compiles to `HelloWorld.class` (bytecode — platform-independent, not machine code)
- `java HelloWorld` → JVM loads the .class file and executes it

> While running simply use: `java HelloWorld` — no `.java` extension, no `.class` extension. Just the class name.
> 
> The java command doesn't take a file path — it takes a fully qualified class name. Internally, the JVM's classloader searches the classpath for a file named HelloWorld.class and loads it.

### Program Structure
```java
public class HelloWorld {
    public static void main(String[] args) {
        System.out.println("Hello, World!");
    }
}
```

- `public` — JVM must access it from outside the class
- `static` — JVM calls it without creating an object first
- `void` — returns nothing
- `String[] args` — command-line arguments get passed here


## Miscellaneous 
### How do servlets work without main?
- They don't need their own main — because the servlet container (Tomcat, Jetty, etc.) already has one, and it's already running as a full JVM process before your servlet code ever executes.
- The container:
  - Starts up (it has its own main)
  - Loads your servlet class
- Manages its lifecycle by calling specific methods it expects your class to implement — init(), service()/doGet()/doPost(), destroy()
- This is a common pattern: you write code conforming to an interface/contract, and some other already-running process calls your methods at the right time. 
- Frameworks like Spring Boot, Android apps, and GUI apps (JavaFX) all work this way — only one main exists at the bottom of the stack; everything else is lifecycle callbacks.


