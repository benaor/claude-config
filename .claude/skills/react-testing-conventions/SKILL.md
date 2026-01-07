---
name: react-testing-conventions
description: Testing conventions for React Native with TypeScript. Use this skill when writing tests, reviewing test code, or setting up test infrastructure. Covers unit tests, integration tests, component tests, test doubles (dummy, stub, spy, mock), builder pattern for fixtures, and the testing pyramid.
---

# Testing Conventions

Testing conventions for React Native applications with TypeScript.

**Stack:** Jest • React Native Testing Library • TypeScript

## Core Principles

### Testing Pyramid

Respect the testing pyramid — more unit tests, fewer integration tests, even fewer E2E tests:

```
        /\
       /  \      E2E (Maestro)
      /----\     Few, slow, expensive
     /      \
    /--------\   Integration
   /          \  Some, medium speed
  /------------\
 /              \ Unit (*.test.ts)
/________________\ Many, fast, cheap
```

### FIRST Principles

Tests must be:

| Principle           | Description                                               |
| ------------------- | --------------------------------------------------------- |
| **F**ast            | Tests run quickly — milliseconds, not seconds             |
| **I**ndependent     | Tests don't depend on each other, can run in any order    |
| **R**epeatable      | Same result every time, no flakiness                      |
| **S**elf-validating | Pass or fail, no manual inspection needed                 |
| **T**imely          | Written at the same time as the code (or before with TDD) |

## File Organization

### Naming

| Type               | Extension  | Example                 |
| ------------------ | ---------- | ----------------------- |
| Unit test          | `.test.ts` | `Login.usecase.test.ts` |
| E2E test (Maestro) | `.yaml`    | `login-flow.yaml`       |

### Colocation (Unit & Integration)

Test files live next to the files they test:

```
modules/authentication/core/usecases/
├── Login.usecase.ts
└── Login.usecase.test.ts

modules/authentication/ui/viewModels/
├── useLogin.viewModel.tsx
└── useLogin.viewModel.test.ts

modules/authentication/infrastructure/adapters/
├── AuthApi.adapter.ts
└── AuthApi.adapter.test.ts
```

### E2E Tests (Maestro)

E2E tests live in a dedicated folder at project root:

```
tests/
├── flows/
│   ├── authentication/
│   │   ├── login-flow.yaml
│   │   └── logout-flow.yaml
│   └── events/
│       ├── create-event-flow.yaml
│       └── delete-event-flow.yaml
└── utils/
    └── common-steps.yaml
```

## Test Anatomy

### AAA Pattern

Every test follows Arrange, Act, Assert:

```typescript
it("should return user when credentials are valid", async () => {
  // Arrange
  const authRepository = new AuthRepositoryStub();
  const useCase = new LoginUseCase(authRepository);
  const credentials = { email: "test@example.com", password: "password123" };

  // Act
  const result = await useCase.execute(credentials);

  // Assert
  expect(result.success).toBe(true);
  expect(result.data.email).toBe("test@example.com");
});
```

### Test Naming

Pattern: `should [expected behavior] when [condition]`

```typescript
// ✅ Good
it("should return failure when email is invalid", ...)
it("should disable button when form is incomplete", ...)
it("should invalidate queries when mutation succeeds", ...)

// ❌ Bad
it("test login", ...)
it("works", ...)
it("handles error", ...)
```

### One logical assertion per test

```typescript
// ✅ Good — one behavior tested
it("should return failure when password is too short", async () => {
  const result = await useCase.execute({ email: "a@b.com", password: "123" });

  expect(result.success).toBe(false);
  expect(result.error.type).toBe("VALIDATION_ERROR");
});

// ❌ Bad — testing multiple unrelated behaviors
it("should validate all fields", async () => {
  // Testing email validation
  const result1 = await useCase.execute({ email: "", password: "valid123" });
  expect(result1.success).toBe(false);

  // Testing password validation
  const result2 = await useCase.execute({ email: "a@b.com", password: "" });
  expect(result2.success).toBe(false);

  // Testing success case
  const result3 = await useCase.execute({
    email: "a@b.com",
    password: "valid123",
  });
  expect(result3.success).toBe(true);
});
```

## Test Doubles

### When to use what

| Test Double | Use Case                             | Layer          |
| ----------- | ------------------------------------ | -------------- |
| **Dummy**   | Placeholder, never actually used     | Core, UI       |
| **Stub**    | Returns predefined data              | Core, UI       |
| **Spy**     | Records calls, verifies interactions | Infrastructure |
| **Mock**    | Spy + predefined behavior            | Infrastructure |

### Rule: Stub the Core, Mock the Infrastructure

```typescript
// ✅ Core/UI tests — use stubs
// We test behavior, not implementation
const authRepository = new AuthRepositoryStub();
const useCase = new LoginUseCase(authRepository);

// ✅ Infrastructure tests — use spies/mocks
// We test the implementation itself
jest.spyOn(global, "fetch").mockResolvedValue(mockResponse);
```

### Dummy

Never actually used, just satisfies the type:

```typescript
class LoggerDummy implements Logger {
  log(_message: string): void {
    // Do nothing
  }
}

// Usage
const useCase = new SomeUseCase(realDependency, new LoggerDummy());
```

### Stub

Returns predefined data:

```typescript
class AuthRepositoryStub implements AuthRepository {
  async login(_params: LoginParams): Promise<Result<User, AuthError>> {
    return ok({
      id: "user-1",
      email: "test@example.com",
      displayName: "Test User",
    });
  }

  async logout(): Promise<Result<void, AuthError>> {
    return ok(undefined);
  }
}

// Configurable stub
class AuthRepositoryStub implements AuthRepository {
  private loginResult: Result<User, AuthError> = ok(userBuilder().build());

  withLoginSuccess(user: User): this {
    this.loginResult = ok(user);
    return this;
  }

  withLoginFailure(error: AuthError): this {
    this.loginResult = fail(error);
    return this;
  }

  async login(_params: LoginParams): Promise<Result<User, AuthError>> {
    return this.loginResult;
  }
}

// Usage
const stub = new AuthRepositoryStub().withLoginFailure({
  type: "INVALID_CREDENTIALS",
});
```

### Spy / Mock (Infrastructure only)

```typescript
// Testing an adapter's implementation
describe("AuthApiAdapter", () => {
  it("should call fetch with correct parameters", async () => {
    const fetchSpy = jest.spyOn(global, "fetch").mockResolvedValue({
      ok: true,
      json: async () => ({ id: "1", email: "test@example.com" }),
    } as Response);

    const adapter = new AuthApiAdapter();
    await adapter.login({ email: "test@example.com", password: "password" });

    expect(fetchSpy).toHaveBeenCalledWith(
      "https://api.example.com/auth/login",
      expect.objectContaining({
        method: "POST",
        body: JSON.stringify({
          email: "test@example.com",
          password: "password",
        }),
      })
    );
  });
});
```

## Builder Pattern

Use builders to create test fixtures (entities, props, etc.):

### Entity Builder

```typescript
// modules/authentication/core/entities/User.builder.ts
import { User } from "./User.entity";

export const userBuilder = () => {
  const user: User = {
    id: "user-123",
    email: "default@example.com",
    displayName: "Default User",
    createdAt: "2024-01-01T00:00:00Z",
  };

  const builder = {
    id: (id: string) => {
      user.id = id;
      return builder;
    },
    email: (email: string) => {
      user.email = email;
      return builder;
    },
    displayName: (displayName: string) => {
      user.displayName = displayName;
      return builder;
    },
    createdAt: (createdAt: string) => {
      user.createdAt = createdAt;
      return builder;
    },
    build: () => user,
  };

  return builder;
};

// Usage
const user = userBuilder()
  .email("custom@test.com")
  .displayName("Custom")
  .build();
```

### Props Builder

```typescript
// modules/events/ui/components/EventCard.props.builder.ts
import { EventCardProps } from "./EventCard";

export const eventCardPropsBuilder = () => {
  const props: EventCardProps = {
    title: "Default Event",
    date: "2024-06-15T10:00:00Z",
    attendees: 10,
    onPress: jest.fn(),
  };

  const builder = {
    title: (title: string) => {
      props.title = title;
      return builder;
    },
    date: (date: string) => {
      props.date = date;
      return builder;
    },
    attendees: (attendees: number) => {
      props.attendees = attendees;
      return builder;
    },
    onPress: (onPress: VoidFunction) => {
      props.onPress = onPress;
      return builder;
    },
    build: () => props,
  };

  return builder;
};
```

### Builder naming

| Type   | File                         | Function                  |
| ------ | ---------------------------- | ------------------------- |
| Entity | `User.builder.ts`            | `userBuilder()`           |
| Props  | `EventCard.props.builder.ts` | `eventCardPropsBuilder()` |
| Params | `LoginParams.builder.ts`     | `loginParamsBuilder()`    |

## Testing Hooks

### Custom renderHook

Use the custom `renderHook` that provides dependencies. See [references/test-utils.md](references/test-utils.md) for the full implementation.

```typescript
// modules/app/react/renderHook.tsx
export function renderHook<Result, Props>(
  renderCallback: (props: Props) => Result,
  options?: {
    initialProps?: Props;
    wrapper?: ComponentType<{ children: ReactNode }>;
    dependencies?: Partial<Dependencies>;
  }
): RenderHookResult<Result, Props>;
```

### Usage

```typescript
import { act } from "@testing-library/react-native";
import { renderHook } from "@app/react/renderHook";
import { useLoginViewModel } from "./useLogin.viewModel";

describe("useLoginViewModel", () => {
  it("should update state to success when login succeeds", async () => {
    const authRepositoryStub = new AuthRepositoryStub().withLoginSuccess(
      userBuilder().build()
    );

    const { result } = renderHook(() => useLoginViewModel(), {
      dependencies: { authRepository: authRepositoryStub },
    });

    await act(async () => {
      await result.current.handlers.login("test@example.com", "password123");
    });

    expect(result.current.state.status).toBe("success");
  });
});
```

## Testing Components

### Custom render

Use the custom `render` that provides dependencies. See [references/test-utils.md](references/test-utils.md) for the full implementation.

```typescript
// modules/app/react/render.tsx
export function render(
  component: ReactElement,
  options?: {
    wrapper?: ComponentType<{ children: ReactNode }>;
    dependencies?: Partial<Dependencies>;
  }
): RenderResult;
```

### Usage

```typescript
import { screen, fireEvent } from "@testing-library/react-native";
import { render } from "@app/react/render";
import { LoginScreen } from "./LoginScreen";
import { TestIDs } from "@/constants/testIDs";

describe("LoginScreen", () => {
  it("should disable submit button when form is incomplete", () => {
    render(<LoginScreen />);

    const submitButton = screen.getByTestId(TestIDs.Login.submitButton);
    expect(submitButton.props.accessibilityState.disabled).toBe(true);
  });

  it("should call login when submit is pressed", async () => {
    const authRepositoryStub = new AuthRepositoryStub();

    render(<LoginScreen />, {
      dependencies: { authRepository: authRepositoryStub },
    });

    fireEvent.changeText(
      screen.getByTestId(TestIDs.Login.emailInput),
      "test@example.com"
    );
    fireEvent.changeText(
      screen.getByTestId(TestIDs.Login.passwordInput),
      "password123"
    );
    fireEvent.press(screen.getByTestId(TestIDs.Login.submitButton));

    // Assert expected behavior
  });
});
```

### Query priority

Prefer queries in this order:

1. `getByRole` — accessible queries
2. `getByTestId` — when role is not enough
3. `getByText` — for static text

```typescript
// ✅ Good
screen.getByRole("button", { name: "Submit" });
screen.getByTestId(TestIDs.Login.submitButton);

// ❌ Avoid
screen.getByText("Submit"); // Fragile, breaks on text changes
```

## Test Isolation

### beforeEach / afterEach

Use `beforeEach` and `afterEach` to ensure test isolation and idempotence:

```typescript
describe("useUserProfile", () => {
  let authRepositoryStub: AuthRepositoryStub;

  beforeEach(() => {
    // Fresh stub for each test
    authRepositoryStub = new AuthRepositoryStub();
  });

  afterEach(() => {
    // Cleanup if needed
    jest.clearAllMocks();
  });

  it("should load user", async () => {
    // Test uses fresh authRepositoryStub
  });
});
```

### Zustand Store Reset

Zustand stores must be reset between tests to ensure isolation. See [references/test-utils.md](references/test-utils.md) for the full store template.

### Testing with Zustand

Use `setState` directly to arrange state and reset:

```typescript
import { renderHook } from "@app/react/renderHook";
import { useProfileViewModel } from "./useProfile.viewModel";
import { createAuthStore } from "../stores/auth.store";

describe("useProfileViewModel", () => {
  let authStore: ReturnType<typeof createAuthStore>;

  beforeEach(() => {
    // Create fresh store for each test
    authStore = createAuthStore();
  });

  afterEach(() => {
    // Reset store — `true` replaces state entirely instead of merging
    authStore.setState({ user: null, isAuthenticated: false }, true);
  });

  it("should display user when authenticated", () => {
    // Arrange — set state directly
    authStore.setState({
      user: userBuilder().displayName("John").build(),
      isAuthenticated: true,
    });

    const { result } = renderHook(() => useProfileViewModel(), {
      wrapper: ({ children }) => (
        <AuthStoreProvider store={authStore}>{children}</AuthStoreProvider>
      ),
    });

    // Assert
    expect(result.current.state.user?.displayName).toBe("John");
  });

  it("should show empty state when not authenticated", () => {
    // Store is fresh (created in beforeEach), no need to set anything

    const { result } = renderHook(() => useProfileViewModel(), {
      wrapper: ({ children }) => (
        <AuthStoreProvider store={authStore}>{children}</AuthStoreProvider>
      ),
    });

    expect(result.current.state.user).toBeNull();
  });
});
```

## Anti-patterns

### ❌ Testing implementation details

```typescript
// ❌ Bad — testing internal state
expect(component.state.isLoading).toBe(true);

// ✅ Good — testing behavior
expect(screen.getByTestId("loading-indicator")).toBeTruthy();
```

### ❌ Over-mocking

```typescript
// ❌ Bad — mocking everything
jest.mock("../hooks/useUser");
jest.mock("../hooks/useAuth");
jest.mock("../utils/format");

// ✅ Good — only mock boundaries (adapters)
const authRepository = new AuthRepositoryStub();
```

### ❌ Tests without assertions

```typescript
// ❌ Bad — no assertion, always passes
it("should render", () => {
  render(<Component />);
});

// ✅ Good — explicit assertion
it("should render title", () => {
  render(<Component />);
  expect(screen.getByText("Welcome")).toBeTruthy();
});
```

### ❌ Flaky tests

```typescript
// ❌ Bad — timing dependent
await new Promise((r) => setTimeout(r, 1000));
expect(result).toBe(expected);

// ✅ Good — use waitFor
await waitFor(() => {
  expect(result).toBe(expected);
});
```

### ❌ Tests coupled to each other

```typescript
// ❌ Bad — tests depend on shared state
let user: User;

it("should create user", () => {
  user = createUser();
  expect(user).toBeDefined();
});

it("should update user", () => {
  user.name = "New Name"; // Depends on previous test!
  expect(user.name).toBe("New Name");
});

// ✅ Good — independent tests
it("should create user", () => {
  const user = createUser();
  expect(user).toBeDefined();
});

it("should update user", () => {
  const user = createUser();
  user.name = "New Name";
  expect(user.name).toBe("New Name");
});
```

## References

- [references/test-utils.md](references/test-utils.md) — Full templates for renderHook, render, stores, builders, and stubs
