---
type: skill
name: subdomain-classification
category: ddd/strategic
---

# Subdomain Classification

## Detect

Scan for these patterns indicating misclassified subdomain:

| Pattern | Example |
|---------|---------|
| Over-engineering generic | Custom-built auth system instead of Auth0/Cognito |
| Under-investing in core | Recommendation engine as simple SQL query |
| Core treated as CRUD | Competitive advantage with just getters/setters |
| Supporting as core | Months spent on "perfect" user management |
| Building what exists | Custom payment processing instead of Stripe |

## The three types

| Type | What it is | Investment |
|------|------------|------------|
| **Core** | Competitive advantage | High — rich DDD, best developers |
| **Supporting** | Enables core, not differentiating | Medium — solid, can outsource |
| **Generic** | Solved problem, same everywhere | Low — buy or use off-the-shelf |

## Fix

### Core — invest heavily

```typescript
// Competitive advantage = rich domain model
class PricingEngine {
  constructor(
    private readonly demandAnalyzer: DemandAnalyzer,
    private readonly competitorMonitor: CompetitorPriceMonitor,
    private readonly marginPolicy: MarginPolicy
  ) {}

  calculateOptimalPrice(
    product: Product,
    customer: Customer,
    marketConditions: MarketConditions
  ): Price {
    const demand = this.demandAnalyzer.analyze(product);
    const competitorPrices = this.competitorMonitor.getPrices(product);
    // Complex proprietary algorithm
    return this.optimizePrice(demand, competitorPrices, customer);
  }
}
```

### Supporting — solid but simpler

```typescript
// Needed but not differentiating
class InventoryService {
  reserve(sku: SKU, quantity: Quantity): ReservationResult {
    const stock = this.repository.findBySku(sku);
    if (!stock.hasAvailable(quantity)) {
      return ReservationResult.insufficientStock();
    }
    stock.reserve(quantity);
    this.repository.save(stock);
    return ReservationResult.success();
  }
}
```

### Generic — don't build

```typescript
// Use third-party
interface PaymentGateway {
  charge(amount: Money, token: PaymentMethodToken): PaymentResult;
}

class StripePaymentGateway implements PaymentGateway {
  constructor(private readonly stripe: Stripe) {}

  async charge(amount: Money, token: PaymentMethodToken): Promise<PaymentResult> {
    const result = await this.stripe.charges.create({
      amount: amount.cents,
      currency: amount.currency.code,
      source: token.value
    });
    return this.mapResult(result);
  }
}
```

## Violations

### Over-engineering generic subdomains
```typescript
// ❌ Detect: months spent on solved problem
class AuthenticationService {
  hashPassword(password: string): HashedPassword { }
  verifyPassword(password: string, hash: HashedPassword): boolean { }
  generateToken(user: User): JwtToken { }
  validateToken(token: JwtToken): TokenValidation { }
  handleMfa(user: User, code: MfaCode): MfaResult { }
  // ... 2000 more lines of custom auth
}

// ✅ Fix: use established solution
class Auth0Adapter implements AuthenticationPort {
  constructor(private readonly auth0: Auth0Client) {}
  
  authenticate(credentials: Credentials): Promise<AuthResult> {
    return this.auth0.authenticate(credentials);
  }
}
```

### Under-investing in core
```typescript
// ❌ Detect: competitive advantage treated as simple CRUD
class RecommendationService {
  getRecommendations(userId: string): Product[] {
    return this.db.query(
      'SELECT * FROM products ORDER BY popularity LIMIT 10'
    );
  }
}

// ✅ Fix: rich domain model for core
class RecommendationEngine {
  constructor(
    private readonly behaviorAnalyzer: UserBehaviorAnalyzer,
    private readonly similarityModel: ProductSimilarityModel,
    private readonly personalization: PersonalizationStrategy
  ) {}

  generateRecommendations(
    customer: Customer,
    context: BrowsingContext
  ): Recommendations {
    const behavior = this.behaviorAnalyzer.analyze(customer);
    const strategy = this.personalization.forCustomer(customer);
    return strategy.recommend(behavior, context);
  }
}
```

### Misclassifying supporting as core
```typescript
// ❌ Detect: over-engineered user management
class User {
  private behaviorStateMachine: UserBehaviorStateMachine;
  private lifecycleManager: UserLifecycleManager;
  private permissionEngine: DynamicPermissionEngine;
  // Complex domain model for basic user ops
}

// ✅ Fix: supporting = solid but pragmatic
class User {
  constructor(
    readonly id: UserId,
    private email: Email,
    private role: Role,
    private status: UserStatus
  ) {}

  activate(): void { this.status = UserStatus.Active; }
  deactivate(): void { this.status = UserStatus.Inactive; }
  changeRole(newRole: Role): void { this.role = newRole; }
}
```

## Decision matrix

| Question | Core | Supporting | Generic |
|----------|------|------------|---------|
| Build custom? | Yes | Maybe | No |
| Full DDD? | Yes | Selective | Minimal |
| Best developers? | Yes | Mixed | Junior OK |

## Skip when

- Early stage — classification emerges as business evolves
- Unclear differentiation — start simple, refactor later
- All feels "core" — step back, talk to business stakeholders
