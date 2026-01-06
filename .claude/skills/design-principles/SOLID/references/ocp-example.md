# OCP Example

## Before — OCP Violation

```typescript
// PaymentProcessor.ts — must modify for each new payment type
class PaymentProcessor {
  process(payment: Payment): Promise<PaymentResult> {
    switch (payment.type) {
      case 'credit_card':
        return this.processCreditCard(payment);
      case 'paypal':
        return this.processPaypal(payment);
      case 'apple_pay':
        return this.processApplePay(payment);
      // Adding new payment = modifying this file
      default:
        throw new Error(`Unknown payment type: ${payment.type}`);
    }
  }

  private processCreditCard(payment: Payment): Promise<PaymentResult> { /* ... */ }
  private processPaypal(payment: Payment): Promise<PaymentResult> { /* ... */ }
  private processApplePay(payment: Payment): Promise<PaymentResult> { /* ... */ }
}
```

## After — OCP Applied

```typescript
// PaymentStrategy.ts — extension point
interface PaymentStrategy {
  readonly type: string;
  process(payment: Payment): Promise<PaymentResult>;
}

// CreditCardStrategy.ts — one file per variant
class CreditCardStrategy implements PaymentStrategy {
  readonly type = 'credit_card';

  async process(payment: Payment): Promise<PaymentResult> {
    // Credit card specific logic
  }
}

// PaymentProcessor.ts — never needs modification
class PaymentProcessor {
  private strategies: Map<string, PaymentStrategy>;

  constructor(strategies: PaymentStrategy[]) {
    this.strategies = new Map(strategies.map(s => [s.type, s]));
  }

  async process(payment: Payment): Promise<PaymentResult> {
    const strategy = this.strategies.get(payment.type);
    if (!strategy) {
      throw new Error(`No strategy for payment type: ${payment.type}`);
    }
    return strategy.process(payment);
  }
}

// Adding GooglePayStrategy = new file only, no modifications
```
