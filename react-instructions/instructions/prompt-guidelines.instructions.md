---
name: Prompt Guidelines
description: How to structure effective prompts for React development tasks
applyTo: ""
---

# Prompt Guidelines for React Development

This file contains guidelines for creating effective prompts when requesting code generation or assistance.

## The CARD Framework

Every prompt should include:

**C** - **Context**: What are you building?  
**A** - **Action**: What do you want created?  
**R** - **Requirements**: What specific features/constraints?  
**D** - **Details**: Any additional specifications?

## Quick Component Prompt Template

```
Create a [ComponentName] component for [purpose].

Props:
- [propName]: [Type] - [Description]

Features:
- [Feature list]

Requirements:
- Use TypeScript
- Include loading/error states
- Add accessibility features
- Write unit tests
```

## Quick Hook Prompt Template

```
Create a use[HookName] hook for [purpose].

Parameters:
- [paramName]: [Type]

Returns:
- [returnValue]: [Type]

Features:
- [Feature list]

Requirements:
- TypeScript generics
- Proper cleanup
- Error handling
```

## Quick Page Prompt Template

```
Create a [PageName] page for [purpose].

Sections:
- [Section list]

Data:
- Fetch from [API endpoint]

Features:
- [Feature list]

State Management:
- Use [approach]
```

## Quick Service Prompt Template

```
Create a [ServiceName] service for [API interactions].

Endpoints:
- [Method] [Endpoint]

Methods:
- [methodName]: [Description]

Features:
- Error handling
- Request cancellation
- TypeScript types
```

## Essential Prompt Modifiers

Add these to any prompt for better results:

**TypeScript:**
- "Use TypeScript with strict types"
- "Define interfaces for all props"
- "Export all types"

**Testing:**
- "Include comprehensive tests"
- "Test user interactions"
- "Mock API calls"

**Accessibility:**
- "Include ARIA labels"
- "Support keyboard navigation"
- "Ensure screen reader compatibility"

**Performance:**
- "Use React.memo where appropriate"
- "Implement code splitting"
- "Lazy load heavy components"

**Error Handling:**
- "Handle all error states"
- "Show user-friendly messages"
- "Implement error boundaries"

## Example Prompts

### Good Prompt (High Success)
```
Create a UserCard component for displaying user profile information.

Props:
- user: User - User object with id, name, email, avatar
- onEdit: () => void - Edit callback
- isLoading?: boolean - Loading state

Features:
- Display avatar with fallback to initials
- Show loading skeleton while loading
- Include edit button
- Support keyboard navigation

Requirements:
- Use TypeScript with strict types
- Use CSS Modules for styling
- Include comprehensive unit tests
- Add ARIA labels for accessibility
```

### Bad Prompt (Low Success)
```
Create a user component
```

## Best Practices

1. **Be Specific**: List exact props, features, requirements
2. **Include Types**: Always specify TypeScript types
3. **State Requirements**: Loading, error, empty states
4. **Mention Tests**: Request test coverage
5. **Add Accessibility**: Include a11y requirements
6. **Specify Styling**: CSS Modules, Tailwind, etc.
7. **Reference Examples**: "Similar to ProductCard component"

## Remember

- Clear prompt = Clean code
- Detailed requirements = Correct implementation
- TypeScript types = Fewer bugs
- Tests included = Better quality
- Accessibility = Inclusive apps

For complete prompt guidelines and templates, see the React-Prompt-Guidelines.md file in the project root.
