# File Templates

Reference templates for each file type.

## Entity

```typescript
// modules/[context]/core/entities/[Name].entity.ts

export interface User {
  id: string;
  email: string;
  displayName: string;
  createdAt: ISO8601;
}
```

## Error Entity

```typescript
// modules/[context]/core/entities/[Name]Error.entity.ts

export type AuthError =
  | { type: "INVALID_CREDENTIALS" }
  | { type: "NETWORK_ERROR" }
  | { type: "UNAUTHORIZED" }
  | { type: "VALIDATION_ERROR"; message: string };
```

## Port

```typescript
// modules/[context]/core/ports/[Name]Repository.port.ts

import { Result } from "@/types/Result";
import { User } from "../entities/User.entity";
import { AuthError } from "../entities/AuthError.entity";

export interface LoginParams {
  email: string;
  password: string;
}

export interface AuthRepository {
  login(params: LoginParams): Promise<Result<User, AuthError>>;
  logout(): Promise<Result<void, AuthError>>;
  getCurrentUser(): Promise<Result<User | null, AuthError>>;
}
```

## Use Case

```typescript
// modules/[context]/core/usecases/[Name].usecase.ts

import { Result, ok, fail } from "@/types/Result";
import { User } from "../entities/User.entity";
import { AuthError } from "../entities/AuthError.entity";
import { AuthRepository, LoginParams } from "../ports/AuthRepository.port";

export class LoginUseCase {
  constructor(private authRepository: AuthRepository) {}

  async execute(params: LoginParams): Promise<Result<User, AuthError>> {
    // Business validation
    if (!params.email.includes("@")) {
      return fail({ type: "VALIDATION_ERROR", message: "Invalid email" });
    }

    if (params.password.length < 8) {
      return fail({ type: "VALIDATION_ERROR", message: "Password too short" });
    }

    // Delegate to repository
    return this.authRepository.login(params);
  }
}
```

## Adapter

```typescript
// modules/[context]/infrastructure/adapters/[Name]Api.adapter.ts

import { Result, ok, fail } from "@/types/Result";
import { User } from "../../core/entities/User.entity";
import { AuthError } from "../../core/entities/AuthError.entity";
import { AuthRepository, LoginParams } from "../../core/ports/AuthRepository.port";
import { LoginApiResponse } from "./LoginApiResponse.model";

export class AuthApiAdapter implements AuthRepository {
  private baseUrl = "https://api.example.com";

  async login(params: LoginParams): Promise<Result<User, AuthError>> {
    try {
      const response = await fetch(`${this.baseUrl}/auth/login`, {
        method: "POST",
        headers: { "Content-Type": "application/json" },
        body: JSON.stringify(params),
      });

      if (response.status === 401) {
        return fail({ type: "INVALID_CREDENTIALS" });
      }

      if (!response.ok) {
        return fail({ type: "NETWORK_ERROR" });
      }

      const data: LoginApiResponse = await response.json();
      return ok(this.mapToEntity(data));
    } catch {
      return fail({ type: "NETWORK_ERROR" });
    }
  }

  private mapToEntity(response: LoginApiResponse): User {
    return {
      id: response.id,
      email: response.email,
      displayName: response.display_name,
      createdAt: response.created_at,
    };
  }

  // ... other methods
}
```

## API Response Model

```typescript
// modules/[context]/infrastructure/adapters/[Name]ApiResponse.model.ts

export interface LoginApiResponse {
  id: string;
  email: string;
  display_name: string; // snake_case from API
  created_at: string;
}
```

## Query Keys

```typescript
// modules/[context]/ui/hooks/[entity].queryKeys.ts

export const usersKeys = {
  all: ["users"] as const,
  lists: () => [...usersKeys.all, "list"] as const,
  list: (filters: UserFilters) => [...usersKeys.lists(), filters] as const,
  details: () => [...usersKeys.all, "detail"] as const,
  detail: (id: string) => [...usersKeys.details(), id] as const,
};
```

## Query Hook

```typescript
// modules/[context]/ui/hooks/use[Entity].query.ts

import { useQuery } from "@tanstack/react-query";
import { useDependencies } from "@app/react/useDependencies";
import { usersKeys } from "./users.queryKeys";

export const useUserQuery = (id: string) => {
  const { userRepository } = useDependencies();

  return useQuery({
    queryKey: usersKeys.detail(id),
    queryFn: () => userRepository.getById(id),
    enabled: !!id,
  });
};
```

## Mutation Hook

```typescript
// modules/[context]/ui/hooks/use[Action][Entity].mutation.ts

import { useMutation, useQueryClient } from "@tanstack/react-query";
import { useDependencies } from "@app/react/useDependencies";
import { UpdateUserUseCase } from "../../core/usecases/UpdateUser.usecase";
import { usersKeys } from "./users.queryKeys";

export const useUpdateUserMutation = () => {
  const { userRepository } = useDependencies();
  const queryClient = useQueryClient();

  const updateUserUseCase = new UpdateUserUseCase(userRepository);

  return useMutation({
    mutationFn: updateUserUseCase.execute.bind(updateUserUseCase),
    onSuccess: (_, variables) => {
      queryClient.invalidateQueries({ queryKey: usersKeys.detail(variables.id) });
      queryClient.invalidateQueries({ queryKey: usersKeys.lists() });
    },
  });
};
```

## ViewModel

```typescript
// modules/[context]/ui/viewModels/use[Feature].viewModel.tsx

import { useState } from "react";
import { useUserQuery } from "../hooks/useUser.query";
import { useUpdateUserMutation } from "../hooks/useUpdateUser.mutation";

export const useUserProfileViewModel = (userId: string) => {
  const userQuery = useUserQuery(userId);
  const updateMutation = useUpdateUserMutation();
  const [isEditing, setIsEditing] = useState(false);

  const state = {
    user: userQuery.data ?? null,
    isLoading: userQuery.isLoading,
    error: userQuery.error,
    isEditing,
    isSaving: updateMutation.isPending,
  };

  const handlers = {
    startEditing: () => setIsEditing(true),
    cancelEditing: () => setIsEditing(false),
    save: (data: UpdateUserParams) => {
      updateMutation.mutate(
        { id: userId, ...data },
        { onSuccess: () => setIsEditing(false) }
      );
    },
    refresh: () => userQuery.refetch(),
  };

  return { state, handlers };
};
```

## Store (Zustand)

```typescript
// modules/[context]/ui/stores/[name].store.ts

import { createStore } from "zustand";

interface UIState {
  sidebarOpen: boolean;
  selectedTab: string;
}

interface UIActions {
  toggleSidebar: () => void;
  selectTab: (tab: string) => void;
}

type UIStore = UIState & UIActions;

export const createUIStore = (initialState?: Partial<UIState>) => {
  const DEFAULT_STATE: UIState = {
    sidebarOpen: false,
    selectedTab: "home",
  };

  return createStore<UIStore>()((set) => ({
    ...DEFAULT_STATE,
    ...initialState,
    toggleSidebar: () => set((state) => ({ sidebarOpen: !state.sidebarOpen })),
    selectTab: (tab) => set({ selectedTab: tab }),
  }));
};
```

## Screen

```typescript
// modules/[context]/ui/screens/[Name]Screen.tsx

import { View, Text } from "react-native";
import { useUserProfileViewModel } from "../viewModels/useUserProfile.viewModel";
import { LoadingSpinner } from "@ui/components/atoms/LoadingSpinner";
import { ErrorMessage } from "@ui/components/molecules/ErrorMessage";

export const UserProfileScreen = ({ route }) => {
  const { userId } = route.params;
  const { state, handlers } = useUserProfileViewModel(userId);

  if (state.isLoading) {
    return <LoadingSpinner />;
  }

  if (state.error) {
    return <ErrorMessage message={state.error.message} onRetry={handlers.refresh} />;
  }

  return (
    <View>
      <Text>{state.user?.displayName}</Text>
      {/* ... */}
    </View>
  );
};
```
