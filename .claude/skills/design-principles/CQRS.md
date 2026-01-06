---
name: cqrs
description: CQRS (Command Query Responsibility Segregation) principle for TypeScript code review and refactoring. Use when reviewing architecture that separates read and write models, evaluating if CQRS is appropriate, or refactoring systems with different read/write scaling needs. Architectural extension of CQS principle.
---

# CQRS — Command Query Responsibility Segregation

Separate the read model (queries) from the write model (commands) at the architectural level. Different models optimized for their specific purpose.

## When CQRS Applies

- Read and write workloads have different scaling requirements
- Complex domain with simple read views (or vice versa)
- Different teams own read vs write paths
- Event sourcing is in use
- Read-heavy applications with complex aggregations

## Architecture Overview

```
┌─────────────┐     ┌──────────────────┐     ┌─────────────┐
│   Client    │────▶│  Command Handler │────▶│ Write Model │
│   (Write)   │     │   (Validation,   │     │  (Domain)   │
└─────────────┘     │    Business)     │     └─────────────┘
                    └──────────────────┘            │
                                                    │ Events/Sync
                                                    ▼
┌─────────────┐     ┌──────────────────┐     ┌─────────────┐
│   Client    │────▶│  Query Handler   │────▶│ Read Model  │
│   (Read)    │     │  (Simple fetch)  │     │ (Optimized) │
└─────────────┘     └──────────────────┘     └─────────────┘
```

## Example

**Before — Single Model (Mixed Concerns):**

```typescript
// ProductRepository.ts — one model for everything
class ProductRepository {
  async getById(id: string): Promise<Product> {
    // Complex domain object with all relationships
    const product = await this.db.query(`
      SELECT p.*, 
             c.name as category_name,
             b.name as brand_name,
             AVG(r.rating) as average_rating,
             COUNT(r.id) as review_count,
             i.quantity as stock_quantity,
             GROUP_CONCAT(t.name) as tags
      FROM products p
      LEFT JOIN categories c ON p.category_id = c.id
      LEFT JOIN brands b ON p.brand_id = b.id
      LEFT JOIN reviews r ON r.product_id = p.id
      LEFT JOIN inventory i ON i.product_id = p.id
      LEFT JOIN product_tags pt ON pt.product_id = p.id
      LEFT JOIN tags t ON t.id = pt.tag_id
      WHERE p.id = ?
      GROUP BY p.id
    `, [id]);

    return this.toDomainModel(product);
  }

  async updatePrice(id: string, price: number): Promise<void> {
    // Write needs domain validation
    const product = await this.getById(id); // Fetches everything!
    product.updatePrice(price);             // Domain logic
    await this.save(product);               // Complex save
  }

  async getProductList(filters: Filters): Promise<ProductListItem[]> {
    // List view doesn't need full domain model
    // but we're forced to use the same structure
    const products = await this.search(filters);
    return products.map(p => ({
      id: p.id,
      name: p.name,
      price: p.price,
      thumbnail: p.images[0]?.thumbnail,
    }));
  }
}
```

**After — CQRS Applied:**

```typescript
// === WRITE SIDE ===

// commands/UpdateProductPriceCommand.ts
interface UpdateProductPriceCommand {
  productId: string;
  newPrice: number;
  updatedBy: string;
}

// handlers/UpdateProductPriceHandler.ts
class UpdateProductPriceHandler {
  constructor(
    private readonly productRepository: ProductWriteRepository,
    private readonly eventBus: EventBus
  ) {}

  async handle(command: UpdateProductPriceCommand): Promise<void> {
    const product = await this.productRepository.getById(command.productId);

    // Rich domain model for writes
    product.updatePrice(command.newPrice, command.updatedBy);

    await this.productRepository.save(product);

    // Notify read side to update
    await this.eventBus.publish(new ProductPriceUpdatedEvent({
      productId: command.productId,
      newPrice: command.newPrice,
    }));
  }
}

// ProductWriteRepository.ts — optimized for domain operations
class ProductWriteRepository {
  async getById(id: string): Promise<Product> {
    // Only load what domain logic needs
    const data = await this.db.query(
      'SELECT id, price, status, version FROM products WHERE id = ?',
      [id]
    );
    return new Product(data);
  }
}

// === READ SIDE ===

// queries/GetProductDetailsQuery.ts
interface GetProductDetailsQuery {
  productId: string;
}

// handlers/GetProductDetailsHandler.ts
class GetProductDetailsHandler {
  constructor(private readonly readDb: ProductReadDatabase) {}

  async handle(query: GetProductDetailsQuery): Promise<ProductDetailsView> {
    // Simple, fast read from denormalized view
    return this.readDb.getProductDetails(query.productId);
  }
}

// ProductReadDatabase.ts — optimized for read patterns
class ProductReadDatabase {
  // Denormalized view, pre-computed aggregates
  async getProductDetails(id: string): Promise<ProductDetailsView> {
    return this.db.query(
      'SELECT * FROM product_details_view WHERE id = ?',
      [id]
    );
  }

  async getProductList(filters: Filters): Promise<ProductListItem[]> {
    // Optimized for list display
    return this.db.query(
      'SELECT id, name, price, thumbnail, rating FROM product_list_view WHERE ...'
    );
  }
}

// Event handler updates read model
class ProductPriceUpdatedHandler {
  constructor(private readonly readDb: ProductReadDatabase) {}

  async handle(event: ProductPriceUpdatedEvent): Promise<void> {
    await this.readDb.updateProductPrice(event.productId, event.newPrice);
  }
}
```

## Anti-Pattern: CQRS Everywhere

CQRS adds complexity. Don't use it for simple CRUD:

```typescript
// ❌ Over-engineering simple operations
interface CreateUserCommand { email: string; name: string; }
interface GetUserQuery { userId: string; }
interface UserCreatedEvent { userId: string; }

// Handlers, event bus, read model sync... for basic CRUD?

// ✅ Simple repository is fine for simple domains
class UserRepository {
  async create(user: CreateUserDTO): Promise<User> { /* ... */ }
  async getById(id: string): Promise<User> { /* ... */ }
}
```

## When NOT to Apply

- **Simple CRUD** — Repository pattern is sufficient
- **Small applications** — Complexity cost exceeds benefits
- **Strong consistency required** — Eventual consistency between models is inherent
- **Single team** — Separation overhead without organizational benefit
- **Uniform read/write patterns** — No different optimization needs

The key question: "Do reads and writes have fundamentally different requirements?"
