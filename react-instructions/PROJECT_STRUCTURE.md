# React Project Directory Structure

```
project-root/
├── .github/
│   ├── copilot-instructions.md              # Workspace-wide instructions
│   ├── instructions/
│   │   ├── react-coding.instructions.md     # React coding standards
│   │   ├── component-patterns.instructions.md
│   │   ├── testing.instructions.md
│   │   ├── performance.instructions.md
│   │   ├── documentation.instructions.md
│   │   └── code-review.instructions.md
│   └── workflows/                            # GitHub Actions
│       ├── ci.yml
│       └── deploy.yml
│
├── public/
│   ├── index.html
│   ├── favicon.ico
│   └── assets/                               # Static assets
│       ├── images/
│       ├── fonts/
│       └── icons/
│
├── src/
│   ├── assets/                               # Application assets
│   │   ├── images/
│   │   ├── icons/
│   │   ├── fonts/
│   │   └── styles/
│   │       ├── global.css
│   │       ├── variables.css
│   │       └── mixins.css
│   │
│   ├── components/                           # Reusable components
│   │   ├── common/                          # Generic components
│   │   │   ├── Button/
│   │   │   │   ├── Button.tsx
│   │   │   │   ├── Button.module.css
│   │   │   │   ├── Button.test.tsx
│   │   │   │   └── index.ts
│   │   │   ├── Input/
│   │   │   ├── Modal/
│   │   │   └── index.ts
│   │   │
│   │   ├── layout/                          # Layout components
│   │   │   ├── Header/
│   │   │   ├── Footer/
│   │   │   ├── Sidebar/
│   │   │   └── index.ts
│   │   │
│   │   └── features/                        # Feature-specific components
│   │       ├── auth/
│   │       ├── dashboard/
│   │       └── index.ts
│   │
│   ├── pages/                                # Page components
│   │   ├── Home/
│   │   │   ├── Home.tsx
│   │   │   ├── Home.module.css
│   │   │   ├── Home.test.tsx
│   │   │   └── index.ts
│   │   ├── About/
│   │   ├── Dashboard/
│   │   └── index.ts
│   │
│   ├── features/                             # Feature modules
│   │   ├── auth/
│   │   │   ├── components/
│   │   │   ├── hooks/
│   │   │   ├── services/
│   │   │   ├── store/
│   │   │   ├── types/
│   │   │   └── utils/
│   │   └── index.ts
│   │
│   ├── hooks/                                # Custom React hooks
│   │   ├── useAuth.ts
│   │   ├── useFetch.ts
│   │   ├── useDebounce.ts
│   │   └── index.ts
│   │
│   ├── context/                              # React Context providers
│   │   ├── AuthContext.tsx
│   │   ├── ThemeContext.tsx
│   │   └── index.ts
│   │
│   ├── store/                                # State management
│   │   ├── slices/                          # Redux slices or Zustand stores
│   │   │   ├── authSlice.ts
│   │   │   ├── userSlice.ts
│   │   │   └── index.ts
│   │   ├── middleware/
│   │   └── index.ts
│   │
│   ├── services/                             # API services
│   │   ├── api/
│   │   │   ├── client.ts                    # Axios/fetch instance
│   │   │   ├── endpoints.ts
│   │   │   └── interceptors.ts
│   │   ├── auth/
│   │   │   ├── authService.ts
│   │   │   └── index.ts
│   │   └── index.ts
│   │
│   ├── utils/                                # Utility functions
│   │   ├── helpers/
│   │   │   ├── formatters.ts
│   │   │   ├── validators.ts
│   │   │   └── index.ts
│   │   ├── constants/
│   │   │   ├── routes.ts
│   │   │   ├── config.ts
│   │   │   └── index.ts
│   │   └── index.ts
│   │
│   ├── types/                                # TypeScript types
│   │   ├── api.types.ts
│   │   ├── user.types.ts
│   │   ├── common.types.ts
│   │   └── index.ts
│   │
│   ├── routes/                               # Routing configuration
│   │   ├── routes.tsx
│   │   ├── PrivateRoute.tsx
│   │   ├── PublicRoute.tsx
│   │   └── index.ts
│   │
│   ├── config/                               # Configuration files
│   │   ├── env.ts
│   │   ├── theme.ts
│   │   └── index.ts
│   │
│   ├── tests/                                # Test utilities
│   │   ├── setup.ts
│   │   ├── mocks/
│   │   ├── helpers/
│   │   └── fixtures/
│   │
│   ├── App.tsx                               # Root component
│   ├── App.test.tsx
│   ├── main.tsx                              # Entry point
│   └── vite-env.d.ts                         # Vite type declarations
│
├── docs/                                     # Documentation
│   ├── architecture.md
│   ├── api.md
│   ├── deployment.md
│   └── contributing.md
│
├── .vscode/                                  # VS Code settings
│   ├── settings.json
│   ├── extensions.json
│   └── launch.json
│
├── .husky/                                   # Git hooks
│   ├── pre-commit
│   └── pre-push
│
├── .eslintrc.js                              # ESLint config
├── .prettierrc                               # Prettier config
├── tsconfig.json                             # TypeScript config
├── vite.config.ts                            # Vite config
├── package.json
├── .gitignore
├── .env.example
└── README.md
```

## Folder Descriptions

### `/src/components/`
Reusable UI components organized by type:
- **common/** - Generic, reusable components (Button, Input, Card)
- **layout/** - Page layout components (Header, Footer, Sidebar)
- **features/** - Feature-specific components

### `/src/pages/`
Route-level page components. Each page should be a folder containing the component, styles, and tests.

### `/src/features/`
Feature modules following a vertical slice architecture. Each feature contains its own components, hooks, services, and state.

### `/src/hooks/`
Custom React hooks for shared logic across components.

### `/src/services/`
API integration and external service communication.

### `/src/store/`
Global state management (Redux, Zustand, or similar).

### `/src/utils/`
Utility functions, constants, and helpers.

### `/src/types/`
TypeScript type definitions and interfaces.

### `/src/routes/`
Routing configuration and route guards.

## Module Organization Pattern

Each component/feature should follow this structure:

```
ComponentName/
├── ComponentName.tsx          # Component logic
├── ComponentName.module.css   # Component styles
├── ComponentName.test.tsx     # Component tests
├── types.ts                   # Local types
├── hooks.ts                   # Local hooks
└── index.ts                   # Public exports
```

## Index Files

Each folder should have an `index.ts` file that exports public APIs:

```typescript
// components/common/index.ts
export { Button } from './Button';
export { Input } from './Input';
export { Modal } from './Modal';
```

This enables clean imports:
```typescript
import { Button, Input, Modal } from '@/components/common';
```
