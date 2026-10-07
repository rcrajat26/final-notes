# Reflection Basics
## What it is
- Reflection is a feature in Java that allows a program to inspect and manipulate its own structure and behavior at runtime. 
- It provides the ability to examine classes, interfaces, fields, methods, and constructors, even if they are private or protected. This can be useful for various purposes, such as debugging, testing, and building frameworks.
- Spring injects into your @Autowired fields, Jackson fills your Client, Hibernate populates your Account entities, and JUnit finds your @Test methods. All of these discover your classes at runtime.

## The Class object: the entry point
- For every class the JVM loads, there is a `java.lang.Class` object holding that class's metadata: name, superclass, interfaces, fields, methods, constructors, annotations and modifiers. Every reflective operation starts from one.
- There are three ways to get it:
```java
Class<?> c1 = Account.class;                       // 1. class literal: compile-time known
Class<?> c2 = account.getClass();                  // 2. from an instance: the RUNTIME type
Class<?> c3 = Class.forName("com.ig.model.Account"); // 3. from a String name: fully qualified
```

- `getClass()` returns the runtime type, not the declared one. With `Account a = new MT4Account();`, `a.getClass()` is `MT4Account.class`. 
- Primitives and arrays have class literals too: `int.class`, `String[].class`. Note that `int.class` != `Integer.class`. 
- One `Class` object exists per class per class-loader. The same class name loaded by two different loaders gives two distinct `Class` objects that are not interchangeable.


## Inspecting a class
- Once you have a `Class` object, you can inspect its structure:
```java
Class<?> c = MT4Account.class;
c.getName();            // "com.ig.model.MT4Account"   (binary name; nested: Outer$Inner)
c.getSimpleName();      // "MT4Account"
c.getPackageName();     // "com.ig.model"               (Java 9+)
c.getSuperclass();      // Account.class  (null for Object, interfaces, primitives)
c.getInterfaces();      // directly implemented interfaces only
c.getModifiers();       // int bit flags; decode with Modifier.toString(...)
c.isInterface(); c.isEnum(); c.isArray(); c.isAnnotation();
c.isRecord();           // Java 16+
c.isSealed();           // Java 17+
```

`getModifiers()` returns packed bits, so use `java.lang.reflect.Modifier` (Modifier.isStatic(m), Modifier.isFinal(m), Modifier.toString(m)) to read them.


## Getting members: `getX` vs `getDeclaredX`
- The same split applies to fields, methods and constructors. It is the most commonly confused part of the API.

### Reflection: `get*` vs `getDeclared*`

| Call | Returns | Inherited members? | Non-public members? |
|---|---|---|---|
| `getFields()` / `getField(name)` | Public fields | Yes (public ones from superclasses and interfaces) | No |
| `getDeclaredFields()` / `getDeclaredField(name)` | All fields declared in this class | No | Yes (private, protected, package) |
| `getMethods()` / `getMethod(name, types...)` | Public methods | Yes (including from `Object`) | No |
| `getDeclaredMethods()` / `getDeclaredMethod(...)` | All methods declared in this class | No | Yes |
| `getConstructors()` / `getConstructor(types...)` | Public constructors | n/a | No |
| `getDeclaredConstructors()` / `getDeclaredConstructor(types...)` | All constructors of this class | n/a | Yes |

Consequences: 
- To reach a private field you almost always need `getDeclaredField`.
- To find a private field defined on a superclass, `getDeclaredField` on the subclass fails with `NoSuchFieldException`.
- You must walk `getSuperclass()` yourself. Frameworks such as Jackson and Hibernate do this loop internally.
- Constructors are never inherited, so there is no "inherited constructors" case.
- The order of returned arrays is not guaranteed. Never depend on it.
- Method lookup is by exact parameter types: `getMethod("setLimit", int.class)` finds `setLimit(int)`. `getMethod("setLimit", Integer.class)` throws `NoSuchMethodException` because the primitive and its wrapper are different types.

The member objects are `java.lang.reflect.Field`, `Method` and `Constructor`, and each has `getName()`, `getModifiers()`, `getType()`/`getReturnType()`/`getParameterTypes()` and `getAnnotation(...)`.


## Using members
```java
public class Account {
    private String status = "ACTIVE";
    private int limit;
    public Account() {}
    private Account(String status) { this.status = status; }
    public void setLimit(int limit) { this.limit = limit; }
    public static Account of(String s) { return new Account(s); }
}
```

### Creating objects 
```java
Constructor<Account> ctor = Account.class.getDeclaredConstructor(String.class); // private one
ctor.setAccessible(true);
Account a = ctor.newInstance("SUSPENDED");
}
```

`Class.newInstance()` is deprecated since Java 9. It propagated checked exceptions from the constructor without declaring them. Use `getDeclaredConstructor().newInstance()` instead.

### Reading and writing fields
```java
Field f = Account.class.getDeclaredField("status");
f.setAccessible(true);                 // needed because the field is private
String s = (String) f.get(a);          // read
f.set(a, "INACTIVE");                  // write
// primitive variants: f.getInt(a), f.setInt(a, 5) avoid boxing
```

`a` here is the instance of `Account` created using the private constructor via reflection. For a static field, pass null as the target: `f.get(null);` As there is no instance needed to fetch a static field.

### Invoking methods
```java
Method m = Account.class.getMethod("setLimit", int.class);
m.invoke(a, 500);                      // instance method: target, then args
Method sm = Account.class.getMethod("of", String.class);
Account b = (Account) sm.invoke(null, "ACTIVE");   // static method: target is null
```

- `invoke` takes `Object...` and returns `Object`, so primitives are auto-boxed on the way in and out, and you cast the result. 
- `InvocationTargetException`: if the invoked method itself throws, `invoke` wraps it in this checked exception. The real error is in `e.getCause()`. Forgetting to unwrap it is a very common debugging mistake. `Constructor.newInstance` does the same. 
- The checked exceptions you must handle: `NoSuchMethodException`/`NoSuchFieldException` (lookup failed), `IllegalAccessException` (not allowed to touch it), `InvocationTargetException` (the target threw), and `InstantiationException` (abstract class or interface).

## Breaking encapsulation: setAccessible, and what stops it
- `setAccessible(true)` turns off the Java access check for that one `Field`/`Method`/`Constructor` object, so private members become usable. Without it, touching a private member throws `IllegalAccessException`.
- This is how a framework fills your private String name without a setter. It is also exactly why reflection is a security and design concern.
- Since Java 9, there are modules and the `--illegal-access` flag that can prevent `setAccessible(true)` from working, adding another layer of encapsulation protection.

Whether `setAccessible(true)` succeeds is governed by the module system:
- Code in the same module, or in the unnamed module (the plain classpath), can generally reflect on anything. 
- Across named modules, deep reflective access to non-public members works only if the target package is `opens`-ed to the caller. A plain `exports` is not enough for private members. 
- When access is denied, `setAccessible` throws `InaccessibleObjectException` (unchecked, Java 9+). `trySetAccessible()` returns false instead of throwing, and `canAccess(obj)` checks without changing anything. 
- The JDK's own internal packages (`java.base`'s `sun.*`, `jdk.internal.*`, and non-public members of `java.*`) are strongly encapsulated.


## A small realistic example
```java
@Retention(RetentionPolicy.RUNTIME) @Target(ElementType.FIELD)
@interface NotBlank { String message() default "must not be blank"; }

class Client {
    @NotBlank private String name;
    @NotBlank private String email;
    private String phone;                 // optional
    Client(String name, String email, String phone) { ... }
}

static List<String> validate(Object target) throws IllegalAccessException {
    List<String> errors = new ArrayList<>();
    for (Field f : target.getClass().getDeclaredFields()) {
        if (f.isAnnotationPresent(NotBlank.class)) {
            f.setAccessible(true);
            Object v = f.get(target);
            if (v == null || v.toString().isBlank()) {
                errors.add(f.getName() + " " + f.getAnnotation(NotBlank.class).message());
            }
        }
    }
    return errors;
}
```


## Dynamic proxies
`java.lang.reflect.Proxy` creates, at runtime, an object that implements one or more interfaces and routes every method call to a single `InvocationHandler`.

```java
interface AccountService { void suspend(String accountId); }

AccountService real = new RealAccountService();

AccountService proxy = (AccountService) Proxy.newProxyInstance(
    AccountService.class.getClassLoader(),
    new Class<?>[] { AccountService.class },
    new InvocationHandler() {
        @Override
        public Object invoke(Object p, Method method, Object[] args) throws Throwable {
            System.out.println("before " + method.getName());
            try {
                return method.invoke(real, args);            // delegate to the real object
            } catch (InvocationTargetException e) {
                throw e.getCause();                          // unwrap
            } finally {
                System.out.println("after " + method.getName());
            }
        }
    });

proxy.suspend("A-1");   // prints before / (real work) / after
```

- A JDK proxy can proxy interfaces only. It cannot proxy a concrete class. 
- Proxying a concrete class needs bytecode generation (CGLIB, ByteBuddy). Spring uses a JDK proxy when a bean implements an interface and a generated subclass otherwise. 
- This is how @Transactional, @Cacheable and @Async work. A proxy wraps your bean, intercepts the call, runs the cross-cutting logic and then delegates. It is also why self-invocation (this.otherMethod() inside the bean) skips the annotation: the call goes straight to this, not through the proxy.

