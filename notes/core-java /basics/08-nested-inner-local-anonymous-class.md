# Nested, Inner, Local, and Anonymous Classes
## Static nested classes 
```java
class Client {
    static class Builder { ... }
}
```

- Static nested classes are declared static and do not have access to instance variables or methods of the outer class. They can be instantiated without an instance of the outer class.
- Declared static inside another class — no implicit reference to any enclosing Client instance. 
- Can be instantiated without an enclosing instance: new Client.Builder(). 
- Behaves like a regular top-level class that just happens to be namespaced inside Client — purely an organizational/naming convenience. Commonly used for builders, or small helper types tightly coupled to the outer class conceptually but not needing any of its instance state.

ex:
```java
class Client {
    private String name;          // instance field of Client
    private String email;

    static class Builder {
        private String name;
        private String email;

        Builder name(String n) { this.name = n; return this; }
        Builder email(String e) { this.email = e; return this; }

        Client build() {
            Client c = new Client();
            c.name = name;          // Builder sets Client's fields directly (same outer class, private access is fine)
            c.email = email;
            return c;
        }
    }
}
```

## Inner class (non-static)
```java
class Client {
    private String name;
    class AccountView { // non-static — an inner class
        void show() { System.out.println(name); } // can access outer's instance fields directly
    }
}

// usage:
Client c = new Client();
Client.AccountView v = c.new AccountView();   // note: c.new, not new Client.AccountView()
```
- Inner classes are non-static and have an implicit reference to an instance of the outer class. They can access the outer class's instance variables and methods directly.
- Can only be instantiated in the context of an existing Client instance: new Client().new AccountView(). 
- Can access outer's instance fields directly, even private ones. Commonly used for event listeners, callbacks, or any situation where the inner class needs to interact closely with a specific instance of the outer class.

## Local class
```java
class Client {
    void process() {
        class LocalHelper { // local class
            void help() { System.out.println("Helping..."); }
        }
        LocalHelper h = new LocalHelper();
        h.help();
    }
}
```
- Local classes are defined within a method and can access final or effectively final local variables of that method. They are only visible within the method they are defined in.
- Can only be instantiated within the method where they are defined.
- Can access final or effectively final local variables of the enclosing method. Commonly used for encapsulating helper logic that is only relevant within a specific method.

## Anonymous class
```java
CountryMatcher matcher = new CountryMatcher() {
    @Override
    public boolean accept(String country) {
        return country.equals("IN");
    }
};
```

- Anonymous classes are a special case of local classes without a name. They are often used to provide a quick implementation of an interface or an abstract class.
- Can only be instantiated at the point of definition.
- Can access final or effectively final local variables of the enclosing method. Commonly used for event handlers, callbacks, or any situation where a one-off implementation is needed without the overhead of a named class.

