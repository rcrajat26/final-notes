# GraalVM Decision: Bank Deposit Payments Application

## Decision

**Run the application on AWS Lambda using GraalVM Native Image.**

ECS with a regular JVM is the fallback if something outside the code (a platform or team constraint) rules Lambda out. Native Image on ECS is not worth doing for this application.

---

## Application Summary

A bank deposit payment application for a small Japanese client base. It supports eight banks through the aggregator BJP (Billing Japan). Clients use it only occasionally, when they want to load or withdraw money.

| Component | What it does |
|---|---|
| Config API | Returns the transaction reference ID, bank list, merchant configuration (identifiers, success and failure callback URLs), transaction token, and customer name |
| Initiate API | Records the transaction as initiated in the database |
| Webhook API | Receives the BJP callback, verifies the transaction token, and sends the deposit to the ledger via Kafka |
| Manual Deposit API | Used by the cash ops team to push through a transaction when no webhook arrived |
| Reconciliation workflow | Reads the BJP transaction file every 2-3 hours, finds transactions still marked initiated, and sends successful ones to the ledger via Kafka |

---

## Options Compared

| Option | Summary |
|---|---|
| **Lambda + GraalVM Native Image** | Pay per request, millisecond startup, no containers to manage |
| **ECS + regular JVM** | Always-on containers, no cold starts, normal JVM build and tooling |

---

## Decision Matrix

| # | Factor | Application situation | Lambda + Native | ECS + JVM | Favors |
|---|---|---|---|---|---|
| 1 | Traffic pattern | Occasional use, not constant | Pay only when used | Pay for containers running all day | Lambda + Native |
| 2 | Cold start | Deposit screen and webhook must respond promptly | Native starts in milliseconds | No cold starts at all | Neutral |
| 3 | Idle cost | Long gaps between requests | Near zero when idle | At least one task always running | Lambda + Native |
| 4 | Webhook reliability | BJP callback must be answered reliably | Fast with native | Always warm | Neutral |
| 5 | Reconciliation job | Short job every 2-3 hours | Natural fit for a scheduled Lambda | Needs a container or scheduled task | Lambda + Native |
| 6 | Kafka connection | Ledger calls go through Kafka | Producer must be reused across invocations | One long-lived producer | ECS + JVM |
| 7 | Code readiness for native | No reflection, proxies, or custom classloaders | Native build should be clean | Not needed | Lambda + Native |
| 8 | Dependency freedom | Rewrite, so new dependencies can be chosen | Pick native-friendly libraries | Any library works | Lambda + Native |
| 9 | Spring Boot upgrade | Moving to a newer version | Needed | Needed | Neutral |
| 10 | Throughput | Low volume, no peak-performance pressure | Native's lower peak speed doesn't matter | Not a differentiator | Lambda + Native |
| 11 | Build time | Native builds take longer | Slower builds, extra native test pass | Normal JVM build | ECS + JVM |
| 12 | Operational overhead | Small app, small team footprint | No cluster or container patching | Cluster, tasks, scaling, and images to manage | Lambda + Native |
| 13 | Debugging | Production issues need to be traceable | Harder with native binaries | Standard JVM tools | ECS + JVM |
| 14 | Japan-specific checks | Encoding, locale, and crypto checked | No blocker | No blocker | Neutral |
| 15 | Pilot result | Native pilot passed | Confirmed working | Not needed | Lambda + Native |
| 16 | Class structure fit | Plain Spring layering: one controller, one service, repository, Kafka publishers | Builds cleanly with Spring's build-time processing | No special handling | Lambda + Native |
| 17 | Reflection registration | Kafka event classes are serialized inside publishers, not in controller signatures | Must register them with `@RegisterReflectionForBinding` | Not needed | ECS + JVM |

**Result:** 9 favor Lambda + Native, 4 favor ECS + JVM, 4 are neutral.

### What each factor means

1. **Traffic pattern:** clients open the app only when depositing or withdrawing, so Lambda's per-request billing matches how it is used.
2. **Cold start:** the first request after idle time is slow on Lambda; native shrinks it to milliseconds, while ECS avoids it by always running, so neither option loses.
3. **Idle cost:** most of the day nothing happens, and Lambda costs almost nothing in that time while ECS keeps charging for running tasks.
4. **Webhook reliability:** BJP retries or fails the callback if you respond slowly, and both options respond fast enough (native by startup speed, ECS by being warm).
5. **Reconciliation job:** a file arriving every 2-3 hours is a short burst of work, which suits a scheduled Lambda better than a container kept running between files.
6. **Kafka connection:** opening a Kafka producer is slow, so on Lambda it must be reused across invocations, whereas ECS holds one connection for the life of the container.
7. **Code readiness for native:** reflection, dynamic proxies, and custom classloaders are what break native builds, and your code has none of them.
8. **Dependency freedom:** since this is a rewrite, you can choose libraries that already support native instead of working around old ones.
9. **Spring Boot upgrade:** the move to a newer version is required for either path, so it does not separate them.
10. **Throughput:** native is usually slower than a warmed-up JVM at peak load, but your volume is low enough that this never matters.
11. **Build time:** native builds take noticeably longer than JVM builds and need a second test run against the binary, a small cost paid once per release.
12. **Operational overhead:** Lambda has no cluster, container images, or task scaling to look after, which matters for a small application.
13. **Debugging:** native binaries give you fewer tools for production investigation than a regular JVM.
14. **Japan-specific checks:** character encoding, locale, and token verification were checked and cause no problem on either option.
15. **Pilot result:** the native pilot already passed, so the main risk of going native has been tested.
16. **Class structure fit:** the design is plain Spring layering with no reflection-heavy patterns, which is the shape Native Image handles best.
17. **Reflection registration:** classes that are only serialized to JSON inside the Kafka publishers need explicit registration for native, a small one-time step that a JVM never requires.

---

## Class Structure and Native Readiness

### Spring Boot class hierarchy

| Layer | Classes | Role |
|---|---|---|
| Controller | `BankDepositController` | All four endpoints: `getConfig`, `initiate`, `handleCallback`, `manualDeposit` |
| Service | `BankDepositService` | Business logic for the four endpoints |
| Shared services | `TransactionTokenService`, `LedgerDepositService`, `EmailNotificationService` | Token generate/verify, single-deposit ledger call, deposit email |
| Repository | `BankDepositRepository` | Saves and loads `Transaction` records |
| Kafka publishers | `LedgerKafkaPublisher`, `EmailKafkaPublisher` | Publish to the ledger topic and the separate email topic |
| Domain | `Transaction`, `TransactionStatus` | The record and its states |
| Configuration | `BankProperties`, `MerchantProperties` | Bank list and merchant settings from backend config |

### Native Image fit by class

| Part | Fit | Note |
|---|---|---|
| `BankDepositController`, `BankDepositService` | Easy | Standard Spring beans handled by build-time processing |
| `TransactionTokenService`, `LedgerDepositService`, `EmailNotificationService` | Easy | Plain logic; token algorithm (HMAC/SHA) already tested |
| `BankProperties`, `MerchantProperties` | Easy | Use constructor-bound `@ConfigurationProperties` (records work well) |
| `LedgerKafkaPublisher`, `EmailKafkaPublisher` | Mostly easy | Kafka client works; the JSON event classes need registration |
| `BankDepositRepository`, `Transaction` | Depends on data layer | JDBC, Spring Data JDBC, or jOOQ are easy; JPA/Hibernate is the likeliest source of build issues |
| DTOs (`LedgerDepositEvent`, `EmailEvent`, etc.) | Needs attention | Jackson reads and writes them through reflection |

The data layer is the one choice that matters most. In a rewrite, JDBC-based access keeps native risk very low.

### Why `@RegisterReflectionForBinding` is needed

Native Image keeps only code it can prove is used at build time. Jackson finds constructors, fields, and getters through reflection at runtime, which Native Image cannot see, so an unregistered class works on the JVM but fails in the native binary (for example, with "no serializer found" errors).

Spring already registers types it can detect, such as controller parameters and return types, `@ConfigurationProperties` classes, and entities. It cannot detect types that are only serialized inside your code:

- `LedgerDepositEvent` and `EmailEvent`, which are turned into JSON inside the Kafka publishers
- BJP callback and transaction file record classes, if they are parsed manually instead of bound by a controller

```java
@Component
@RegisterReflectionForBinding({LedgerDepositEvent.class, EmailEvent.class})
public class LedgerKafkaPublisher {
    // serializes LedgerDepositEvent to JSON and sends to Kafka
}
```

- Place the annotation on the class that uses the type, so the registration sits next to the code that needs it.
- Over-registering is harmless, so when in doubt, register.
- Running the tests against the native binary (`nativeTest`) exposes any missing registration before release.

---

## Why Not the Other Options

| Option | Reason |
|---|---|
| JVM on Lambda | A standard Spring Boot JVM takes several seconds to start, and with occasional traffic most requests would hit a cold function |
| ECS + JVM | Works fine, but costs for idle containers and adds container management with no benefit this code needs |
| Native Image on ECS | Always-on containers have no cold start problem, so native would give only a small memory saving |

**Fallbacks:** ECS + JVM if Lambda is ruled out, or JVM with Lambda SnapStart if a native build cannot be made to work.

---

## Lambda Layout

| Component | Lambda setup |
|---|---|
| Config, Initiate, and Manual Deposit APIs | One native Lambda behind API Gateway |
| BJP webhook | Same function or a separate one; acknowledge BJP quickly |
| Reconciliation | Separate native Lambda, triggered by an S3 file event or an EventBridge schedule |

---

## Miscellaneous

- **One transaction, one deposit:** the webhook, manual deposit, and reconciliation can all trigger the same ledger call. Use the transaction reference ID so each transaction is deposited only once.
- **Kafka producer:** reuse it across invocations rather than creating one per request.
- **Large reconciliation files:** process in chunks to stay within the 15-minute Lambda limit.
- **Email topic:** `EmailKafkaPublisher` writes to a different Kafka topic from the ledger; reuse its producer the same way.
- **Reflection hints:** register every JSON event class that is serialized only inside a publisher.
- **Testing:** run the tests against the native binary, not only the JVM build, before each release.
