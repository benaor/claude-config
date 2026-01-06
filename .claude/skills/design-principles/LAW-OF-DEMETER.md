---
name: law-of-demeter
description: Law of Demeter (Principle of Least Knowledge) for TypeScript code review and refactoring. Use when detecting deep property access chains, tight coupling between modules, or objects that know too much about other objects' internals. Helps reduce coupling.
---

# Law of Demeter — Principle of Least Knowledge

A method should only talk to its immediate friends. Don't reach through objects to access their internals.

## Allowed Calls

A method `M` of object `O` may only call methods on:

1. `O` itself (this)
2. Objects passed as parameters to `M`
3. Objects created within `M`
4. `O`'s direct properties
5. Global/static objects accessible in `M`'s scope

## Violation Signs

- Long dot chains: `a.b.c.d.method()`
- Accessing nested properties: `user.address.city.zipCode`
- Passing objects just to extract sub-properties
- Changes in one class ripple to many dependent classes
- Tests require deep object mocking

## Example

**Before — Law of Demeter Violation:**

```typescript
// OrderService.ts — knows too much about object internals
class OrderService {
  constructor(private readonly orderRepository: OrderRepository) {}

  calculateShipping(order: Order): number {
    // Violation: reaching deep into object graph
    const country = order.customer.address.country;
    const weight = order.items.reduce(
      (sum, item) => sum + item.product.shipping.weight * item.quantity,
      0
    );
    const carrier = order.customer.preferences.shipping.preferredCarrier;

    // This method now depends on:
    // - Order structure
    // - Customer structure  
    // - Address structure
    // - Item structure
    // - Product structure
    // - ShippingInfo structure
    // - Preferences structure
    
    return this.calculateRate(country, weight, carrier);
  }

  async sendConfirmation(order: Order): Promise<void> {
    // More violations
    const email = order.customer.contact.email;
    const name = order.customer.profile.firstName;
    const trackingUrl = order.shipment.carrier.tracking.url;
    
    await this.emailService.send(email, `Hi ${name}, track: ${trackingUrl}`);
  }
}

// Any change to Customer, Address, Product structure breaks OrderService
```

**After — Law of Demeter Applied:**

```typescript
// Order.ts — encapsulates its knowledge
class Order {
  constructor(
    private readonly customer: Customer,
    private readonly items: OrderItem[],
    private readonly shipment?: Shipment
  ) {}

  getShippingCountry(): string {
    return this.customer.getShippingCountry();
  }

  getTotalWeight(): number {
    return this.items.reduce((sum, item) => sum + item.getShippingWeight(), 0);
  }

  getPreferredCarrier(): string {
    return this.customer.getPreferredCarrier();
  }

  getCustomerEmail(): string {
    return this.customer.getEmail();
  }

  getCustomerFirstName(): string {
    return this.customer.getFirstName();
  }

  getTrackingUrl(): string | null {
    return this.shipment?.getTrackingUrl() ?? null;
  }
}

// Customer.ts — encapsulates its structure
class Customer {
  getShippingCountry(): string {
    return this.address.country;
  }

  getPreferredCarrier(): string {
    return this.preferences.shipping.preferredCarrier;
  }

  getEmail(): string {
    return this.contact.email;
  }

  getFirstName(): string {
    return this.profile.firstName;
  }
}

// OrderService.ts — talks only to Order
class OrderService {
  calculateShipping(order: Order): number {
    return this.calculateRate(
      order.getShippingCountry(),
      order.getTotalWeight(),
      order.getPreferredCarrier()
    );
  }

  async sendConfirmation(order: Order): Promise<void> {
    const trackingUrl = order.getTrackingUrl();
    if (!trackingUrl) return;

    await this.emailService.send(
      order.getCustomerEmail(),
      `Hi ${order.getCustomerFirstName()}, track: ${trackingUrl}`
    );
  }
}
```

## Anti-Pattern: Train Wreck

Long chains of calls that expose internal structure:

```typescript
// ❌ Train wreck — each dot is a coupling point
const city = user.getProfile().getAddress().getCity().getName();

// ✅ Ask for what you need
const city = user.getCityName();

// Or pass the minimum needed data
interface ShippingDestination {
  city: string;
  country: string;
  postalCode: string;
}

function calculateShipping(destination: ShippingDestination): number {
  // Receives exactly what it needs, doesn't know about User
}
```

## React / React Native

```typescript
// ❌ Component knows too much about state shape
function OrderSummary({ state }: { state: AppState }) {
  return (
    <View>
      <Text>{state.checkout.order.customer.name}</Text>
      <Text>{state.checkout.order.totals.withTax}</Text>
      <Text>{state.checkout.shipping.address.formatted}</Text>
    </View>
  );
}

// ✅ Component receives only what it needs
interface OrderSummaryProps {
  customerName: string;
  totalWithTax: number;
  formattedAddress: string;
}

function OrderSummary({ customerName, totalWithTax, formattedAddress }: OrderSummaryProps) {
  return (
    <View>
      <Text>{customerName}</Text>
      <Text>{formatCurrency(totalWithTax)}</Text>
      <Text>{formattedAddress}</Text>
    </View>
  );
}

// Container/selector handles the extraction
function OrderSummaryContainer() {
  const props = useAppSelector((state) => ({
    customerName: selectCustomerName(state),
    totalWithTax: selectTotalWithTax(state),
    formattedAddress: selectFormattedShippingAddress(state),
  }));

  return <OrderSummary {...props} />;
}
```

## When NOT to Apply

Strict LoD compliance can lead to wrapper method explosion. Acceptable violations:

- **Data Transfer Objects** — Plain DTOs without behavior can expose properties
- **Fluent APIs** — Builder patterns intentionally chain: `builder.setX().setY().build()`
- **Functional pipelines** — `array.filter().map().reduce()` is idiomatic
- **Configuration objects** — `config.database.connection.timeout` is readable

The key question: "If I change this internal structure, how many files break?"
