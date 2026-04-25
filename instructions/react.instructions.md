---
description: "React component patterns, hooks, state management, error boundaries, accessibility, and performance best practices. Functional components only."
applyTo: "**/*.tsx,**/components/**/*.ts"
---

# React Best Practices

## Component Philosophy

- Functional components ONLY — no class components
- Components should be small, focused, and composable
- Separate logic from presentation (custom hooks for logic, components for UI)
- Prefer composition over configuration (children/render props over mega-prop components)

## Component Structure

Follow this consistent ordering inside every component:

```tsx
// 1. Type definitions
interface UserCardProps {
  user: User;
  onSelect: (userId: string) => void;
  className?: string;
}

// 2. Component
export function UserCard({ user, onSelect, className }: UserCardProps) {
  // 3. Hooks (in consistent order)
  const [isExpanded, setIsExpanded] = useState(false);
  const theme = useTheme();

  // 4. Derived state (useMemo for expensive)
  const fullName = `${user.firstName} ${user.lastName}`;

  // 5. Callbacks (useCallback when passed to children)
  const handleClick = useCallback(() => {
    onSelect(user.id);
  }, [user.id, onSelect]);

  // 6. Effects
  useEffect(() => {
    // side effect
  }, [dependency]);

  // 7. Early returns (loading, error, empty states)
  if (!user) return null;

  // 8. Render
  return <div>...</div>;
}
```

## Naming

- **Components:** PascalCase (`UserCard`, `DashboardLayout`)
- **Hooks:** camelCase with `use` prefix (`useAuth`, `useDebounce`)
- **Event handlers:** `handle` prefix in component, `on` prefix in props (`onClick`, `handleClick`)
- **Boolean props:** `is`/`has`/`should` prefix (`isLoading`, `hasError`)
- **Files:** PascalCase matching component name (`UserCard.tsx`)

## Hooks Best Practices

- Follow Rules of Hooks — always at top level, never conditional
- Custom hooks for reusable logic — extract when used in 2+ components
- `useState`: keep state minimal — derive what you can
- `useEffect`: one effect per concern, clean up subscriptions
- `useMemo`: only for expensive computations — don't overuse
- `useCallback`: only when passed to memoized children — don't wrap everything
- `useRef`: for DOM access and values that don't trigger re-render

## State Management

- **Local state** (`useState`) for component-specific state
- **Context** for cross-cutting concerns (theme, auth, locale) — not for everything
- **External store** (Zustand, React Query) for server state / complex shared state
- Never lift state higher than necessary
- Prefer React Query / TanStack Query for server data (caching, refetching, optimistic updates)

## Error Handling

- Use Error Boundaries for catching render errors
- Use `try/catch` in async event handlers
- Show meaningful error states to users (not blank screens)
- Log errors with context for debugging
- Provide retry mechanisms for transient failures

## Performance

- Use `React.memo()` for components that receive stable props but re-render often
- Use `useMemo`/`useCallback` when preventing unnecessary child re-renders
- Lazy load routes and heavy components with `React.lazy()` + `Suspense`
- Virtualize long lists (`react-window` or `@tanstack/react-virtual`)
- Avoid inline object/array creation in JSX props (creates new references each render)

## Accessibility (a11y)

- Use semantic HTML elements (`<button>` not `<div onClick>`)
- All images need `alt` attributes
- Form inputs need associated `<label>` elements
- Interactive elements must be keyboard-navigable
- Use ARIA attributes only when semantic HTML is insufficient
- Color must not be the only way to convey information
- Test with screen reader and keyboard navigation

## Patterns to Avoid

- ❌ Props drilling > 3 levels — use Context or composition
- ❌ `useEffect` for data fetching — use React Query or data loader
- ❌ `useEffect` for derived state — compute directly or `useMemo`
- ❌ Index as key in dynamic lists — use stable unique IDs
- ❌ Inline styles for complex styling — use CSS modules or styled components
- ❌ Mixing business logic in components — extract to hooks or services
