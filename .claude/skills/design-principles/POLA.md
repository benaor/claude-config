---
name: pola
description: PoLA (Principle of Least Authority) principle for TypeScript code review and refactoring. Use when detecting overly broad permissions, functions receiving more access than needed, or security concerns around excessive privileges. Helps improve security and reduce attack surface.
---

# PoLA — Principle of Least Authority

Grant only the minimum permissions and access necessary to perform a task. Reduce the blast radius of bugs and security issues.

## Also Known As

- Principle of Least Privilege (PoLP)
- Need-to-know basis
- Minimal authority

## Violation Signs

- Functions receiving entire objects when they need one property
- Admin/root access used for routine operations
- API keys with full access when read-only would suffice
- Components with access to entire app state
- Services that can modify data they only need to read

## Example

**Before — PoLA Violation:**

```typescript
// ReportGenerator.ts — has access to way more than needed
class ReportGenerator {
  constructor(
    private readonly db: Database,           // Full DB access!
    private readonly userService: UserService // Can modify users!
  ) {}

  async generateSalesReport(userId: string): Promise<Report> {
    // Only needs to READ sales data for ONE user
    // But has access to:
    // - All database tables (can drop tables!)
    // - All users (can delete accounts!)
    
    const user = await this.userService.getById(userId);
    const sales = await this.db.query('SELECT * FROM sales WHERE user_id = ?', [userId]);
    
    return this.formatReport(user.name, sales);
  }
}

// If this class has a bug or is compromised:
// - Could read sensitive data from any table
// - Could modify or delete user accounts
// - Could corrupt or drop sales data
```

**After — PoLA Applied:**

```typescript
// Ports define minimal required access
interface SalesReadPort {
  getSalesByUserId(userId: string): Promise<Sale[]>;
}

interface UserNameResolver {
  getNameById(userId: string): Promise<string>;
}

// ReportGenerator.ts — minimal authority
class ReportGenerator {
  constructor(
    private readonly salesReader: SalesReadPort,    // Read-only, sales only
    private readonly userNames: UserNameResolver     // Can only read names
  ) {}

  async generateSalesReport(userId: string): Promise<Report> {
    const [userName, sales] = await Promise.all([
      this.userNames.getNameById(userId),
      this.salesReader.getSalesByUserId(userId),
    ]);
    
    return this.formatReport(userName, sales);
  }
}

// If this class has a bug or is compromised:
// - Can only read sales data (not modify)
// - Can only read user names (no emails, passwords, etc.)
// - Cannot access other tables
// - Cannot perform any writes
```

## Function Parameters: Pass Only What's Needed

```typescript
// ❌ Function receives more than needed
interface User {
  id: string;
  email: string;
  passwordHash: string;
  creditCard: CreditCardInfo;
  ssn: string;
  // ... 20 more sensitive fields
}

function formatGreeting(user: User): string {
  // Only uses name, but has access to everything
  return `Hello, ${user.name}!`;
}

// If formatGreeting has a bug that logs its input...
// All sensitive data is exposed

// ✅ Receive only what's needed
function formatGreeting(name: string): string {
  return `Hello, ${name}!`;
}

// Or use a minimal interface
interface Named {
  name: string;
}

function formatGreeting(entity: Named): string {
  return `Hello, ${entity.name}!`;
}
```

## React / React Native

```typescript
// ❌ Component receives entire state
interface AppState {
  user: User;
  cart: Cart;
  orders: Order[];
  paymentMethods: PaymentMethod[];
  adminSettings: AdminSettings;
}

function CartSummary({ state }: { state: AppState }) {
  // Only needs cart, but can access admin settings
  return <Text>Total: {state.cart.total}</Text>;
}

// ✅ Component receives minimal props
interface CartSummaryProps {
  total: number;
  itemCount: number;
}

function CartSummary({ total, itemCount }: CartSummaryProps) {
  return <Text>Total: {formatCurrency(total)} ({itemCount} items)</Text>;
}

// ✅ Or use scoped selectors
function CartSummary() {
  // Hook only exposes cart data, not entire state
  const { total, itemCount } = useCartSummary();
  return <Text>Total: {formatCurrency(total)} ({itemCount} items)</Text>;
}
```

## API Keys and Tokens

```typescript
// ❌ Full admin access for a read operation
const storage = new CloudStorage({
  apiKey: process.env.ADMIN_API_KEY, // Can delete buckets!
});

async function getUserAvatar(userId: string): Promise<Buffer> {
  return storage.download(`avatars/${userId}.png`);
}

// ✅ Scoped credentials for specific operation
const avatarStorage = new CloudStorage({
  apiKey: process.env.AVATAR_READ_KEY, // Read-only, avatars bucket only
  bucket: 'avatars',
  permissions: ['read'],
});

async function getUserAvatar(userId: string): Promise<Buffer> {
  return avatarStorage.download(`${userId}.png`);
}
```

## Anti-Pattern: God Service

```typescript
// ❌ Service with authority over everything
class AppService {
  constructor(
    private readonly db: Database,
    private readonly cache: Cache,
    private readonly email: EmailService,
    private readonly payment: PaymentProcessor,
    private readonly analytics: Analytics,
    private readonly admin: AdminService
  ) {}

  // Every method in this class can access everything
  // A bug in formatDate() could theoretically process payments
}

// ✅ Focused services with minimal authority
class SalesReportService {
  constructor(
    private readonly salesReader: SalesReadPort,
    private readonly analytics: AnalyticsReadPort
  ) {}
  // Can only read sales and analytics
}

class PaymentService {
  constructor(
    private readonly paymentProcessor: PaymentPort,
    private readonly orderWriter: OrderWritePort
  ) {}
  // Can only process payments and update orders
}
```

## When to Grant Broader Access

- **Admin interfaces** — Legitimate need for broad access, but isolate to admin modules
- **Migration scripts** — Temporary broad access, run under supervision
- **Development mode** — Broader access acceptable for debugging, not production
- **Audit logging** — May need read access to many systems

The key question: "If this code is compromised, what's the worst it can do?"
