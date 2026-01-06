---
name: dry
description: DRY (Don't Repeat Yourself) principle for TypeScript code review and refactoring. Use when detecting duplicated logic, copy-pasted code, repeated patterns, or redundant implementations. Helps identify extraction opportunities. See also WET principle for when duplication is acceptable.
---

# DRY — Don't Repeat Yourself

Every piece of knowledge should have a single, unambiguous representation in the system.

## Violation Signs

- Copy-pasted code blocks with minor variations
- Same validation logic in multiple places
- Repeated data transformations
- Identical error handling patterns across files
- Multiple sources of truth for the same concept

## Example

**Before — DRY Violation:**

```typescript
// CreateUserService.ts
class CreateUserService {
  async execute(data: CreateUserDTO): Promise<User> {
    // Validation duplicated
    if (!data.email) {
      throw new ValidationError('Email is required');
    }
    if (!data.email.includes('@')) {
      throw new ValidationError('Invalid email format');
    }
    if (data.email.length > 254) {
      throw new ValidationError('Email too long');
    }

    return this.userRepository.create(data);
  }
}

// UpdateUserService.ts — same validation duplicated
class UpdateUserService {
  async execute(id: string, data: UpdateUserDTO): Promise<User> {
    // Same validation logic repeated
    if (!data.email) {
      throw new ValidationError('Email is required');
    }
    if (!data.email.includes('@')) {
      throw new ValidationError('Invalid email format');
    }
    if (data.email.length > 254) {
      throw new ValidationError('Email too long');
    }

    return this.userRepository.update(id, data);
  }
}

// InviteUserService.ts — and again...
class InviteUserService {
  // Same validation logic repeated a third time
}
```

**After — DRY Applied:**

```typescript
// validators/EmailValidator.ts — single source of truth
interface ValidationResult {
  isValid: boolean;
  error?: string;
}

class EmailValidator {
  validate(email: string): ValidationResult {
    if (!email) {
      return { isValid: false, error: 'Email is required' };
    }
    if (!email.includes('@')) {
      return { isValid: false, error: 'Invalid email format' };
    }
    if (email.length > 254) {
      return { isValid: false, error: 'Email too long' };
    }
    return { isValid: true };
  }

  validateOrThrow(email: string): void {
    const result = this.validate(email);
    if (!result.isValid) {
      throw new ValidationError(result.error!);
    }
  }
}

// CreateUserService.ts — clean usage
class CreateUserService {
  constructor(
    private readonly userRepository: UserRepository,
    private readonly emailValidator: EmailValidator
  ) {}

  async execute(data: CreateUserDTO): Promise<User> {
    this.emailValidator.validateOrThrow(data.email);
    return this.userRepository.create(data);
  }
}

// UpdateUserService.ts — same validator, reused
class UpdateUserService {
  constructor(
    private readonly userRepository: UserRepository,
    private readonly emailValidator: EmailValidator
  ) {}

  async execute(id: string, data: UpdateUserDTO): Promise<User> {
    this.emailValidator.validateOrThrow(data.email);
    return this.userRepository.update(id, data);
  }
}
```

## Anti-Pattern: Shotgun Surgery

When one change requires modifications in multiple places:

```typescript
// ❌ Price calculation duplicated
// CartScreen.tsx
const total = items.reduce((sum, item) => sum + item.price * item.quantity, 0);
const withTax = total * 1.2;
const withDiscount = coupon ? withTax * 0.9 : withTax;

// CheckoutScreen.tsx
const subtotal = cartItems.reduce((sum, item) => sum + item.price * item.quantity, 0);
const tax = subtotal * 0.2;
const discount = hasCoupon ? (subtotal + tax) * 0.1 : 0;
const finalPrice = subtotal + tax - discount;

// OrderConfirmationScreen.tsx — slightly different again...

// ✅ Single source of truth
// domain/pricing.ts
interface PriceBreakdown {
  subtotal: number;
  tax: number;
  discount: number;
  total: number;
}

function calculatePrice(items: CartItem[], couponCode?: string): PriceBreakdown {
  const subtotal = items.reduce((sum, item) => sum + item.price * item.quantity, 0);
  const tax = subtotal * 0.2;
  const discount = couponCode ? (subtotal + tax) * 0.1 : 0;
  return {
    subtotal,
    tax,
    discount,
    total: subtotal + tax - discount,
  };
}
```

## React / React Native

```typescript
// ❌ Validation duplicated across screens
// CreateUserScreen.tsx
function CreateUserScreen() {
  const [email, setEmail] = useState('');
  const [emailError, setEmailError] = useState('');

  const validateEmail = () => {
    if (!email) {
      setEmailError('Email is required');
      return false;
    }
    if (!email.includes('@')) {
      setEmailError('Invalid email format');
      return false;
    }
    setEmailError('');
    return true;
  };
  // ...
}

// EditProfileScreen.tsx — same validation duplicated
function EditProfileScreen() {
  const [email, setEmail] = useState(user.email);
  const [emailError, setEmailError] = useState('');

  const validateEmail = () => {
    // Same validation logic repeated...
  };
}

// ✅ Extract to reusable hook
// hooks/useEmailField.ts
function useEmailField(initialValue = '') {
  const [email, setEmail] = useState(initialValue);
  const [error, setError] = useState('');

  const validate = useCallback(() => {
    const result = validateEmail(email); // Uses shared validator
    setError(result.error ?? '');
    return result.isValid;
  }, [email]);

  return { email, setEmail, error, validate };
}

// CreateUserScreen.tsx — clean usage
function CreateUserScreen() {
  const emailField = useEmailField();
  // ...
}

// EditProfileScreen.tsx — same hook, different initial value
function EditProfileScreen() {
  const emailField = useEmailField(user.email);
  // ...
}
```

## When NOT to Apply

DRY is about knowledge duplication, not code duplication. Don't extract when:

- **Coincidental similarity** — Two functions look alike now but represent different concepts that may diverge
- **Premature abstraction** — See WET principle: wait for 3 occurrences before extracting
- **Coupling cost** — Extraction creates tight coupling between unrelated modules
- **Readability loss** — The abstraction is harder to understand than the duplication

The key question: "Is this the same knowledge, or just similar-looking code?"
