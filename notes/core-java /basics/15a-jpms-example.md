# JPMS example 
## Directory layout
```
src/
├── com.brokerage.api/
│   ├── module-info.java
│   └── com/brokerage/api/AccountValidator.java
├── com.brokerage.accounts/
│   ├── module-info.java
│   └── com/brokerage/accounts/
│       ├── internal/DefaultAccountValidator.java
│       └── model/Account.java
└── com.brokerage.app/
    ├── module-info.java
    └── com/brokerage/app/Main.java
```

Here's a minimal three-module application that exercises all six keywords. Three modules: 
- `com.brokerage.api` (the contract)
- `com.brokerage.accounts` (the provider)
- `com.brokerage.app` (the consumer)

### Module 1 — `com.brokerage.api` (the contract, nothing else)

```java
// src/com.brokerage.api/module-info.java
module com.brokerage.api {
    exports com.brokerage.api;
}
```
```java
// src/com.brokerage.api/com/brokerage/api/AccountValidator.java
package com.brokerage.api;

public interface AccountValidator {
    boolean isValid(String accountNumber);
}
```

### Module 2 — `com.brokerage.accounts` (the provider)

```java
// src/com.brokerage.accounts/module-info.java
module com.brokerage.accounts {
    requires com.brokerage.api;

    opens com.brokerage.accounts.model to com.brokerage.app;

    provides com.brokerage.api.AccountValidator
        with com.brokerage.accounts.internal.DefaultAccountValidator;
}
```
```java
// src/com.brokerage.accounts/com/brokerage/accounts/internal/DefaultAccountValidator.java
package com.brokerage.accounts.internal;

import com.brokerage.api.AccountValidator;

public class DefaultAccountValidator implements AccountValidator {
    public boolean isValid(String accountNumber) {
        return accountNumber != null && accountNumber.startsWith("ACC");
    }
}
```
```java
// src/com.brokerage.accounts/com/brokerage/accounts/model/Account.java
package com.brokerage.accounts.model;

public class Account {
    private String accountNumber = "ACC123";   // never exported — only reachable via reflection, and only because opened
}
```

Notice `accounts.model` is **never `exports`ed** — only `opens`ed. So `com.brokerage.app` can never `import com.brokerage.accounts.model.Account;` directly (compile error if it tried), but it *can* reach the private field reflectively, purely because of the `opens ... to` grant.

### Module 3 — `com.brokerage.app` (the consumer)

```java
// src/com.brokerage.app/module-info.java
module com.brokerage.app {
    requires com.brokerage.api;
    uses com.brokerage.api.AccountValidator;
}
```
```java
// src/com.brokerage.app/com/brokerage/app/Main.java
package com.brokerage.app;

import com.brokerage.api.AccountValidator;
import java.lang.reflect.Field;
import java.util.ServiceLoader;

public class Main {
    public static void main(String[] args) throws Exception {
        // uses + provides + with: ServiceLoader finds accounts' registered implementation,
        // even though this module never 'requires com.brokerage.accounts' at all
        ServiceLoader<AccountValidator> loader = ServiceLoader.load(AccountValidator.class);
        for (AccountValidator validator : loader) {
            System.out.println("Found provider: " + validator.getClass().getName());
            System.out.println("Is ACC999 valid? " + validator.isValid("ACC999"));
        }

        // opens: reach into a package that was never exported, only opened, via reflection
        Class<?> accountClass = Class.forName("com.brokerage.accounts.model.Account");
        Object account = accountClass.getDeclaredConstructor().newInstance();
        Field field = accountClass.getDeclaredField("accountNumber");
        field.setAccessible(true);                       // fails with InaccessibleObjectException without 'opens'
        System.out.println("Reflected private field: " + field.get(account));
    }
}
```

### Compiling and running

```bash
javac -d out --module-source-path src $(find src -name "*.java")

java --module-path out -m com.brokerage.app/com.brokerage.app.Main
```

**Expected output:**
```
Found provider: com.brokerage.accounts.internal.DefaultAccountValidator
Is ACC999 valid? true
Reflected private field: ACC123
```

### What each keyword is doing here

| Keyword             | Where                                                                 | What it demonstrates                                                                                                                   |
|---------------------|-----------------------------------------------------------------------|----------------------------------------------------------------------------------------------------------------------------------------|
| `exports`           | `api` module exports `com.brokerage.api`                              | The interface is the only thing genuinely public across module boundaries                                                              |
| `requires`          | `accounts` and `app` both `requires com.brokerage.api`                | Both need the interface to compile against                                                                                             |
| `provides ... with` | `accounts` provides `AccountValidator` with `DefaultAccountValidator` | Registers a concrete implementation without the consumer ever naming that class                                                        |
| `uses`              | `app` declares `uses AccountValidator`                                | Asks `ServiceLoader` for *some* implementation, never hard-codes which module supplies it                                              |
| `opens ... to`      | `accounts` opens `accounts.model` to `com.brokerage.app`              | Grants reflective access to a package that is deliberately **not** exported — ordinary `import` still fails, reflection still succeeds |

Try removing the `opens` line and rerunning — `field.setAccessible(true)` throws `InaccessibleObjectException`, which is the cleanest way to see the `exports`-vs-`opens` distinction fail loudly rather than silently.
