# Understanding Repositories in Domain-Driven Design with Go

_A reading note for Chapter 12 of Implementing Domain-Driven Design by Vaughn Vernon._

---

## 1. The Missing Link Between Domain and Data

Many engineers see a Repository as nothing more than a rebranded DAO (Data Access Object)—but Vaughn Vernon disagrees.

Vernon defines repositories as bridges between domain and data layers, simulating in-memory aggregate collections.

Repositories are part of the domain model, separating domain logic from persistence infrastructure. The repository interface lives in the domain layer, while its implementation sits inside infrastructure. They let you write expressive domain code, such as:

```go
tenant.Activate()
tenantRepo.Save(ctx, tenant)
```

Aggregates seem in-memory despite being persisted.

---

## 2. What a Repository Really Represents

Eric Evans originally defined a repository as:

> “A Repository mediates between the domain and data-mapping layers, giving the illusion of an in-memory collection of all objects of a given type.”  
> — Evans, Domain-Driven Design (2003)

Vernon describes the pattern in the same spirit and extends the idea:

- Provide repositories only for aggregate roots.
- Hide technical concerns such as persistence details.
- Accept and return domain objects, never raw database records.

In Go, this looks like:

```go
type TenantRepository interface {
    Add(ctx context.Context, tenant *Tenant) error
    ByID(ctx context.Context, id TenantID) (*Tenant, error)
    Remove(ctx context.Context, id TenantID) error
}
```

A simple interface defines the aggregate lifecycle.

---

## 3. Two Repository Styles

### 3.1 Collection-Oriented

This approach uses the collection-illusion model. You treat aggregates as if all of them were loaded in memory, while a Unit of Work quietly tracks their state changes:

```go
uow.Add(ctx, tenant)
tenant.RegisterUser("jane@example.com")
uow.Commit()
```

Frameworks detect dirty objects—common in Java and .NET, rare in Go. Collection-oriented repositories depend on change-tracking plus the surrounding Unit of Work to spot those updates. If Go ever had an ORM with state tracking, this is the model it would follow.

### 3.2 Persistence-Oriented

Go developers typically adopt this explicit style—each repository method performs direct reads and writes against MongoDB, Redis, or Postgres, with no hidden state:

```go
type TenantRepository interface {
    Save(ctx context.Context, tenant *Tenant) error
    Delete(ctx context.Context, id TenantID) error
    ByID(ctx context.Context, id TenantID) (*Tenant, error)
}
```

This explicit approach suits Go’s design and avoids hidden transactions.

---

## 4. Example: Tenant and User Aggregates

In Vernon’s example, a Tenant aggregate root manages its collection of User entities within a single boundary:

```go
type Tenant struct {
    ID       TenantID
    Name     string
    IsActive bool
    Users    []*User
}

func (t *Tenant) Activate() { t.IsActive = true }

func (t *Tenant) RegisterUser(email string) *User {
    u := &User{Email: email, TenantID: t.ID}
    t.Users = append(t.Users, u)
    return u
}
```

Each Tenant uses one repository. User data is always accessed through it:

```go
tenant, _ := tenantRepo.ByID(ctx, tenantID)
tenant.RegisterUser("amy@example.com")
tenantRepo.Save(ctx, tenant)
```

No UserRepository exists—the Tenant aggregate manages persistence.

---

## 5. How DAO and Repository Differ

| Aspect  | DAO                | Repository                                      |
|---------|--------------------|-------------------------------------------------|
| Focus   | Tables / Records   | Aggregates                                      |
| Returns | DTOs / Raw rows    | Domain objects                                  |
| Layer   | Infrastructure     | Domain interface (implemented in infrastructure)|
| Concern | CRUD operations    | Aggregate consistency                           |

In Go:

```go
// DAO
func (dao *TenantDAO) Insert(record TenantRecord) error

// Repository
func (repo *TenantRepo) Save(ctx context.Context, t *Tenant) error
```

A DAO handles data structures. A Repository manages aggregate lifecycles, safeguards invariants, and hides persistence details instead of wrapping CRUD calls.

---

## 6. Coexistence in Practice

Real systems often use both.

The Repository interface lives in the domain layer, while a DAO encapsulates low-level database access:

```go
type TenantDAO struct{ col *mongo.Collection }

func (d *TenantDAO) Upsert(ctx context.Context, doc any) error {
    _, err := d.col.UpdateByID(ctx, doc.(bson.M)["_id"], doc, options.Update().SetUpsert(true))
    return err
}

type MongoTenantRepository struct{ dao *TenantDAO }

func (r *MongoTenantRepository) Save(ctx context.Context, t *Tenant) error {
    doc := bson.M{"_id": t.ID, "name": t.Name, "isActive": t.IsActive}
    return r.dao.Upsert(ctx, doc)
}
```

The domain only relies on `TenantRepository`, never its database-specific implementation. This keeps the domain pure and the infrastructure easily replaceable.

---

## 7. Transaction Boundaries

Repositories do not manage transactions. Application services coordinate them:

```go
func (s *TenantService) ActivateTenant(ctx context.Context, id TenantID) error {
    tenant, err := s.repo.ByID(ctx, id)
    if err != nil {
        return err
    }
    tenant.Activate()
    return s.repo.Save(ctx, tenant)
}
```

The service controls transaction scope. The repository’s only job is to persist aggregates.

---

## 8. Type Hierarchies and Polymorphism

Sometimes aggregates share behavior (e.g., Tenant vs. ExternalTenant). Vernon cautions against generic repositories—separate them unless polymorphism is genuinely needed.

```go
type TenantRepository interface { /* ... */ }
type ExternalTenantRepository interface { /* ... */ }
```

Separate repositories clarify responsibility and avoid subtype leakage.

---

## 9. Testing Repositories

For unit tests, use in-memory implementations; for integration tests, use real databases:

```go
type InMemoryTenantRepo struct {
    store map[TenantID]*Tenant
}

func (r *InMemoryTenantRepo) Save(ctx context.Context, t *Tenant) error {
    r.store[t.ID] = t
    return nil
}

func (r *InMemoryTenantRepo) ByID(ctx context.Context, id TenantID) (*Tenant, error) {
    return r.store[id], nil
}
```

The domain logic behaves identically across both, proving the abstraction’s integrity.

---

## 10. When Not to Use Repositories

Vernon ends the chapter with a warning:

> Don’t create repositories simply because the pattern exists.

If the concept isn’t an aggregate root—like a read-only report—use a DAO or query service instead. Use repositories for lifecycles, not analytics or logging.

When you need cross-aggregate reports or projections, prefer a dedicated read model or query service so repository contracts stay focused on a single aggregate boundary.

---

## 11. Designing for Flexibility

Prefer the persistence-oriented style—it’s adaptable and explicit.

You can swap storage technologies without rewriting business logic:

```go
var repo TenantRepository
if useMongo {
    repo = NewMongoTenantRepo(client)
} else {
    repo = NewSQLTenantRepo(db)
}
```

As long as both satisfy the same domain contract, the application layer remains untouched.

Keep query operations scoped to aggregate identity or other local keys. Broader, reporting-style queries belong in read models tailored to consumers, which keeps the write model small and explicit.

---

## 12. Key Takeaways

- Use one repository for each aggregate root.
- Keep repository interfaces small and focused on domain needs.
- The application layer owns transactions.
- Distinguish DAOs (low-level data access) from Repositories (domain logic and lifecycle).
- Avoid generic repositories; keep type hierarchies specific to each aggregate.
- Use in-memory repository implementations for effective unit testing.
- Consider DAOs or query services for strictly read-only data requirements.

---

## 13. Why It Matters to Go Developers

Go lacks an ORM with implicit state tracking, so boundaries must be explicitly designed. DDD repositories isolate business logic from data access.

By following Vernon’s approach, your Go services become clearer, more testable, and better aligned with real business concepts rather than just database schemas.

That’s the essence of Chapter 12: Repositories sustain your domain between requests—not just wrap persistence.
