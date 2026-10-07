# OOP — Four Pillars
- The four pillars of object-oriented programming (OOP) are the fundamental concepts that define the structure and behavior of object-oriented systems. These pillars are:
  1. Encapsulation
  2. Abstraction
  3. Inheritance
  4. Polymorphism

## 1. Encapsulation
> Definition: Control what can change your object's state, and how.
- Encapsulation is the practice of bundling data (attributes) and methods (functions) that operate on that data into a single unit, typically a class. 
- It restricts direct access to some of an object's components, which can prevent the accidental modification of data.
- This is usually achieved through access modifiers (private, protected, public) and providing public getter and setter methods to access and modify the private data.

Without encapsulation a field can be modified by anyone directly:
```java
Account acc = new Account();
acc.status = "ACTIVE";   // no checks done. Directly activating the account
```

With encapsulation:
```java
public class Account {
    private String status;   // private — no direct outside access
    private String type;

    public String activateAccount() {
        validateActivationRules();  // some internal logic to check if activation is allowed
        this.status = "ACTIVE";
        return status;
    }
}
```
- Account can only be activated using the `activateAccount()` method. This way any other developer will not be able to randomly set an account status to `active`.

### Access modifiers
| Modifier                 | Same class | Same package | Subclass (different package) | Everywhere |
|--------------------------|:----------:|:------------:|:----------------------------:|:----------:|
| `private`                |     ✅     |      ❌      |              ❌              |     ❌     |
| *(default, no modifier)* |     ✅     |      ✅      |              ❌              |     ❌     |
| `protected`              |     ✅     |      ✅      |              ✅              |     ❌     |
| `public`                 |     ✅     |      ✅      |              ✅              |     ✅     |


### Why encapsulation matters practically
- **Validation** — activateAccount() can refuse invalid state transitions; a public field cannot.
- **Flexibility** — you can later change status from a String to an enum internally, and callers using getStatus() are unaffected. If status were a public field, every caller touching it directly would break.
- **Read-only exposure** — provide only a getter (no setter) for fields that shouldn't change after construction. (Unlike final, these are not completely constant variables. They can be set via constructor or some specialized method can change it, like date of birth)

> Practical test for encapsulation: "If I hand this object to code I don't trust, can they corrupt its internal state?"
## 2. Inheritance
> Definition: use it when a subclass is a genuine specialization of the parent — same core identity, extra or adjusted behavior — not merely to "reuse some code."

- Inheritance is a mechanism where one class (child/subclass) can inherit the properties and methods of another class (parent/superclass). This promotes code reusability and establishes a hierarchical relationship between classes.
```java
public abstract class Account {
    public void acceptsTransferTo(Account toAccount, BigDecimal amount) throws FundsTransferException {
        validateAccountsAreLinked(toAccount);
        validateHasBaseCurrency(toAccount.getBaseIsoCurrency());
        validateAccountHasSufficientFunds(amount);
        validateAccountTransfersAreEnabledForTheSite();
        validateIfBaseCurrencyIsJPYThenNoDecimalPlaces(amount);
    }
}

public class MT4Account extends Account {
    @Override
    public void acceptsTransferTo(Account toAccount, BigDecimal amount) throws FundsTransferException {
        validateMT4IsAvailable();                           // MT4-specific, before
        super.acceptsTransferTo(toAccount, amount);         // run the shared logic unchanged
        validateHasEnoughFundsToWithdrawInMetatrader(amount); // MT4-specific, after
    }
}
```

> Practical test for "should this be inheritance?": "Is MT4Account fundamentally an Account, used everywhere an Account is expected, just with extra rules?" Yes — so extends is correct here. Contrast with SurchargeRule, which does not inherit from any matcher — it's not "a kind of" CountryMatcher, it just uses one. That's a composition relationship, not an inheritance one, and the code reflects that correctly.

## 3. Polymorphism
> Definition: the caller holds a reference to the contract type, and the correct concrete behavior runs automatically — without the explicitly knowing which subclass is actually in play.
  
- Polymorphism means "many forms" — the same method call can behave differently depending on context.
  - The most common use of polymorphism in OOP is when a parent class reference is used to refer to a child class object.
    - There are two distinct kinds of polymorphism in Java - 
      1. Compile-time polymorphism (method overloading): Same method name, different parameter list, in the same class. The compiler decides which version to call based on the arguments you pass — before the program even runs.
      ```java
      public class Client {
        public void addAccount(String type) {
            System.out.println("Opening default account of type: " + type);
        }
  
        public void addAccount(String type, double initialDeposit) {
            System.out.println("Opening account of type: " + type + " with deposit: " + initialDeposit);
        }
      }
      ```
     2. Runtime polymorphism (method overriding): Same method name and signature, redefined in a subclass. Which version actually executes is decided at runtime, based on the object's actual type — not the reference type.
    ```java
      public class Account {
          public void printSummary() {
              System.out.println("Generic account, status: " + status);
          }
      }
      
      public class MarginAccount extends Account {
          @Override
          public void printSummary() {
              System.out.println("Margin account, limit: " + marginLimit);
          }
      }
      ```
      
      ### Rules for overriding 
      - Method name + parameter types must match exactly.
      - Return type must be the same, or a subtype (e.g. overriding a method returning Account with one returning MarginAccount is legal).
      - Cannot throw broader checked exceptions than the overridden method.
      - Cannot reduce visibility (can't override a public method with a protected one — but widening, protected → public, is fine).
      - Cannot override static, final, or private methods — these aren't part of the dynamic dispatch mechanism at all.

### Fields are NOT polymorphic (an important trap)
```java
public class Account {
    protected String category = "GENERIC";
}

public class MarginAccount extends Account {
    protected String category = "MARGIN";   // this HIDES the parent field, doesn't override it
}

Account acc = new MarginAccount("IGCFD", 50000.0);
System.out.println(acc.category);   // prints "GENERIC" — field access uses the REFERENCE type, not actual type!
```

> **Why**: unlike methods, field access is resolved statically, at compile time, based on the declared type of the reference variable. This is the opposite of method overriding. Two separate category fields actually exist in memory — one from Account, one from MarginAccount — and which one you see depends purely on how the variable is declared, not what the object actually is.

### Static methods are also NOT polymorphic — they're hidden, not overridden
```java
public class Account {
    public static void printInfo() {
        System.out.println("Account class");
    }
}

public class MarginAccount extends Account {
    public static void printInfo() {   // this HIDES the parent's static method
        System.out.println("MarginAccount class");
    }
}

Account acc = new MarginAccount("IGCFD", 50000.0);
acc.printInfo();   // prints "Account class" — resolved by REFERENCE type, not actual object!
```

> Practical test for polymorphism: "Can I add a new implementation of this contract without touching any code that already uses the contract?" If yes, you have real polymorphism at work, not just method overriding for its own sake.

## 4. Abstraction
> Definition: Depend on a contract, not a concrete implementation — so the implementation can be swapped or multiplied freely.
- Abstraction is the concept of hiding the complex implementation details from the caller and showing only the essential features of an object. It allows focusing on what an object does instead of how it does it.
- Hence we can swap any implementation at any given point in time as the implementation is abstracted away. The caller doesn't depend on the logic of the implementer, hence swapping it won't do any harm.
- In Java, abstraction can be achieved using abstract classes and interfaces.

> Practical test for abstraction: "If I introduce a brand-new way of fulfilling this behavior, how many existing files do I have to edit?" For SurchargeRule + CountryMatcher: zero files touched, one new file added. That's abstraction doing real work.


## Miscellaneous 
### Do we encourage inheritance
- Generally, no — not as a first instinct. Modern Java practice leans toward "favor composition over inheritance." 
- Inheritance is powerful but creates tight coupling: a subclass depends on the exact internal behavior of its superclass, and changes to the superclass can silently break subclasses in ways that are hard to trace.
- Use inheritance primarily for "is-a" relationships where the subclass truly represents a specialized version of the superclass.
- Example given below is a strict "is-a" relationship, so inheritance is appropriate:
```java
public abstract class Account {
    public void acceptsTransferTo(Account toAccount, BigDecimal amount) throws FundsTransferException {
        validateAccountsAreLinked(toAccount);
        validateHasBaseCurrency(toAccount.getBaseIsoCurrency());
        validateAccountHasSufficientFunds(amount);
        validateAccountTransfersAreEnabledForTheSite();
        validateIfBaseCurrencyIsJPYThenNoDecimalPlaces(amount);
    }
}

public class MT4Account extends Account {
    @Override
    public void acceptsTransferTo(Account toAccount, BigDecimal amount) throws FundsTransferException {
        validateMT4IsAvailable();                           // MT4-specific, before
        super.acceptsTransferTo(toAccount, amount);         // run the shared logic unchanged
        validateHasEnoughFundsToWithdrawInMetatrader(amount); // MT4-specific, after
    }
}
```

### Why are fields resolved by reference type, but methods by actual object type? (The internals)
### Fields
- When `new MarginAccount()` runs, the JVM must allocate enough space for every field in the entire inheritance chain — the subclass's own fields, plus every field declared anywhere in its superclasses. 
- It does not collapse same-named fields into one slot. 
- Field identity in the JVM isn't just "the name" — it's (declaring class + field name + type) as a combined identity.
- So `Account.category` and `MarginAccount.category` are two entirely distinct fields that happen to share a name.

```
MarginAccount object (one object, one memory block):
┌─────────────────────────────┐
│  object header (mark word,  │
│  class pointer, etc.)       │
├─────────────────────────────┤
│  Account's slot:            │
│    category → "GENERIC"     │   ← inherited storage, from Account's layout
├─────────────────────────────┤
│  MarginAccount's slot:      │
│    category → "MARGIN"      │   ← MarginAccount's own declared field
└─────────────────────────────┘
```

- **One object. Two separate storage locations** for "category." Both values genuinely exist simultaneously, in the same object instance, taking up real memory twice.
- **How access decides which slot you hit**: This is resolved at compile time, exactly like we discussed for field access generally — but now we can be specific about why two different reads can both compile and both succeed, hitting different slots:
```java
Account accRef = ma;
MarginAccount marginRef = ma;    // same object, two different reference types

System.out.println(accRef.category);     // reads Account's slot    → "GENERIC"
System.out.println(marginRef.category);  // reads MarginAccount's slot → "MARGIN"
```

**Accessing hidden field from inside the subclass**:
```java
public class MarginAccount extends Account {
    protected String category = "MARGIN";

    public void printBoth() {
        System.out.println(this.category);     // "MARGIN" — closest declaration wins by default
        System.out.println(super.category);    // "GENERIC" — explicitly reach the hidden parent slot
    }
}
```

### Methods
- Unlike fields, method bytecode is never duplicated per object. There's exactly one copy of each method's compiled instructions, stored once in Metaspace, as part of the class's metadata — regardless of how many objects of that class you create

```
┌──────────────── METASPACE ────────────────────────┐
│  Account.class metadata:                          │
│    printSummary() bytecode   ← ONE copy           │
│    vtable: [printSummary → Account.printSummary]  │
└───────────────────────────────────────────────────┘

┌──────────────────── HEAP ──────────────────────────────────────┐
│  a1 { category: "GENERIC" }   ← own field, NO method code copy │
│  a2 { category: "GENERIC" }   ← own field, NO method code copy │
│  a3 { category: "GENERIC" }   ← own field, NO method code copy │
└────────────────────────────────────────────────────────────────┘

┌──────────── METASPACE ──────────────────────────┐
│ Account's vtable:                               │
│   [0] printSummary → Account.printSummary       │
│                                                 │
│ MarginAccount's vtable:                         │
│   [0] printSummary → MarginAccount.printSummary │  ← overwritten, not duplicated
└─────────────────────────────────────────────────┘
```

When we invoke the below code:
```java
Account acc = new MarginAccount();
acc.printSummary();
```

At the call site, the JVM executes following:
- Look at the actual object acc points to on the heap → find its class pointer in the object header → it's MarginAccount.
- Go to MarginAccount's vtable (not Account's) in Metaspace.
- Jump to whatever that slot points to → MarginAccount.printSummary's bytecode.
- Execute that code.

Note:
- Every object carries a pointer to its class (inside the object header) — this is precisely how, given any object, the JVM can always find the correct vtable to consult, regardless of what reference type is being used to access it.
- During compile time we only have metadata hence even the static methods behave in a similar manner they pick from the class within which it is defined.
- During runtime we go to the heap and get the object an object always belongs to a given class as mentioned above, when we access something from an object, it goes to the given class's method definition.

### Can an abstract class have zero abstract methods?
Yes — completely legal, and a reasonably common pattern.
```java
public abstract class Account {
    protected String status;

    public Account(String type) {
        this.status = "ACTIVE";
    }

    public void close() {
        this.status = "CLOSED";
    }

    public boolean isFundsTransferEnabledForAccount() {
        return true;   // fully implemented default
    }
    // no abstract methods at all
}
```

**Why you'd do this**: 
- sometimes the only thing you need abstract for — preventing instantiation of an incomplete concept — even when there's no specific method that must vary per subclass.
- If `Account` had no abstract methods at all, it would still make sense to keep it abstract, purely to express: "this class represents a concept too general to exist on its own — you must always work through a specific subtype," even if, technically, every method already has usable default behavior.

