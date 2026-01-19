# Project Coding Standards

This file contains workspace-wide coding standards that apply to all code in this React project.

## Project Overview

This is a modern React application built with TypeScript, Vite, and best practices for scalable, maintainable code. The project follows a feature-based architecture with emphasis on component reusability, type safety, and performance optimization.

## Technology Stack

- **Framework:** React 18+
- **Language:** TypeScript 5+
- **Build Tool:** Vite
- **State Management:** Redux Toolkit / Zustand
- **Routing:** React Router v6
- **Styling:** CSS Modules / Tailwind CSS
- **Testing:** Vitest + React Testing Library
- **Code Quality:** ESLint + Prettier
- **Package Manager:** npm / pnpm / yarn

## General Coding Standards

### Naming Conventions

- Use **PascalCase** for component names, interfaces, type aliases, and enums
- Use **camelCase** for variables, functions, methods, and props
- Use **SCREAMING_SNAKE_CASE** for constants and environment variables
- Prefix private class members with underscore (_)
- Use descriptive, meaningful names that convey intent
- Avoid abbreviations unless widely recognized (e.g., URL, API, HTTP)

### File Naming

- Component files: **PascalCase** (e.g., `UserProfile.tsx`)
- Utility files: **camelCase** (e.g., `formatDate.ts`)
- Test files: **ComponentName.test.tsx** or **fileName.test.ts**
- Style files: **ComponentName.module.css**
- Type files: **camelCase.types.ts**

### Error Handling

- Always use try-catch blocks for async operations
- Implement proper error boundaries in React components
- Log errors with contextual information (timestamp, user action, error details)
- Never silently catch errors - always handle or propagate them
- Use custom error classes for different error types
- Provide user-friendly error messages in the UI

### Code Quality

- Keep functions small and focused (single responsibility)
- Limit function parameters to 3-4 maximum (use object destructuring for more)
- Write self-documenting code with clear variable and function names
- Add comments only when explaining "why", not "what"
- Use TypeScript strict mode
- Avoid any type unless absolutely necessary
- Prefer const over let, never use var
- Use optional chaining (?.) and nullish coalescing (??) operators

### Import Organization

Organize imports in this order:
1. External libraries (React, third-party packages)
2. Internal modules (components, hooks, utils)
3. Types and interfaces
4. Styles
5. Assets

```typescript
// External
import React, { useState, useEffect } from 'react';
import { useNavigate } from 'react-router-dom';

// Internal
import { Button } from '@/components/common';
import { useAuth } from '@/hooks';

// Types
import type { User } from '@/types';

// Styles
import styles from './Component.module.css';

// Assets
import logo from '@/assets/logo.svg';
```

### Commenting Guidelines

- Use JSDoc comments for public APIs and complex functions
- Explain business logic and non-obvious decisions
- Keep comments up-to-date with code changes
- Avoid redundant comments that simply restate the code

### Security Best Practices

- Never commit API keys, tokens, or credentials
- Validate and sanitize all user inputs
- Use environment variables for configuration
- Implement proper authentication and authorization
- Escape output to prevent XSS attacks
- Use HTTPS for all API calls
- Implement CSRF protection for forms

### Accessibility (a11y)

- Use semantic HTML elements
- Provide meaningful alt text for images
- Ensure keyboard navigation works properly
- Maintain proper heading hierarchy
- Use ARIA attributes when necessary
- Test with screen readers
- Ensure sufficient color contrast

### Git Commit Messages

Follow conventional commits format:
```
<type>(<scope>): <subject>

<body>

<footer>
```

Types: feat, fix, docs, style, refactor, test, chore

Example:
```
feat(auth): add password reset functionality

Implemented password reset flow with email verification
and token-based authentication

Closes #123
```

## Development Workflow

1. Create feature branch from main/develop
2. Write code following these standards
3. Write tests for new features
4. Run linter and fix all issues
5. Ensure all tests pass
6. Create pull request with clear description
7. Address code review feedback
8. Merge after approval

## Performance Considerations

- Lazy load routes and heavy components
- Memoize expensive calculations
- Use React.memo() for pure components
- Implement virtual scrolling for long lists
- Optimize images and assets
- Code split at route level
- Monitor bundle size
