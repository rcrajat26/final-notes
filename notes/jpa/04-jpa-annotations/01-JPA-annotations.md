## Annotations
### @Entity
```java
@Entity(name = "Client")
public class Client { ... }
```

Parameters:
- `name` — the entity name used in JPQL/HQL queries. Defaults to the simple class name if omitted. If you have two `Client` classes in different packages, you'd disambiguate here. This is not the table name — that's `@Table`.

**What it actually does**: marks the class for Hibernate's bootstrap-time metadata scan. During `SessionFactory` construction, Hibernate's `MetadataSources` picks up every `@Entity`-annotated class, builds a `PersistentClass` model for each, and eventually compiles that into the `EntityPersister`. No `@Entity` → no persister → Hibernate has no idea the class exists, even if every other annotation is present.

Hard requirements (Hibernate will throw at startup if violated):
- Must have a no-arg constructor (public or protected — Byte Buddy proxy generation needs it)
- Must not be final (proxying again — a final class can't be subclassed for lazy loading)
- Must have exactly one @Id (or @EmbeddedId/@IdClass for composite keys)

### @Table
```java
@Table(
    name = "client",
    schema = "trading",
    catalog = "ig_db",
    uniqueConstraints = {
        @UniqueConstraint(name = "uk_client_email", columnNames = "email")
    },
    indexes = {
        @Index(name = "idx_client_status", columnList = "status")
    }
)
```
Parameters:
- `name` — actual DB table name. Defaults to the entity name if omitted entirely. 
- `schema` / `catalog` — for multi-schema DBs (you'd use this if trading and payments schemas coexist in one database). 
- `uniqueConstraints` — generates `ALTER TABLE ... ADD CONSTRAINT` in DDL, and matters for `hbm2ddl.auto` schema generation. Doesn't affect runtime behavior if you're not using Hibernate to generate schema. 
- `indexes` — same story, DDL-generation only. In a real setup you're managing indexes via Flyway/Liquibase anyway, so these two are mostly documentation unless you lean on `hibernate.hbm2ddl.auto=update`.

**Optional annotation** — if omitted, table name = entity name, no schema/catalog override, default DB collation/constraints.

### @Id
```java
@Id
private Long id;
```

No parameters. 

- Marks the primary key field. 
- This is the field that makes an object's identity independent of its Java equals()/hashCode() — two Client references with the same @Id value loaded in the same session are the same object reference, guaranteed by the identity map.

- Can be placed on a field or a getter — placement determines access type (FIELD vs PROPERTY). 
- If @Id is on the field, Hibernate reads/writes all persistent state via reflection on fields, bypassing your getters/setters entirely for every property in that entity.

### @GeneratedValue
```java
@GeneratedValue(strategy = GenerationType.IDENTITY)

@GeneratedValue(strategy = GenerationType.SEQUENCE, generator = "client_seq")
@SequenceGenerator(name = "client_seq", sequenceName = "client_id_seq", allocationSize = 50)

@GeneratedValue(strategy = GenerationType.TABLE, generator = "client_gen")
@TableGenerator(name = "client_gen", table = "id_gen", pkColumnValue = "client", allocationSize = 50)

@GeneratedValue(strategy = GenerationType.AUTO)
```

| Strategy | Mechanism | Batch insert friendly? |
|---|---|---|
| `IDENTITY` | DB auto-increment column | **No** — Hibernate must `INSERT` immediately to get the ID back, which disables JDBC batching for inserts on that entity |
| `SEQUENCE` | DB sequence object, pre-fetched in chunks (`allocationSize`) | **Yes** — IDs allocated in memory ahead of the actual insert |
| `TABLE` | A table simulating a sequence, row-locked per allocation | Yes, but slower — extra `SELECT ... FOR UPDATE` round-trip |
| `AUTO` | Hibernate picks based on the dialect (usually `SEQUENCE` for Postgres/Oracle, `IDENTITY` for MySQL) | Depends on resolution |

- **`generator`** — a name pointing at a `@SequenceGenerator` or `@TableGenerator` bean defined elsewhere on the class. Without it, Hibernate uses dialect defaults.
- **`@SequenceGenerator` params:** `name`, `sequenceName`, `initialValue`, `allocationSize` (default 50 — this is the classic gotcha: if your Oracle DB sequence itself increments by 1 but Hibernate's `allocationSize` is 50, you get gaps of 49 because Hibernate grabs a batch of IDs into memory per round trip).