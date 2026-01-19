---
name: Testing Standards
description: Testing guidelines and best practices
applyTo: "**/*.test.tsx,**/*.test.ts,**/*.spec.tsx,**/*.spec.ts"
---

# Testing Standards and Best Practices

## Testing Philosophy

- Write tests that give confidence, not just coverage
- Test behavior, not implementation details
- Write tests that are easy to maintain
- Follow the Testing Trophy approach: Many integration tests, some unit tests, few E2E tests

## Test Structure

### Arrange-Act-Assert Pattern

```typescript
describe('ComponentName', () => {
  it('should perform expected behavior', () => {
    // Arrange - Set up test data and conditions
    const mockData = { id: 1, name: 'Test' };
    const handleClick = jest.fn();
    
    // Act - Execute the behavior
    render(<Component data={mockData} onClick={handleClick} />);
    fireEvent.click(screen.getByRole('button'));
    
    // Assert - Verify the outcome
    expect(handleClick).toHaveBeenCalledWith(mockData);
  });
});
```

## Testing Library Setup

### Test File Organization

```
ComponentName/
├── ComponentName.tsx
├── ComponentName.test.tsx        # Component tests
├── ComponentName.integration.test.tsx  # Integration tests
└── __mocks__/
    └── mockData.ts              # Test data
```

### Test Utilities Setup

```typescript
// src/tests/test-utils.tsx
import { render, RenderOptions } from '@testing-library/react';
import { BrowserRouter } from 'react-router-dom';
import { Provider } from 'react-redux';
import { store } from '@/store';

interface WrapperProps {
  children: React.ReactNode;
}

const AllTheProviders: React.FC<WrapperProps> = ({ children }) => {
  return (
    <Provider store={store}>
      <BrowserRouter>
        {children}
      </BrowserRouter>
    </Provider>
  );
};

const customRender = (
  ui: React.ReactElement,
  options?: Omit<RenderOptions, 'wrapper'>
) => render(ui, { wrapper: AllTheProviders, ...options });

export * from '@testing-library/react';
export { customRender as render };
```

## Component Testing

### Testing Functional Components

```typescript
import { render, screen, fireEvent, waitFor } from '@/tests/test-utils';
import { UserProfile } from './UserProfile';

describe('UserProfile', () => {
  const mockUser = {
    id: '1',
    name: 'John Doe',
    email: 'john@example.com'
  };
  
  it('renders user information correctly', () => {
    render(<UserProfile user={mockUser} />);
    
    expect(screen.getByText(mockUser.name)).toBeInTheDocument();
    expect(screen.getByText(mockUser.email)).toBeInTheDocument();
  });
  
  it('calls onEdit when edit button is clicked', () => {
    const handleEdit = jest.fn();
    render(<UserProfile user={mockUser} onEdit={handleEdit} />);
    
    fireEvent.click(screen.getByRole('button', { name: /edit/i }));
    
    expect(handleEdit).toHaveBeenCalledTimes(1);
    expect(handleEdit).toHaveBeenCalledWith(mockUser.id);
  });
  
  it('shows loading state while fetching data', () => {
    render(<UserProfile user={null} isLoading={true} />);
    
    expect(screen.getByTestId('loading-spinner')).toBeInTheDocument();
    expect(screen.queryByText(mockUser.name)).not.toBeInTheDocument();
  });
  
  it('displays error message when error occurs', () => {
    const errorMessage = 'Failed to load user';
    render(<UserProfile error={errorMessage} />);
    
    expect(screen.getByRole('alert')).toHaveTextContent(errorMessage);
  });
});
```

### Testing User Interactions

```typescript
import { render, screen, fireEvent, waitFor } from '@/tests/test-utils';
import userEvent from '@testing-library/user-event';

describe('LoginForm', () => {
  it('submits form with valid credentials', async () => {
    const user = userEvent.setup();
    const handleSubmit = jest.fn();
    
    render(<LoginForm onSubmit={handleSubmit} />);
    
    // Type into inputs
    await user.type(screen.getByLabelText(/email/i), 'test@example.com');
    await user.type(screen.getByLabelText(/password/i), 'password123');
    
    // Submit form
    await user.click(screen.getByRole('button', { name: /sign in/i }));
    
    // Verify submission
    await waitFor(() => {
      expect(handleSubmit).toHaveBeenCalledWith({
        email: 'test@example.com',
        password: 'password123'
      });
    });
  });
  
  it('shows validation errors for invalid inputs', async () => {
    const user = userEvent.setup();
    render(<LoginForm onSubmit={jest.fn()} />);
    
    // Submit without filling fields
    await user.click(screen.getByRole('button', { name: /sign in/i }));
    
    // Check for error messages
    expect(await screen.findByText(/email is required/i)).toBeInTheDocument();
    expect(await screen.findByText(/password is required/i)).toBeInTheDocument();
  });
});
```

### Testing Async Behavior

```typescript
import { render, screen, waitFor } from '@/tests/test-utils';
import { fetchUsers } from '@/services/api';

jest.mock('@/services/api');
const mockFetchUsers = fetchUsers as jest.MockedFunction<typeof fetchUsers>;

describe('UserList', () => {
  it('loads and displays users', async () => {
    const mockUsers = [
      { id: '1', name: 'Alice' },
      { id: '2', name: 'Bob' }
    ];
    
    mockFetchUsers.mockResolvedValueOnce(mockUsers);
    
    render(<UserList />);
    
    // Initially shows loading
    expect(screen.getByText(/loading/i)).toBeInTheDocument();
    
    // Wait for users to load
    await waitFor(() => {
      expect(screen.getByText('Alice')).toBeInTheDocument();
      expect(screen.getByText('Bob')).toBeInTheDocument();
    });
    
    // Loading should be gone
    expect(screen.queryByText(/loading/i)).not.toBeInTheDocument();
  });
  
  it('handles API errors gracefully', async () => {
    mockFetchUsers.mockRejectedValueOnce(new Error('API Error'));
    
    render(<UserList />);
    
    await waitFor(() => {
      expect(screen.getByText(/failed to load users/i)).toBeInTheDocument();
    });
  });
});
```

## Hook Testing

### Testing Custom Hooks

```typescript
import { renderHook, act, waitFor } from '@testing-library/react';
import { useAuth } from './useAuth';

describe('useAuth', () => {
  it('initializes with unauthenticated state', () => {
    const { result } = renderHook(() => useAuth());
    
    expect(result.current.isAuthenticated).toBe(false);
    expect(result.current.user).toBeNull();
  });
  
  it('authenticates user on login', async () => {
    const { result } = renderHook(() => useAuth());
    
    await act(async () => {
      await result.current.login({ email: 'test@example.com', password: 'pass' });
    });
    
    expect(result.current.isAuthenticated).toBe(true);
    expect(result.current.user).toBeTruthy();
  });
  
  it('clears user on logout', async () => {
    const { result } = renderHook(() => useAuth());
    
    // Login first
    await act(async () => {
      await result.current.login({ email: 'test@example.com', password: 'pass' });
    });
    
    // Then logout
    act(() => {
      result.current.logout();
    });
    
    expect(result.current.isAuthenticated).toBe(false);
    expect(result.current.user).toBeNull();
  });
});
```

## Mocking

### Mocking API Calls

```typescript
// Option 1: Mock entire module
jest.mock('@/services/api', () => ({
  fetchUser: jest.fn(),
  updateUser: jest.fn()
}));

// Option 2: Mock specific implementation
import * as api from '@/services/api';
jest.spyOn(api, 'fetchUser').mockResolvedValue(mockUser);

// Option 3: Mock with MSW (Mock Service Worker)
import { rest } from 'msw';
import { setupServer } from 'msw/node';

const server = setupServer(
  rest.get('/api/users', (req, res, ctx) => {
    return res(ctx.json([{ id: 1, name: 'John' }]));
  })
);

beforeAll(() => server.listen());
afterEach(() => server.resetHandlers());
afterAll(() => server.close());
```

### Mocking Context

```typescript
import { render, screen } from '@/tests/test-utils';
import { AuthContext } from '@/context/AuthContext';

const mockAuthContext = {
  user: { id: '1', name: 'Test User' },
  isAuthenticated: true,
  login: jest.fn(),
  logout: jest.fn()
};

describe('ProtectedComponent', () => {
  it('renders for authenticated users', () => {
    render(
      <AuthContext.Provider value={mockAuthContext}>
        <ProtectedComponent />
      </AuthContext.Provider>
    );
    
    expect(screen.getByText(/welcome/i)).toBeInTheDocument();
  });
});
```

### Mocking Hooks

```typescript
import { useNavigate } from 'react-router-dom';

jest.mock('react-router-dom', () => ({
  ...jest.requireActual('react-router-dom'),
  useNavigate: jest.fn()
}));

const mockNavigate = useNavigate as jest.MockedFunction<typeof useNavigate>;

describe('Component', () => {
  it('navigates on button click', () => {
    const navigate = jest.fn();
    mockNavigate.mockReturnValue(navigate);
    
    render(<Component />);
    fireEvent.click(screen.getByRole('button'));
    
    expect(navigate).toHaveBeenCalledWith('/destination');
  });
});
```

## Accessibility Testing

```typescript
import { render } from '@/tests/test-utils';
import { axe, toHaveNoViolations } from 'jest-axe';

expect.extend(toHaveNoViolations);

describe('Button', () => {
  it('should not have accessibility violations', async () => {
    const { container } = render(<Button label="Click me" onClick={jest.fn()} />);
    const results = await axe(container);
    
    expect(results).toHaveNoViolations();
  });
  
  it('has proper ARIA attributes', () => {
    render(<Button label="Submit" loading={true} onClick={jest.fn()} />);
    
    const button = screen.getByRole('button');
    expect(button).toHaveAttribute('aria-busy', 'true');
  });
});
```

## Snapshot Testing

Use sparingly and with purpose:

```typescript
import { render } from '@/tests/test-utils';

describe('Card', () => {
  it('matches snapshot', () => {
    const { container } = render(
      <Card title="Test Card">
        <p>Content</p>
      </Card>
    );
    
    expect(container.firstChild).toMatchSnapshot();
  });
});
```

**When to use snapshots:**
- Complex UI structures that rarely change
- Error messages and validation states
- Generated markup from data

**When NOT to use snapshots:**
- Rapidly changing components
- Components with dynamic content
- As primary test method

## Integration Testing

```typescript
import { render, screen, waitFor } from '@/tests/test-utils';
import { UserDashboard } from './UserDashboard';
import { setupServer } from 'msw/node';
import { rest } from 'msw';

const server = setupServer(
  rest.get('/api/user/:userId', (req, res, ctx) => {
    return res(ctx.json({ id: '1', name: 'John Doe' }));
  }),
  rest.get('/api/user/:userId/posts', (req, res, ctx) => {
    return res(ctx.json([
      { id: '1', title: 'Post 1' },
      { id: '2', title: 'Post 2' }
    ]));
  })
);

beforeAll(() => server.listen());
afterEach(() => server.resetHandlers());
afterAll(() => server.close());

describe('UserDashboard Integration', () => {
  it('loads user data and posts together', async () => {
    render(<UserDashboard userId="1" />);
    
    // Check loading state
    expect(screen.getByText(/loading/i)).toBeInTheDocument();
    
    // Wait for all data to load
    await waitFor(() => {
      expect(screen.getByText('John Doe')).toBeInTheDocument();
      expect(screen.getByText('Post 1')).toBeInTheDocument();
      expect(screen.getByText('Post 2')).toBeInTheDocument();
    });
  });
});
```

## Testing Best Practices

### 1. Query Priority

Use queries in this order:
1. `getByRole` (most accessible)
2. `getByLabelText` (forms)
3. `getByPlaceholderText` (forms)
4. `getByText` (content)
5. `getByDisplayValue` (form fields)
6. `getByTestId` (last resort)

```typescript
// Good
screen.getByRole('button', { name: /submit/i })
screen.getByLabelText(/email/i)

// Avoid if possible
screen.getByTestId('submit-button')
```

### 2. Async Testing

Always use `waitFor` for async operations:

```typescript
// Good
await waitFor(() => {
  expect(screen.getByText('Loaded')).toBeInTheDocument();
});

// Bad - flaky
setTimeout(() => {
  expect(screen.getByText('Loaded')).toBeInTheDocument();
}, 1000);
```

### 3. Cleanup

Tests automatically cleanup between runs, but for manual cleanup:

```typescript
import { cleanup } from '@testing-library/react';

afterEach(() => {
  cleanup();
  jest.clearAllMocks();
});
```

### 4. Test Isolation

Each test should be independent:

```typescript
// Good - independent tests
describe('Counter', () => {
  it('increments count', () => {
    render(<Counter />);
    // test logic
  });
  
  it('decrements count', () => {
    render(<Counter />);
    // test logic
  });
});

// Bad - tests depend on each other
let count = 0;
it('increments', () => {
  count++;
  expect(count).toBe(1);
});
it('increments again', () => {
  count++;
  expect(count).toBe(2); // Depends on previous test
});
```

### 5. Descriptive Test Names

```typescript
// Good
it('displays error message when email is invalid', () => {});
it('disables submit button while form is submitting', () => {});

// Bad
it('works correctly', () => {});
it('test 1', () => {});
```

### 6. Coverage Goals

- Aim for >80% code coverage
- 100% coverage of critical paths
- Focus on meaningful tests, not just coverage numbers

### 7. Test Organization

```typescript
describe('UserProfile', () => {
  describe('when user is authenticated', () => {
    it('shows user information', () => {});
    it('allows editing profile', () => {});
  });
  
  describe('when user is not authenticated', () => {
    it('redirects to login', () => {});
  });
  
  describe('error handling', () => {
    it('shows error message on API failure', () => {});
  });
});
```

## Performance Testing

```typescript
import { render } from '@/tests/test-utils';

describe('LargeList Performance', () => {
  it('renders large lists efficiently', () => {
    const items = Array.from({ length: 1000 }, (_, i) => ({
      id: i,
      name: `Item ${i}`
    }));
    
    const startTime = performance.now();
    render(<VirtualList items={items} />);
    const endTime = performance.now();
    
    expect(endTime - startTime).toBeLessThan(100); // Should render in <100ms
  });
});
```

## Common Testing Patterns

### Testing Forms

```typescript
it('validates and submits form', async () => {
  const user = userEvent.setup();
  const handleSubmit = jest.fn();
  
  render(<ContactForm onSubmit={handleSubmit} />);
  
  await user.type(screen.getByLabelText(/name/i), 'John Doe');
  await user.type(screen.getByLabelText(/email/i), 'invalid-email');
  await user.click(screen.getByRole('button', { name: /submit/i }));
  
  // Validation error should appear
  expect(await screen.findByText(/invalid email/i)).toBeInTheDocument();
  expect(handleSubmit).not.toHaveBeenCalled();
  
  // Fix email and resubmit
  await user.clear(screen.getByLabelText(/email/i));
  await user.type(screen.getByLabelText(/email/i), 'john@example.com');
  await user.click(screen.getByRole('button', { name: /submit/i }));
  
  await waitFor(() => {
    expect(handleSubmit).toHaveBeenCalledWith({
      name: 'John Doe',
      email: 'john@example.com'
    });
  });
});
```

### Testing Modal Dialogs

```typescript
it('opens and closes modal', async () => {
  const user = userEvent.setup();
  render(<ModalExample />);
  
  // Modal should be closed initially
  expect(screen.queryByRole('dialog')).not.toBeInTheDocument();
  
  // Open modal
  await user.click(screen.getByRole('button', { name: /open modal/i }));
  expect(screen.getByRole('dialog')).toBeInTheDocument();
  
  // Close modal
  await user.click(screen.getByRole('button', { name: /close/i }));
  await waitFor(() => {
    expect(screen.queryByRole('dialog')).not.toBeInTheDocument();
  });
});
```
