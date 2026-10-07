# OOP Concepts in the Payments Codebase

Real examples pulled from the payments monorepo, illustrating the four core OOP pillars: **Encapsulation**, **Abstraction**, **Inheritance**, and **Polymorphism**. Each section starts with a generic framework-level example, then three deeper examples pulled from actual business logic spread across the estate: the `bank-withdrawal` payment-run sign-off workflow and bank-file field generation, the `payments` funds-transfer account hierarchy, and the `card-payments` surcharge rule-matching engine.

---

## 1. Abstraction

### Example 1 — Framework-level (`wallet-payments`)

**File:** `wallet-payments/integration/src/main/java/com/iggroup/wt/wallet/payments/web/filter/AbstractFilter.java`

```java
@Slf4j
public abstract class AbstractFilter implements Filter {

   @Setter
   private RequestExclusionStrategy requestExclusionStrategy;

   @Override
   public void doFilter(ServletRequest request, ServletResponse response, FilterChain filterChain) throws IOException, ServletException {
      HttpServletRequest httpServletRequest = (HttpServletRequest) request;
      HttpServletResponse httpServletResponse = (HttpServletResponse) response;
      if (isExcluded(httpServletRequest)) {
         log.debug("{} not running for URL={}", getClass().getSimpleName(), httpServletRequest.getRequestURI());
         filterChain.doFilter(request, response);
      } else {
         doFilter(httpServletRequest, httpServletResponse, filterChain);
      }
   }

   public abstract void doFilter(HttpServletRequest httpServletRequest, HttpServletResponse httpServletResponse, FilterChain filterChain) throws IOException, ServletException;

   private boolean isExcluded(HttpServletRequest request) {
      return requestExclusionStrategy != null && requestExclusionStrategy.isExcluded(request);
   }

   @Override
   public void destroy() { }

   @Override
   public void init(FilterConfig filterConfig) { }
}
```

`AbstractFilter` implements the servlet `Filter` interface but leaves `doFilter(HttpServletRequest, ...)` abstract. This is a **Template Method pattern**: the skeleton (check exclusion → run filter logic) lives in the base class, and the variable part is abstracted away for subclasses to fill in. Callers never see the casting, exclusion-checking, or lifecycle boilerplate — just "a filter that runs unless excluded."

---

### Example 2 — Business logic: `AbstractPaymentRunStage` (`bank-withdrawal`)

**File:** `bank-withdrawal/impl/src/main/java/com/iggroup/wt/bankwithdrawal/domain/paymentrun/stages/AbstractPaymentRunStage.java`

```java
public abstract class AbstractPaymentRunStage implements PaymentRunStage {
   private final PaymentRunRepository paymentRunRepository;
   private final BankWithdrawalProcessRepository bankWithdrawalProcessRepository;
   private final WithdrawalStage nextStage;

   @Override
   public final void progressPaymentRunStage(Long regionId, Long bankId, User user)
      throws PaymentRunException, PaymentRunUnAuthorisedException {
      Audit createdAudit = Audit.auditEntry(user.getName(), new Date());
      PaymentRun currentPaymentRun = loadCurrentPaymentRunDetails(regionId, createdAudit);
      validateCurrentPaymentRun(currentPaymentRun, regionId, user);
      updatePaymentRunStage(currentPaymentRun, createdAudit);
      updateWithdrawalRequests(currentPaymentRun, createdAudit);
      performStageSpecificOperations(currentPaymentRun, bankId);
   }

   protected abstract void validateCurrentPaymentRun(PaymentRun currentPaymentRun, Long regionId, User user)
      throws PaymentRunException, PaymentRunUnAuthorisedException;

   protected abstract void updateWithdrawalRequests(PaymentRun currentPaymentRun, Audit updatedAudit) throws PaymentRunException;

   protected void performStageSpecificOperations(PaymentRun currentPaymentRun, Long bankId) throws PaymentRunException {
      // overridden in subclasses that have stage-specific operations
   }
   // ... rollback mirror methods omitted for brevity
}
```

### What's happening

This is a **business-rule template**, not a framework hook. A bank withdrawal payment run moves through fixed stages (first credit sign-off → second credit sign-off → bank file → completed). `progressPaymentRunStage(...)` is declared `final` — **no subclass can ever change the sequence**: load the run, validate it's in the right stage, advance the stage, update the underlying withdrawal requests, then do anything stage-specific. What *is* allowed to vary (how a stage validates itself, how it decides what to do with the in-flight withdrawal requests) is abstracted into `validateCurrentPaymentRun(...)` and `updateWithdrawalRequests(...)`.

### Why this is abstraction

A caller invoking `progressPaymentRunStage(regionId, bankId, user)` doesn't need to know anything about sign-off rules, ledger postings, or audit-trail construction — it just knows "advance this region's payment run." The entire multi-step financial workflow (loading, auditing, validating, persisting, cascading to withdrawal requests) is hidden behind one method call per stage. This also structurally **prevents a dangerous bug class**: no stage implementation can skip validation or mutate state out of order, because the orchestrating method isn't overridable.

---

### Example 3 — Business logic: `Account` (`payments` funds-transfer module)

**File:** `payments/service/src/main/java/uk/co/igindex/payments/service/newfundstransfer/internal/Account.java`

```java
/**
 * Base account used to represent the different types of accounts ... from the point of view of funds transfers.
 */
public abstract class Account {
   private Client account;
   private ClientDetails clientDetails;
   private Money money;
   private FundingRestrictionType fundingRestrictionType;

   public void acceptsTransferTo(Account toAccount, BigDecimal amount) throws FundsTransferException {
      validateAccountsAreLinked(toAccount);
      validateHasBaseCurrency(toAccount.getBaseIsoCurrency());
      validateAccountHasSufficientFunds(amount);
      validateAccountTransfersAreEnabledForTheSite();
      validateIfBaseCurrencyIsJPYThenNoDecimalPlaces(amount);
   }

   public void acceptsTransferFrom(Account fromAccount, BigDecimal amount) throws FundsTransferException {
      validateAccountTransfersAreEnabledForTheSite();
   }

   public void validateAccountHasSufficientFunds(BigDecimal amountToTransfer) throws FundsTransferException {
      BigDecimal availableToWithdraw = getAvailableToWithdraw();
      if (amountToTransfer.compareTo(availableToWithdraw) > 0) {
         throw new FundsTransferException("Insufficient funds. The account has " + availableToWithdraw + "  available to transfer",
                 FundsTransferResult.FUNDS_TRANSFER_FAILED_INSUFFICIENT_FUNDS);
      }
   }
   // ... validateAccountsAreLinked, validateHasBaseCurrency, validateIfBaseCurrencyIsJPYThenNoDecimalPlaces are all private
}
```

### What's happening

`acceptsTransferTo(...)` is the single gate every funds transfer between two accounts must pass through, regardless of what kind of account is involved (a standard IG trading account, an Invest-Your-Way account, an MT4 account — see the inheritance example below). It composes five private validation steps — linked-account check, currency match, sufficient-funds check, site-level transfer toggle, JPY decimal-place rule — into one readable public contract. None of those five private methods is reachable from outside the class.

### Why this is abstraction

Code in the funds-transfer service calls `fromAccount.acceptsTransferTo(toAccount, amount)` and either gets a clean return or a `FundsTransferException` with a precise `FundsTransferResult` reason code. It never needs to know (or be able to accidentally skip) any of the five underlying checks — the concept exposed is simply "is this transfer allowed," not "here are five things you must remember to check."

---

### Example 4 — Business logic: `SurchargeRule` and its matchers (`card-payments`)

**File:** `card-payments/impl/src/main/java/com/iggroup/wt/cardpayments/domain/deposits/surcharge/rules/SurchargeRule.java`

```java
public class SurchargeRule {
   private final WebsiteMatcher websiteMatcher;
   private final SchemeMatcher schemeMatcher;
   private final CountryMatcher countryMatcher;
   private final OfficeMatcher officeMatcher;
   private final SurchargePolicy surchargePolicy;

   public boolean matches(DepositSurchargeRequest depositSurchargeRequest) {
      return schemeMatcher.accept(convertToUpperIfNotNull(depositSurchargeRequest.getCardScheme())) &&
             countryMatcher.accept(convertToUpperIfNotNull(depositSurchargeRequest.getCardIssuerCountry())) &&
             officeMatcher.accept(depositSurchargeRequest.getAccountOffice()) &&
             websiteMatcher.accept(depositSurchargeRequest.getWebsiteId());
   }
}
```

### What's happening

Deciding whether a card surcharge applies to a given deposit depends on matching the card scheme, issuing country, office, and website against a huge, business-maintained rule table (over 200 Spring-wired rule beans in `cardpayments-surcharges.xml`, covering every scheme/country/office/website combination IG supports). `SurchargeRule.matches(...)` abstracts all of that complexity into one boolean method, delegating each dimension to its own matcher without caring how any individual matcher decides to match.

### Why this is abstraction

The code that iterates the full rule table to find the applicable surcharge rule (`matches(request)`) has zero knowledge of *how* a scheme or country is matched — whether by exact-list membership or an unconditional pass. That knowledge is deliberately abstracted away into the matcher implementations (see the polymorphism example below), which is what lets IG's commercial/compliance team's rule table grow to hundreds of entries without the rule-matching engine itself ever changing.

---

### Example 5 — Business logic: `AbstractTmsPaymentField<T>` (`bank-withdrawal`)

**File:** `bank-withdrawal/impl/src/main/java/com/iggroup/wt/bankwithdrawal/eai/tms/field/AbstractTmsPaymentField.java`

```java
public abstract class AbstractTmsPaymentField<T> implements TmsPaymentField {
   private final T value;

   protected AbstractTmsPaymentField(T rawValue) {
      T formatted = format(rawValue);
      validate(formatted);
      this.value = formatted;
   }

   protected T format(T rawValue) throws FieldValidationException {
      if (rawValue == null) {
         throw new FieldValidationException(this.getClass().getSimpleName(), "rawValue is null");
      }
      return rawValue;
   }

   protected abstract void validate(T formattedValue) throws FieldValidationException;

   @Override
   public final String getValue() {
      return value.toString();
   }
}
```

### What's happening

This is the base type for every field that gets written into a bank's TMS (Treasury Management System) withdrawal file — bank names, account numbers, currencies, country codes, transaction dates. Each field type must be formatted (strip disallowed characters, normalize whitespace) and validated (length limits, required patterns) **before it's allowed to exist as an object at all** — the constructor runs `format()` then `validate()` and only then assigns `value`. There is no code path to get an `AbstractTmsPaymentField` instance holding unvalidated or unformatted data.

### Why this is abstraction

Code that assembles a TMS payment file doesn't need to know the formatting/validation rules for a `BankName` versus a `Currency` versus a `BeneficiaryAccountNumber` — it just calls `getValue()` on any `TmsPaymentField` and gets a guaranteed-valid string. The abstract `validate(T)` method is the one piece of information each concrete field type must supply; everything else (the "format-then-validate-at-construction" lifecycle, the final `getValue()`) is fixed and hidden from callers.

---

## 2. Inheritance

### Example 1 — Framework-level (`wallet-payments`)

**Parent — `wallet-payments/integration/src/main/java/com/iggroup/wt/wallet/payments/web/filter/AbstractFilter.java`:**

```java
public abstract class AbstractFilter implements Filter {
   @Setter
   private RequestExclusionStrategy requestExclusionStrategy;

   @Override
   public void doFilter(ServletRequest request, ServletResponse response, FilterChain filterChain) throws IOException, ServletException {
      HttpServletRequest httpServletRequest = (HttpServletRequest) request;
      HttpServletResponse httpServletResponse = (HttpServletResponse) response;
      if (isExcluded(httpServletRequest)) {
         filterChain.doFilter(request, response);
      } else {
         doFilter(httpServletRequest, httpServletResponse, filterChain);   // delegates to the subclass
      }
   }

   public abstract void doFilter(HttpServletRequest httpServletRequest, HttpServletResponse httpServletResponse, FilterChain filterChain) throws IOException, ServletException;

   private boolean isExcluded(HttpServletRequest request) {
      return requestExclusionStrategy != null && requestExclusionStrategy.isExcluded(request);
   }
   // init()/destroy() no-ops also live here
}
```

**Child — `wallet-payments/integration/src/main/java/com/iggroup/wt/wallet/payments/web/filter/ServiceAuthenticationFilter.java`:**

```java
public class ServiceAuthenticationFilter extends AbstractFilter {
   // supplies its own doFilter(HttpServletRequest, HttpServletResponse, FilterChain)
   // with authentication-specific logic
}
```

### Why this matters

Lay the two side by side and the benefit is concrete: `ServiceAuthenticationFilter` is a handful of lines because it inherits the exclusion check, the `ServletRequest`→`HttpServletRequest` casting, and the no-op lifecycle methods from `AbstractFilter`. It only writes the one thing that's actually its job — authenticating the request — and the servlet-level wiring (`isExcluded`, `init`, `destroy`) is guaranteed identical to every other filter in the chain because it's literally the same inherited code, not a copy.

---

### Example 2 — Business logic: the credit sign-off stage hierarchy (`bank-withdrawal`)

**Files:**
- `bank-withdrawal/impl/.../domain/paymentrun/stages/CreditSignOffStage.java` (abstract, extends `AbstractPaymentRunStage`)
- `bank-withdrawal/impl/.../domain/paymentrun/stages/FirstCreditSignOff.java` (concrete)
- `bank-withdrawal/impl/.../domain/paymentrun/stages/SecondCreditSignOff.java` (concrete)

```java
/**
 * Class contains common operations in {@link FirstCreditSignOff}
 * and {@link SecondCreditSignOff}
 */
public abstract class CreditSignOffStage extends AbstractPaymentRunStage {

   @Override
   protected void updateWithdrawalRequests(PaymentRun currentPaymentRun, Audit updatedAudit) {
      loadStateForWithdrawalProcesses(currentPaymentRun, updatedAudit);
      final Map<UpdateAction, List<BankWithdrawalProcess>> requestsByUpdateAction = getRequestsByUpdateAction(currentPaymentRun);

      handleRejectedRequests(requestsByUpdateAction.getOrDefault(REJECT, EMPTY_LIST));
      handleAmendedRequests(currentPaymentRun, requestsByUpdateAction.getOrDefault(AMEND, EMPTY_LIST));
      handleCheckedRequests(requestsByUpdateAction.getOrDefault(CHECK, EMPTY_LIST));
      handleUnmodifiedRequests(requestsByUpdateAction.getOrDefault(DEFAULT, EMPTY_LIST));
   }

   private void rejectCompletely(BankWithdrawalProcess withdrawalProcess) {
      // creates reversal ledger entries, publishes WithdrawalRequestRejected event, etc.
   }

   protected abstract void handleAmendedRequests(PaymentRun currentPaymentRun, List<BankWithdrawalProcess> orDefault);
   protected abstract void handleUnmodifiedRequests(List<BankWithdrawalProcess> unModifiedRequests);
}
```

```java
@Component
public class FirstCreditSignOff extends CreditSignOffStage {

   @Override
   protected void validateCurrentPaymentRun(PaymentRun currentPaymentRun, Long regionId, User user) throws PaymentRunException {
      if (FIRST_CREDIT_SIGNOFF != currentPaymentRun.getProcessStage()) {
         throw new PaymentRunException(format(INVALID_STAGE_PROCESSING, currentPaymentRun.getProcessStage()));
      }
   }

   @Override
   protected void handleAmendedRequests(PaymentRun currentPaymentRun, List<BankWithdrawalProcess> amendedRequests) {
      amendedRequests.forEach(r -> {
         r.getCurrentState().completeAmend();
         getBankWithdrawalProcessRepository().add(r);
      });
   }
   // ... handleUnmodifiedRequests similarly
}
```

### What's happening

This is a genuine **two-level inheritance chain**, driven entirely by business process, not framework plumbing:

1. `AbstractPaymentRunStage` — defines the universal stage-progression algorithm (any stage).
2. `CreditSignOffStage` — adds a second layer shared only by the two credit sign-off stages: grouping in-flight withdrawal requests by what the user did to them (rejected / amended / checked / left unmodified) and handling rejection (including ledger reversal and event publishing) identically for both sign-off steps.
3. `FirstCreditSignOff` / `SecondCreditSignOff` — each supplies only what's truly different: which `WithdrawalStage` enum value is "currently valid" for it to run, and what "amended" or "unmodified" actually means at that specific point in the sign-off process (e.g. `SecondCreditSignOff` additionally has to deal with batching and MetaTrader withdrawals, which `FirstCreditSignOff` never touches).

### Why this matters

Ledger-reversal logic for a rejected withdrawal is **security/finance-critical** and is written exactly once, in `CreditSignOffStage`, then inherited by both concrete sign-off stages. If `FirstCreditSignOff` and `SecondCreditSignOff` had been written as unrelated classes, that reversal logic would almost certainly have drifted between the two over time — a classic source of reconciliation bugs in payment systems.

---

### Example 3 — Business logic: `BankName extends AbstractTmsPaymentField<String>` (`bank-withdrawal`)

**Parent — `bank-withdrawal/impl/src/main/java/com/iggroup/wt/bankwithdrawal/eai/tms/field/AbstractTmsPaymentField.java`:**

```java
public abstract class AbstractTmsPaymentField<T> implements TmsPaymentField {
   private final T value;

   protected AbstractTmsPaymentField(T rawValue) {
      T formatted = format(rawValue);   // every subclass constructor runs through this same pipeline
      validate(formatted);
      this.value = formatted;
   }

   protected T format(T rawValue) throws FieldValidationException {
      if (rawValue == null) {
         throw new FieldValidationException(this.getClass().getSimpleName(), "rawValue is null");
      }
      return rawValue;
   }

   protected abstract void validate(T formattedValue) throws FieldValidationException;

   @Override
   public final String getValue() {
      return value.toString();
   }
}
```

**Child — `bank-withdrawal/impl/src/main/java/com/iggroup/wt/bankwithdrawal/eai/tms/field/BankName.java`:**

```java
public abstract class BankName extends AbstractTmsPaymentField<String> {
   protected static final int MAX_LENGTH = 35;
   private static final String ALLOWED = "\\p{L}\\p{M}- '/&\\.\\(\\)"; // letters, marks, hyphen, space, etc.

   public BankName(String rawValue) {
      super(rawValue);
   }

   @Override
   protected String format(String rawValue) {
      rawValue = super.format(rawValue);
      String cleaned = rawValue.replaceAll(String.format(NOT_ALLOWED_REGEX_PATTERN, ALLOWED), " ");
      cleaned = cleaned.replaceAll(" +", " ");
      return cleaned.trim();
   }

   @Override
   protected void validate(String formattedValue) {
      if (formattedValue.isEmpty()) {
         throw new FieldValidationException(getFieldName(), "cannot be empty");
      }
      if (formattedValue.length() > MAX_LENGTH) {
         throw new FieldValidationException(getFieldName(),
            String.format("exceeds maximum length of %d (actual: %d)", MAX_LENGTH, formattedValue.length()));
      }
   }
}
```

### What's happening

Look at the two side by side: the parent's constructor — `format(rawValue)` then `validate(formatted)` then assign — runs for **every** field type, unconditionally, because `BankName`'s constructor does nothing but call `super(rawValue)`. `BankName` inherits that entire lifecycle plus the null-check inside the base `format()` (notice its own `format()` calls `super.format(rawValue)` first, then layers its own character-stripping logic on top). It only needs to define what "valid" and "formatted" mean *specifically for a bank name field*: strip disallowed characters, collapse whitespace, enforce a 35-character limit. Sibling classes like `Currency`, `CountryCode`, and `BeneficiaryAccountNumber` extend the exact same parent and each define their own formatting/length rules in the same two methods.

### Why this matters

This is the real payoff of the parent/child split: every one of the ~6 field types in this package gets null-safety and the construct-time validation guarantee "for free" from `AbstractTmsPaymentField`, and it is *structurally impossible* to construct any of them without going through `format()` then `validate()` — there's no alternate constructor path that skips it. A new field type (say, a new beneficiary reference format for a newly-onboarded country) is added by writing *only* `format()` and `validate()` — a few lines — rather than re-deriving the whole safety guarantee from scratch.

---

### Example 4 — Business logic: `IGAccount` / `InvestYourWayAccount` / `MT4Account extends Account` (`payments` funds-transfer module)

**Parent — `payments/service/src/main/java/uk/co/igindex/payments/service/newfundstransfer/internal/Account.java`:**

```java
/**
 * Base account used to represent the different types of accounts ... from the point of view of funds transfers.
 */
public abstract class Account {
   private Client account;
   private ClientDetails clientDetails;
   private Money money;
   private FundingRestrictionType fundingRestrictionType;
   private String productCode;

   public void acceptsTransferTo(Account toAccount, BigDecimal amount) throws FundsTransferException {
      validateAccountsAreLinked(toAccount);
      validateHasBaseCurrency(toAccount.getBaseIsoCurrency());
      validateAccountHasSufficientFunds(amount);
      validateAccountTransfersAreEnabledForTheSite();
      validateIfBaseCurrencyIsJPYThenNoDecimalPlaces(amount);
   }

   public void acceptsTransferFrom(Account fromAccount, BigDecimal amount) throws FundsTransferException {
      validateAccountTransfersAreEnabledForTheSite();
   }

   public String withdraw(LedgerService ledgerService, Account associatedAccount, BigDecimal amount) throws FundsTransferException {
      return ledgerService.withdraw(this, new LedgerNarrative(FUNDS_TRANSFER_TO, associatedAccount), this.getBaseIsoCurrency(), amount);
   }

   public boolean isFundsTransferEnabledForAccount() {
      return true;   // default: every account type is transfer-enabled unless it says otherwise
   }

   public void validateAccountHasSufficientFunds(BigDecimal amountToTransfer) throws FundsTransferException {
      BigDecimal availableToWithdraw = getAvailableToWithdraw();
      if (amountToTransfer.compareTo(availableToWithdraw) > 0) {
         throw new FundsTransferException("Insufficient funds. The account has " + availableToWithdraw + "  available to transfer",
                 FundsTransferResult.FUNDS_TRANSFER_FAILED_INSUFFICIENT_FUNDS);
      }
   }

   public TaxWrapperAccount getTaxWrapperAccount() {
      return null;   // overridden meaningfully only where a tax wrapper concept applies
   }

   // validateAccountsAreLinked, validateHasBaseCurrency, validateAccountTransfersAreEnabledForTheSite,
   // validateIfBaseCurrencyIsJPYThenNoDecimalPlaces are all PRIVATE — unreachable by any subclass or caller
}
```

**Children:**

```java
public class MT4Account extends Account {
   private final MetaTraderService metaTraderService;

   @Override
   public void acceptsTransferTo(Account toAccount, BigDecimal amount) throws FundsTransferException {
      validateMT4IsAvailable();
      super.acceptsTransferTo(toAccount, amount);
      validateHasEnoughFundsToWithdrawInMetatrader(amount);
   }

   @Override
   public String withdraw(LedgerService ledgerService, Account associatedAccount, BigDecimal amount) throws FundsTransferException {
      metaTraderService.withdraw(getAccountId(), getBaseIsoCurrency(), amount);
      return super.withdraw(ledgerService, associatedAccount, amount);
   }
}
```

```java
public class InvestYourWayAccount extends Account {
   @Override
   public void acceptsTransferTo(Account toAccount, BigDecimal amount) throws FundsTransferException {
      validateAccountIsNotFundAccount();
      super.acceptsTransferTo(toAccount, amount);
   }

   @Override
   public boolean isFundsTransferEnabledForAccount() {
      // false specifically when the linked client account is a FUND-type account
   }
}
```

```java
public class IGAccount extends Account {
   @Override
   public void acceptsTransferFrom(Account fromAccount, BigDecimal amount) throws FundsTransferException {
      super.acceptsTransferFrom(fromAccount, amount);
      if (isOversubscribeValidationRequired(getTaxWrapperAccount(), fromAccount.getTaxWrapperAccount())) {
         validateDepositWillNotOversubscribeTaxWrapperAccount(fromAccount.getBaseIsoCurrency(), amount);
      }
   }
}
```

### What's happening

Put the parent and children side by side and the inheritance benefit is concrete. `Account.acceptsTransferTo(...)` already runs five validation steps — linked-account check, currency match, sufficient-funds check, site-level transfer toggle, JPY decimal-place rule. All three subclasses inherit that entire pipeline *unchanged* and layer their own extra, account-type-specific rule **on top** by calling `super.acceptsTransferTo(...)` partway through:

- `MT4Account` checks the external MetaTrader bridge is up, then runs the parent's five checks, then additionally checks MetaTrader itself has enough funds.
- `InvestYourWayAccount` blocks transfers for FUND-type sub-accounts *before* deferring to the parent's checks.
- `IGAccount` defers straight to the parent (`acceptsTransferFrom`) and then adds ISA/LISA oversubscription checks on top.

None of the three re-implements the linked-account check, the currency check, or the JPY-decimal rule — those five private methods only exist in `Account` and are literally unreachable from the subclasses (Java enforces this: `private` methods aren't inherited, only the public method that calls them is). A subclass physically cannot skip or duplicate that logic even if it wanted to.

### Why this matters

This is inheritance doing real work in a regulated domain: ISA oversubscription rules, MetaTrader availability, and fund-account restrictions are genuinely different per account type and change independently (e.g. HMRC changes ISA rules without touching MetaTrader logic). Extending `Account` lets each rule set live only where it's relevant, while the shared, money-safety-critical checks (sufficient funds, currency match) are guaranteed — by the language, not by convention — to run for every account type without being re-typed three times. If a new account type is added tomorrow (say, a crypto-wrapper account), it inherits the five safety checks automatically just by extending `Account`; the only way to lose them would be to not extend `Account` at all.

---

## 3. Polymorphism

### Example 1 — Framework-level (`wallet-payments`)

**Context:** Same `AbstractFilter` / `Filter` hierarchy as above — the servlet container holds a reference typed as `Filter`, and at runtime dispatches to whichever concrete subclass (e.g. `ServiceAuthenticationFilter`) is wired into the chain.

---

### Example 2 — Business logic: the withdrawal process state machine (`bank-withdrawal`)

**File:** `bank-withdrawal/impl/src/main/java/com/iggroup/wt/bankwithdrawal/domain/processstates/WithdrawalProcessState.java`

```java
public interface WithdrawalProcessState {
   default void amend(WithdrawalUpdateDetails withdrawalUpdateDetails) {
      throw new UnsupportedOperationException("Amend operation is not applicable to the current state " + this);
   }
   default void completeReject(String withdrawalCancelledLedgerReference, String surchargeCancelledLedgerReference) {
      throw new UnsupportedOperationException("Reject to complete operation is not applicable to the current state " + this);
   }
   default void completeAmend() {
      throw new UnsupportedOperationException("Final amendment operation is not applicable to the current state " + this);
   }
   default void completeCheck() {
      throw new UnsupportedOperationException("Final check operation is not applicable to the current state " + this);
   }
   // ... check(), reject(), validateReset(), gotoReadyForBankFile(), gotoReadyForSecondCreditSignOff()
}
```

11 concrete classes implement this — `SubmittedState`, `ReadyForCreditSignOffOne`, `CheckedAtCreditSignOffOne`, `AmendedAtCreditSignOffOne`, `RejectedAtCreditSignOffOne`, `ReadyForCreditSignOffTwo`, `CheckedAtCreditSignOffTwo`, `AmendedAtCreditSignOffTwo`, `RejectedAtCreditSignOffTwo`, `ReadyForTmsState`, `BankFileState` — each overriding *only* the operations that are legal for that specific point in the withdrawal's lifecycle. And it's called exactly the way you'd expect polymorphism to be used, from `CreditSignOffStage`:

```java
checkedRequests.forEach(r -> {
   r.getCurrentState().completeCheck();   // dispatches to whichever concrete state r is actually in
   getBankWithdrawalProcessRepository().add(r);
});
```

### Why this is polymorphism

`r.getCurrentState()` returns a value typed only as `WithdrawalProcessState` — the calling code has no idea, and doesn't need to know, whether the withdrawal is in `CheckedAtCreditSignOffOne` or `CheckedAtCreditSignOffTwo`. Calling `completeCheck()` dispatches to the correct state's behavior at runtime. If the withdrawal happens to be in a state where "complete check" doesn't make sense (e.g. it was already rejected), the inherited default implementation throws — **the state pattern replaces a long chain of `if (stage == X) ... else if (stage == Y) ...` business-logic conditionals** with pure dynamic dispatch, and makes illegal operations fail loudly by construction rather than silently doing the wrong thing.

---

### Example 3 — Business logic: `FirstCreditSignOff` vs `SecondCreditSignOff` (`bank-withdrawal`)

**Context:** Same classes as the inheritance example above.

Both classes are held and invoked through the same `PaymentRunStage` interface reference by the orchestrating code (e.g. a controller or scheduled job that just calls `stage.progressPaymentRunStage(regionId, bankId, user)`). But `validateCurrentPaymentRun`, `handleAmendedRequests`, and `handleUnmodifiedRequests` resolve to completely different logic depending on which concrete stage object is actually being driven — `FirstCreditSignOff` checks for the `FIRST_CREDIT_SIGNOFF` stage and hands amended requests off for a simple "complete amend" state transition; `SecondCreditSignOff` additionally has to coordinate with `BatchingServiceFactory`, `MetaTraderWithdrawalService`, and `RoleService` because it's the final sign-off before money actually leaves via a bank file.

### Why this matters

The calling code that advances payment runs through their stages is written once, against `PaymentRunStage`, and never needs an `instanceof` check or a switch statement to know it's dealing with the first sign-off versus the second. Each stage object polymorphically supplies its own notion of "valid" and "amended," which is exactly what lets new stages (e.g. a hypothetical third sign-off tier) be added without touching the orchestration code at all.

---

### Example 4 — Business logic: pluggable rule matchers (`card-payments`)

**Files:** `card-payments/impl/src/main/java/com/iggroup/wt/cardpayments/domain/deposits/surcharge/rules/{RuleMatcher,CountryMatcher,AllCountriesMatcher,CountryListMatcher}.java`

```java
public interface RuleMatcher<T> {
   boolean accept(T condition);
}

public interface CountryMatcher extends RuleMatcher<String> { }

public class AllCountriesMatcher implements CountryMatcher {
   @Override
   public boolean accept(String condition) {
      return true;              // matches any country — used for "applies everywhere" rules
   }
}

public class CountryListMatcher implements CountryMatcher {
   private Set<String> countries = new HashSet<>();

   @Override
   public boolean accept(String country) {
      return countries.contains(country);   // matches only an explicit, configured set of countries
   }
}
```

These (and their `Scheme`/`Office`/`Website`/`PaymentServiceProvider` counterparts) are wired together declaratively in `cardpayments-surcharges.xml` — over 200 Spring bean definitions, one per real-world surcharge rule IG's commercial team maintains (e.g. `europeanCountryGroupMatcher`, `visaCreditCardSchemesMatcher`, `chinaWebsiteMatcher`), each choosing whichever matcher implementation fits that rule.

### Why this is polymorphism

`SurchargeRule.matches(...)` (see the abstraction example above) calls `countryMatcher.accept(...)`, `schemeMatcher.accept(...)`, etc. on fields typed only as the interface (`CountryMatcher`, `SchemeMatcher`, ...). At runtime, depending on which bean Spring wired in for that particular rule, `accept(...)` either does an unconditional "yes" (`AllCountriesMatcher`) or a set-membership check against a business-configured list (`CountryListMatcher`). `SurchargeRule` itself never changes, never knows how many matcher implementations exist, and never needs modifying when a new matching strategy (e.g. a regex-based matcher, or a "all except these" matcher) is introduced.

### Why this matters

This is polymorphism directly enabling a non-technical team to maintain complex business rules: product/compliance can add, remove, or reconfigure hundreds of surcharge rules purely through Spring XML bean wiring, because every rule's matching behavior is just "some `RuleMatcher<T>`" from the engine's point of view.

---

## 4. Encapsulation

### Example 1 — Framework-adjacent (`psp-maintenance`)

**File:** `psp-maintenance/domain/src/main/java/com/iggroup/wt/psp/maintenance/domain/PaymentServiceProvider.java`

```java
@Builder
@EqualsAndHashCode(onlyExplicitlyIncluded = true)
@Getter
@Setter
@ToString(exclude = {"username", "password", "refundPassword"})
public class PaymentServiceProvider {
   @Setter(PRIVATE)
   @Include
   private String name;
   @Setter(PRIVATE)
   private String username;
   private String password;
   private String refundPassword;
   // ... createdBy, createdAt, updatedBy, updatedAt
}
```

Identity fields (`name`, `username`) are only mutable via `private` setters (locked after construction), sensitive fields are excluded from `toString()` to stop credentials leaking into logs, and equality is deliberately scoped to just the `name` field rather than comparing every field.

---

### Example 2 — Business logic: `Money` (`bank-withdrawal`)

**File:** `bank-withdrawal/impl/src/main/java/com/iggroup/wt/bankwithdrawal/domain/Money.java`

```java
public class Money {
   private final Currency currency;
   private final BigDecimal amount;

   public Money(Currency currency, BigDecimal amount) {
      if (currency == null) {
         throw new IllegalArgumentException("Currency cannot be null");
      }
      if (amount == null) {
         throw new IllegalArgumentException("Amount cannot be null");
      }
      this.currency = currency;
      this.amount = amount;
   }

   public Currency getCurrency() { return currency; }
   public BigDecimal getAmount() { return amount; }
   // equals()/hashCode()/toString() compare and print both fields together
}
```

### What's happening

Both fields are `private final` — there are **no setters at all**. Once a `Money` object exists, it is guaranteed to represent one specific, valid, unchangeable amount-plus-currency pairing for its entire lifetime. The constructor is the single gatekeeper: it's structurally impossible to construct a `Money` with a `null` currency or a `null` amount, because the constructor throws before the object is ever assigned to a field.

### Why this matters for a payments system

Money values flow through ledger postings, surcharge calculations, and withdrawal validations across the whole `bank-withdrawal` application. If `Money` were mutable (e.g. a `setAmount()` existed), a reference to a `Money` object passed into three different methods could be silently changed by any one of them, corrupting the other two callers' view of "how much money this represents" — a bug class that's catastrophic in a financial system. Immutability-by-construction removes that entire category of bug by design, not by convention.

---

### Example 3 — Business logic: guarded mutation in `BankWithdrawalProcess.amend(...)` (`bank-withdrawal`)

**File:** `bank-withdrawal/impl/src/main/java/com/iggroup/wt/bankwithdrawal/domain/bankwithdrawal/BankWithdrawalProcess.java`

```java
public void amend(WithdrawalUpdateDetails withdrawalUpdateDetails) throws WithdrawalRequestUpdateException {
   final BigDecimal amendedAmount = withdrawalUpdateDetails.getAmount();

   if ((amendedAmount == null)
           || isGreaterThanOrEqualCurrentAmount(amendedAmount)
           || isLessThanMinThreshold(amendedAmount)) {
      throw new WithdrawalRequestUpdateException(
         String.format("Amended amount should be less than current amount %s and greater than zero. Amended amount = %s",
            withdrawAmount, amendedAmount),
         INVALID_AMEND_AMOUNT);
   }
   currentState.amend(withdrawalUpdateDetails);
}

private boolean isGreaterThanOrEqualCurrentAmount(BigDecimal amendedAmount) {
   return amendedAmount.compareTo(withdrawAmount) >= 0;
}

private boolean isLessThanMinThreshold(BigDecimal amendedAmount) {
   return (amendedAmount.compareTo(ZERO) <= 0);
}
```

### What's happening

`amend(...)` is the only sanctioned way application code is meant to reduce a withdrawal's amount mid-sign-off. It encapsulates a real business invariant — *an amended withdrawal amount must be strictly less than the current amount and strictly greater than zero* — behind a single public method, with the two comparison rules (`isGreaterThanOrEqualCurrentAmount`, `isLessThanMinThreshold`) kept entirely `private`. Callers can't accidentally bypass the rule by reaching for a raw field mutator for this operation; they go through `amend()` or they get a `WithdrawalRequestUpdateException`.

### Why this matters

This shows encapsulation protecting **business correctness**, not just data hiding: without this guard, a caller could amend a £500 withdrawal up to £5,000, or down to a negative amount, both of which would be financially nonsensical (and in the up-case, a potential money-laundering/limit-bypass vector). The validation lives in exactly one place, so every caller — current and future — gets the same guarantee for free.

---

### Example 4 — Business logic: `SurchargeRule`'s hidden matching criteria (`card-payments`)

**File:** `card-payments/impl/src/main/java/com/iggroup/wt/cardpayments/domain/deposits/surcharge/rules/SurchargeRule.java`

```java
public class SurchargeRule {
   private final WebsiteMatcher websiteMatcher;
   private final SchemeMatcher schemeMatcher;
   private final CountryMatcher countryMatcher;
   private final OfficeMatcher officeMatcher;
   private final SurchargePolicy surchargePolicy;

   // constructor is the only way in — all five fields are private and final, no setters exist

   public boolean matches(DepositSurchargeRequest depositSurchargeRequest) { /* ... */ }

   public Money calculateSurcharge(DepositSurchargeRequest depositSurchargeRequest, TotalDailyDeposit totalDailyDeposit, CurrencyConverter currencyConverter) {
      return surchargePolicy.calculateSurcharge(depositSurchargeRequest, totalDailyDeposit, currencyConverter);
   }
}
```

### What's happening

A `SurchargeRule` bundles together everything that defines one surcharge rule — four matching criteria plus the policy for calculating the actual surcharge amount — behind exactly two public operations: "does this rule apply to this deposit" (`matches`) and "what's the surcharge" (`calculateSurcharge`). All five collaborators are `private final`, assigned once at construction (by Spring, from the XML configuration) and never exposed or reassigned.

### Why this matters

Calling code elsewhere in the deposit-processing flow (the code that loops over all configured `SurchargeRule`s looking for the first match) only ever calls `.matches(...)` and `.calculateSurcharge(...)`. It has no way to reach into a rule and inspect or swap out its `countryMatcher` or `surchargePolicy` — which matters because a `SurchargeRule`'s identity *is* that specific combination of matchers and policy; letting outside code mutate one piece after construction would silently turn one configured business rule into a different, unintended one.

---

## Summary Table

| Concept | Framework example | Business-logic examples |
|---|---|---|
| Abstraction | `AbstractFilter` (`wallet-payments`) — template method hides filtering mechanics | `AbstractPaymentRunStage` (`bank-withdrawal`) · `Account` (`payments`) · `SurchargeRule` (`card-payments`) · `AbstractTmsPaymentField<T>` (`bank-withdrawal`) |
| Inheritance | `ServiceAuthenticationFilter extends AbstractFilter` (`wallet-payments`) | `CreditSignOffStage` → `FirstCreditSignOff`/`SecondCreditSignOff` (`bank-withdrawal`) · `BankName extends AbstractTmsPaymentField<String>` (`bank-withdrawal`) · `IGAccount`/`InvestYourWayAccount`/`MT4Account extends Account` (`payments`) |
| Polymorphism | `Filter` reference dispatching to concrete filter (`wallet-payments`) | `WithdrawalProcessState` 11-way state pattern (`bank-withdrawal`) · `FirstCreditSignOff` vs `SecondCreditSignOff` via `PaymentRunStage` (`bank-withdrawal`) · `AllCountriesMatcher`/`CountryListMatcher` via `RuleMatcher<T>` (`card-payments`) |
| Encapsulation | `PaymentServiceProvider` (`psp-maintenance`) — private setters, log-safe `toString` | `Money` immutable value object (`bank-withdrawal`) · `BankWithdrawalProcess.amend(...)` guarded mutation (`bank-withdrawal`) · `SurchargeRule`'s hidden matcher/policy collaborators (`card-payments`) |

---

*Source modules: `wallet-payments`, `psp-maintenance`, `bank-withdrawal` (payment-run sign-off workflow in `domain/paymentrun`, bank-file field generation in `eai/tms/field`, process state machine in `domain/processstates`), `payments` (funds-transfer account hierarchy in `service/newfundstransfer/internal`), and `card-payments` (surcharge rule-matching engine in `domain/deposits/surcharge/rules`). Scope was intentionally kept broad-but-shallow across modules rather than exhaustive — this is a representative sampling, not a full audit of OOP usage across the monorepo.*
