# 04 — Transactions

## M12. Declarative Transaction Management

### T1. Transactions three ways — `MK` `[P→S→B]`
- Plain JDBC: `setAutoCommit(false)`, commit and rollback, passing the `Connection` through every call, then a `ThreadLocal` holder.
- Spring: `@Transactional`.
- Boot: `TransactionAutoConfiguration` and the transaction manager it picks.
- Demo: suspending all of a client's accounts atomically.

### T2. The abstraction — `MK`
- `PlatformTransactionManager`, `TransactionDefinition`, `TransactionStatus`.
- Implementations: DataSource, `JdbcTransactionManager` (5.3), JPA, Kafka (see M36), and JTA (`GTH`).

### T3. @Transactional mechanics — `MK`
- The interceptor flow.
- The connection is bound to the thread via `TransactionSynchronizationManager` and found again through `DataSourceUtils.getConnection` — which is why the transaction is "ambient".

### T4. Attributes — `MK`
- Propagation, isolation, `timeout` (seconds; default `-1` = none), `rollbackFor` / `noRollbackFor`, choosing a transaction manager.
- `readOnly`, and its three effects:
  - The JDBC connection is set read-only.
  - Hibernate's flush mode becomes `MANUAL`, so dirty checking is skipped.
  - The transaction is marked read-only for synchronizations.

### T5. Propagation — `MK`
- A table of all seven values.
- `REQUIRES_NEW`: the outer transaction is suspended and a second connection is taken.
- `NESTED`: a savepoint inside the same connection.

### T6. Rollback rules — `MK`
- By default, only `RuntimeException` and `Error` trigger rollback — the EJB CMT heritage.
- `rollbackFor`, and the most-specific-rule-wins matching.
- A composed `@BusinessTransactional` that sets `rollbackFor` once for the team.

### T7. Placement — `MK`
- On the service layer, at class or method level — not on interfaces.
- `private` and `final` methods are ignored.
- `@EnableTransactionManagement` vs Boot's automatic enablement.

### T8. Programmatic transactions — `MK`
- `TransactionTemplate` (`execute`, `executeWithoutResult`).
- `TransactionalOperator` for reactive code (see M33).

### Traps
- A checked exception **commits**.
- `this.` calls skip the transaction.
- `readOnly` on a class that has write methods.
- A transaction spanning an HTTP call.

---

## M13. Transactions in Practice and Internals

### T1. Rollback-only contagion — `MK`
- `setRollbackOnly`; reproducing `UnexpectedRollbackException`.
- Catching an exception and continuing inside the same method: partial work commits.
- `globalRollbackOnParticipationFailure`.

### T2. Propagation in practice — `MK`
- An audit row that must survive a rollback: `REQUIRES_NEW` vs an `AFTER_COMPLETION` listener vs an outbox row.
- Connection-pool exhaustion arithmetic for `REQUIRES_NEW`; the inner transaction deadlocking against the suspended outer one.
- A nesting outcome matrix, and `MANDATORY` used as an assertion.

### T3. Transaction synchronization — `MK`
- `TransactionSynchronization` callbacks; `registerSynchronization`; side effects in `afterCommit`.

### T4. Boundaries and failure timing — `MK`
- No remote calls inside a transaction; the cost of long transactions.
- Constraint violations can surface only at commit time, outside your method; `TransactionSystemException`.

### T5. Multiple transaction managers — `GTH`
- Qualifiers and `@Primary`.
- `ChainedTransactionManager` is deprecated; why XA is usually avoided; pointers to outbox and saga.

### T6. Validating existing transactions — `GTH`
- `validateExistingTransaction`: turns a silently ignored inner isolation setting into an error.

### T7. Internals — `MK`
- `TransactionAspectSupport.invokeWithinTransaction`.
- Where Spring looks for `@Transactional`: method → class → interface.
- `RuleBasedTransactionAttribute`.
- `AbstractPlatformTransactionManager`: `getTransaction`, `handleExistingTransaction`, suspend/resume, `processCommit`.
- `GTH` AspectJ mode, where self-invocation works.

### T8. Diagnostics — `MK`
- Turn on TRACE for `org.springframework.transaction`.
- Read the log lines: "Creating new transaction", "Participating", "Suspending", "Initiating rollback".

### T9. Build it: mini transaction manager — `MK`
- A `ThreadLocal` connection holder plus an interceptor supporting `REQUIRED`, `REQUIRES_NEW` and `NESTED`, with rollback rules.

### Traps
- A `@Transactional` test rolls back, so your `AFTER_COMMIT` behaviour is never exercised.
- `REQUIRES_NEW` for logging doubles connection usage.
