# Date & Time (java.time)
## Why the old API was replaced
- The old `java.util.Date` and `java.util.Calendar` classes had several design flaws:
  - `Date` is mutable, which can lead to bugs when used in collections or as keys in maps.
  - `Date` has a confusing API, with methods that are deprecated or inconsistent (e.g., months are zero-based, years are offset from 1900).
  - `Calendar` is also mutable and has a similarly confusing API.
- `java.time` (Java 8) replaced them with an immutable, clearly-separated set of types modeled on the same design discipline as String/BigDecimal: every "change" method returns a new instance.

### The core type split
- Java 8 introduced a new date and time API in the `java.time` package, which is more comprehensive and user-friendly than the old `java.util.Date` and `java.util.Calendar` classes.
- The new API is based on the ISO-8601 calendar system and provides classes for dates, times, date-times, durations, periods, and more.

**Java Date and Time Types**

| Type            | Represents                                           | Example                                   |
|-----------------|------------------------------------------------------|-------------------------------------------|
| `LocalDate`     | Date only, no time, no zone                          | `2024-03-15`                              |
| `LocalTime`     | Time only, no date, no zone                          | `14:30:00`                                |
| `LocalDateTime` | Date + time, no zone                                 | `2024-03-15T14:30:00`                     |
| `Instant`       | A single point on the UTC timeline (machine-centric) | `2024-03-15T09:00:00Z`                    |
| `ZonedDateTime` | Date + time + explicit time zone                     | `2024-03-15T14:30:00+05:30[Asia/Kolkata]` |
| `Duration`      | A time-based amount (hours/minutes/seconds/nanos)    | `PT2H30M`                                 |
| `Period`        | A date-based amount (years/months/days)              | `P1Y2M10D`                                |

- The `Local`* types are deliberately zone-less — they represent a date/time as a human would write it on a calendar or clock, with no claim about which actual instant in physical time that corresponds to. 
- Instant is the opposite: a precise point on the UTC timeline, with no human-calendar representation at all (no "year/month" fields — just seconds-since-epoch + nanos). 
- ZonedDateTime bridges the two: a LocalDateTime plus a ZoneId, letting you ask "what UTC instant is this?"

### Domain example — Account audit fields:
```java
public class Account {
    private final LocalDate openedDate;      // just a calendar date — no time needed
    private final Instant lastModifiedAt;    // precise audit timestamp — compare across zones safely
}
```

- Using `LocalDate` for `openedDate` is deliberate: an account-opening date doesn't need a time-of-day or time zone component, and giving it one invites bugs (e.g. comparing two "same day" timestamps that differ by milliseconds due to zone conversion). 
- Using Instant for lastModifiedAt is deliberate too: it's unambiguous regardless of which server/zone wrote it — the right choice for anything compared or sorted across systems.

## Immutability and the fluent "with/plus/minus" idiom
Every instance method that looks like a mutator returns a new object:
```java
LocalDate today = LocalDate.now();
LocalDate nextWeek = today.plusDays(7);     // today is unchanged
LocalDate lastMonth = today.minusMonths(1);
LocalDate specificDay = today.withDayOfMonth(1); // first of this month
```

## Parsing, formatting, and DateTimeFormatter
- The new API uses `DateTimeFormatter` for parsing and formatting, which is thread-safe and immutable.
```java
DateTimeFormatter fmt = DateTimeFormatter.ofPattern("dd/MM/yyyy");
LocalDate parsed = LocalDate.parse("15/03/2024", fmt);
String formatted = parsed.format(fmt); // "15/03/2024"
```
- `DateTimeFormatter` instances are immutable and thread-safe (unlike the old `SimpleDateFormat`). Default toString()/parse() without a formatter uses ISO-8601 (2024-03-15, 2024-03-15T14:30:00).


## `Duration` vs `Period` — the time-based/date-based split
- `Duration` is for time-based amounts (hours, minutes, seconds, nanoseconds) and is measured in seconds/nanos. It is suitable for measuring elapsed time or intervals.
- `Period` is for date-based amounts (years, months, days) and is measured in terms of calendar units. It is suitable for representing age, anniversaries, or other date-based calculations.
```java
Duration oneHour = Duration.ofHours(1);
Duration diff = Duration.between(start, end);      // works on time-based types (Instant, LocalTime, LocalDateTime)

Period oneMonth = Period.ofMonths(1);
Period diff2 = Period.between(startDate, endDate); // works on LocalDate only
```
- `Duration` measures in exact time units (seconds/nanos) — "25 hours" always means exactly 25×3600 seconds. 
- `Period` measures in calendar units (years/months/days) — "1 month" means "the same day next month," which is a variable number of actual seconds depending on which month, and interacts with leap years/month-length irregularities. 
- Mixing them up (e.g. trying to add a `Period` to an `Instant`) is a compile error — `Instant` has no calendar-field concept to apply a `Period` against.

## Time zones — `ZoneId` and the DST trap
```java
ZoneId zone = ZoneId.of("Asia/Kolkata");
ZonedDateTime zdt = ZonedDateTime.of(LocalDateTime.now(), zone);
```

- `ZoneId` is a named zone ("Asia/Kolkata", "Europe/London"), which encodes historical and future DST transition rules — not a fixed offset. 
- `ZoneOffset` (e.g. +05:30) is a fixed offset with no DST awareness; `ZoneId` is almost always the right choice for anything user-facing, since a fixed offset silently becomes wrong the moment a DST boundary is crossed. 
- This is why converting a `LocalDateTime` to an Instant requires a zone: the same clock-face time means a different actual instant depending on zone and even depending on time of year in zones that observe DST.

- Time zones are represented by `ZoneId` (e.g., `ZoneId.of("America/New_York")`), which encapsulates the rules for converting between local date-times and UTC instants, including daylight saving time transitions.
- A common trap is to assume that adding 24 hours to a `ZonedDateTime` always results in the same local time the next day. This is not true during DST transitions, where the local time may shift by an hour. Always use `plusDays(1)` or `minusDays(1)` for calendar-based adjustments, rather than adding a fixed number of hours.

### Legacy interop — toInstant() / bridge methods
Since a huge amount of existing code (older libraries, some JDK APIs, DB drivers) still speaks java.util.Date, the JDK added bridge methods rather than forcing a hard cutover:
```java
Date legacyDate = new Date();
Instant instant = legacyDate.toInstant(); // convert to Instant
Date backToLegacy = Date.from(instant);    // convert back to Date
```

## Comparison and equality

- `LocalDate`/`LocalDateTime`/etc. implement `equals()`/`hashCode()`/`compareTo()` based on actual field values — no `BigDecimal`-style scale trap here, since there's no analogous "same value, different representation" concept. `isBefore()`/`isAfter()`/`isEqual()` are provided as more readable alternatives to `compareTo()` comparisons.


---
# Miscellaneous 
## Exhaustive trap list (including old ↔ new conversion traps)
1. Discarding immutable return values — `today.plusDays(7)`; alone does nothing; must assign.
2. Month indexing mismatch during migration — old `Calendar` months are 0-indexed (`Calendar.JANUARY` == 0); `java.time`'s `Month` enum / `getMonthValue()` are 1-indexed. Direct numeric reuse across old/new code is a classic off-by-one.
3. Deprecated `Date.getYear()`/`getMonth()` — still callable, still deprecated since Java 1.1: `getYear()` returns years since 1900, a completely different number space from `LocalDate.getYear()`. Any legacy code still calling these instance methods (not `Calendar`) is a landmine during interop.
4. Instant has no calendar fields — `instant.plus(Period.ofMonths(1))` throws `UnsupportedTemporalTypeException`; must convert to `ZonedDateTime` first, since "a month" requires calendar context `Instant` doesn't have.
5. Duration on `LocalDate` — similarly throws `UnsupportedTemporalTypeException`; `LocalDate` has no sub-day precision for `Duration` to operate on. Use `Period` or `plusDays`.
6. DST gap (spring-forward) — constructing a `ZonedDateTime` at a local time that doesn't exist (e.g. 2:30 AM on a spring-forward day) doesn't throw — Java silently shifts it forward past the gap. Easy to not notice.
7. DST overlap (fall-back) — the same local time occurs twice; `ZonedDateTime.of(...)` silently picks the earlier offset by default unless you call `withEarlierOffsetAtOverlap()`/`withLaterOffsetAtOverlap()` explicitly.
8. `ZonedDateTime.equals()` vs `isEqual()` — `equals()` compares local date-time + offset + zone-id all together; two `ZonedDateTime`s representing the same instant in different zones are not `equals()`-equal. Use `isEqual()` (or compare `.toInstant()`) for true instant equality — directly analogous to the `BigDecimal` `equals`/`compareTo` gotcha.
9. Period not auto-normalized — `Period.of(1, 13, 0).getMonths()` returns 13, not 1; must call `.normalized()` explicitly.
10. Period direction is silently signed — `Period.between(later, earlier)` returns a negative Period without throwing; easy to pass arguments in the wrong order unnoticed.
11. Month-end clamping drift — `LocalDate.of(2024,1,31).plusMonths(1)` → 2024-02-29 (clamped to Feb's actual last day), not an "overflowed" March date. Chaining `plusMonths(1)` repeatedly from a 31st doesn't reliably land back on a 31st.
12. Pattern letter case sensitivity — `MM` = month, `mm` = minute; `yyyy` = calendar year, `YYYY` = ISO week-based year, which can silently differ by one near year boundaries (e.g., Dec 31 sometimes belongs to week 1 of the next ISO week-year). Mixing these up is a very common, very silent bug.
13. SimpleDateFormat not thread-safe — legacy formatter; sharing one static instance across threads corrupts output under concurrency. `DateTimeFormatter` instances are immutable and thread-safe — a real reason to migrate, not just a style preference.
14. `Date.toString()` implies a zone it doesn't really have — `Date` internally is just epoch-millis with no zone field at all; `toString()` renders it using the JVM's default zone, which misleads people into thinking the `Date` object itself carries zone information.
15. `Calendar.getInstance()` defaults to system zone/locale silently — a classic "works on my machine, breaks on the CI server" bug when the environment's default zone differs.
16. Leap seconds ignored by `Instant` (ties back to the corrected table row) — every day is modeled as exactly 86,400 seconds; this is a deliberate simplification (Java Time-Scale), not a bug, but it means `Instant` arithmetic is not millisecond-perfect against real-world UTC on the rare day a leap second is inserted.
17. Naive hour-math across DST instead of `Duration.between` — manually computing elapsed hours via wall-clock subtraction on zone-aware values gives a wrong answer across a DST boundary (a "24-hour day" might really be 23 or 25 hours); `Duration.between` on zone-aware types gets this right automatically.
18. Bridging Calendar↔java.time needs `GregorianCalendar` specifically — `Date` has direct `toInstant()`/`Date.from()` bridges, but generic `Calendar` doesn't convert as cleanly to `ZonedDateTime`; the bridge is `GregorianCalendar.from(zonedDateTime)` / `((GregorianCalendar) cal).toZonedDateTime()` — plain `Calendar` only has `toInstant()`, which loses the zone.

## Predefined ISO/RFC Constants

Static fields on `DateTimeFormatter`:

| Constant               | Format                                                      |
|------------------------|-------------------------------------------------------------|
| `BASIC_ISO_DATE`       | `20240315`                                                  |
| `ISO_LOCAL_DATE`       | `2024-03-15`                                                |
| `ISO_OFFSET_DATE`      | `2024-03-15+05:30`                                          |
| `ISO_DATE`             | Local or offset date, whichever fields are present          |
| `ISO_LOCAL_TIME`       | `14:30:00`                                                  |
| `ISO_OFFSET_TIME`      | `14:30:00+05:30`                                            |
| `ISO_TIME`             | Local or offset time                                        |
| `ISO_LOCAL_DATE_TIME`  | `2024-03-15T14:30:00`                                       |
| `ISO_OFFSET_DATE_TIME` | `2024-03-15T14:30:00+05:30`                                 |
| `ISO_ZONED_DATE_TIME`  | `2024-03-15T14:30:00+05:30[Asia/Kolkata]`                   |
| `ISO_DATE_TIME`        | Widest ISO variant, any of the above                        |
| `ISO_ORDINAL_DATE`     | `2024-075` (year + day-of-year)                             |
| `ISO_WEEK_DATE`        | `2024-W11-5` (ISO week-based date)                          |
| `ISO_INSTANT`          | `2024-03-15T09:00:00Z`                                      |
| `RFC_1123_DATE_TIME`   | `Fri, 15 Mar 2024 14:30:00 +0530` (HTTP header date format) |


---

## Period: June→July vs July→August — does it differ?
- No — `Period`'s representation doesn't differ, even though the actual elapsed days do. 
- `Period` normalizes to calendar units (years/months/days), not elapsed time, so "one calendar month" is treated identically regardless of how many actual days that month contains.

```java
Period.between(LocalDate.of(2024, 6, 15), LocalDate.of(2024, 7, 15)); // P1M  (30 actual days)
Period.between(LocalDate.of(2024, 7, 15), LocalDate.of(2024, 8, 15)); // P1M  (31 actual days)
```

### If you actually want total elapsed days, use ChronoUnit.DAYS
```java
long days1 = ChronoUnit.DAYS.between(LocalDate.of(2024, 6, 15), LocalDate.of(2024, 7, 15)); // 30
long days2 = ChronoUnit.DAYS.between(LocalDate.of(2024, 7, 15), LocalDate.of(2024, 8, 15)); // 31
```

### Duration, by contrast, measures exact elapsed time — so it does differ:
```java
Duration.between(Instant.parse("2024-06-15T00:00:00Z"), Instant.parse("2024-07-15T00:00:00Z")); // PT720H (30*24h)
Duration.between(Instant.parse("2024-07-15T00:00:00Z"), Instant.parse("2024-08-15T00:00:00Z")); // PT744H (31*24h)
```
These are genuinely different `Duration` values. This is the cleanest way to internalize the split: `Period` is calendar-unit-shaped and ignores actual duration; `Duration` is time-unit-shaped and is exact.

### More examples covering the full depth
**Period — month-end clamping trap:**
```java
Period.between(LocalDate.of(2024, 1, 31), LocalDate.of(2024, 2, 29)); // P29D, not P1M
// Feb 2024 (leap year) has no "31st" to land on, so the algorithm can't express
// a clean 1-month jump and falls back to counting days instead.
```

**Period — un-normalized construction:**
```java
Period p = Period.of(1, 13, 0);   // literally 1 year, 13 months, 0 days — NOT auto-normalized
p.getMonths();                    // 13 — still 13, not reduced to 1 year
p.normalized().getMonths();       // 1  — normalized() rolls excess months into years
p.toTotalMonths();                // 25 — flattens years+months into one long, ignoring days
p.normalized().getYears();        // 2  — normalized() rolls excess months into years
```

**Period — reversed direction is silently negative, not an exception:**
```java
Period.between(LocalDate.of(2024, 8, 15), LocalDate.of(2024, 7, 15)); // P-1M (negative, no throw)
```

**Duration — arithmetic and conversions:**
```java
Duration d = Duration.ofHours(25);
d.toDays();     // 1   (truncates, doesn't round)
d.toHours();    // 25
d.plusMinutes(30); // PT25H30M
d.multipliedBy(2); // PT50H
d.negated();    // PT-25H
d.isZero(); d.isNegative(); // false false
```

**Duration — the DST-aware elapsed-time case (why Duration beats naive hour math):**
```java
ZonedDateTime before = ZonedDateTime.of(2024, 3, 9, 0, 0, 0, 0, ZoneId.of("America/New_York"));
ZonedDateTime after  = ZonedDateTime.of(2024, 3, 10, 0, 0, 0, 0, ZoneId.of("America/New_York"));
Duration.between(before, after); // PT23H — spring-forward day is genuinely 23 hours, not 24
```

If you'd instead computed ChronoUnit.HOURS.between(...) assuming calendar-day × 24, or done naive wall-clock subtraction, you'd get the wrong answer across a DST boundary. Duration.between on zone-aware types always reflects true elapsed seconds.

### Which types each one works on:
- `Period.between(...)` — only `LocalDate` (calendar-field types).
- `Duration.between(...)` — anything with a time-of-day/instant concept: `Instant`, `LocalTime`, `LocalDateTime`, `ZonedDateTime`. Not bare `LocalDate`.
- Trying to add a `Period` to an `Instant`, or a `Duration` to a `LocalDate`, throws `UnsupportedTemporalTypeException` at runtime — a real trap.

---


## Decision Table — Which Type for Which Use Case

| Use case                                                                                                     | Type                                                                 | Why                                                                                                                                                                    |
|--------------------------------------------------------------------------------------------------------------|----------------------------------------------------------------------|------------------------------------------------------------------------------------------------------------------------------------------------------------------------|
| Account-opened date, birthday, anniversary                                                                   | `LocalDate`                                                          | Calendar date only — no time-of-day or zone is meaningful                                                                                                              |
| Daily recurring local event ("standup at 9:30 AM")                                                           | `LocalTime`                                                          | Time-of-day only, no specific date, no zone                                                                                                                            |
| A specific appointment on a calendar, no cross-zone concern                                                  | `LocalDateTime`                                                      | Date + time as a human would write it, zone irrelevant to the use case                                                                                                 |
| Audit/log timestamp compared or sorted across servers/zones                                                  | `Instant`                                                            | Unambiguous machine point in time — the right type for "when did this really happen"                                                                                   |
| User-facing appointment that must respect DST rules ("3 PM in Asia/Kolkata, even if DST rules change later") | `ZonedDateTime`                                                      | Carries a `ZoneId` with full DST-transition rules, not just a fixed offset                                                                                             |
| DB column of type `TIMESTAMP WITH TIME ZONE` / JDBC mapping                                                  | `OffsetDateTime`                                                     | JDBC 4.2+ spec recommends `OffsetDateTime` over `ZonedDateTime` for DB interop — a fixed offset is a simpler, storage-friendly contract than a named zone's rule table |
| Session/token time-to-live, elapsed processing time                                                          | `Duration`                                                           | Exact elapsed time in seconds/nanos — not a calendar concept                                                                                                           |
| Subscription/billing cycle length ("1 month", "1 year")                                                      | `Period`                                                             | Calendar-unit concept — "1 month" must mean the same thing regardless of which month                                                                                   |
| Passing a timestamp into legacy `java.util.Date`-based API                                                   | `Instant → Date.from(instant)`                                       | `Instant` is the direct, lossless bridge point to the old API                                                                                                          |
| Testable "current time" in business logic                                                                    | `Clock` injected, then `LocalDate.now(clock)` / `Instant.now(clock)` | Avoids hardcoding `now()` and makes time-dependent business logic testable                                                                                             |

---

# Full Method Tables by Class

## `LocalDate`

| Method                                                        | Purpose                                       |
|---------------------------------------------------------------|-----------------------------------------------|
| `now()`, `now(Clock)`                                         | Current date (system clock or injected clock) |
| `of(y, m, d)`, `parse(text)`, `parse(text, formatter)`        | Construction                                  |
| `getYear()`, `getMonthValue()`, `getMonth()`,                 | Field access                                  |
| `getDayOfMonth()`, `getDayOfWeek()`, `getDayOfYear()`         | Field access                                  |
| `lengthOfMonth()`, `lengthOfYear()`, `isLeapYear()`           | Calendar metadata                             |
| `plusDays/Weeks/Months/Years(n)`                              | Calendar arithmetic (immutable, returns new)  |
| `minusDays/Weeks/Months/Years(n)`                             | Calendar arithmetic (immutable, returns new)  |
| `withYear/Month/DayOfMonth(n)`                                | Field replacement                             |
| `isBefore()`, `isAfter()`, `isEqual()`, `compareTo()`         | Comparison                                    |
| `until(Temporal, ChronoUnit)`, `until(Temporal)` → `Period`   | Difference measurement                        |
| `atStartOfDay()`, `atStartOfDay(ZoneId)`, `atTime(LocalTime)` | Promote to `LocalDateTime` / `ZonedDateTime`  |
| `toEpochDay()`                                                | Days since epoch (`long`)                     |

## `LocalTime`

| Method                                                 | Purpose                |
|--------------------------------------------------------|------------------------|
| `now()`, `of(h, m, s, nano)`, `parse(text)`            | Construction           |
| `getHour()`, `getMinute()`, `getSecond()`, `getNano()` | Field access           |
| `plusHours/Minutes/Seconds/Nanos`, `minus...`          | Arithmetic             |
| `withHour/Minute/Second/Nano(n)`                       | Field replacement      |
| `isBefore()`, `isAfter()`                              | Comparison             |
| `toSecondOfDay()`                                      | Seconds since midnight |
| `MIDNIGHT`, `NOON`                                     | Constants              |

## `LocalDateTime`

| Method                                                          | Purpose                            |
|-----------------------------------------------------------------|------------------------------------|
| `now()`, `of(date, time)`, `of(y, m, d, h, min)`, `parse(text)` | Construction                       |
| `toLocalDate()`, `toLocalTime()`                                | Decomposition                      |
| `plus/minusDays/Hours/Minutes/...` (full unit set)              | Arithmetic                         |
| `withX(...)`                                                    | Field replacement                  |
| `atZone(ZoneId)`, `atOffset(ZoneOffset)`                        | Promote to zone/offset-aware types |
| `isBefore()`, `isAfter()`, `isEqual()`                          | Comparison                         |

## `Instant`

| Method                                          | Purpose                    |
|-------------------------------------------------|----------------------------|
| `now()`                                         | Construction               |
| `ofEpochSecond(s)`, `ofEpochSecond(s, nanoAdj)` | Construction               |
| `ofEpochMilli(ms)`, `parse(text)`               | Construction               |
| `getEpochSecond()`, `getNano()`                 | Field access               |
| `plusSeconds/Millis/Nanos`, `minus...`          | Arithmetic                 |
| `isBefore()`, `isAfter()`                       | Comparison                 |
| `toEpochMilli()`                                | Legacy-style millis value  |
| `atZone(ZoneId)`                                | Promote to `ZonedDateTime` |

## `ZonedDateTime`

| Method                             | Purpose                                                                     |
|------------------------------------|-----------------------------------------------------------------------------|
| `now()`,`of(LDT,ZoneId)`           | Construction                                                                |
| `parse(text)`                      | Construction                                                                |
| `toInstant()`, `toLocalDateTime()` | Decomposition                                                               |
| `toLocalDate()`, `toLocalTime()`   | Decomposition                                                               |
| `withZoneSameInstant(ZoneId)`      | Re-express same instant in another zone (changes wall-clock time)           |
| `withZoneSameLocal(ZoneId)`        | Keep same wall-clock numbers, reinterpret in another zone (changes instant) |
| `withLaterOffsetAtOverlap()`       | Resolve DST-overlap ambiguity explicitly                                    |
| `withEarlierOffsetAtOverlap()`     | Resolve DST-overlap ambiguity explicitly                                    |
| `getZone()`, `getOffset()`         | Zone/offset access                                                          |
| `plus/minus...`                    | Arithmetic                                                                  |

## `OffsetDateTime`

| Method                                   | Purpose                                                              |
|------------------------------------------|----------------------------------------------------------------------|
| `now()`, `of(LocalDateTime, ZoneOffset)` | Construction                                                         |
| `parse(text)`                            | Construction                                                         |
| `toInstant()`, `toZonedDateTime()`       | Conversion                                                           |
| `withOffsetSameLocal(ZoneOffset)`        | Same two variants as `ZonedDateTime`, but offset-only (no DST rules) |
| `withOffsetSameInstant(ZoneOffset)`      | Same two variants as `ZonedDateTime`, but offset-only (no DST rules) |

## `Duration`

| Method                                                            | Purpose                           |
|-------------------------------------------------------------------|-----------------------------------|
| `ofSeconds/Minutes/Hours/Days/Millis/Nanos(n)`, `between(t1, t2)` | Construction                      |
| `getSeconds()`, `getNano()`                                       | Raw component access              |
| `toDays/Hours/Minutes/Seconds/Millis/Nanos()`                     | Total-converted value (truncated) |
| `plus/minus(Duration)`, `plusSeconds/...`                         | Arithmetic                        |
| `multipliedBy(n)`, `dividedBy(n)`                                 | Scaling                           |
| `negated()`, `abs()`                                              | Sign operations                   |
| `isZero()`, `isNegative()`, `compareTo()`                         | Checks/comparison                 |

## `Period`

| Method                                                 | Purpose                                            |
|--------------------------------------------------------|----------------------------------------------------|
| `of(y,m,d)`,`ofYears/Months/Days(n)`,`between(d1, d2)` | Construction                                       |
| `getYears()`, `getMonths()`, `getDays()`               | Component access (not normalized)                  |
| `toTotalMonths()`                                      | Flattens years + months (ignores days) into `long` |
| `normalized()`                                         | Rolls excess months into years                     |
| `plus/minus(Period)`                                   | Arithmetic                                         |
| `isZero()`, `isNegative()`, `negated()`                | Sign checks                                        |

## `DateTimeFormatter`

| Method                                                 | Purpose                         |
|--------------------------------------------------------|---------------------------------|
| `ofPattern(pattern)`, `ofPattern(pattern, locale)`     | Custom pattern                  |
| `ofLocalizedDate/Time/DateTime(FormatStyle)`           | Locale-sensitive, non-hardcoded |
| `format(temporal)`                                     | Object → string                 |
| `parse(text)` (via `LocalDate.parse(text, fmt)`, etc.) | String → object                 |
| `withZone(ZoneId)`, `withLocale(Locale)`               | Derive a variant formatter      |

---

## ISO-8601 Duration Format
### The basic shape
```
P<date-part>T<time-part>
```

- **`P`** = "Period" designator. It marks the start of the duration string. Every ISO-8601 duration/period starts with `P`.
- **`T`** = "Time" separator. It marks the boundary between the date-based part and the time-based part. It only appears if there's a time component.

So `P` by itself handles years/months/days (date-based), and `T` introduces hours/minutes/seconds (time-based).

### `Period` — only uses the date part (Y, M, D)
`Period` represents **date-based** amounts: years, months, days. It never has a `T` section because it has no concept of hours/minutes/seconds.

| Code       | Meaning                   |
|------------|---------------------------|
| `P1Y`      | 1 year                    |
| `P2M`      | 2 months                  |
| `P10D`     | 10 days                   |
| `P1Y2M10D` | 1 year, 2 months, 10 days |
| `P0D`      | zero period               |

```java
Period p = Period.of(1, 2, 10);
System.out.println(p); // P1Y2M10D
```

### `Duration` — only uses the time part (H, M, S), but still keeps the `P`
`Duration` represents **time-based** amounts: hours, minutes, seconds, nanos. Since it's fundamentally about exact machine time, not calendar units, its string always has a `T` right after `P` (there's no date part at all, just `PT...`).

| Code       | Meaning                  |
|------------|--------------------------|
| `PT1H`     | 1 hour                   |
| `PT30M`    | 30 minutes               |
| `PT45S`    | 45 seconds               |
| `PT1H30M`  | 1 hour 30 minutes        |
| `PT0.500S` | 0.5 seconds (500 millis) |
| `PT-6H`    | negative 6 hours         |

```java
Duration d = Duration.ofMinutes(90);
System.out.println(d); // PT1H30M
```

Notice: **`M` means different things in each part.** In the date part (before `T`), `M` = months. In the time part (after `T`), `M` = minutes. The `T` is exactly what disambiguates them — that's its main job.

### Why the format matters
This is the standard output of `toString()` and the required input format for `Period.parse(...)` / `Duration.parse(...)`:

```java
Period.parse("P1Y2M10D");   // works
Duration.parse("PT1H30M");  // works
Duration.parse("P1H30M");   // throws DateTimeParseException — missing T!
```

If you forget the `T` before time units, parsing fails — a common gotcha.

### Quick reference table

| Symbol | Unit                                                               | Which part             |
|--------|--------------------------------------------------------------------|------------------------|
| `P`    | Period designator (always first)                                   | —                      |
| `Y`    | Years                                                              | date part              |
| `M`    | Months                                                             | date part (before `T`) |
| `W`    | Weeks (converted to days when parsed)                              | date part              |
| `D`    | Days                                                               | date part              |
| `T`    | Time separator                                                     | —                      |
| `H`    | Hours                                                              | time part              |
| `M`    | Minutes                                                            | time part (after `T`)  |
| `S`    | Seconds (can include fractional, e.g. `S` with decimals for nanos) | time part              |

### Mixed example (valid ISO-8601, but not directly a `Period` or `Duration`)
The full ISO-8601 duration spec technically allows combining both, e.g. `P1Y2M3DT4H5M6S` ("1 year, 2 months, 3 days, 4 hours, 5 minutes, 6 seconds"). Java splits this concept across two classes:
- `Period` only ever parses/produces the `P...` date-only form.
- `Duration` only ever parses/produces the `PT...` time-only form.

Neither class alone handles the fully combined string — if you need both calendar and clock granularity together, you typically keep a `Period` and a `Duration` side by side, or use something like `Period.plus` logic manually against a `LocalDateTime`.



