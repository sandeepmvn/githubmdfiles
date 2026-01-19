---
name: Code Review Checklist
description: Comprehensive code review guidelines and checklist
applyTo: "**/*"
---

# Code Review Checklist

## Code Review Philosophy

- Reviews are for learning, not just finding bugs
- Be kind, be constructive, be specific
- Focus on the code, not the person
- Ask questions instead of making demands
- Recognize good work
- Everyone's code can be improved

## Review Process

### For Reviewers

1. **Read the PR description** - Understand what's being changed and why
2. **Check CI/CD status** - Ensure all automated checks pass
3. **Review commit history** - Look for logical, atomic commits
4. **Test locally if needed** - For complex changes, pull and test
5. **Leave constructive feedback** - Be specific and helpful
6. **Approve when ready** - Don't nitpick minor style issues

### For Authors

1. **Write clear PR description** - Explain what, why, and how
2. **Keep PRs small** - Aim for <400 lines of changes
3. **Self-review first** - Review your own code before submitting
4. **Respond to feedback** - Address comments or explain reasoning
5. **Update documentation** - Include doc changes in the same PR
6. **Request re-review** - After addressing feedback

## Code Review Checklist

### Functionality

- [ ] **Does the code work?**
  - Are there any obvious bugs or logical errors?
  - Does it handle edge cases?
  - Are there race conditions or concurrency issues?

- [ ] **Does it meet requirements?**
  - Does it solve the stated problem?
  - Are all acceptance criteria met?
  - Does it match the design/mockups?

- [ ] **Error handling**
  - Are errors caught and handled appropriately?
  - Are error messages user-friendly?
  - Are errors logged with sufficient context?
  - Are error boundaries in place for React components?

- [ ] **Loading states**
  - Are loading states shown during async operations?
  - Are skeletons/spinners appropriately placed?
  - Is there a timeout for long operations?

- [ ] **Empty states**
  - What happens when there's no data?
  - Are empty states helpful and actionable?

### Code Quality

- [ ] **Readability**
  - Is the code self-documenting?
  - Are variable and function names descriptive?
  - Is the logic easy to follow?
  - Are comments necessary and helpful?

- [ ] **Maintainability**
  - Is the code DRY (Don't Repeat Yourself)?
  - Are functions small and focused?
  - Is the code properly organized?
  - Will this be easy to change in the future?

- [ ] **Complexity**
  - Is the solution unnecessarily complex?
  - Could it be simplified?
  - Are there too many nested conditionals?
  - Is there excessive abstraction?

- [ ] **Naming**
  - Are names consistent with existing code?
  - Do names follow project conventions?
  - Are magic numbers replaced with named constants?

### React-Specific

- [ ] **Component structure**
  - Are components kept small and focused?
  - Is logic separated from presentation where appropriate?
  - Are compound components used effectively?

- [ ] **Hooks usage**
  - Are hooks called at the top level?
  - Are dependency arrays complete and correct?
  - Are custom hooks named with 'use' prefix?
  - Is useCallback/useMemo used appropriately (not over-used)?

- [ ] **Props**
  - Are prop types properly defined with TypeScript?
  - Are required vs optional props clear?
  - Are default values provided for optional props?
  - Is prop drilling avoided (using Context or state management)?

- [ ] **State management**
  - Is state kept as local as possible?
  - Is global state used only when necessary?
  - Is state structure normalized and efficient?

- [ ] **Performance**
  - Are expensive operations memoized?
  - Are lists virtualized if needed?
  - Is code splitting implemented for large components/routes?
  - Are images optimized and lazy loaded?

### TypeScript

- [ ] **Type safety**
  - Are types properly defined (no implicit 'any')?
  - Are type assertions used sparingly and correctly?
  - Are generics used where appropriate?
  - Are union types and type guards used correctly?

- [ ] **Type organization**
  - Are types exported for reuse?
  - Are types defined close to usage?
  - Are shared types in appropriate locations?

### Testing

- [ ] **Test coverage**
  - Are there tests for new functionality?
  - Are edge cases tested?
  - Are error scenarios tested?
  - Is coverage above project threshold (typically 80%)?

- [ ] **Test quality**
  - Do tests test behavior, not implementation?
  - Are tests readable and maintainable?
  - Are test names descriptive?
  - Are mocks used appropriately?

- [ ] **Test types**
  - Are there unit tests for utilities/hooks?
  - Are there component tests for UI?
  - Are there integration tests for features?

### Security

- [ ] **Input validation**
  - Are all user inputs validated?
  - Are inputs sanitized before use?
  - Is XSS prevention in place?

- [ ] **Authentication/Authorization**
  - Are protected routes properly guarded?
  - Are API calls authenticated?
  - Are user permissions checked?

- [ ] **Sensitive data**
  - Are no secrets committed to the repository?
  - Are environment variables used correctly?
  - Is sensitive data encrypted?
  - Are API keys properly secured?

- [ ] **Dependencies**
  - Are dependencies from trusted sources?
  - Are there known vulnerabilities?
  - Are dependencies up to date?

### Performance

- [ ] **Bundle size**
  - Are new dependencies necessary?
  - Is tree-shaking utilized?
  - Are heavy libraries code-split?

- [ ] **Runtime performance**
  - Are there unnecessary re-renders?
  - Are loops and iterations efficient?
  - Is debouncing/throttling used where needed?
  - Are API calls optimized (caching, deduplication)?

- [ ] **Memory leaks**
  - Are event listeners cleaned up?
  - Are subscriptions unsubscribed?
  - Are timers cleared?
  - Are AbortControllers used for fetch requests?

### Accessibility (a11y)

- [ ] **Semantic HTML**
  - Are semantic elements used appropriately?
  - Is heading hierarchy correct?
  - Are landmarks (nav, main, aside) used?

- [ ] **ARIA**
  - Are ARIA attributes used correctly?
  - Are roles defined where needed?
  - Are labels provided for interactive elements?

- [ ] **Keyboard navigation**
  - Can all functionality be accessed via keyboard?
  - Is tab order logical?
  - Are focus states visible?

- [ ] **Screen readers**
  - Are images given alt text?
  - Are complex interactions announced?
  - Are dynamic content changes announced?

### Documentation

- [ ] **Code comments**
  - Are complex sections explained?
  - Are "why" comments present (not just "what")?
  - Are TODOs tracked in issue tracker?

- [ ] **API documentation**
  - Are new APIs documented?
  - Are examples provided?
  - Are edge cases noted?

- [ ] **README/Docs**
  - Are documentation updates included?
  - Are setup instructions current?
  - Are breaking changes noted?

### Git Practices

- [ ] **Commit messages**
  - Are commits atomic and logical?
  - Do messages follow conventional commits format?
  - Are messages descriptive and clear?

- [ ] **PR description**
  - Is the problem clearly stated?
  - Is the solution explained?
  - Are screenshots included for UI changes?
  - Are breaking changes highlighted?

- [ ] **Branch management**
  - Is the branch name descriptive?
  - Is the branch up-to-date with target?
  - Are merge conflicts resolved?

### Styling

- [ ] **CSS quality**
  - Are styles scoped (CSS Modules, styled-components)?
  - Are magic values replaced with variables?
  - Is responsive design implemented?
  - Are animations performant (use transform/opacity)?

- [ ] **Consistency**
  - Does styling match design system?
  - Are spacing and typography consistent?
  - Are colors from design tokens?

- [ ] **Browser support**
  - Does it work in target browsers?
  - Are fallbacks provided for newer features?
  - Is vendor prefixing handled?

## Review Comments Examples

### Good Comments

✅ **Be specific and actionable:**
```
Consider extracting this logic into a custom hook:

```typescript
function useUserPermissions(userId: string) {
  const [permissions, setPermissions] = useState<Permission[]>([]);
  
  useEffect(() => {
    fetchPermissions(userId).then(setPermissions);
  }, [userId]);
  
  return permissions;
}
```

This would make it reusable across other components and easier to test.
```

✅ **Ask questions:**
```
What happens if the API returns null here? Should we add a null check
or handle it in the error boundary?
```

✅ **Recognize good work:**
```
Nice use of React.memo here! This should prevent unnecessary re-renders
when the parent updates.
```

✅ **Provide context:**
```
We typically use the useQuery hook from React Query for data fetching
to get automatic caching and refetching. See the UserList component
for an example.
```

### Bad Comments

❌ **Vague:**
```
This doesn't look right.
```

❌ **Demanding:**
```
Change this immediately.
```

❌ **Nitpicky:**
```
Use single quotes instead of double quotes.
(This should be handled by linter, not code review)
```

❌ **Personal:**
```
I would never write code like this.
```

## Approval Criteria

### Ready to Approve ✅

- All automated checks pass (CI/CD, linting, tests)
- Code meets functionality requirements
- No major bugs or security issues
- Code quality is acceptable (readable, maintainable)
- Tests provide adequate coverage
- Documentation is updated if needed
- Minor style issues can be addressed in follow-up or left to linter

### Request Changes 🔄

- Functionality is broken or incomplete
- Security vulnerabilities present
- Major performance issues
- Tests are missing or failing
- Breaking changes without migration path
- Code quality severely impacts maintainability

### Comment Only 💬

- Minor suggestions for improvement
- Questions about approach
- Suggestions for future improvements
- Style nitpicks already handled by linter

## PR Size Guidelines

### Ideal PR Sizes

- **Small (< 200 lines):** ✅ Easy to review, quick feedback
- **Medium (200-400 lines):** ⚠️ Acceptable, might take longer
- **Large (> 400 lines):** ❌ Consider splitting

### When Large PRs are OK

- Generated code (migrations, types)
- Major refactoring (well-documented)
- New feature with comprehensive tests
- Initial project setup

### How to Split Large PRs

1. **By feature:** Each PR is a complete, testable feature
2. **By layer:** Infrastructure → Backend → Frontend
3. **By file type:** Types → Logic → Tests → UI
4. **Incremental:** Base → Iteration 1 → Iteration 2

## Review Turnaround Time

- **Critical fixes:** < 2 hours
- **Small PRs:** Same day
- **Medium PRs:** 1-2 business days
- **Large PRs:** 2-3 business days

## Self-Review Checklist

Before requesting review, check:

- [ ] I've tested my changes locally
- [ ] All tests pass
- [ ] I've added tests for new functionality
- [ ] Linter passes with no errors
- [ ] I've updated relevant documentation
- [ ] I've reviewed my own code line-by-line
- [ ] I've removed debug code and console.logs
- [ ] I've checked for unintended file changes
- [ ] PR description is clear and complete
- [ ] Breaking changes are highlighted
- [ ] Screenshots included for UI changes

## Common Review Findings

### Anti-patterns to Watch For

**State Management:**
```typescript
// ❌ Bad - Prop drilling through multiple levels
<Parent prop={value}>
  <Child prop={value}>
    <GrandChild prop={value}>
      <GreatGrandChild prop={value} />
    </GrandChild>
  </Child>
</Parent>

// ✅ Good - Use Context or state management
<Provider value={value}>
  <Parent>
    <Child>
      <GrandChild>
        <GreatGrandChild />
      </GrandChild>
    </Child>
  </Parent>
</Provider>
```

**Error Handling:**
```typescript
// ❌ Bad - Silent error
try {
  await fetchData();
} catch (error) {
  // Nothing
}

// ✅ Good - Proper error handling
try {
  await fetchData();
} catch (error) {
  logError('Failed to fetch data', error);
  setError('Unable to load data. Please try again.');
}
```

**useEffect Dependencies:**
```typescript
// ❌ Bad - Missing dependencies
useEffect(() => {
  fetchUser(userId);
}, []); // userId is missing!

// ✅ Good - Complete dependencies
useEffect(() => {
  fetchUser(userId);
}, [userId]);
```

## Post-Review Actions

### After Approval

1. Squash or rebase commits if needed
2. Ensure CI/CD passes
3. Merge using appropriate strategy
4. Delete feature branch
5. Move ticket to "Done"
6. Monitor production for issues

### After Requesting Changes

1. Address all feedback or provide reasoning
2. Make requested changes in new commits
3. Reply to each comment
4. Request re-review
5. Thank reviewer for their time

## Resources for Better Reviews

- [Google Engineering Practices - Code Review](https://google.github.io/eng-practices/review/)
- [The Art of Giving and Receiving Code Reviews (Gracefully)](https://www.alexandra-hill.com/2018/06/25/the-art-of-giving-and-receiving-code-reviews/)
- [How to Make Your Code Reviewer Fall in Love with You](https://mtlynch.io/code-review-love/)
