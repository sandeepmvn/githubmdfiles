---
name: React Coding Standards
description: React-specific coding guidelines and best practices
applyTo: "**/*.tsx,**/*.jsx"
---

# React Coding Standards

Apply the [general coding guidelines](../copilot-instructions.md) to all code.

## Component Architecture

### Functional Components

- Always use functional components with hooks
- Avoid class components unless absolutely necessary
- Use React.FC type for components with children (TypeScript)
- Keep components focused on a single responsibility
- Limit component size to 200-300 lines maximum

### Component Structure

Follow this order within components:

```typescript
// 1. Imports
import React, { useState, useEffect } from 'react';
import type { ComponentProps } from './types';

// 2. Types/Interfaces
interface Props extends ComponentProps {
  // component-specific props
}

// 3. Constants
const DEFAULT_TIMEOUT = 3000;

// 4. Component Definition
export const ComponentName: React.FC<Props> = ({ prop1, prop2 }) => {
  // 5. Hooks (in this order)
  // - useState
  // - useReducer
  // - useContext
  // - useRef
  // - Custom hooks
  // - useEffect
  
  // 6. Event Handlers
  const handleClick = () => {
    // logic
  };
  
  // 7. Helper Functions
  const formatData = (data: Data) => {
    // logic
  };
  
  // 8. Early Returns
  if (loading) return <Loader />;
  if (error) return <Error message={error} />;
  
  // 9. Main Render
  return (
    <div>
      {/* JSX */}
    </div>
  );
};

// 10. Default Props (if needed)
ComponentName.defaultProps = {
  // defaults
};
```

## React Hooks Guidelines

### Rules of Hooks

- Only call hooks at the top level (never in loops, conditions, or nested functions)
- Only call hooks from React function components or custom hooks
- Name custom hooks with "use" prefix
- Always include all dependencies in useEffect dependency arrays

### useState

- Use descriptive names for state variables
- Initialize state with appropriate default values
- Use functional updates when new state depends on previous state
- Consider useReducer for complex state logic

```typescript
// Good
const [isLoading, setIsLoading] = useState(false);
const [user, setUser] = useState<User | null>(null);

// Functional update
setCount(prevCount => prevCount + 1);

// Bad
const [data, setData] = useState(); // unclear name
const [x, setX] = useState(0); // non-descriptive
```

### useEffect

- Keep effects focused on a single concern
- Always clean up side effects (timers, subscriptions, listeners)
- Use separate useEffect hooks for different concerns
- Add comments explaining what the effect does
- Avoid infinite loops by properly managing dependencies

```typescript
// Good - Focused effect with cleanup
useEffect(() => {
  // Fetch user data on mount
  const controller = new AbortController();
  
  fetchUser(userId, { signal: controller.signal })
    .then(setUser)
    .catch(handleError);
  
  return () => {
    controller.abort(); // Cleanup
  };
}, [userId]);

// Bad - Multiple concerns in one effect
useEffect(() => {
  fetchUser();
  subscribeToNotifications();
  setupWebSocket();
  // Hard to track dependencies and cleanup
}, []);
```

### Custom Hooks

- Extract reusable logic into custom hooks
- Return objects or arrays with clear, descriptive names
- Document hook parameters and return values
- Include all necessary dependencies

```typescript
// Good custom hook
function useAuth() {
  const [user, setUser] = useState<User | null>(null);
  const [isAuthenticated, setIsAuthenticated] = useState(false);
  
  const login = async (credentials: Credentials) => {
    const user = await authService.login(credentials);
    setUser(user);
    setIsAuthenticated(true);
  };
  
  const logout = () => {
    authService.logout();
    setUser(null);
    setIsAuthenticated(false);
  };
  
  return { user, isAuthenticated, login, logout };
}
```

### useMemo and useCallback

- Use useMemo for expensive calculations only
- Use useCallback to prevent unnecessary re-renders of child components
- Don't over-optimize - measure before memoizing
- Include all dependencies

```typescript
// Good - Memoize expensive calculation
const sortedItems = useMemo(
  () => items.sort((a, b) => a.price - b.price),
  [items]
);

// Good - Prevent child re-render
const handleSubmit = useCallback(
  (data: FormData) => {
    submitForm(data, userId);
  },
  [userId]
);
```

## Props Management

### Props Definition

- Define props interface at the top of the file
- Use TypeScript interfaces for props
- Mark optional props with ? operator
- Provide default values for optional props
- Use destructuring in function parameters

```typescript
interface ButtonProps {
  label: string;
  onClick: () => void;
  variant?: 'primary' | 'secondary' | 'danger';
  disabled?: boolean;
  icon?: React.ReactNode;
}

export const Button: React.FC<ButtonProps> = ({
  label,
  onClick,
  variant = 'primary',
  disabled = false,
  icon
}) => {
  // component logic
};
```

### Props Drilling

- Avoid passing props through more than 2-3 levels
- Use Context API for deeply nested components
- Consider component composition over prop drilling
- Use state management (Redux/Zustand) for global state

## JSX Best Practices

### Conditional Rendering

```typescript
// Good - Clear and concise
{isLoading && <Spinner />}
{error ? <ErrorMessage error={error} /> : <Content />}
{items.length > 0 ? (
  <ItemList items={items} />
) : (
  <EmptyState />
)}

// Bad - Confusing ternaries
{condition ? component1 : condition2 ? component2 : component3}
```

### List Rendering

- Always provide unique, stable keys
- Never use array index as key unless list is static
- Use semantic keys (IDs from data)

```typescript
// Good
{users.map(user => (
  <UserCard key={user.id} user={user} />
))}

// Bad
{users.map((user, index) => (
  <UserCard key={index} user={user} />
))}
```

### Event Handlers

- Use arrow functions in JSX for handlers with arguments
- Define handler functions outside JSX when possible
- Use descriptive handler names (handleClick, handleSubmit)

```typescript
// Good
const handleDeleteUser = (userId: string) => {
  deleteUser(userId);
};

return (
  <button onClick={() => handleDeleteUser(user.id)}>
    Delete
  </button>
);

// Bad - Inline logic
<button onClick={() => {
  // multiple lines of logic
  // hard to read and test
}}>
  Delete
</button>
```

## Component Composition

### Container/Presentational Pattern

- Separate logic (containers) from UI (presentational components)
- Presentational components receive data via props
- Container components manage state and side effects

```typescript
// Presentational Component
interface UserListProps {
  users: User[];
  onSelectUser: (user: User) => void;
}

export const UserList: React.FC<UserListProps> = ({ users, onSelectUser }) => (
  <ul>
    {users.map(user => (
      <li key={user.id} onClick={() => onSelectUser(user)}>
        {user.name}
      </li>
    ))}
  </ul>
);

// Container Component
export const UserListContainer: React.FC = () => {
  const [users, setUsers] = useState<User[]>([]);
  const [selectedUser, setSelectedUser] = useState<User | null>(null);
  
  useEffect(() => {
    fetchUsers().then(setUsers);
  }, []);
  
  return <UserList users={users} onSelectUser={setSelectedUser} />;
};
```

### Compound Components

Use compound components for related UI elements:

```typescript
// Good compound component pattern
<Tabs>
  <TabList>
    <Tab>Profile</Tab>
    <Tab>Settings</Tab>
  </TabList>
  <TabPanels>
    <TabPanel>Profile content</TabPanel>
    <TabPanel>Settings content</TabPanel>
  </TabPanels>
</Tabs>
```

## Performance Optimization

### React.memo

- Use for components that render often with same props
- Provide custom comparison function if needed
- Don't memoize everything - measure first

```typescript
export const ExpensiveComponent = React.memo<Props>(
  ({ data, onAction }) => {
    // component logic
  },
  (prevProps, nextProps) => {
    // Custom comparison
    return prevProps.data.id === nextProps.data.id;
  }
);
```

### Code Splitting

- Use React.lazy for route-level code splitting
- Implement Suspense boundaries with fallbacks
- Split on route boundaries, not component boundaries

```typescript
import { lazy, Suspense } from 'react';

const Dashboard = lazy(() => import('./pages/Dashboard'));
const Profile = lazy(() => import('./pages/Profile'));

function App() {
  return (
    <Suspense fallback={<LoadingSpinner />}>
      <Routes>
        <Route path="/dashboard" element={<Dashboard />} />
        <Route path="/profile" element={<Profile />} />
      </Routes>
    </Suspense>
  );
}
```

## Error Handling

### Error Boundaries

- Implement error boundaries for critical UI sections
- Provide fallback UI for errors
- Log errors to monitoring service

```typescript
import { Component, ErrorInfo, ReactNode } from 'react';

interface Props {
  children: ReactNode;
  fallback?: ReactNode;
}

interface State {
  hasError: boolean;
  error?: Error;
}

export class ErrorBoundary extends Component<Props, State> {
  state: State = { hasError: false };
  
  static getDerivedStateFromError(error: Error): State {
    return { hasError: true, error };
  }
  
  componentDidCatch(error: Error, errorInfo: ErrorInfo) {
    console.error('Error boundary caught:', error, errorInfo);
    // Log to error tracking service
  }
  
  render() {
    if (this.state.hasError) {
      return this.props.fallback || <ErrorFallback />;
    }
    
    return this.props.children;
  }
}
```

## Testing Guidelines

- Write tests for all components
- Test user interactions, not implementation details
- Use React Testing Library best practices
- Mock external dependencies
- Aim for >80% code coverage

```typescript
import { render, screen, fireEvent } from '@testing-library/react';
import { Button } from './Button';

describe('Button', () => {
  it('calls onClick when clicked', () => {
    const handleClick = jest.fn();
    render(<Button label="Click me" onClick={handleClick} />);
    
    fireEvent.click(screen.getByRole('button'));
    
    expect(handleClick).toHaveBeenCalledTimes(1);
  });
});
```
