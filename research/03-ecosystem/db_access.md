# Database access patterns for Indent: a comprehensive design guide

A new programming language targeting backend developers must provide **type-safe, performant, and ergonomic** database access. Analysis of Go's database/sql, Rust's sqlx, Python's asyncpg, and TypeScript's Prisma/Drizzle reveals that the most successful implementations share three key traits: compile-time safety, async-first design, and built-in connection pooling. This report synthesizes best practices from these implementations into concrete recommendations for Indent's database layer.

## Connection pooling should be built-in with safe defaults

The most critical lesson from analyzing existing implementations is that **connection pooling must be built-in, not optional**. Go's database/sql gets this right—the `sql.DB` object _is_ the pool—while Drizzle's delegation to underlying drivers creates configuration confusion. However, Go's default of unlimited connections is dangerous; production incidents commonly stem from exhausted database connections.

**Recommended default configuration** based on HikariCP's battle-tested formula and cross-implementation analysis:

| Setting | Recommended Default | Rationale |
|---------|---------------------|-----------|
| `max_connections` | 10 | HikariCP default; `cores * 2 + 1` for tuning |
| `min_connections` | 0 | Lazy creation; set equal to max for fixed pools |
| `acquire_timeout` | 30 seconds | SQLx, HikariCP consensus |
| `idle_timeout` | 10 minutes | Prevents stale connections |
| `max_lifetime` | 30 minutes | Forces connection recycling |
| `test_on_checkout` | true | SQLx default; prevents bad connections |

**Connection checkout should use RAII semantics**. Both Rust's sqlx and Python's asyncpg demonstrate that context-manager patterns eliminate connection leaks:

```
// Recommended Indent API
with db.connection() as conn:
    conn.execute("UPDATE accounts SET balance = balance - ?", amount)
// Connection automatically returned to pool

// For explicit control
conn = db.acquire()
defer conn.release()
```

For **multiple database connections**, provide a named registry pattern rather than forcing users to manage separate pool instances:

```
databases = DatabaseRegistry({
    "primary": Pool.create(url: PRIMARY_URL),
    "replica": Pool.create(url: REPLICA_URL, readonly: true),
    "analytics": Pool.create(url: ANALYTICS_URL, max_connections: 30),
})

// Access by name with type safety
users = databases["primary"].query("SELECT * FROM users")
```

## Type-safe queries require a hybrid approach

The query pattern landscape divides into three tiers: raw SQL (maximum control), query builders (type safety with SQL familiarity), and ORMs (productivity with abstraction costs). **Indent should support all three** while making the type-safe middle tier the default experience.

### How existing implementations achieve type safety

**Rust sqlx uses compile-time database verification**. The `query!` macro connects to the database during compilation, validates SQL syntax and types against the actual schema, then generates anonymous structs matching result columns. This catches errors before runtime but requires a database connection (or cached metadata) at build time.

**Prisma generates a fully-typed client from schema files**. The `.prisma` schema file defines models declaratively; `prisma generate` produces TypeScript types for all operations. Type safety comes from TypeScript's compiler validating against generated types.

**Drizzle infers types from TypeScript schema definitions**. No code generation needed—schema defined as TypeScript objects, and the type system propagates through query builder chains. Instant feedback when schema changes, but no database verification.

**JOOQ generates Java classes from database introspection**. Like Prisma but database-first: connect to existing database, generate typed table/column constants, use those constants in type-safe DSL queries.

### Recommended query approach for Indent

Provide three abstraction levels users can choose based on their needs:

```
// Level 1: Compile-time verified raw SQL (like sqlx)
users = sql!("SELECT id, name FROM users WHERE age > ?", min_age)
         .fetch_all(&pool)
// Compiler verifies SQL at build time, generates typed result

// Level 2: Type-safe query builder (like Drizzle/JOOQ)
users = db.select()
          .from(Users)
          .where(Users.age > min_age)
          .fetch_all()
// IDE autocomplete, refactoring support, type inference

// Level 3: ORM-style for simple CRUD
user = db.users.find(id: 42)
user.name = "Updated"
db.users.save(user)
```

**Dynamic queries require careful design**. Type-safe systems struggle with runtime-constructed WHERE clauses. The recommended pattern uses conditional predicate building:

```
conditions = []
if name_filter:
    conditions.append(Users.name == name_filter)
if age_filter:
    conditions.append(Users.age > age_filter)

result = db.select().from(Users).where(and(...conditions))
```

For maximum flexibility, support parameterized NULL checks:

```sql
SELECT * FROM users 
WHERE ($1::text IS NULL OR name = $1)
  AND ($2::int IS NULL OR age > $2)
```

**Prepared statement caching** should happen at the driver level, per-connection. HikariCP explicitly recommends against pool-level caching because prepared statements are bound to specific connections. Use LRU eviction with configurable cache size (default 256 statements).

## NULL handling needs first-class language support

Go's `sql.NullString` wrapper pattern is universally criticized as verbose. Rust's `Option<T>` is the gold standard—nullable columns map to `Option<T>`, the compiler enforces handling. **Indent should adopt Option<T> semantics** with compile-time nullability inference from database schema.

```
// Schema: name VARCHAR NOT NULL, bio VARCHAR NULL
struct User {
    name: String,      // Non-nullable column
    bio: Option<String> // Nullable column
}

// Compiler error if mapping mismatch
let user = sql!("SELECT name, bio FROM users").fetch_one()
// user.bio is Option<String>, must handle None case
```

**Type mapping recommendations** based on cross-language analysis:

| Database Type | Indent Type | Notes |
|---------------|-------------|-------|
| INTEGER, INT4 | i32 | |
| BIGINT, INT8 | i64 | |
| VARCHAR, TEXT | String | |
| TIMESTAMP WITH TZ | DateTime<Utc> | Require timezone-aware type |
| NUMERIC, DECIMAL | Decimal | Never use float for money |
| BOOLEAN | bool | |
| UUID | Uuid | First-class stdlib support |
| JSONB | Json<T> | Generic over contained type |
| ARRAY | Vec<T> | Native array mapping |
| ENUM | enum | Direct mapping to language enums |

**Row-to-struct mapping should use derive macros** (compile-time) rather than runtime reflection:

```
#[derive(FromRow)]
struct User {
    id: i64,
    #[column("full_name")]  // Rename column
    name: String,
    #[default]              // Use default if missing
    location: Option<String>,
    #[flatten]              // Flatten nested struct
    address: Address,
}
```

This generates efficient code at compile time while preserving type safety. Runtime reflection (Go's sqlx) is more flexible but loses compile-time guarantees and incurs performance overhead.

## Transaction API should enforce connection affinity through types

The fundamental challenge with async transactions is **connection affinity**—all operations in a transaction must use the same database connection. Rust sqlx solves this elegantly: the `Transaction` type holds an exclusive reference to the connection, making it impossible to accidentally use a different connection.

**Recommended transaction API for Indent**:

```
// Callback-based (preferred for safety)
result = db.transaction(|tx| {
    tx.execute("UPDATE accounts SET balance = balance - ? WHERE id = ?", amount, from_id)
    tx.execute("UPDATE accounts SET balance = balance + ? WHERE id = ?", amount, to_id)
    return TransferResult { from_id, to_id, amount }
})
// Auto-commit on return, auto-rollback on panic/error

// With options
result = db.transaction(
    isolation: Serializable,
    readonly: false,
    timeout: 30.seconds,
    |tx| { ... }
)

// Manual control when needed
tx = db.begin()
defer tx.rollback()  // No-op if committed
tx.execute(...)
tx.commit()
```

**Savepoints must be supported** for composable transactions. Prisma's lack of savepoint support is frequently criticized:

```
db.transaction(|tx| {
    tx.execute("INSERT INTO orders ...")
    
    // Nested begin creates savepoint
    tx.savepoint("items", || {
        tx.execute("INSERT INTO items ...")
        // If this fails, order still exists
    })
    
    tx.execute("INSERT INTO logs ...")
})
```

**Isolation levels** should be explicit, not hidden behind defaults:

| Level | Use Case |
|-------|----------|
| `ReadCommitted` | Default; prevents dirty reads |
| `RepeatableRead` | Consistent reads within transaction |
| `Serializable` | Full isolation; use for financial operations |

## Driver interface should be minimal with optional extensions

Go's driver interface evolved over time, adding optional interfaces (`Pinger`, `SessionResetter`, `Validator`) that complicate implementations. **Start minimal** with a clear extension mechanism.

**Core driver trait**:

```
trait Driver {
    fn connect(config: ConnectOptions) -> Result<Connection>
}

trait Connection {
    fn execute(query: &str, params: &[Value]) -> Result<QueryResult>
    fn prepare(query: &str) -> Result<Statement>
    fn begin() -> Result<Transaction>
    fn close()
}

trait Statement {
    fn execute(params: &[Value]) -> Result<QueryResult>
    fn fetch_one(params: &[Value]) -> Result<Row>
    fn fetch_all(params: &[Value]) -> Result<Vec<Row>>
}
```

**Optional extension traits** for advanced capabilities:

```
trait Pinger {
    fn ping() -> Result<()>  // Connection health check
}

trait SessionResetter {
    fn reset_session() -> Result<()>  // Clean state before pool reuse
}

trait StreamingFetch {
    fn fetch_stream(query: &str) -> Stream<Row>  // Row-by-row streaming
}
```

**Connection configuration should support both URL and structured forms**:

```
// URL form (convenient)
Pool.create("postgres://user:pass@host:5432/db?sslmode=verify-full")

// Structured form (explicit)
Pool.create(ConnectOptions {
    host: "host",
    port: 5432,
    username: "user",
    password: "pass",
    database: "db",
    ssl_mode: VerifyFull,
    ssl_root_cert: "/path/to/ca.pem",
})
```

**TLS/SSL handling** should follow PostgreSQL's standard modes: `disable`, `prefer`, `require`, `verify-ca`, `verify-full`. Default to `prefer` for development convenience, recommend `verify-full` for production.

## Migration tooling belongs outside stdlib

Every analyzed implementation keeps migrations in external tools (golang-migrate, sqlx-cli, Alembic, Prisma Migrate). This is correct—migration workflows vary significantly across teams, and stdlib updates are slower than ecosystem iteration.

**Recommended approach**:
- **External CLI** for migration creation, application, rollback
- **Library support** for embedding migrations in binaries (like sqlx's `migrate!` macro)
- **Hybrid schema model**: declarative schema definition generating imperative SQL migrations

```
// Schema definition (declarative)
model User {
    id: Int @id @autoincrement
    email: String @unique
    posts: Post[]
}

// Generated migration (imperative, customizable)
-- 20260117_create_users.up.sql
CREATE TABLE users (
    id SERIAL PRIMARY KEY,
    email VARCHAR(255) NOT NULL UNIQUE
);

-- 20260117_create_users.down.sql
DROP TABLE users;
```

**Compile-time schema validation** (like sqlx) should be optional but supported:

```
// Development: connect to DATABASE_URL at compile time
// CI: use cached schema metadata (indent db prepare)
let users = sql!("SELECT * FROM users WHERE id = ?", id)
// Compiler verifies table/column existence and types
```

## Database priorities should focus on PostgreSQL, SQLite, and MySQL

Based on adoption patterns and feature sets, **prioritize in this order**:

1. **PostgreSQL**: Most SQL-compliant (160/179 SQL:2011 features), richest type system (JSON, arrays, ranges, custom types), growing rapidly, reference implementation for standards
2. **SQLite**: Essential for testing, embedded use cases, zero-config development
3. **MySQL/MariaDB**: Market share, web application deployment ubiquity

**NoSQL databases should have separate interfaces**. Attempting to unify SQL and document/key-value stores under one abstraction loses the strengths of both. Common patterns (connection pooling, async primitives) can be shared, but query interfaces should be purpose-built.

**Drivers should live in the ecosystem, not stdlib**:
- Define standard interfaces in stdlib
- Let community implement drivers
- Provide reference implementation for PostgreSQL
- Include connection pooling in stdlib (not driver responsibility)

## Complete API proposal summary

### Pool configuration

```
pool = Pool.create(
    url: "postgres://...",
    max_connections: 10,
    min_connections: 0,
    acquire_timeout: 30.seconds,
    idle_timeout: 10.minutes,
    max_lifetime: 30.minutes,
    test_on_checkout: true,
    on_connect: |conn| conn.execute("SET timezone = 'UTC'"),
)
```

### Query execution

```
// Compile-time verified
users = sql!("SELECT id, name FROM users WHERE age > ?", 18)
         .fetch_all(&pool)

// Query builder
users = pool.select()
            .from(Users)
            .where(Users.age > 18)
            .order_by(Users.name.asc())
            .fetch_all()

// Simple ORM
user = pool.users.find(42)
```

### Transactions

```
pool.transaction(|tx| {
    tx.execute("UPDATE ...")
    tx.savepoint("nested", || {
        tx.execute("INSERT ...")
    })
    tx.commit()
})
```

### Type mapping

```
#[derive(FromRow)]
struct User {
    id: i64,
    name: String,
    bio: Option<String>,        // Nullable
    metadata: Json<Metadata>,   // JSONB
    tags: Vec<String>,          // Array
    status: UserStatus,         // Enum
}
```

The design philosophy throughout should be: **make safe things easy, make unsafe things explicit, never hide SQL**. Backend developers chose SQL for its power—Indent's database layer should enhance that power with type safety and ergonomics, not abstract it away.