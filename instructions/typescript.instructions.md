---
description: "TypeScript coding standards covering strict type safety, naming conventions, async patterns, error handling, and modern TypeScript features. No 'any' allowed."
applyTo: "**/*.ts,**/*.tsx,**/*.mts"
---

# TypeScript Coding Standards

## Strict Mode — Non-Negotiable

- `"strict": true` in tsconfig.json — always
- `"noImplicitAny": true` — never use `any`
- `"strictNullChecks": true` — handle null/undefined explicitly
- `"noUncheckedIndexedAccess": true` — array/object access returns `T | undefined`
- If you need to escape the type system, use `unknown` + type guards, NEVER `any`
- The ONLY acceptable use of `any` is in `.d.ts` files for third-party type definitions

## Naming Conventions

- **PascalCase:** types, interfaces, enums, classes, React components
- **camelCase:** variables, functions, methods, properties, parameters
- **UPPER_SNAKE_CASE:** module-level constants (`const MAX_RETRIES = 3`)
- **Interfaces:** DO NOT prefix with `I` (TypeScript convention — `UserService` not `IUserService`)
- **Type aliases:** PascalCase (`type UserRole = 'admin' | 'user'`)
- **File names:** kebab-case for modules (`user-service.ts`), PascalCase for React components (`UserCard.tsx`)
- **Barrel exports:** `index.ts` per feature directory

## Type Safety

- Prefer `interface` for object shapes (extendable), `type` for unions/intersections
- Use discriminated unions for state management (`type Result = Success | Error`)
- Use `as const` for literal types
- Use `satisfies` operator to validate types without widening
- Use `Readonly<T>`, `ReadonlyArray<T>` for immutable data
- Use template literal types for string patterns
- Prefer `Map<K, V>` over `Record<string, V>` when keys are dynamic
- Use `unknown` for values from external sources (API responses, user input)

## Async Patterns

- Always use `async/await` — never raw `.then()` chains
- Handle errors with try/catch at the boundary, not every call
- Use `AbortController` / `AbortSignal` for cancellable operations
- Use `Promise.all()` for independent parallel operations (not sequential awaits)
- Use `Promise.allSettled()` when partial failure is acceptable
- Return early for guard clauses — avoid deep nesting

## Error Handling

- Use custom error classes extending `Error` for domain errors
- Type your errors:
  ```typescript
  class NotFoundError extends Error {
    code = 'NOT_FOUND' as const;
  }
  ```
- Use `Result<T, E>` pattern for expected failures (no throw for business logic)
- Always include context in error messages
- Log errors with structured data, not string concatenation

## Imports & Modules

- Use ES modules (`import/export`) exclusively
- Prefer named exports over default exports (better refactoring, tree-shaking)
- Group imports: external libs → internal modules → relative imports (with blank lines between)
- Use path aliases (`@/components/...`) instead of deep relative paths (`../../../`)
- Avoid circular dependencies — if detected, restructure

## Code Quality

- Functions: max 30 lines, single responsibility
- Files: max 300 lines — split into modules
- Function parameters: max 3 — use an options object for more
- Use `readonly` on function parameters when the function shouldn't mutate them
- Prefer pure functions — same input → same output, no side effects
- Use early returns over deeply nested conditionals

## Documentation

- TSDoc (`/** */`) on all exported functions and types
- Document parameters with `@param`, return values with `@returns`
- Document thrown errors with `@throws`
- Use `@example` for non-obvious usage patterns
- Inline comments for WHY, not WHAT
