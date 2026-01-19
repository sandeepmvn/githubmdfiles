---
name: Performance Guidelines
description: Performance optimization best practices
applyTo: "**/*.tsx,**/*.ts,**/*.jsx,**/*.js"
---

# Performance Optimization Guidelines

## Core Performance Principles

1. **Measure First** - Use profiling tools before optimizing
2. **Optimize What Matters** - Focus on bottlenecks that impact users
3. **Avoid Premature Optimization** - Don't sacrifice code clarity for marginal gains
4. **Monitor Continuously** - Track performance metrics over time

## React Performance Optimization

### 1. React.memo

Use `React.memo` to prevent unnecessary re-renders of components:

```typescript
// Without memo - re-renders on every parent render
export const ExpensiveComponent: React.FC<Props> = ({ data }) => {
  // expensive calculations
  return <div>{/* JSX */}</div>;
};

// With memo - only re-renders when props change
export const ExpensiveComponent = React.memo<Props>(({ data }) => {
  // expensive calculations
  return <div>{/* JSX */}</div>;
});

// Custom comparison function
export const ExpensiveComponent = React.memo<Props>(
  ({ data, metadata }) => {
    return <div>{/* JSX */}</div>;
  },
  (prevProps, nextProps) => {
    // Return true if passing nextProps to render would return same result as prevProps
    return prevProps.data.id === nextProps.data.id;
  }
);
```

**When to use React.memo:**
- Component renders often with same props
- Component has expensive render logic
- Component receives complex props that don't change frequently

**When NOT to use:**
- Props change frequently
- Component is already fast
- Component has children that change

### 2. useMemo

Memoize expensive calculations:

```typescript
// Without useMemo - recalculates on every render
const sortedData = items.sort((a, b) => a.value - b.value);

// With useMemo - only recalculates when items change
const sortedData = useMemo(
  () => items.sort((a, b) => a.value - b.value),
  [items]
);

// Complex example
const expensiveValue = useMemo(() => {
  console.log('Calculating...');
  return items
    .filter(item => item.active)
    .map(item => item.value * 2)
    .reduce((sum, val) => sum + val, 0);
}, [items]);
```

**When to use useMemo:**
- Expensive calculations (sorting, filtering large arrays)
- Creating new object/array references that cause child re-renders
- Calculations that depend on props/state

**When NOT to use:**
- Simple calculations (they're fast anyway)
- Values that change on every render
- Everywhere (measure first!)

### 3. useCallback

Memoize callback functions to prevent child re-renders:

```typescript
// Without useCallback - new function on every render
const handleClick = () => {
  console.log('Clicked', userId);
};

// With useCallback - same function reference unless userId changes
const handleClick = useCallback(() => {
  console.log('Clicked', userId);
}, [userId]);

// Passing to memoized child
const MemoizedChild = React.memo(Child);

const Parent = () => {
  const [count, setCount] = useState(0);
  
  // Without useCallback, Child re-renders on every count change
  const handleAction = useCallback(() => {
    doSomething();
  }, []); // No dependencies - function never changes
  
  return (
    <>
      <button onClick={() => setCount(count + 1)}>Count: {count}</button>
      <MemoizedChild onAction={handleAction} />
    </>
  );
};
```

**When to use useCallback:**
- Passing callbacks to memoized children
- Callbacks used in dependency arrays of other hooks
- Event handlers in frequently re-rendering components

**When NOT to use:**
- Simple inline handlers
- Callbacks not passed to children
- Before measuring performance impact

### 4. Code Splitting and Lazy Loading

Split code at route boundaries:

```typescript
import { lazy, Suspense } from 'react';
import { Routes, Route } from 'react-router-dom';

// Eager loading (everything loads upfront)
import Dashboard from './pages/Dashboard';
import Profile from './pages/Profile';

// Lazy loading (loads on demand)
const Dashboard = lazy(() => import('./pages/Dashboard'));
const Profile = lazy(() => import('./pages/Profile'));
const Settings = lazy(() => import('./pages/Settings'));

function App() {
  return (
    <Suspense fallback={<PageLoader />}>
      <Routes>
        <Route path="/dashboard" element={<Dashboard />} />
        <Route path="/profile" element={<Profile />} />
        <Route path="/settings" element={<Settings />} />
      </Routes>
    </Suspense>
  );
}
```

**Component-level lazy loading:**

```typescript
const HeavyChart = lazy(() => import('./components/HeavyChart'));

function Dashboard() {
  const [showChart, setShowChart] = useState(false);
  
  return (
    <div>
      <button onClick={() => setShowChart(true)}>Show Chart</button>
      
      {showChart && (
        <Suspense fallback={<Spinner />}>
          <HeavyChart data={chartData} />
        </Suspense>
      )}
    </div>
  );
}
```

### 5. Virtualization for Long Lists

Use virtualization for lists with >50 items:

```typescript
import { FixedSizeList } from 'react-window';

interface ItemData {
  id: string;
  name: string;
}

const Row: React.FC<{ index: number; style: React.CSSProperties; data: ItemData[] }> = ({
  index,
  style,
  data
}) => (
  <div style={style}>
    {data[index].name}
  </div>
);

function VirtualList({ items }: { items: ItemData[] }) {
  return (
    <FixedSizeList
      height={600}
      itemCount={items.length}
      itemSize={50}
      width="100%"
      itemData={items}
    >
      {Row}
    </FixedSizeList>
  );
}
```

**Libraries:**
- `react-window` (lightweight, recommended)
- `react-virtualized` (feature-rich but larger)
- `@tanstack/react-virtual` (modern, flexible)

### 6. Debouncing and Throttling

**Debounce** - Wait until user stops typing:

```typescript
import { debounce } from 'lodash';

function SearchInput() {
  const [query, setQuery] = useState('');
  
  // Debounce search - only search after 300ms of no typing
  const debouncedSearch = useMemo(
    () => debounce((searchQuery: string) => {
      performSearch(searchQuery);
    }, 300),
    []
  );
  
  const handleChange = (e: React.ChangeEvent<HTMLInputElement>) => {
    const value = e.target.value;
    setQuery(value);
    debouncedSearch(value);
  };
  
  // Cleanup
  useEffect(() => {
    return () => {
      debouncedSearch.cancel();
    };
  }, [debouncedSearch]);
  
  return <input value={query} onChange={handleChange} />;
}
```

**Throttle** - Limit function calls to once per interval:

```typescript
import { throttle } from 'lodash';

function ScrollTracker() {
  const handleScroll = useMemo(
    () => throttle((event: Event) => {
      const scrollPosition = window.scrollY;
      trackScrollPosition(scrollPosition);
    }, 100), // Maximum once per 100ms
    []
  );
  
  useEffect(() => {
    window.addEventListener('scroll', handleScroll);
    return () => {
      window.removeEventListener('scroll', handleScroll);
      handleScroll.cancel();
    };
  }, [handleScroll]);
  
  return <div>{/* content */}</div>;
}
```

### 7. Image Optimization

**Lazy loading images:**

```typescript
// Native lazy loading
<img src="large-image.jpg" loading="lazy" alt="Description" />

// Custom lazy loading with Intersection Observer
function LazyImage({ src, alt }: { src: string; alt: string }) {
  const [imageSrc, setImageSrc] = useState<string>();
  const imgRef = useRef<HTMLImageElement>(null);
  
  useEffect(() => {
    if (!imgRef.current) return;
    
    const observer = new IntersectionObserver(
      ([entry]) => {
        if (entry.isIntersecting) {
          setImageSrc(src);
          observer.disconnect();
        }
      },
      { threshold: 0.1 }
    );
    
    observer.observe(imgRef.current);
    
    return () => observer.disconnect();
  }, [src]);
  
  return <img ref={imgRef} src={imageSrc} alt={alt} />;
}
```

**Responsive images:**

```typescript
<picture>
  <source media="(min-width: 1200px)" srcSet="image-large.webp" type="image/webp" />
  <source media="(min-width: 768px)" srcSet="image-medium.webp" type="image/webp" />
  <source media="(min-width: 1200px)" srcSet="image-large.jpg" />
  <source media="(min-width: 768px)" srcSet="image-medium.jpg" />
  <img src="image-small.jpg" alt="Description" loading="lazy" />
</picture>
```

### 8. State Management Optimization

**Keep state close to where it's used:**

```typescript
// Bad - Global state for local concern
const globalState = {
  modalOpen: false,
  selectedTab: 0,
  // ... other state
};

// Good - Local state for local concern
function Modal() {
  const [isOpen, setIsOpen] = useState(false);
  return <>{/* modal */}</>;
}
```

**Split context to avoid unnecessary re-renders:**

```typescript
// Bad - Single context causes all consumers to re-render
const AppContext = createContext({
  user: null,
  theme: 'light',
  notifications: [],
  settings: {}
});

// Good - Separate contexts
const UserContext = createContext(null);
const ThemeContext = createContext('light');
const NotificationsContext = createContext([]);

// Even better - Use state management library (Redux, Zustand)
```

**Normalize state structure:**

```typescript
// Bad - Nested, hard to update
const state = {
  posts: [
    { id: 1, author: { id: 1, name: 'John' }, comments: [...] },
    { id: 2, author: { id: 1, name: 'John' }, comments: [...] }
  ]
};

// Good - Normalized
const state = {
  posts: {
    byId: {
      1: { id: 1, authorId: 1, commentIds: [1, 2] },
      2: { id: 2, authorId: 1, commentIds: [3] }
    },
    allIds: [1, 2]
  },
  users: {
    byId: {
      1: { id: 1, name: 'John' }
    }
  },
  comments: {
    byId: {
      1: { id: 1, text: 'Great post!' },
      2: { id: 2, text: 'Thanks!' },
      3: { id: 3, text: 'Awesome!' }
    }
  }
};
```

## Bundle Optimization

### 1. Analyze Bundle Size

```bash
# Install bundle analyzer
npm install --save-dev @bundle-analyzer/webpack-plugin

# For Vite
npm install --save-dev rollup-plugin-visualizer
```

```typescript
// vite.config.ts
import { visualizer } from 'rollup-plugin-visualizer';

export default defineConfig({
  plugins: [
    react(),
    visualizer({
      open: true,
      gzipSize: true,
      brotliSize: true
    })
  ]
});
```

### 2. Tree Shaking

Import only what you need:

```typescript
// Bad - Imports entire library
import _ from 'lodash';
import * as dateFns from 'date-fns';

// Good - Imports only needed functions
import debounce from 'lodash/debounce';
import { format } from 'date-fns';
```

### 3. Dynamic Imports

Load heavy libraries only when needed:

```typescript
// Load chart library only when showing chart
const loadChart = async () => {
  const { Chart } = await import('chart.js');
  return Chart;
};

function ChartComponent() {
  const [Chart, setChart] = useState<any>(null);
  
  useEffect(() => {
    loadChart().then(setChart);
  }, []);
  
  if (!Chart) return <Spinner />;
  
  return <Chart {...chartProps} />;
}
```

## Network Performance

### 1. API Request Optimization

**Request caching:**

```typescript
const cache = new Map<string, { data: any; timestamp: number }>();
const CACHE_DURATION = 5 * 60 * 1000; // 5 minutes

async function fetchWithCache<T>(url: string): Promise<T> {
  const cached = cache.get(url);
  
  if (cached && Date.now() - cached.timestamp < CACHE_DURATION) {
    return cached.data;
  }
  
  const data = await fetch(url).then(res => res.json());
  cache.set(url, { data, timestamp: Date.now() });
  
  return data;
}
```

**Request deduplication:**

```typescript
const pendingRequests = new Map<string, Promise<any>>();

async function fetchWithDedup<T>(url: string): Promise<T> {
  if (pendingRequests.has(url)) {
    return pendingRequests.get(url);
  }
  
  const request = fetch(url)
    .then(res => res.json())
    .finally(() => pendingRequests.delete(url));
  
  pendingRequests.set(url, request);
  return request;
}
```

**Pagination:**

```typescript
function InfiniteList() {
  const [items, setItems] = useState<Item[]>([]);
  const [page, setPage] = useState(1);
  const [hasMore, setHasMore] = useState(true);
  
  const loadMore = async () => {
    const newItems = await fetchItems(page);
    setItems(prev => [...prev, ...newItems]);
    setPage(prev => prev + 1);
    setHasMore(newItems.length > 0);
  };
  
  return (
    <InfiniteScroll
      loadMore={loadMore}
      hasMore={hasMore}
      loader={<Spinner />}
    >
      {items.map(item => <ItemCard key={item.id} item={item} />)}
    </InfiniteScroll>
  );
}
```

### 2. Prefetching

```typescript
import { useEffect } from 'react';
import { Link } from 'react-router-dom';

// Prefetch on hover
function PrefetchLink({ to, children }: { to: string; children: React.ReactNode }) {
  const prefetchRoute = () => {
    // Prefetch route component
    import(`./pages${to}`);
  };
  
  return (
    <Link to={to} onMouseEnter={prefetchRoute}>
      {children}
    </Link>
  );
}

// Prefetch data on route
function Dashboard() {
  useEffect(() => {
    // Prefetch data for likely next pages
    prefetchData('/api/user/profile');
    prefetchData('/api/user/settings');
  }, []);
  
  return <>{/* content */}</>;
}
```

## Rendering Performance

### 1. Avoid Inline Objects and Arrays

```typescript
// Bad - Creates new object on every render
<Component style={{ margin: 10 }} items={[1, 2, 3]} />

// Good - Define outside component or use useMemo
const style = { margin: 10 };
const items = [1, 2, 3];
<Component style={style} items={items} />

// Or with useMemo for computed values
const items = useMemo(() => computeItems(), [dependencies]);
```

### 2. Key Prop Optimization

```typescript
// Bad - Index as key (causes issues on reorder)
{items.map((item, index) => (
  <Item key={index} {...item} />
))}

// Good - Stable unique identifier
{items.map(item => (
  <Item key={item.id} {...item} />
))}

// For static lists, index is OK
{STATIC_MENU_ITEMS.map((item, index) => (
  <MenuItem key={index} {...item} />
))}
```

### 3. Conditional Rendering Optimization

```typescript
// Bad - Component always mounts
<div>
  {showModal && <HeavyModal />}
</div>

// Good - Component only mounts when needed
{showModal && <HeavyModal />}

// Better - With Suspense for code splitting
{showModal && (
  <Suspense fallback={<Spinner />}>
    <LazyModal />
  </Suspense>
)}
```

## Performance Monitoring

### 1. React DevTools Profiler

```typescript
import { Profiler } from 'react';

function onRenderCallback(
  id: string,
  phase: 'mount' | 'update',
  actualDuration: number
) {
  console.log(`${id} (${phase}) took ${actualDuration}ms`);
}

function App() {
  return (
    <Profiler id="App" onRender={onRenderCallback}>
      <Dashboard />
    </Profiler>
  );
}
```

### 2. Web Vitals

```typescript
import { getCLS, getFID, getFCP, getLCP, getTTFB } from 'web-vitals';

getCLS(console.log); // Cumulative Layout Shift
getFID(console.log); // First Input Delay
getFCP(console.log); // First Contentful Paint
getLCP(console.log); // Largest Contentful Paint
getTTFB(console.log); // Time to First Byte
```

### 3. Performance Budgets

Set and enforce performance budgets:

```json
// performance-budget.json
{
  "bundles": [
    {
      "resourceSizes": [
        {
          "resourceType": "script",
          "budget": 300
        },
        {
          "resourceType": "total",
          "budget": 500
        }
      ]
    }
  ]
}
```

## Performance Checklist

- [ ] Use React.memo for frequently rendered components
- [ ] Memoize expensive calculations with useMemo
- [ ] Memoize callbacks with useCallback
- [ ] Implement code splitting for routes
- [ ] Virtualize long lists (>50 items)
- [ ] Debounce input handlers
- [ ] Lazy load images
- [ ] Optimize images (WebP, proper sizing)
- [ ] Analyze and reduce bundle size
- [ ] Cache API responses
- [ ] Implement pagination for large datasets
- [ ] Avoid inline object/array creation
- [ ] Use stable keys for lists
- [ ] Monitor Core Web Vitals
- [ ] Profile components with React DevTools
