# EF Core Review Guide

A structured refresher and study map for developers who want to review Entity Framework Core, or who need a keyword map before going deeper.

The guide starts with the mental model and moves through modeling, querying, change tracking, transactions, performance, diagnostics, security, and advanced features. Each section is labeled so you can skip ahead.

**Assumed baseline:** EF Core 8 or later on a relational provider (SQL Server examples are used where the API is provider-specific). Features that arrived later are marked **EF Core 10+**.

---

## How to use this guide

| If you are… | Start here | Goal |
| --- | --- | --- |
| New to EF Core | [Mental model](#1-ef-core-mental-model) → [Study order](#2-recommended-study-order) → sections 3–9 | Write correct CRUD and simple queries |
| Returning after a break | [Decision guide](#33-quick-decision-guide) and skim sections 6, 9, 11, 21 | Recover the rules you forget under pressure |
| Reviewing a pull request | [Performance checklist](#32-performance-checklist) and [Security](#20-multi-tenant-security) | Spot tracking, N+1, filter, and concurrency mistakes |
| Going deeper | [Keywords](#34-important-keywords-for-further-study) | Search official docs with the right terms |

Official reference: [EF Core documentation](https://learn.microsoft.com/en-us/ef/core/).

---

## Contents

1. [EF Core mental model](#1-ef-core-mental-model)
2. [Recommended study order](#2-recommended-study-order)
3. [DbContext and unit of work](#3-dbcontext-and-unit-of-work)
4. [Entity configuration](#4-entity-configuration)
5. [CRUD operations](#5-crud-operations)
6. [Tracking and no-tracking queries](#6-tracking-and-no-tracking-queries)
7. [Queries and projections](#7-queries-and-projections)
8. [What LINQ can and cannot translate](#8-what-linq-can-and-cannot-translate)
9. [Relationships and loading](#9-relationships-and-loading)
10. [Split queries and Cartesian explosion](#10-split-queries-and-cartesian-explosion)
11. [Change-tracking states](#11-change-tracking-states)
12. [Concurrency tokens](#12-concurrency-tokens)
13. [Sequences](#13-sequences)
14. [Owned types and complex types](#14-owned-types-and-complex-types)
15. [Value semantics](#15-value-semantics)
16. [Value converters](#16-value-converters)
17. [Value comparers](#17-value-comparers)
18. [JSON columns](#18-json-columns)
19. [Global query filters](#19-global-query-filters)
20. [Multi-tenant security](#20-multi-tenant-security)
21. [Set-based updates and deletes](#21-set-based-updates-and-deletes)
22. [Transactions](#22-transactions)
23. [Execution strategies and retries](#23-execution-strategies-and-retries)
24. [Interceptors](#24-interceptors)
25. [What belongs in an interceptor](#25-what-belongs-in-an-interceptor)
26. [Diagnostics and logging](#26-diagnostics-and-logging)
27. [Compiled queries](#27-compiled-queries)
28. [Context pooling](#28-context-pooling)
29. [Migrations](#29-migrations)
30. [Testing](#30-testing)
31. [Seeding and raw SQL](#31-seeding-and-raw-sql)
32. [Performance checklist](#32-performance-checklist)
33. [Quick decision guide](#33-quick-decision-guide)
34. [Keywords for further study](#34-important-keywords-for-further-study)
35. [Practical rules](#35-final-practical-rules)

---

## 1. EF Core mental model

**Level:** beginner

Entity Framework Core is an object-relational mapper (ORM) for .NET. It maps application objects to relational tables and LINQ queries to SQL.

| Application | Database |
| --- | --- |
| Entity class | Table |
| Property | Column |
| Navigation property | Relationship |
| LINQ query | SQL query |
| `DbContext` | Unit of work / session |
| Migration | Schema change script |

A minimal model:

```csharp
public class Product
{
    public int Id { get; set; }
    public string Name { get; set; } = "";
    public decimal Price { get; set; }
}

public class AppDbContext : DbContext
{
    public AppDbContext(DbContextOptions<AppDbContext> options)
        : base(options)
    {
    }

    public DbSet<Product> Products => Set<Product>();

    protected override void OnModelCreating(ModelBuilder modelBuilder)
    {
        modelBuilder.ApplyConfigurationsFromAssembly(typeof(AppDbContext).Assembly);
    }
}
```

A typical update:

```csharp
var product = await db.Products.SingleAsync(x => x.Id == productId);
product.Price = 25;
await db.SaveChangesAsync();
```

What EF Core does:

1. Executes SQL to load the product.
2. Tracks the entity.
3. Detects the changed property.
4. Generates an `UPDATE` for changed columns only.
5. Sends it inside a transaction.
6. Updates the tracked entity state to `Unchanged`.

EF Core is not a substitute for understanding the database. Generated SQL, indexes, and execution plans still decide whether a query is fast.

---

## 2. Recommended study order

### Beginner

1. `DbContext` and lifetime
2. `DbSet` and entity classes
3. CRUD and `SaveChanges`
4. LINQ filters, ordering, pagination
5. Tracking and `AsNoTracking`
6. Relationships and `Include`
7. Migrations

### Intermediate

1. Fluent configuration (`IEntityTypeConfiguration`)
2. Projections into DTOs
3. Loading strategies and split queries
4. Transactions and isolation
5. Concurrency tokens
6. Value converters and comparers
7. Global query filters
8. `ExecuteUpdate` / `ExecuteDelete`
9. Integration tests against a real database

### Advanced

1. Owned types and complex types
2. JSON columns
3. Interceptors
4. Diagnostics, query tags, and plans
5. Execution strategies
6. Compiled queries and compiled models
7. Multi-tenancy and row-level security
8. Context pooling
9. Provider-specific features (sequences, `rowversion`, JSON functions)

---

## 3. `DbContext` and unit of work

**Level:** beginner

A `DbContext` is a short-lived unit of work. It holds tracked entities, the model, and the connection/command pipeline for one piece of work.

Typical web-request lifetime:

```text
Request begins
  Create DbContext          (scoped)
  Query or modify entities
  SaveChanges
  Dispose DbContext
Request ends
```

Register it as scoped:

```csharp
services.AddDbContext<AppDbContext>(options =>
    options.UseSqlServer(connectionString));
```

A context is:

- Not thread-safe
- Intended to be short-lived
- Responsible for change tracking
- Responsible for database interaction
- Normally scoped to one request, command, or job

Do not:

- Store a `DbContext` in a singleton
- Use one context across threads
- Keep one context for the whole application
- Reuse one context for a long-running workflow
- Inject `DbContext` into a singleton service (inject `IDbContextFactory<T>` instead and create a context per operation)

`DbContext` implements `IDisposable` / `IAsyncDisposable`. In ASP.NET Core the container disposes the scoped instance. Elsewhere, dispose it yourself (`await using`).

---

## 4. Entity configuration

**Level:** intermediate

Prefer data annotations for trivial constraints. Prefer Fluent API configuration classes once a model is non-trivial. Configuration classes keep persistence details out of the domain type and scale better across a large model.

```csharp
public sealed class ProductConfiguration : IEntityTypeConfiguration<Product>
{
    public void Configure(EntityTypeBuilder<Product> builder)
    {
        builder.ToTable("Products");
        builder.HasKey(x => x.Id);

        builder.Property(x => x.Name)
            .HasMaxLength(200)
            .IsRequired();

        builder.Property(x => x.Price)
            .HasPrecision(18, 2);

        builder.HasIndex(x => x.Name);
    }
}
```

Register every configuration in the assembly:

```csharp
protected override void OnModelCreating(ModelBuilder modelBuilder)
{
    modelBuilder.ApplyConfigurationsFromAssembly(typeof(AppDbContext).Assembly);
}
```

Configuration you will meet often:

- Primary keys, alternate keys, composite keys
- Required and optional properties
- Maximum lengths and decimal precision
- Indexes and unique constraints
- Foreign keys and delete behavior
- Default values and computed columns
- Concurrency tokens
- Value converters and value comparers
- Global query filters
- Table names and schemas
- Inheritance (TPH, TPT, TPC)
- Owned types, complex types, and JSON mapping

Unique index:

```csharp
builder.HasIndex(x => x.Email).IsUnique();
```

Tenant-scoped unique index:

```csharp
builder.HasIndex(x => new { x.TenantId, x.Email }).IsUnique();
```

Delete behavior matters. `Cascade` is convenient and dangerous across aggregates. `Restrict` or `ClientSetNull` is often safer for relationships you want the application to handle explicitly. Review every cascade in a migration before it ships.

---

## 5. CRUD operations

**Level:** beginner

### Create

```csharp
var product = new Product { Name = "Keyboard", Price = 100 };
db.Products.Add(product);
await db.SaveChangesAsync();
// product.Id is populated if the database generated it
```

### Read

```csharp
var product = await db.Products.SingleOrDefaultAsync(x => x.Id == id);
```

Pick the operator on purpose:

| Method | No row | Many rows |
| --- | --- | --- |
| `SingleAsync` | throws | throws |
| `SingleOrDefaultAsync` | `null` | throws |
| `FirstAsync` | throws | returns first |
| `FirstOrDefaultAsync` | `null` | returns first |
| `FindAsync(id)` | `null` | n/a (key lookup, checks tracker first) |

Use `Single` when the predicate should match at most one row. That turns a data bug into an exception instead of a silent wrong row.

### Update

```csharp
var product = await db.Products.SingleAsync(x => x.Id == id);
product.Price = 120;
await db.SaveChangesAsync();
```

### Delete

```csharp
var product = await db.Products.SingleAsync(x => x.Id == id);
db.Products.Remove(product);
await db.SaveChangesAsync();
```

One `SaveChanges` batches pending inserts, updates, and deletes in one transaction. For bulk changes, prefer [set-based operations](#21-set-based-updates-and-deletes) instead of loading every row.

---

## 6. Tracking and no-tracking queries

**Level:** beginner

By default, queries track entities:

```csharp
var product = await db.Products.SingleAsync(x => x.Id == id);
product.Price = 150;
await db.SaveChangesAsync(); // EF detects the change
```

For read-only work, turn tracking off:

```csharp
var products = await db.Products
    .AsNoTracking()
    .ToListAsync();
```

No-tracking queries use less memory and skip change-tracker work. They are the right default for API responses, read-only screens, and reports.

| Use tracking when | Use no tracking when |
| --- | --- |
| You will modify the entity and call `SaveChanges` | You are returning a DTO or read model |
| You need identity resolution in one graph | You are running a report |
| You rely on relationship fix-up | The query is a projection |

`AsNoTrackingWithIdentityResolution()` is the middle ground: no change tracking, but the same entity instance is reused when it appears more than once in the result. Use it when a no-tracking graph contains repeated references and you still want one object per key.

Projections (`Select` into a non-entity type) are not tracked. You do not need `AsNoTracking` on a DTO projection, though it does no harm.

---

## 7. Queries and projections

**Level:** beginner → intermediate

### Filtering and ordering

```csharp
var products = await db.Products
    .Where(x => x.Price >= 100)
    .OrderByDescending(x => x.Price)
    .ThenBy(x => x.Name)
    .ToListAsync();
```

Always order before paging. Unordered `Skip`/`Take` is not stable.

```csharp
var page = await db.Products
    .AsNoTracking()
    .OrderBy(x => x.Id)
    .Skip((pageNumber - 1) * pageSize)
    .Take(pageSize)
    .ToListAsync();
```

Offset paging (`Skip`/`Take`) gets slower on deep pages. For large feeds, study **keyset pagination** (`WHERE Id > @lastId ORDER BY Id`).

### Projection

Projection selects only what the caller needs:

```csharp
var products = await db.Products
    .AsNoTracking()
    .Select(x => new ProductDto
    {
        Id = x.Id,
        Name = x.Name,
        Price = x.Price
    })
    .ToListAsync();
```

Projection can:

- Reduce transferred columns
- Avoid tracking
- Avoid loading whole object graphs
- Produce simpler SQL
- Match an API response shape directly

Aggregation in the projection stays in SQL:

```csharp
var summaries = await db.Orders
    .Select(x => new OrderSummary
    {
        Id = x.Id,
        CustomerName = x.Customer.Name,
        Total = x.Lines.Sum(line => line.Quantity * line.UnitPrice)
    })
    .ToListAsync();
```

If a projection cannot be translated, EF Core may throw, or (in older versions / with client evaluation enabled) silently finish the work in memory. Treat client evaluation of a large query as a bug.

---

## 8. What LINQ can and cannot translate

**Level:** intermediate

EF Core translates expression trees into SQL. It cannot translate arbitrary .NET methods.

This fails, because `IsEligible` is a delegate, not an expression EF can open:

```csharp
bool IsEligible(User user) => user.IsActive && user.Age >= 18;

var users = await db.Users.Where(IsEligible).ToListAsync();
```

Write the predicate inline, or pass an expression:

```csharp
Expression<Func<User, bool>> eligible = x => x.IsActive && x.Age >= 18;

var users = await db.Users.Where(eligible).ToListAsync();
```

If the rule truly cannot be translated, filter in the database as far as you can, then finish in memory:

```csharp
var users = await db.Users
    .Where(x => x.IsActive)
    .AsNoTracking()
    .ToListAsync();

var eligible = users.Where(IsEligible).ToList();
```

Do not materialize a large table just to run application code. Push filters, projections, and aggregations into SQL.

Reusable shapes that translate well: `Expression<Func<...>>`, `IQueryable` extension methods that return `IQueryable`, and specification objects that expose an expression. Plain `Func<T, bool>` runs in memory only.

---

## 9. Relationships and loading

**Level:** beginner → intermediate

### Eager loading

```csharp
var blogs = await db.Blogs
    .Include(x => x.Posts)
    .ThenInclude(x => x.Author)
    .ToListAsync();
```

`Include` loads entity graphs. It is the right tool when you will mutate those entities. It is often the wrong tool for an API read model.

### Explicit loading

```csharp
var blog = await db.Blogs.SingleAsync(x => x.Id == id);

await db.Entry(blog)
    .Collection(x => x.Posts)
    .LoadAsync();
```

Useful when the decision to load depends on earlier application logic. Easy to turn into N+1 if called inside a loop.

### Lazy loading

Lazy loading loads a navigation when code first touches it. It requires proxies or the `ILazyLoader` pattern, and it hides queries:

```csharp
foreach (var blog in blogs)
{
    // May execute one query per blog
    Console.WriteLine(blog.Posts.Count);
}
```

That is the N+1 problem. Prefer eager loading or a projection unless you have measured a case where lazy loading is safer.

### Projection (usually best for reads)

```csharp
var blogs = await db.Blogs
    .Select(x => new BlogDto
    {
        Id = x.Id,
        Name = x.Name,
        PostCount = x.Posts.Count
    })
    .ToListAsync();
```

### Relationship keywords

`HasOne`, `HasMany`, `WithOne`, `WithMany`, `HasForeignKey`, `DeleteBehavior`, skip navigations (many-to-many without an explicit join entity). Configure the foreign key explicitly when the convention would guess wrong.

---

## 10. Split queries and Cartesian explosion

**Level:** intermediate

Loading several collections with `Include` joins them into one result set:

```csharp
var blogs = await db.Blogs
    .Include(x => x.Posts)
    .Include(x => x.Tags)
    .ToListAsync();
```

If a blog has 10 posts and 5 tags, one join can produce 50 intermediate rows for that blog, repeating parent columns. That is a Cartesian explosion.

```csharp
var blogs = await db.Blogs
    .Include(x => x.Posts)
    .Include(x => x.Tags)
    .AsSplitQuery()
    .ToListAsync();
```

EF Core then runs one query for the roots and one query per collection include.

| Prefer a single query when | Prefer `AsSplitQuery()` when |
| --- | --- |
| The graph is small | You include more than one collection |
| One round trip matters more | The join duplicates large columns |
| You need a single consistent snapshot | The result shows a Cartesian explosion |

Split queries are not one atomic read. Rows can change between the separate statements unless you wrap them in a transaction with a suitable isolation level. For read models, a projection is often better than either include style:

```csharp
var result = await db.Blogs
    .Select(x => new BlogSummary
    {
        Id = x.Id,
        Name = x.Name,
        PostCount = x.Posts.Count
    })
    .ToListAsync();
```

`AsSingleQuery()` forces the join strategy. The default is single-query unless you configure split queries globally.

---

## 11. Change-tracking states

**Level:** intermediate

| State | Meaning |
| --- | --- |
| `Detached` | Not tracked |
| `Unchanged` | Tracked, matches the database snapshot |
| `Added` | Will be inserted |
| `Modified` | Will be updated |
| `Deleted` | Will be deleted |

Inspect the tracker:

```csharp
foreach (var entry in db.ChangeTracker.Entries())
{
    Console.WriteLine($"{entry.Entity.GetType().Name} - {entry.State}");
}
```

You can set state yourself:

```csharp
db.Entry(entity).State = EntityState.Modified;
```

Be careful with disconnected graphs:

```csharp
db.Update(clientEntity);
```

`Update` marks every property modified. A client can then overwrite columns it was never allowed to change. Prefer:

1. Load the existing row.
2. Copy only allowed fields.
3. `SaveChanges`.

If you must attach a disconnected entity, set modified properties explicitly (`entry.Property(x => x.Price).IsModified = true`) and ignore server-owned fields such as concurrency tokens you did not just read.

---

## 12. Concurrency tokens

**Level:** intermediate

A concurrency token stops a second writer from silently overwriting a first writer's change.

```text
User A reads Product
User B reads Product
User A updates Product
User B updates the stale Product  -> conflict, not a silent overwrite
```

### SQL Server `rowversion`

```csharp
public class Product
{
    public int Id { get; set; }
    public decimal Price { get; set; }
    public byte[] Version { get; set; } = [];
}
```

```csharp
builder.Property(x => x.Version).IsRowVersion();
```

`IsRowVersion()` already configures the property as database-generated on add and update, and as a concurrency token. Adding `.IsConcurrencyToken()` is redundant.

EF Core emits SQL shaped like:

```sql
UPDATE Products
SET Price = @price
WHERE Id = @id AND Version = @originalVersion;
```

If zero rows are updated:

```csharp
try
{
    await db.SaveChangesAsync();
}
catch (DbUpdateConcurrencyException ex)
{
    // Reload, merge, retry, or return a conflict to the caller.
}
```

### Custom concurrency token

Use `IsConcurrencyToken()` for an application-managed value:

```csharp
builder.Property(x => x.Version)
    .IsConcurrencyToken()
    .ValueGeneratedNever();
```

```csharp
entity.Version = Guid.NewGuid();
```

| API | Use for |
| --- | --- |
| `IsRowVersion()` | SQL Server `rowversion` / `timestamp`, generated by the database |
| `IsConcurrencyToken()` | `Guid`, integer versions, application timestamps, other providers |

Sending the original token back from the client is part of the contract. If the client can omit it, you do not have concurrency control.

---

## 13. Sequences

**Level:** advanced · provider-specific

A sequence is a database object that generates numbers independently of a table.

```csharp
modelBuilder.HasSequence<int>("OrderNumbers")
    .StartsAt(1000)
    .IncrementsBy(1);

modelBuilder.Entity<Order>()
    .Property(x => x.OrderNumber)
    .HasDefaultValueSql("NEXT VALUE FOR OrderNumbers");
```

| | Identity column | Sequence |
| --- | --- | --- |
| Scope | Usually one column | Independent object |
| Reuse across tables | No | Yes |
| Typical use | Surrogate keys | Human-readable numbers, shared counters |

Sequence SQL is provider-specific. Gaps are normal (rolled-back transactions still consume values). Do not use a sequence if the business rule is "no gaps."

---

## 14. Owned types and complex types

**Level:** advanced

Both map a nested object. They are not the same thing.

### Owned type

An owned type is a dependent entity with no independent identity exposed to you.

```csharp
public class Order
{
    public int Id { get; set; }
    public Address ShippingAddress { get; set; } = new();
}

public class Address
{
    public string Street { get; set; } = "";
    public string City { get; set; } = "";
}
```

```csharp
builder.OwnsOne(x => x.ShippingAddress);
```

By default, columns land on the owner table: `ShippingAddress_Street`, `ShippingAddress_City`. Owned types can also be mapped to a separate table. They can have their own configuration, indexes, and (owned) relationships.

### Complex type

**EF Core 8+.** A complex type is not an entity. It has no key and no identity.

```csharp
public class Customer
{
    public int Id { get; set; }
    public Name Name { get; set; } = new();
}

public class Name
{
    public string First { get; set; } = "";
    public string Last { get; set; } = "";
}
```

```csharp
builder.ComplexProperty(x => x.Name);
```

Use a complex type when the object is a group of values: no id, no independent lifecycle, never queried on its own.

| | Owned type | Complex type |
| --- | --- | --- |
| Identity | Dependent entity | None |
| Key | Internal dependent key | No key |
| Independent lifecycle | No | No |
| Separate table | Yes | No |
| Best fit | Dependent object that still needs entity configuration | Value object |

---

## 15. Value semantics

**Level:** intermediate

Value semantics means two instances are the same when their contents are the same.

```csharp
var first = new Money(10, "USD");
var second = new Money(10, "USD");
// same value, even if they are different objects
```

Typical value objects: money, address, email, coordinates, date range, person name.

Entities use identity semantics. Customer 1 and Customer 2 are different even if every property matches.

Complex types fit value objects because they have no identity. Equality is still your job on the CLR type (`record`, `IEquatable<T>`). EF's change tracking also needs a [value comparer](#17-value-comparers) if the default structural comparison is wrong.

---

## 16. Value converters

**Level:** intermediate

A value converter maps a CLR value to the column value.

### Enum as string

```csharp
builder.Property(x => x.Status).HasConversion<string>();
```

The application uses `OrderStatus.Paid`. The database stores `"Paid"`. Strings survive enum reordering. Integers are smaller and fragile if members are renumbered.

### Strongly typed id

```csharp
public readonly record struct OrderId(Guid Value);

var converter = new ValueConverter<OrderId, Guid>(
    id => id.Value,
    value => new OrderId(value));

builder.Property(x => x.Id).HasConversion(converter);
```

Converters are also used for encrypted columns, custom date representations, and domain types. Conversion runs in the application, so the database sees the provider type. Indexes and filters still work, but database expressions must use the stored form.

---

## 17. Value comparers

**Level:** advanced

A value comparer tells EF how to detect changes. The default comparer is wrong for many mutable collections and some converted values: it compares references, not contents.

```csharp
var comparer = new ValueComparer<List<string>>(
    (a, b) => a!.SequenceEqual(b!),
    value => value.Aggregate(0, (hash, item) => HashCode.Combine(hash, item.GetHashCode())),
    value => value.ToList());

builder.Property(x => x.Tags)
    .HasConversion(
        tags => JsonSerializer.Serialize(tags, (JsonSerializerOptions?)null),
        json => JsonSerializer.Deserialize<List<string>>(json, (JsonSerializerOptions?)null) ?? new List<string>())
    .Metadata.SetValueComparer(comparer);
```

Without a comparer, mutating `Tags` in place may not be detected, or a no-op replacement may be detected. Prefer primitive collections (`List<string>` mapped as a JSON collection in EF Core 8+) when the provider supports them, and still confirm snapshot behavior in a test.

---

## 18. JSON columns

**Level:** advanced · provider-specific

JSON columns store a structured document in one column.

```csharp
builder.OwnsOne(x => x.Details, details =>
{
    details.ToJson();
    details.OwnsOne(x => x.Dimensions);
});
```

```csharp
var redProducts = await db.Products
    .Where(x => x.Details.Color == "Red")
    .ToListAsync();
```

| Use JSON when | Prefer columns or tables when |
| --- | --- |
| The payload is document-like | Nested fields are filtered or sorted constantly |
| Shape changes often | You need foreign keys |
| The object is loaded as one unit | Rows are edited independently |
| Relational constraints are not required | You need strong integrity |

JSON query support depends on the provider (SQL Server JSON functions, PostgreSQL `jsonb`, and so on). A filter on a JSON property can be a scan unless you add a computed column or an expression index.

---

## 19. Global query filters

**Level:** intermediate

A global filter is added to every query for that entity, including queries reached through a navigation.

```csharp
builder.HasQueryFilter(x => !x.IsDeleted);
```

Common uses: soft delete, tenant isolation, "active only" flags.

### Named filters (EF Core 10+)

Before EF Core 10, an entity had one filter. A second `HasQueryFilter` call replaced the first. Named filters compose, and you can ignore one without ignoring the others.

```csharp
builder.HasQueryFilter("SoftDelete", x => !x.IsDeleted);
builder.HasQueryFilter("Tenant", x => x.TenantId == tenantId);
```

```csharp
var deletedOrders = await db.Orders
    .IgnoreQueryFilters(["SoftDelete"])
    .ToListAsync();
```

Rules:

- Do not mix named and unnamed filters on the same entity. Model building throws.
- `IgnoreQueryFilters()` with no arguments disables every filter.
- On EF Core 9 and earlier, combine predicates with `&&` and accept that you cannot turn one off independently.

Required navigations and filters interact badly: a required relationship to a filtered-out row can make the parent disappear from a query. Prefer optional navigations, or filter explicitly, when a filter can hide the target.

Treat `IgnoreQueryFilters()` as privileged. In a multi-tenant system it is a data leak unless another boundary still applies.

---

## 20. Multi-tenant security

**Level:** advanced

A query filter is one layer, not the security boundary.

```csharp
builder.HasQueryFilter(x =>
    tenantContext.TenantId != null &&
    x.TenantId == tenantContext.TenantId);
```

Also do all of this:

1. **Trusted tenant identity.** Derive the tenant from authenticated claims, a validated token, server-side membership, or a trusted host mapping. Never from a body field or query string alone.
2. **Authorization.** Check that this user may perform this operation in this tenant.
3. **Least-privilege database credentials.**
4. **Database row-level security** where the engine supports it, so a missing filter is not a full breach.
5. **Tenant-aware unique indexes** so two tenants can share an email without colliding, and one tenant cannot.
6. **Explicit tenant context in background jobs.** A job with no tenant must not mean "all tenants."
7. **Fail closed.**

Avoid:

```csharp
x => tenantId == null || x.TenantId == tenantId
```

If `tenantId` is accidentally null, that predicate matches every row. Require a tenant before building the query, and throw if it is missing.

---

## 21. Set-based updates and deletes

**Level:** intermediate · EF Core 7+

### Tracked update

```csharp
var user = await db.Users.SingleAsync(x => x.Id == id);
user.IsActive = false;
user.UpdatedAt = DateTime.UtcNow;
await db.SaveChangesAsync();
```

Tracked updates load entities, run setters, participate in interceptors, and honor concurrency tokens. Use them for one or a few entities, or when domain logic must run.

### Set-based update

```csharp
await db.Users
    .Where(x => !x.IsActive)
    .ExecuteUpdateAsync(setters => setters
        .SetProperty(x => x.UpdatedAt, DateTime.UtcNow));
```

### Set-based delete

```csharp
await db.Orders
    .Where(x => x.Status == OrderStatus.Expired && x.CreatedAt < cutoff)
    .ExecuteDeleteAsync();
```

Set-based commands:

- Do not load entities
- Do not use the change tracker
- Do not call property setters
- Do not call `SaveChanges`
- Do not run `SaveChanges` interceptors
- Leave already-tracked instances stale

Do not keep using a tracked entity after a set-based update of the same row. Reload it, or do not track it in that context.

---

## 22. Transactions

**Level:** intermediate

A transaction makes several database operations atomic.

```csharp
await using var transaction = await db.Database.BeginTransactionAsync();
try
{
    db.Orders.Add(order);
    await db.SaveChangesAsync();

    db.OutboxMessages.Add(new OutboxMessage
    {
        Type = "OrderCreated",
        Payload = $"Order {order.Id}"
    });
    await db.SaveChangesAsync();

    await transaction.CommitAsync();
}
catch
{
    await transaction.RollbackAsync();
    throw;
}
```

A single `SaveChanges` is already transactional. You need an explicit transaction when:

- Several `SaveChanges` calls must commit together
- EF commands and raw SQL must be atomic
- Several contexts enlist in one transaction (prefer one context if you can)

### Isolation

```csharp
await using var transaction = await db.Database.BeginTransactionAsync(
    IsolationLevel.Serializable);
```

Higher isolation reduces anomalies and increases locking, blocking, and deadlocks. The usual default (`ReadCommitted` on SQL Server) is right until you have a concrete anomaly to prevent. Serializable workflows should be retried.

### Outbox

Writing the business row and an outbox message in the same transaction is the standard way to publish an integration event without a dual write. A separate processor publishes the message. Handlers should be idempotent, because delivery is at-least-once.

---

## 23. Execution strategies and retries

**Level:** advanced

An execution strategy retries transient failures. It is not a transaction.

```csharp
options.UseSqlServer(connectionString, sql => sql.EnableRetryOnFailure(
    maxRetryCount: 5,
    maxRetryDelay: TimeSpan.FromSeconds(10),
    errorNumbersToAdd: null));
```

Transient failures include brief network drops, cloud failovers, throttling, and some deadlocks.

| | Transaction | Execution strategy |
| --- | --- | --- |
| Purpose | Commit or roll back together | Retry a temporary failure |
| Retries work | No | Yes |
| Implies the other | No | No |

User-created transactions are not retried automatically. Retry the whole delegate:

```csharp
var strategy = db.Database.CreateExecutionStrategy();

await strategy.ExecuteAsync(async () =>
{
    await using var transaction = await db.Database.BeginTransactionAsync();
    try
    {
        db.Orders.Add(order);
        await db.SaveChangesAsync();

        db.OutboxMessages.Add(message);
        await db.SaveChangesAsync();

        await transaction.CommitAsync();
    }
    catch
    {
        await transaction.RollbackAsync();
        throw;
    }
});
```

Do not charge a card, send an email, or call another service inside the retried delegate unless that call is idempotent. A retry can run it twice.

---

## 24. Interceptors

**Level:** advanced

Interceptors observe or change EF operations. Common types: `DbCommandInterceptor`, `SaveChangesInterceptor`, `DbConnectionInterceptor`, `DbTransactionInterceptor`.

Register them once, ideally as singletons with no per-request fields:

```csharp
options.AddInterceptors(new AuditInterceptor());
```

### Command timing

Measure after the command finishes. `ReaderExecutingAsync` runs *before* execution, so a stopwatch there does not include the query.

```csharp
public sealed class CommandTimingInterceptor : DbCommandInterceptor
{
    public override ValueTask<DbDataReader> ReaderExecutedAsync(
        DbCommand command,
        CommandExecutedEventData eventData,
        DbDataReader result,
        CancellationToken cancellationToken = default)
    {
        Console.WriteLine($"{eventData.Duration.TotalMilliseconds:0} ms");
        return base.ReaderExecutedAsync(command, eventData, result, cancellationToken);
    }
}
```

### Auditing on save

```csharp
public sealed class AuditInterceptor : SaveChangesInterceptor
{
    public override ValueTask<InterceptionResult<int>> SavingChangesAsync(
        DbContextEventData eventData,
        InterceptionResult<int> result,
        CancellationToken cancellationToken = default)
    {
        var context = eventData.Context;
        if (context is not null)
        {
            foreach (var entry in context.ChangeTracker.Entries())
            {
                // Set CreatedAt / UpdatedAt, or write an audit row.
            }
        }

        return base.SavingChangesAsync(eventData, result, cancellationToken);
    }
}
```

Good uses: auditing, timestamps, current-user columns, command timing, command tags, persistence-wide validation, OpenTelemetry enrichment.

`ExecuteUpdate` and `ExecuteDelete` do not go through `SaveChanges`, so a `SaveChangesInterceptor` does not see them.

---

## 25. What belongs in an interceptor

**Level:** advanced

Interceptors fit cross-cutting persistence rules: set `CreatedAt`, stamp a correlation id, measure SQL, reject an empty tenant.

They are a poor place for business workflows:

```text
When an order is paid:
  charge the card
  send email
  publish a domain event
```

Those belong in an application service, command handler, or explicit event handler. Interceptors are implicit. Hidden side effects are hard to test and surprising during a retry.

---

## 26. Diagnostics and logging

**Level:** intermediate

### SQL logging

```csharp
optionsBuilder
    .LogTo(Console.WriteLine, LogLevel.Information)
    .EnableDetailedErrors();
```

```csharp
optionsBuilder.EnableSensitiveDataLogging();
```

Sensitive logging prints parameter values. Use it only in a controlled development environment. Parameters can contain passwords, tokens, and personal data.

### `ToQueryString`

```csharp
var sql = db.Products
    .Where(x => x.Price > 100)
    .Select(x => new { x.Id, x.Name })
    .ToQueryString();
```

This shows SQL without executing the query. Parameter rendering is not identical to what the server receives, but it is enough to spot joins, missing filters, and client-evaluation surprises.

### Query tags

```csharp
var users = await db.Users
    .TagWith("Admin dashboard - active users")
    .Where(x => x.IsActive)
    .ToListAsync();
```

Tags show up in the SQL text, which makes them searchable in database monitoring.

### Plans, not just SQL

Look at the execution plan for table scans, missing indexes, implicit conversions, large sorts, bad cardinality estimates, and unexpected residual predicates. A pretty LINQ query can still scan.

EF Core logging categories and `DiagnosticListener` / `ActivitySource` integrate with OpenTelemetry. That is the production path; `LogTo(Console.WriteLine)` is for local investigation.

---

## 27. Compiled queries

**Level:** advanced

EF Core already caches translation by query shape. A compiled query skips extra work for a hot shape:

```csharp
private static readonly Func<AppDbContext, int, Task<User?>> FindUser =
    EF.CompileAsyncQuery((AppDbContext db, int id) =>
        db.Users.SingleOrDefault(x => x.Id == id));

var user = await FindUser(db, userId);
```

Compiled queries do not fix missing indexes, bad joins, N+1 queries, lock contention, or network latency. Profile first.

**Compiled models** (`dotnet ef dbcontext optimize`) are a different feature: they prebuild the model to shorten startup. Useful for large models and serverless cold starts. They are not a query-performance tool.

---

## 28. Context pooling

**Level:** advanced

```csharp
services.AddDbContextPool<AppDbContext>(options =>
    options.UseSqlServer(connectionString));
```

Pooling reuses context instances across requests. That cuts allocation. It also means a context is not a fresh object.

For code that creates contexts itself (workers, background jobs), use `IDbContextFactory<T>` or `AddPooledDbContextFactory<T>`.

Request-specific state that must be reset on every lease:

- Tenant id
- Current user id
- Correlation id
- Command timeout overrides
- Mutable interceptor fields
- Session settings (`SET` options)

Fail closed when the tenant is missing, and set it every time the context is taken:

```csharp
public AppDbContext Create()
{
    var db = factory.CreateDbContext();
    db.TenantId = tenantContext.TenantId
        ?? throw new InvalidOperationException("Tenant is missing.");
    return db;
}
```

Do not store request state in fields of a shared interceptor. Test that a second lease does not see the first lease's tenant.

---

## 29. Migrations

**Level:** beginner → intermediate

```bash
dotnet ef migrations add AddProducts
dotnet ef database update
dotnet ef migrations script
dotnet ef migrations script --idempotent
```

Practices that avoid production incidents:

- Read the generated `Up` and `Down` before committing.
- Test on a copy of production-like data, including nulls and large tables.
- Split destructive changes (add column, backfill, switch readers, drop old column).
- Do not edit a migration that has already been applied in a shared environment.
- Keep migrations in source control.
- Deploy with an idempotent script or a controlled migrator, not an ad-hoc laptop.
- Be careful with `Migrate()` at application startup when you run more than one instance.

Data motion belongs in SQL inside the migration when possible, so it is transactional with the schema change and reviewable.

---

## 30. Testing

**Level:** intermediate

| Test type | Cover | Do not use it for |
| --- | --- | --- |
| Unit | Domain rules, value objects, handlers with a fake | SQL translation |
| Integration | Mappings, constraints, transactions, concurrency, filters | Replacing the domain tests |

The EF Core InMemory provider is not a relational database. It will not catch foreign keys, transactions, null semantics, translation failures, or provider functions. SQLite in-memory is closer and still not SQL Server or PostgreSQL. Use the real provider in a container (Testcontainers is the usual choice) for tests that claim to cover database behavior.

Worth asserting:

- Tenant isolation, including a missing tenant
- Soft-delete filter, and the privileged path that ignores it
- Pagination order
- Concurrency conflict on a stale token
- Query count, so an N+1 does not sneak in

---

## 31. Seeding and raw SQL

**Level:** intermediate

### Seeding

`HasData` is for reference data that belongs in migrations. It generates `InsertData` / `UpdateData` operations and tracks changes by primary key. It is awkward for large or environment-specific data.

For dev and test data, a dedicated seeder that runs after migrate is easier to reason about. Make it idempotent.

### Raw SQL

```csharp
var orders = await db.Orders
    .FromSqlInterpolated($"SELECT * FROM Orders WHERE Status = {status}")
    .ToListAsync();
```

Use interpolation / `FromSqlInterpolated` so values become parameters. Do not concatenate user input into SQL.

`FromSql` composes with LINQ, but the SQL must be a composable query on that entity (correct columns, no extra projection unless you map a keyless type). Non-query commands go through `ExecuteSqlInterpolatedAsync`. Raw SQL bypasses some model conventions. Keep it narrow and tested.

---

## 32. Performance checklist

**Level:** all

Before changing code:

1. Measure the query (duration, rows, round trips).
2. Read the generated SQL (`ToQueryString`, logs, or a profiler).
3. Read the execution plan.
4. Check indexes against the actual predicate and order.
5. Check for N+1 (a loop that queries).

Rules that are usually right:

- Project only required columns.
- `AsNoTracking()` for read-only entity queries.
- No lazy loading in loops.
- Deterministic `OrderBy` before `Skip`/`Take`.
- Indexes that match real filters, not every column.
- `ExecuteUpdate` / `ExecuteDelete` for bulk work.
- Avoid large `Include` graphs; split or project.
- Keep contexts short-lived.
- Compiled queries only after a profile says translation is the cost.
- Benchmark with realistic data volume.

---

## 33. Quick decision guide

| Question | Default answer |
| --- | --- |
| Tracking? | Yes if you will `SaveChanges`. `AsNoTracking()` for read-only entities. |
| `Include`? | When you need a tracked graph. Projection for a read model. |
| Split query? | When several collection includes explode into a Cartesian product. |
| `ExecuteUpdate`? | Bulk changes that do not need setters, interceptors, or domain logic. |
| Interceptor? | Cross-cutting persistence and diagnostics. Not business workflows. |
| Explicit transaction? | When multiple `SaveChanges` or raw commands must be atomic. One `SaveChanges` is already a transaction. |
| Compiled query? | Only after profiling shows compilation cost. |
| Complex type? | Value object, no identity, EF Core 8+. |
| Owned type? | Dependent object that may need its own table or entity configuration. |
| JSON column? | Document-shaped data you usually load as a unit. |
| Context pooling? | When allocation shows up in a profile, and every lease resets request state. |
| Ignore a query filter? | Privileged, named, and still authorized. Never "because it was easier." |

---

## 34. Important keywords for further study

### Fundamentals

`DbContext`, `DbSet`, `ChangeTracker`, `EntityState`, `SaveChanges`, `AsNoTracking`, `AsNoTrackingWithIdentityResolution`, `IDbContextFactory`

### Modeling

`IEntityTypeConfiguration`, `ModelBuilder`, Fluent API, data annotations, owned entities, complex types, value objects, TPH / TPT / TPC, table splitting, JSON mapping, shadow properties, backing fields, `HasData`

### Relationships

`HasOne`, `HasMany`, `WithOne`, `WithMany`, `HasForeignKey`, `DeleteBehavior`, cascade delete, many-to-many, skip navigations

### Querying

LINQ translation, expression trees, projection, `Include`, `ThenInclude`, split queries, keyset pagination, `GroupBy` translation, compiled queries, compiled models

### Updates

Change tracking, concurrency tokens, `rowversion`, `ExecuteUpdate`, `ExecuteDelete`, disconnected entities, store-generated values

### Transactions

`BeginTransaction`, `IsolationLevel`, execution strategy, retry, savepoints, `TransactionScope`, outbox, idempotency

### Performance

Generated SQL, `ToQueryString`, query tags, execution plans, indexes, N+1, Cartesian explosion, context pooling, connection pooling

### Extensibility

`SaveChangesInterceptor`, `DbCommandInterceptor`, `DbTransactionInterceptor`, `DiagnosticListener`, `ActivitySource`, OpenTelemetry

### Security

Global query filters, named query filters, multi-tenancy, row-level security, fail-closed design, tenant-aware indexes

---

## 35. Final practical rules

1. Keep `DbContext` short-lived and scoped. Never singleton, never shared across threads.
2. Put non-trivial mapping in `IEntityTypeConfiguration` classes.
3. Project only what you need.
4. Use no-tracking queries for read-only entities.
5. Read generated SQL for any query that matters.
6. Read the execution plan before adding hints or rewriting LINQ.
7. Use concurrency tokens on data that more than one writer can edit.
8. Use `IsRowVersion()` for SQL Server `rowversion`. Use `IsConcurrencyToken()` for application-managed versions.
9. A single `SaveChanges` is already a transaction. Open an explicit one only when you need a wider boundary.
10. Retry the whole transaction delegate, and keep external side effects out of it unless they are idempotent.
11. Use interceptors for cross-cutting persistence concerns, not for business workflows.
12. Global filters are one security layer. Fail closed if the tenant is missing.
13. Treat `IgnoreQueryFilters()` as privileged. Prefer named filters (EF Core 10+) so you can disable one predicate.
14. Use `AsSplitQuery()` for large multi-collection includes, and remember the reads are not one snapshot.
15. Use `ExecuteUpdate` / `ExecuteDelete` for bulk changes, and do not trust tracked instances afterward.
16. Use complex types for value objects and owned types for dependent entities.
17. Supply a value comparer for mutable converted values.
18. Do not compile queries or pool contexts until you have a measurement and a reset story.
19. Test database behavior on the real provider.
20. Optimize only after measuring.
