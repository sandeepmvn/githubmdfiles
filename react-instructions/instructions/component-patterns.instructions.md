---
name: Component Patterns
description: Component architecture and design patterns
applyTo: "**/components/**/*.tsx,**/components/**/*.jsx"
---

# Component Patterns and Architecture

## Component Organization

### Component Folder Structure

Each component should be organized in its own folder:

```
ComponentName/
├── ComponentName.tsx          # Main component file
├── ComponentName.module.css   # Component styles
├── ComponentName.test.tsx     # Unit tests
├── types.ts                   # Component-specific types
├── hooks.ts                   # Component-specific hooks
├── utils.ts                   # Component-specific utilities
└── index.ts                   # Public exports
```

### Index File Pattern

Every component folder must have an index.ts file that exports the public API:

```typescript
// index.ts
export { ComponentName } from './ComponentName';
export type { ComponentNameProps } from './types';
```

This enables clean imports:
```typescript
import { ComponentName } from '@/components/common/ComponentName';
```

## Component Categories

### 1. Common Components (Generic/Reusable)

Location: `src/components/common/`

These are generic, reusable UI components with no business logic.

**Characteristics:**
- Highly reusable across the application
- No direct API calls or business logic
- Controlled via props
- Well-documented prop interface
- Comprehensive test coverage

**Examples:**

```typescript
// Button.tsx
interface ButtonProps {
  label: string;
  variant?: 'primary' | 'secondary' | 'danger' | 'ghost';
  size?: 'small' | 'medium' | 'large';
  disabled?: boolean;
  loading?: boolean;
  icon?: React.ReactNode;
  onClick: () => void;
}

export const Button: React.FC<ButtonProps> = ({
  label,
  variant = 'primary',
  size = 'medium',
  disabled = false,
  loading = false,
  icon,
  onClick
}) => {
  return (
    <button
      className={`btn btn-${variant} btn-${size}`}
      disabled={disabled || loading}
      onClick={onClick}
    >
      {loading && <Spinner />}
      {icon && <span className="btn-icon">{icon}</span>}
      {label}
    </button>
  );
};
```

### 2. Layout Components

Location: `src/components/layout/`

Components that define page structure and layout.

**Examples:**
- Header
- Footer
- Sidebar
- Navigation
- PageLayout

```typescript
// PageLayout.tsx
interface PageLayoutProps {
  children: React.ReactNode;
  sidebar?: React.ReactNode;
  header?: React.ReactNode;
}

export const PageLayout: React.FC<PageLayoutProps> = ({
  children,
  sidebar,
  header
}) => {
  return (
    <div className="page-layout">
      {header && <header className="page-header">{header}</header>}
      <div className="page-content">
        {sidebar && <aside className="page-sidebar">{sidebar}</aside>}
        <main className="page-main">{children}</main>
      </div>
    </div>
  );
};
```

### 3. Feature Components

Location: `src/components/features/` or `src/features/[feature-name]/components/`

Components specific to a particular feature or domain.

**Characteristics:**
- Contains business logic
- May include API calls
- Uses domain-specific hooks and services
- Less reusable than common components

```typescript
// UserProfile.tsx
import { useUser } from '@/features/auth/hooks';
import { Avatar, Card } from '@/components/common';

export const UserProfile: React.FC = () => {
  const { user, updateProfile } = useUser();
  const [isEditing, setIsEditing] = useState(false);
  
  const handleSave = async (data: ProfileData) => {
    await updateProfile(data);
    setIsEditing(false);
  };
  
  if (!user) return <Skeleton />;
  
  return (
    <Card>
      <Avatar src={user.avatar} alt={user.name} />
      {isEditing ? (
        <ProfileForm user={user} onSave={handleSave} />
      ) : (
        <ProfileView user={user} onEdit={() => setIsEditing(true)} />
      )}
    </Card>
  );
};
```

## Design Patterns

### 1. Compound Components

Use for components with tightly coupled children:

```typescript
// Tabs.tsx
interface TabsContextValue {
  selectedIndex: number;
  setSelectedIndex: (index: number) => void;
}

const TabsContext = React.createContext<TabsContextValue | undefined>(undefined);

export const Tabs: React.FC<{ children: React.ReactNode }> = ({ children }) => {
  const [selectedIndex, setSelectedIndex] = useState(0);
  
  return (
    <TabsContext.Provider value={{ selectedIndex, setSelectedIndex }}>
      <div className="tabs">{children}</div>
    </TabsContext.Provider>
  );
};

export const TabList: React.FC<{ children: React.ReactNode }> = ({ children }) => (
  <div className="tab-list" role="tablist">
    {children}
  </div>
);

export const Tab: React.FC<{ children: React.ReactNode; index: number }> = ({
  children,
  index
}) => {
  const context = React.useContext(TabsContext);
  if (!context) throw new Error('Tab must be used within Tabs');
  
  const { selectedIndex, setSelectedIndex } = context;
  
  return (
    <button
      role="tab"
      aria-selected={selectedIndex === index}
      onClick={() => setSelectedIndex(index)}
    >
      {children}
    </button>
  );
};

// Usage
<Tabs>
  <TabList>
    <Tab index={0}>Overview</Tab>
    <Tab index={1}>Details</Tab>
  </TabList>
  <TabPanels>
    <TabPanel>Overview content</TabPanel>
    <TabPanel>Details content</TabPanel>
  </TabPanels>
</Tabs>
```

### 2. Render Props

Use when you need to share logic but keep rendering flexible:

```typescript
interface DataFetcherProps<T> {
  url: string;
  children: (data: {
    data: T | null;
    loading: boolean;
    error: Error | null;
  }) => React.ReactNode;
}

export function DataFetcher<T>({ url, children }: DataFetcherProps<T>) {
  const [data, setData] = useState<T | null>(null);
  const [loading, setLoading] = useState(true);
  const [error, setError] = useState<Error | null>(null);
  
  useEffect(() => {
    fetchData(url)
      .then(setData)
      .catch(setError)
      .finally(() => setLoading(false));
  }, [url]);
  
  return <>{children({ data, loading, error })}</>;
}

// Usage
<DataFetcher<User> url="/api/user/123">
  {({ data, loading, error }) => {
    if (loading) return <Spinner />;
    if (error) return <Error message={error.message} />;
    if (!data) return null;
    return <UserCard user={data} />;
  }}
</DataFetcher>
```

### 3. Higher-Order Components (HOC)

Use sparingly - prefer hooks for most cases:

```typescript
// withAuth.tsx
export function withAuth<P extends object>(
  Component: React.ComponentType<P>
): React.FC<P> {
  return (props: P) => {
    const { isAuthenticated, user } = useAuth();
    
    if (!isAuthenticated) {
      return <Redirect to="/login" />;
    }
    
    return <Component {...props} user={user} />;
  };
}

// Usage
const ProtectedDashboard = withAuth(Dashboard);
```

### 4. Custom Hook Pattern

Extract logic into custom hooks for reusability:

```typescript
// useToggle.ts
export function useToggle(initialValue = false) {
  const [value, setValue] = useState(initialValue);
  
  const toggle = useCallback(() => {
    setValue(v => !v);
  }, []);
  
  const setTrue = useCallback(() => {
    setValue(true);
  }, []);
  
  const setFalse = useCallback(() => {
    setValue(false);
  }, []);
  
  return { value, toggle, setTrue, setFalse };
}

// Usage
const Modal = () => {
  const { value: isOpen, toggle, setFalse } = useToggle();
  
  return (
    <>
      <button onClick={toggle}>Open Modal</button>
      <ModalDialog isOpen={isOpen} onClose={setFalse} />
    </>
  );
};
```

### 5. Controlled vs Uncontrolled Components

**Controlled Components** (Preferred):
```typescript
interface InputProps {
  value: string;
  onChange: (value: string) => void;
}

export const Input: React.FC<InputProps> = ({ value, onChange }) => (
  <input
    value={value}
    onChange={(e) => onChange(e.target.value)}
  />
);

// Usage
const [name, setName] = useState('');
<Input value={name} onChange={setName} />
```

**Uncontrolled Components** (Use for file inputs or when integration with non-React code):
```typescript
export const FileInput: React.FC = () => {
  const fileInputRef = useRef<HTMLInputElement>(null);
  
  const handleSubmit = () => {
    const files = fileInputRef.current?.files;
    // process files
  };
  
  return (
    <>
      <input type="file" ref={fileInputRef} />
      <button onClick={handleSubmit}>Upload</button>
    </>
  );
};
```

## Component Composition

### Favor Composition Over Props

Instead of many conditional props:

```typescript
// Bad - Too many props
<Button 
  withIcon 
  iconPosition="left" 
  icon={<Icon />}
  withSpinner
  variant="primary"
/>

// Good - Composition
<Button variant="primary">
  <Icon />
  <Spinner />
  Submit
</Button>
```

### Slot Pattern

```typescript
interface CardProps {
  header?: React.ReactNode;
  footer?: React.ReactNode;
  children: React.ReactNode;
}

export const Card: React.FC<CardProps> = ({ header, footer, children }) => (
  <div className="card">
    {header && <div className="card-header">{header}</div>}
    <div className="card-body">{children}</div>
    {footer && <div className="card-footer">{footer}</div>}
  </div>
);

// Usage
<Card
  header={<h2>Title</h2>}
  footer={<Button onClick={handleSave}>Save</Button>}
>
  <p>Content here</p>
</Card>
```

## Component Communication

### Parent-Child Communication

**Parent to Child:** Use props
```typescript
<ChildComponent data={parentData} onAction={handleAction} />
```

**Child to Parent:** Use callback props
```typescript
interface ChildProps {
  onComplete: (result: Result) => void;
}

// In parent
const handleComplete = (result: Result) => {
  console.log('Child completed with:', result);
};

<Child onComplete={handleComplete} />
```

### Sibling Communication

Use a common parent to coordinate:

```typescript
const Parent = () => {
  const [sharedState, setSharedState] = useState<State>();
  
  return (
    <>
      <SiblingA state={sharedState} onChange={setSharedState} />
      <SiblingB state={sharedState} onChange={setSharedState} />
    </>
  );
};
```

### Deep/Distant Component Communication

Use Context API or state management:

```typescript
// Context approach
const ThemeContext = createContext<ThemeContextValue | undefined>(undefined);

export const ThemeProvider: React.FC<{ children: ReactNode }> = ({ children }) => {
  const [theme, setTheme] = useState<Theme>('light');
  
  return (
    <ThemeContext.Provider value={{ theme, setTheme }}>
      {children}
    </ThemeContext.Provider>
  );
};

export const useTheme = () => {
  const context = useContext(ThemeContext);
  if (!context) throw new Error('useTheme must be used within ThemeProvider');
  return context;
};
```

## Component Best Practices

### 1. Single Responsibility
Each component should do one thing well.

### 2. Prop Drilling Limit
Don't pass props through more than 2-3 levels. Use Context or state management instead.

### 3. Component Size
Keep components under 300 lines. Extract sub-components or logic into hooks.

### 4. Pure Components
Strive for pure components that produce the same output for the same props.

### 5. Accessibility
- Use semantic HTML
- Include ARIA attributes
- Ensure keyboard navigation
- Test with screen readers

### 6. Error Handling
Wrap feature areas with error boundaries:

```typescript
<ErrorBoundary fallback={<ErrorPage />}>
  <FeatureArea />
</ErrorBoundary>
```

### 7. Loading States
Always handle loading states:

```typescript
if (loading) return <Skeleton />;
if (error) return <ErrorMessage error={error} />;
if (!data) return <EmptyState />;
return <DataView data={data} />;
```

### 8. Type Safety
- Define prop types clearly
- Use TypeScript generics for flexible components
- Avoid `any` type
- Export prop types for consumers
