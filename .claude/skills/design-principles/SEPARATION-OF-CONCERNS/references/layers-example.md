# Separation of Concerns Example

## Before — Mixed Concerns

```typescript
// UserProfileScreen.tsx — does everything
function UserProfileScreen({ route }: Props) {
  const { userId } = route.params;
  const [user, setUser] = useState<User | null>(null);
  const [loading, setLoading] = useState(true);
  const [editing, setEditing] = useState(false);
  const [formData, setFormData] = useState({ name: '', email: '' });

  useEffect(() => {
    // Data fetching in component
    const fetchUser = async () => {
      try {
        const response = await fetch(`https://api.example.com/users/${userId}`, {
          headers: { 
            'Authorization': `Bearer ${await AsyncStorage.getItem('token')}` 
          }
        });
        const data = await response.json();
        
        // Business logic in component
        if (data.status === 'suspended') {
          Alert.alert('Account Suspended', 'Contact support');
          navigation.goBack();
          return;
        }
        
        // Data transformation in component
        setUser({
          ...data,
          fullName: `${data.firstName} ${data.lastName}`,
          memberSince: new Date(data.createdAt).toLocaleDateString(),
        });
        setFormData({ name: data.firstName, email: data.email });
      } catch (error) {
        // Error handling mixed with UI
        Alert.alert('Error', 'Failed to load profile');
      } finally {
        setLoading(false);
      }
    };
    fetchUser();
  }, [userId]);

  const handleSave = async () => {
    // Validation in component
    if (!formData.name.trim()) {
      Alert.alert('Error', 'Name is required');
      return;
    }
    if (!formData.email.includes('@')) {
      Alert.alert('Error', 'Invalid email');
      return;
    }

    // More data fetching
    try {
      await fetch(`https://api.example.com/users/${userId}`, {
        method: 'PATCH',
        headers: {
          'Content-Type': 'application/json',
          'Authorization': `Bearer ${await AsyncStorage.getItem('token')}`
        },
        body: JSON.stringify(formData)
      });
      
      // Analytics in component
      analytics.track('profile_updated', { userId });
      
      setEditing(false);
    } catch (error) {
      Alert.alert('Error', 'Failed to save');
    }
  };

  // 200 more lines of mixed concerns...
  
  return (
    <View>
      {/* Rendering */}
    </View>
  );
}
```

## After — Separated Concerns

### Domain Layer

```typescript
// domain/entities/User.ts — pure domain object
interface User {
  id: string;
  firstName: string;
  lastName: string;
  email: string;
  status: UserStatus;
  createdAt: Date;
}

// domain/ports/UserPort.ts — abstract data access
interface UserPort {
  getById(id: string): Promise<User>;
  update(id: string, data: UpdateUserDTO): Promise<User>;
}

// domain/usecases/GetUserProfileUseCase.ts — business logic
class GetUserProfileUseCase {
  constructor(private readonly userPort: UserPort) {}

  async execute(userId: string): Promise<UserProfile> {
    const user = await this.userPort.getById(userId);
    
    if (user.status === 'suspended') {
      throw new UserSuspendedError(userId);
    }

    return {
      ...user,
      fullName: `${user.firstName} ${user.lastName}`,
      memberSince: user.createdAt,
    };
  }
}

// domain/usecases/UpdateUserProfileUseCase.ts
class UpdateUserProfileUseCase {
  constructor(
    private readonly userPort: UserPort,
    private readonly validator: UserValidator
  ) {}

  async execute(userId: string, data: UpdateProfileDTO): Promise<User> {
    const validation = this.validator.validateProfileUpdate(data);
    if (!validation.isValid) {
      throw new ValidationError(validation.errors);
    }

    return this.userPort.update(userId, data);
  }
}
```

### Infrastructure Layer

```typescript
// infrastructure/adapters/ApiUserAdapter.ts — concrete data access
class ApiUserAdapter implements UserPort {
  constructor(private readonly httpClient: HttpClient) {}

  async getById(id: string): Promise<User> {
    const response = await this.httpClient.get<UserDTO>(`/users/${id}`);
    return this.toDomain(response);
  }

  async update(id: string, data: UpdateUserDTO): Promise<User> {
    const response = await this.httpClient.patch<UserDTO>(`/users/${id}`, data);
    return this.toDomain(response);
  }

  private toDomain(dto: UserDTO): User {
    return {
      ...dto,
      createdAt: new Date(dto.createdAt),
    };
  }
}
```

### Presentation Layer

```typescript
// presentation/hooks/useUserProfile.ts — state management
function useUserProfile(userId: string) {
  const getUserProfile = useGetUserProfileUseCase();
  const updateUserProfile = useUpdateUserProfileUseCase();
  const navigation = useNavigation();

  const profileQuery = useQuery({
    queryKey: ['user', userId],
    queryFn: () => getUserProfile.execute(userId),
  });

  const updateMutation = useMutation({
    mutationFn: (data: UpdateProfileDTO) => 
      updateUserProfile.execute(userId, data),
    onSuccess: () => {
      profileQuery.refetch();
      analytics.track('profile_updated', { userId });
    },
  });

  useEffect(() => {
    if (profileQuery.error instanceof UserSuspendedError) {
      Alert.alert('Account Suspended', 'Contact support');
      navigation.goBack();
    }
  }, [profileQuery.error]);

  return {
    profile: profileQuery.data,
    isLoading: profileQuery.isLoading,
    error: profileQuery.error,
    updateProfile: updateMutation.mutate,
    isUpdating: updateMutation.isPending,
  };
}

// presentation/screens/UserProfileScreen.tsx — pure UI
function UserProfileScreen({ route }: Props) {
  const { userId } = route.params;
  const { profile, isLoading, updateProfile, isUpdating } = useUserProfile(userId);

  if (isLoading) return <LoadingSpinner />;
  if (!profile) return <ErrorView />;

  return (
    <ScrollView>
      <ProfileHeader profile={profile} />
      <ProfileForm
        initialValues={profile}
        onSubmit={updateProfile}
        isSubmitting={isUpdating}
      />
    </ScrollView>
  );
}

// presentation/components/ProfileForm.tsx — reusable UI component
function ProfileForm({ initialValues, onSubmit, isSubmitting }: ProfileFormProps) {
  const [formData, setFormData] = useState(initialValues);

  return (
    <View>
      <TextInput
        value={formData.name}
        onChangeText={(name) => setFormData(prev => ({ ...prev, name }))}
        placeholder="Name"
      />
      <TextInput
        value={formData.email}
        onChangeText={(email) => setFormData(prev => ({ ...prev, email }))}
        placeholder="Email"
        keyboardType="email-address"
      />
      <Button
        title="Save"
        onPress={() => onSubmit(formData)}
        disabled={isSubmitting}
      />
    </View>
  );
}
```
