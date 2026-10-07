# OOP — Classes & Objects
## Classes & Objects — the basic distinction
- A class is a blueprint. An object is a concrete instance created from that blueprint, living on the heap.
- The class itself lives on Metaspace (not the heap). It contains the class's bytecode, static fields, static methods and constant pool (string literals, constant values, references to other classes/methods).

```java
public class Client {
    String name;      // instance field — each Client object gets its own copy
    String email;
    String phone;
    List<Account> accounts;

    static int totalClients;  // static field — shared, ONE copy across all Client objects
}
```

```java
Client client1 = new Client("John","john@therateemail.com","123-456-7890");   // client1 is an object — an actual instance in memory
Client client2 = new Client("Jean","jean@therateemail.com","987-654-3210");   // a completely separate object
```

### Fields
- Fields are the data members of a class. They represent the state of an object.
- There are two kinds of fields: instance fields and static fields 
  - Instance fields belong to an object. Each object has its own copy of instance fields.
  - Static fields belong to the class itself. There is only one copy of a static field, shared by all instances of the class.
- Note: `Client.totalClients` is the correct way to reference it — accessing it via an instance (`client1.totalClients`) also works, but is misleading and discouraged, since it makes it look instance-specific when it isn't.

### Constructors
- A constructor is a special method that is called when an object is created. It initializes the object's state.
- A constructor has the same name as the class and does not have a return type.
- If you do not define any constructors, the compiler provides a default no-argument constructor.
- If you define any constructor, the default no-argument constructor is not provided. You must explicitly define it if you want it.
```java
public class Client {
    String name;
    String email;
    String phone;
    List<Account> accounts;

    // constructor
    public Client(String name, String email, String phone) {
        this.name = name;
        this.email = email;
        this.phone = phone;
        this.accounts = new ArrayList<>();
    }
}
```

**Trap (shadowing)**: if you forget this., the assignment just assigns the parameter to itself — the field stays null forever, silently:
```java
public Account(String status, String type) {
    status = status;   // does NOTHING useful — assigns parameter to itself
    type = type;       // field "type" is never actually set!
}
```

### Instance and static methods
- Instance methods operate on an object. They can access instance fields and static fields.
- Static methods belong to the class. They can only access static fields and other static methods.

> Trap: you can call a static method through an instance reference — ```c1.getTotalClients();``` -> compiles, works, but misleading

### Static Initialization Blocks & Instance Initialization Blocks
- Beyond constructors, Java allows initializer blocks — code that runs automatically, separate from any specific constructor.
- Static initialization blocks run once when the class is first loaded. They are used to initialize static fields.
- Instance initialization blocks run every time an object is created, before the constructor. They are used to initialize instance fields.

```java
public class Account {
    String status;
    String type;
    static int accountCounter;

    // static block — runs ONCE, when the class is first loaded, before any object is created
    static {
        accountCounter = 1000;
        System.out.println("Account class loaded.");
    }

    // instance initializer block — runs every time an object is created, BEFORE the constructor body
    {
        System.out.println("Preparing new account...");
    }

    public Account(String status, String type) {
        this.status = status;
        this.type = type;
        accountCounter++;
        System.out.println("Account created: " + type);
    }
}
```
```java
new Account("ACTIVE", "IGSTK");
new Account("ACTIVE", "IGCFD");
```

```
Account class loaded.          // static block — only once per class, ever
Preparing new account...       // instance block — runs every time
Account created: IGSTK         // constructor body
Preparing new account...       // instance block — runs every time
Account created: IGCFD         // constructor body
```

### Inheritance case 
```java
public class Client {
    String name;

    static {
        System.out.println("Client: static block");
    }

    {
        System.out.println("Client: instance block");
    }

    public Client(String name) {
        System.out.println("Client: constructor");
        this.name = name;
    }
}

public class PremiumClient extends Client {
    static {
        System.out.println("PremiumClient: static block");
    }

    {
        System.out.println("PremiumClient: instance block");
    }

    public PremiumClient(String name) {
        super(name);   // calls Client's constructor FIRST
        System.out.println("PremiumClient: constructor");
    }
}
```

```
Client: static block            // superclass static — loaded first, once
PremiumClient: static block     // subclass static — loaded once
Client: instance block          // superclass instance init
Client: constructor              // superclass constructor body
PremiumClient: instance block   // subclass instance init
PremiumClient: constructor      // subclass constructor body
```
