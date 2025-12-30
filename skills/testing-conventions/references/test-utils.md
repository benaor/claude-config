# Test Utilities

Templates for test utilities with dependency injection.

## renderHook

Custom `renderHook` that provides dependencies via context:

```typescript
// modules/app/react/renderHook.tsx
import {
  renderHook as originalRenderHook,
  RenderHookResult,
} from "@testing-library/react-native";
import { ComponentType, ReactNode } from "react";
import { Dependencies } from "@app/dependencies/Dependencies.type";
import { DependenciesProvider } from "@app/react/useDependencies";

const Wrapper: ComponentType<{
  children: ReactNode;
  dependencies?: Partial<Dependencies>;
  ExtendedWrapper?: ComponentType<{ children: ReactNode }>;
}> = ({ children, dependencies, ExtendedWrapper }) => (
  <DependenciesProvider dependencies={dependencies}>
    {ExtendedWrapper ? (
      <ExtendedWrapper>{children}</ExtendedWrapper>
    ) : (
      children
    )}
  </DependenciesProvider>
);

export type RenderHookOptions<Props> = {
  initialProps?: Props;
  wrapper?: ComponentType<{ children: ReactNode }>;
  dependencies?: Partial<Dependencies>;
};

export function renderHook<Result, Props>(
  renderCallback: (props: Props) => Result,
  options?: RenderHookOptions<Props>,
): RenderHookResult<Result, Props> {
  return originalRenderHook(renderCallback, {
    initialProps: options?.initialProps,
    wrapper: ({ children }) => (
      <Wrapper
        dependencies={options?.dependencies}
        ExtendedWrapper={options?.wrapper}
      >
        {children}
      </Wrapper>
    ),
  });
}
```

## render

Custom `render` that provides dependencies via context:

```typescript
// modules/app/react/render.tsx
import {
  render as originalRender,
  RenderResult,
} from "@testing-library/react-native";
import { ComponentType, ReactElement, ReactNode } from "react";
import { Dependencies } from "@app/dependencies/Dependencies.type";
import { DependenciesProvider } from "@app/react/useDependencies";

const Wrapper: ComponentType<{
  children: ReactNode;
  dependencies?: Partial<Dependencies>;
  ExtendedWrapper?: ComponentType<{ children: ReactNode }>;
}> = ({ children, dependencies, ExtendedWrapper }) => (
  <DependenciesProvider dependencies={dependencies}>
    {ExtendedWrapper ? (
      <ExtendedWrapper>{children}</ExtendedWrapper>
    ) : (
      children
    )}
  </DependenciesProvider>
);

export type RenderOptions = {
  wrapper?: ComponentType<{ children: ReactNode }>;
  dependencies?: Partial<Dependencies>;
};

export function render(
  component: ReactElement,
  options?: RenderOptions,
): RenderResult {
  return originalRender(component, {
    wrapper: ({ children }) => (
      <Wrapper
        dependencies={options?.dependencies}
        ExtendedWrapper={options?.wrapper}
      >
        {children}
      </Wrapper>
    ),
  });
}
```

## Zustand Store Template

Store template with initial state for easy reset in tests:

```typescript
// modules/[context]/ui/stores/[name].store.ts
import { createStore } from "zustand";

interface AuthState {
  user: User | null;
  isAuthenticated: boolean;
}

interface AuthActions {
  setUser: (user: User) => void;
  logout: VoidFunction;
}

type AuthStore = AuthState & AuthActions;

// Export for test reset
export const initialAuthState: AuthState = {
  user: null,
  isAuthenticated: false,
};

export const createAuthStore = (preloadedState?: Partial<AuthState>) => {
  return createStore<AuthStore>()((set) => ({
    ...initialAuthState,
    ...preloadedState,
    setUser: (user) => set({ user, isAuthenticated: true }),
    logout: () => set(initialAuthState),
  }));
};
```

## Builder Template

Generic builder pattern for test fixtures:

```typescript
// modules/[context]/core/entities/[Entity].builder.ts
import { User } from "./User.entity";

export const userBuilder = () => {
  const entity: User = {
    id: "user-123",
    email: "default@example.com",
    displayName: "Default User",
    createdAt: "2024-01-01T00:00:00Z",
  };

  const builder = {
    id: (value: string) => {
      entity.id = value;
      return builder;
    },
    email: (value: string) => {
      entity.email = value;
      return builder;
    },
    displayName: (value: string) => {
      entity.displayName = value;
      return builder;
    },
    createdAt: (value: string) => {
      entity.createdAt = value;
      return builder;
    },
    build: () => entity,
  };

  return builder;
};
```

## Stub Template

Configurable stub for repositories:

```typescript
// modules/[context]/core/ports/[Name]Repository.stub.ts
import { Result, ok, fail } from "@/types/Result";
import { User } from "../entities/User.entity";
import { AuthError } from "../entities/AuthError.entity";
import { AuthRepository, LoginParams } from "./AuthRepository.port";
import { userBuilder } from "../entities/User.builder";

export class AuthRepositoryStub implements AuthRepository {
  private loginResult: Result<User, AuthError> = ok(userBuilder().build());
  private logoutResult: Result<void, AuthError> = ok(undefined);

  // Configurable responses
  withLoginSuccess(user: User): this {
    this.loginResult = ok(user);
    return this;
  }

  withLoginFailure(error: AuthError): this {
    this.loginResult = fail(error);
    return this;
  }

  withLogoutSuccess(): this {
    this.logoutResult = ok(undefined);
    return this;
  }

  withLogoutFailure(error: AuthError): this {
    this.logoutResult = fail(error);
    return this;
  }

  // Port implementation
  async login(_params: LoginParams): Promise<Result<User, AuthError>> {
    return this.loginResult;
  }

  async logout(): Promise<Result<void, AuthError>> {
    return this.logoutResult;
  }
}
```
