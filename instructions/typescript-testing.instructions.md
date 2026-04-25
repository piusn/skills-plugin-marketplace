---
description: "TypeScript and React testing standards using Jest/Vitest, React Testing Library, and user-event. Covers component testing, hook testing, and API mocking."
applyTo: "**/*.test.ts,**/*.test.tsx,**/*.spec.ts,**/*.spec.tsx,**/__tests__/**"
---

# TypeScript & React Testing Standards

## Framework Stack

- **Test runner:** Vitest (preferred) or Jest
- **Component testing:** React Testing Library (`@testing-library/react`)
- **User interactions:** `@testing-library/user-event` (over `fireEvent`)
- **API mocking:** MSW (Mock Service Worker) for network mocking
- **Assertions:** Vitest/Jest built-in + `@testing-library/jest-dom` matchers

## Testing Philosophy

- Test what the USER sees and does — not implementation details
- Query by accessibility roles first: `getByRole`, `getByLabelText`, `getByText`
- Avoid `getByTestId` — only as last resort
- Never test internal state directly — test the rendered output

## Component Test Structure

```tsx
import { render, screen } from '@testing-library/react';
import userEvent from '@testing-library/user-event';

describe('LoginForm', () => {
  it('should display error when submitting empty form', async () => {
    // Arrange
    const user = userEvent.setup();
    render(<LoginForm onSubmit={vi.fn()} />);

    // Act
    await user.click(screen.getByRole('button', { name: /submit/i }));

    // Assert
    expect(screen.getByRole('alert')).toHaveTextContent(/email is required/i);
  });

  it('should call onSubmit with form data when valid', async () => {
    // Arrange
    const handleSubmit = vi.fn();
    const user = userEvent.setup();
    render(<LoginForm onSubmit={handleSubmit} />);

    // Act
    await user.type(screen.getByLabelText(/email/i), 'test@example.com');
    await user.type(screen.getByLabelText(/password/i), 'password123');
    await user.click(screen.getByRole('button', { name: /submit/i }));

    // Assert
    expect(handleSubmit).toHaveBeenCalledWith({
      email: 'test@example.com',
      password: 'password123',
    });
  });
});
```

## Query Priority (Testing Library)

Use queries in this order of preference:

1. `getByRole` — accessible role (button, heading, textbox)
2. `getByLabelText` — form inputs
3. `getByPlaceholderText` — when no label exists
4. `getByText` — non-interactive elements
5. `getByDisplayValue` — current input value
6. `getByAltText` — images
7. `getByTitle` — title attribute
8. `getByTestId` — LAST RESORT only

## Custom Hook Testing

```tsx
import { renderHook, act } from '@testing-library/react';

describe('useCounter', () => {
  it('should increment', () => {
    const { result } = renderHook(() => useCounter());

    act(() => result.current.increment());

    expect(result.current.count).toBe(1);
  });
});
```

## API Mocking with MSW

```tsx
import { setupServer } from 'msw/node';
import { http, HttpResponse } from 'msw';

const server = setupServer(
  http.get('/api/users', () => HttpResponse.json([{ id: 1, name: 'Test' }]))
);

beforeAll(() => server.listen());
afterEach(() => server.resetHandlers());
afterAll(() => server.close());
```

## Rules

- ⛔ Never use `container.querySelector` — use Testing Library queries
- ⛔ Never test implementation details (state, internal methods)
- ⛔ Never use `waitFor` with side effects — only for assertions
- ✅ Use `user-event` over `fireEvent` (simulates real user behavior)
- ✅ Use `screen` for queries (not destructured from render)
- ✅ Wrap state updates in `act()` when testing hooks
- ✅ Test loading, error, and empty states — not just happy path
- ✅ Test keyboard navigation for interactive components
