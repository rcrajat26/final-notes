# Pass-by-Value Semantics
## The single rule, stated precisely
Java is always pass-by-value — no exceptions, no special case for objects. What gets copied when you pass an argument depends on what kind of value it is:
- Primitives (int, double, boolean, etc.) — the actual value itself is copied.
- Object references — the reference (the memory address/pointer stored in the variable) is copied. The object it points to is never copied.

### Object references — where the confusion starts
- When you pass an object reference to a method, the reference itself is copied. 
- This means that both the original variable and the parameter in the method point to the same object in memory. 

```java
void freeze(Account account) {
    account.setStatus(AccountStatus.FROZEN);   // mutates the OBJECT the reference points to
}

Account myAccount = new Account("ACC123", AccountStatus.ACTIVE);
freeze(myAccount);
System.out.println(myAccount.getStatus());   // FROZEN — the caller sees the change
```

- This looks like `pass-by-reference` because the caller's object did change. 
- But look closer at what was actually copied: the reference value (the address pointing to the one `Account` object on the heap) was copied into the method's parameter. 
- Both `myAccount` (caller's variable) and `account` (parameter) now hold the same address — two separate variables pointing at one shared heap object. 
- Calling `.setStatus(...)` through either reference mutates that one shared object — which is why the caller observes the change. 
- Nothing about the reference itself was passed "by reference" — the reference's value was copied, same as a primitive would be; it just happens that the value being copied is an address, so both copies point at the same place.

### Reassigning a parameter does NOT affect the caller
```java
void replace(Account account) {
    account = new Account("ACC999", AccountStatus.CLOSED);   // reassigns the LOCAL parameter
}

Account myAccount = new Account("ACC123", AccountStatus.ACTIVE);
replace(myAccount);
System.out.println(myAccount.getAccountNumber());   // "ACC123" — unchanged!
```
- In this example, the `account` parameter is reassigned to point to a new `Account` object.
- This reassignment only affects the local copy of the reference inside the method.
- The caller's `myAccount` variable still points to the original `Account` object, so its state remains unchanged. 
- This demonstrates that Java does not allow you to change the caller's reference to point to a new object — the reference itself is passed by value, and reassigning it does not affect the caller's variable.
