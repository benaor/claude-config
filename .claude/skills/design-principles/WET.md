---
name: wet
description: WET (Write Everything Twice) principle for TypeScript code review and refactoring. Use when evaluating whether to extract abstractions, deciding if duplication is acceptable, or reviewing premature DRY refactors. Helps prevent over-abstraction. Companion to DRY principle.
---

# WET — Write Everything Twice

Also known as "AHA" (Avoid Hasty Abstractions). Accept duplication until patterns become clear. The cost of the wrong abstraction is higher than the cost of duplication.

## When WET is Right

- First and second occurrence: duplicate freely
- Third occurrence: consider extraction (Rule of Three)
- Unclear pattern: keep duplicating until the abstraction reveals itself
- Different contexts: what looks similar may need to diverge

## Example

**Before — Premature DRY:**

```typescript
// Extracted too early based on superficial similarity
// services/DataExporter.ts
interface ExportOptions {
  format: 'csv' | 'json' | 'xml' | 'pdf';
  fields?: string[];
  filters?: Record<string, unknown>;
  includeHeaders?: boolean;
  dateFormat?: string;
  delimiter?: string;
  encoding?: string;
  transform?: (data: unknown) => unknown;
  onProgress?: (percent: number) => void;
  // Options keep growing as use cases diverge...
}

class DataExporter<T> {
  constructor(private readonly repository: Repository<T>) {}

  async export(options: ExportOptions): Promise<Buffer> {
    const data = await this.repository.findAll(options.filters);
    
    switch (options.format) {
      case 'csv':
        return this.toCsv(data, options);
      case 'json':
        return this.toJson(data, options);
      case 'xml':
        return this.toXml(data, options);
      case 'pdf':
        return this.toPdf(data, options);
      // Every new format adds more complexity
      // Bug fixes in one format often break others
    }
  }

  private toCsv(data: T[], options: ExportOptions): Buffer { /* 100 lines */ }
  private toJson(data: T[], options: ExportOptions): Buffer { /* 50 lines */ }
  private toXml(data: T[], options: ExportOptions): Buffer { /* 80 lines */ }
  private toPdf(data: T[], options: ExportOptions): Buffer { /* 150 lines */ }
}

// Usage: forced to use same interface for very different needs
const userExporter = new DataExporter(userRepo);
const reportExporter = new DataExporter(reportRepo);
```

**After — WET Applied:**

```typescript
// Separate exporters that can evolve independently
// services/UserCsvExporter.ts
class UserCsvExporter {
  constructor(private readonly userRepository: UserRepository) {}

  async export(filters?: UserFilters): Promise<Buffer> {
    const users = await this.userRepository.findAll(filters);
    const headers = ['id', 'name', 'email', 'createdAt'];
    const rows = users.map(u => [u.id, u.name, u.email, u.createdAt.toISOString()]);
    return this.toCsv(headers, rows);
  }

  private toCsv(headers: string[], rows: string[][]): Buffer {
    // Simple, focused implementation
  }
}

// services/ReportPdfExporter.ts
class ReportPdfExporter {
  constructor(
    private readonly reportRepository: ReportRepository,
    private readonly pdfGenerator: PdfGenerator
  ) {}

  async export(reportId: string): Promise<Buffer> {
    const report = await this.reportRepository.getById(reportId);
    return this.pdfGenerator.generate({
      title: report.title,
      sections: report.sections,
      charts: report.charts,
      // PDF-specific options only
    });
  }
}

// services/AnalyticsJsonExporter.ts
class AnalyticsJsonExporter {
  constructor(private readonly analyticsRepository: AnalyticsRepository) {}

  async export(dateRange: DateRange): Promise<Buffer> {
    const data = await this.analyticsRepository.getByDateRange(dateRange);
    return Buffer.from(JSON.stringify(data, null, 2));
  }
}

// Each exporter: ~30-50 lines, focused, easy to modify
// Adding new export: new file, no risk to existing exporters
```

## Anti-Pattern: Wrong Abstraction

A bad abstraction is worse than duplication:

```typescript
// ❌ Forced abstraction creates coupling
class BaseService<T> {
  constructor(
    protected readonly repository: Repository<T>,
    protected readonly validator: Validator<T>,
    protected readonly logger: Logger,
    protected readonly cache?: Cache,
    protected readonly eventBus?: EventBus
  ) {}

  async create(data: Partial<T>): Promise<T> {
    this.logger.info('Creating entity');
    const validated = await this.validator.validate(data);
    const entity = await this.repository.create(validated);
    this.cache?.invalidate(this.getCacheKey());
    this.eventBus?.publish(this.getCreatedEvent(entity));
    return entity;
  }

  // Every service forced into same pattern
  // What if OrderService needs saga? What if UserService needs hooks?
}

// Every service inherits unwanted complexity
class UserService extends BaseService<User> { /* constrained */ }
class OrderService extends BaseService<Order> { /* constrained */ }
class NotificationService extends BaseService<Notification> { /* constrained */ }

// ✅ Purpose-specific services
class UserService {
  // Exactly what users need, nothing more
}

class OrderService {
  // Can add saga pattern without affecting others
}

class NotificationService {
  // Different caching strategy, no compromises
}
```

## React / React Native

```typescript
// ❌ Premature component abstraction
// components/Card.tsx
interface CardProps {
  variant: 'user' | 'product' | 'notification' | 'settings';
  title: string;
  subtitle?: string;
  image?: string;
  badge?: string;
  footer?: React.ReactNode;
  onPress?: () => void;
  showChevron?: boolean;
  disabled?: boolean;
}

function Card({ variant, title, image, badge, ... }: CardProps) {
  // 150 lines of conditional rendering based on variant
  return (
    <Pressable onPress={onPress} disabled={disabled}>
      {variant === 'user' && <Avatar source={image} />}
      {variant === 'product' && <ProductImage source={image} />}
      {variant === 'notification' && badge && <Badge text={badge} />}
      {/* More variant-specific conditionals... */}
    </Pressable>
  );
}

// ✅ Separate components that can evolve independently
// components/UserCard.tsx
function UserCard({ user, onPress }: UserCardProps) {
  return (
    <Pressable onPress={onPress}>
      <Avatar source={user.avatar} size="medium" />
      <View>
        <Text style={styles.name}>{user.name}</Text>
        <Text style={styles.role}>{user.role}</Text>
      </View>
      <ChevronRight />
    </Pressable>
  );
}

// components/ProductCard.tsx
function ProductCard({ product, onAddToCart }: ProductCardProps) {
  return (
    <Pressable onPress={() => navigate('Product', { id: product.id })}>
      <ProductImage source={product.image} />
      <Text style={styles.productName}>{product.name}</Text>
      <Text style={styles.price}>{formatPrice(product.price)}</Text>
      <Button title="Add to Cart" onPress={onAddToCart} />
    </Pressable>
  );
}

// Each component: ~30 lines, focused, easy to modify
```

**Wrong hook abstraction:**

```typescript
// ❌ Forced abstraction creates coupling
function useDataFetcher<T>(
  endpoint: string,
  options?: {
    transform?: (data: unknown) => T;
    cache?: boolean;
    pollInterval?: number;
    onError?: (error: Error) => void;
    retries?: number;
    // Options keep growing...
  }
) {
  // 200 lines trying to handle every case
}

// Every screen uses it differently, none are satisfied
const users = useDataFetcher('/users', { transform: parseUsers, cache: true });
const messages = useDataFetcher('/messages', { pollInterval: 5000 });

// ✅ Purpose-specific hooks
function useUsers() {
  // Exactly what users list needs
}

function useMessages() {
  // Can add polling without affecting other hooks
}
```

## The Rule of Three

```
1st occurrence → Just write it
2nd occurrence → Copy it, note the similarity  
3rd occurrence → Now extract (you see the real pattern)
```

## When to Extract Anyway

Even with <3 occurrences, extract when:

- **Bug-prone logic** — Complex calculations that must stay in sync
- **Business rules** — Domain logic that's definitionally the same
- **Test burden** — Same logic needs identical test coverage in multiple places
- **Proven stable** — Pattern is well-established and unlikely to diverge

The key question: "Am I sure these will always change together?"
