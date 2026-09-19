## Annotations
### @Basic
```java
@Basic(fetch = FetchType.EAGER, optional = true)
private String contactNumber;
```

This is actually the implicit default for every simple field (String, Long, int, enums, etc.) — you almost never write it explicitly. Every field that isn't @Transient, isn't a relationship, and isn't @Embedded gets @Basic applied implicitly by Hibernate.

Parameters:
- `fetch` — EAGER (default) or LAZY. Note: lazy loading of a @Basic field only actually works with bytecode enhancement turned on (Byte Buddy field-level interception at build time). Without bytecode enhancement enabled, @Basic(fetch = LAZY) is silently ignored and Hibernate loads it eagerly anyway — a classic "I set this and nothing changed" trap. 
- `optional` — true (default) means the field can be null, which affects the generated column's NOT NULL constraint during DDL generation, and also affects whether Hibernate treats a null as valid unset state vs a validation-worthy absence.

You'd write @Basic explicitly mainly to try lazy-loading a large CLOB/BLOB column (e.g., a big JSON blob or scanned document field) without pulling it into the initial SELECT.

### @Column
```java
@Column(
    name = "contact_number",
    nullable = false,
    unique = false,
    length = 20,
    precision = 10,
    scale = 2,
    insertable = true,
    updatable = true,
    columnDefinition = "VARCHAR(20)"
)
```

Parameters:
- `name` — DB column name. Defaults to the Java field name (Hibernate applies whatever NamingStrategy/PhysicalNamingStrategy is configured — commonly camelCase → snake_case). 
- `nullable` — DDL-generation hint (NOT NULL). Does not enforce anything at runtime by itself — Hibernate won't stop you from persist()-ing a null value; you'd get a DB-level constraint violation on flush, or you'd need Bean Validation (@NotNull) for an earlier runtime check.
- `unique` — same story, DDL-only, generates a unique constraint.
- `length` — for String/VARCHAR columns, default 255. 
- `precision` / `scale` — for BigDecimal/numeric columns. precision = total digits, scale = digits after decimal. Relevant for anything money-related — for a CardPayments amount field you'd want something like precision = 19, scale = 4 to avoid floating-point-style rounding disasters. 
- `insertable` / `updatable` — both default true. Setting insertable = false, updatable = false makes a field read-only from Hibernate's perspective — it's excluded from the generated INSERT/UPDATE SQL entirely. Common pattern: a column maintained by a DB trigger or generated column, or the "many" side's duplicate mapping of a value already managed elsewhere.
- `columnDefinition` — raw DDL fragment, bypasses Hibernate's type-to-SQL-type mapping entirely. Escape hatch for DB-specific types (e.g., Oracle NUMBER(1) for a boolean, or Postgres JSONB).

### @Enumerated 
```java
@Enumerated(EnumType.STRING)
private AccountType accountType;
```

Parameter: 
- `value` — `EnumType.ORDINAL` (default, stores the enum's positional index as an int) or `EnumType.STRING` (stores the enum constant's name as text).

**Why `ORDINAL` is a trap** : 
- if `AccountType` is { `IGCFD`, `IGSTK` } and someone later inserts a new type in the middle — say { `IGCFD`, `IGCRY`, `IGSTK`  } — every existing `IGSTK` row (ordinal 1) silently becomes `IGCRY` on read. 
- STRING is immune to this but costs more storage and a marginally slower comparison. 
- For a regulated platform where AccountType/AccountStatus values likely show up in audit trails or compliance reports too, STRING is close to mandatory in practice — self-documenting DB rows matter when someone's doing a manual DB query during an incident.

**Alternative** implementing AttributeConverter<AccountType, String> via @Convert gives you full control (e.g., mapping to a custom short code like "IGSTK" → "STK") instead of relying on either ordinal or the enum's literal name.

example:
```java
public enum AccountType {
    IG, CFT, IGSTK
}

// -------------------- //

@Converter(autoApply = true)
public class AccountTypeConverter implements AttributeConverter<AccountType, String> {

    @Override
    public String convertToDatabaseColumn(AccountType attribute) {
        if (attribute == null) {
            return null;
        }
        return switch (attribute) {
            case IG    -> "IG";
            case CFD   -> "CFD";
            case IGSTK -> "STK";   // custom short code, not the enum name
        };
    }

    @Override
    public AccountType convertToEntityAttribute(String dbData) {
        if (dbData == null) {
            return null;
        }
        return switch (dbData) {
            case "IG"  -> AccountType.IG;
            case "CFD" -> AccountType.CFD;
            case "STK" -> AccountType.IGSTK;
            default -> throw new IllegalArgumentException("Unknown account_type code: " + dbData);
        };
    }
}

// -------------------- //

@Entity
public class Account {

    @Id
    @GeneratedValue(strategy = GenerationType.IDENTITY)
    private Long id;

    // No @Enumerated here at all.
    // autoApply = true on the converter means this just works:
    @Column(name = "account_type", nullable = false)
    private AccountType accountType;

    @Enumerated(EnumType.STRING)
    @Column(name = "status", nullable = false)
    private AccountStatus status;   // still using STRING here, untouched

    @ManyToOne(fetch = FetchType.LAZY)
    @JoinColumn(name = "client_id")
    private Client client;
}

```

`autoApply`. If you set `autoApply = true` on `@Converter`, Hibernate scans all entity attributes of type `AccountType` and applies this converter automatically — no per-field `@Convert` needed, which is why `accountType` above has no annotation on it at all. If you leave it false, you'd need to be explicit:
```java
@Convert(converter = AccountTypeConverter.class)
@Column(name = "account_type")
private AccountType accountType;
```

### @Transient 
```java
@Transient
private String cachedDisplayLabel;
```

No parameters. 

Marks a field as non-persistent — Hibernate excludes it entirely from mapping.

- Internals detail worth having, at bootstrap, when Hibernate builds the property arrays that back dirty-checking (`propertyNames`, `propertyTypes`, the `EntityMetamodel`'s state-array indices), `@Transient` fields are filtered out before those arrays are built. 
- They're not "checked and ignored" — they don't occupy a slot at all. 
- Contrast this with `insertable = false`, `updatable = false` from `@Column`, where the property does get a slot and is dirty-checked, just excluded from SQL generation. 
- Two different exclusion mechanisms that look similar from the outside but operate at different layers.

>Gotcha: Java's own transient keyword (lowercase, for Serializable) is a completely different thing and does not exclude a field from JPA mapping. People conflate these because they read as the same word — Hibernate only respects @Transient (the annotation).

**Common use cases**: computed/derived fields, caches, or a field that only makes sense in-memory (e.g., a boolean isNewlyRegistered flag set during a request but never persisted).

### @Embeddable / @Embedded
```java
@Embeddable
public class Address {
    private String line1;
    private String city;
    private String postcode;
}

@Entity
public class Client {
    @Embedded
    @AttributeOverrides({
        @AttributeOverride(name = "postcode", column = @Column(name = "registered_postcode"))
    })
    private Address registeredAddress;
}
```

- `@Embeddable` — no parameters. Marks a class as a value type with no identity of its own and no separate table. Its fields get flattened into whatever entity embeds it.
- `@Embedded` — no parameters. Marks a field as an embedded value type. The field's type must be annotated with `@Embeddable`.
- `@AttributeOverrides`/`@AttributeOverride` — lets you rename columns per-usage. Useful if the same Address embeddable is used twice in one entity (e.g., registeredAddress and correspondenceAddress on Client) — without overrides, both would try to map to the same column names (line1, city, postcode) and collide.

Internals note: an embeddable has no EntityPersister of its own. Its fields are absorbed directly into the owning entity's persister at the same level as any other @Basic property — which also means an embeddable's fields participate in the owning entity's dirty-checking state array, not their own. If you mutate address.setCity(...) on a managed Client, that's dirty-checked as part of Client's state, exactly like changing contactNumber directly.

Composite key variant — `@EmbeddedId` uses this same mechanism but for the primary key:
```java
@Embeddable
public class AccountKey implements Serializable {
    private Long clientId;
    private String accountType;
}

@Entity
public class Account {
    @EmbeddedId
    private AccountKey id;
}
```

### @ManyToOne
```java
@ManyToOne(
    fetch = FetchType.LAZY,
    optional = false,
    cascade = {},
    targetEntity = Client.class
)
@JoinColumn(name = "client_id", nullable = false)
private Client client;
```

This is the owning side of the Client–Account relationship — the side whose table actually holds the foreign key column. That's not a convention, it's structural: whichever side has `@JoinColumn` (not `mappedBy`) is the one Hibernate consults when generating `INSERT`/`UPDATE` SQL for the FK.

Parameters:
- `fetch` — default is EAGER for @ManyToOne (this trips people up — @OneToMany/@ManyToMany default to LAZY, but @ManyToOne/@OneToOne default to EAGER). In practice almost everyone overrides this to LAZY explicitly, because eager-by-default on the "many" side is exactly how N+1 query storms happen silently.
- `optional` — default true. optional = false tells Hibernate the association can't be null, which lets it use an inner join instead of an outer join when fetching, and generates NOT NULL on the FK column at DDL time. 
- `cascade` — array of CascadeType values. Rare on the @ManyToOne side in practice — cascading from child to parent (e.g., saving an Account cascades to save its Client) is usually not what you want. 
- `targetEntity` — only needed if the field type is an interface or otherwise ambiguous; Hibernate infers it from the field's declared type otherwise.


- when `fetch = LAZY`, Hibernate doesn't set client to the real loaded `Client` — it sets it to a proxy holding just the FK value.
- The proxy's method interceptor triggers `EntityPersister.load()` on first access of any field other than the ID. This is why `account.getClient().getId()` doesn't trigger a query (ID is already known from the FK) but `account.getClient().getName()` does.