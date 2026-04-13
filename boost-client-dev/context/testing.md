# Testing Patterns (Jest + Testing Library)

## File locations

| What | Where |
|------|-------|
| Component tests | Co-located: `src/components/<domain>/<name>/__tests__/<name>.test.tsx` |
| Hook tests | `src/hooks/__tests__/<name>.test.ts` |
| Helper tests | `src/helpers/__tests__/<name>.test.ts` |
| E2E tests | `cypress/e2e/` |
| Test setup | `config/jest/setupAfterEnv.js` |

---

## Component test structure

```tsx
import React from 'react';
import { render, screen, waitFor } from '@testing-library/react';
import userEvent from '@testing-library/user-event';
import { MockedProvider } from '@apollo/client/testing';
import { Provider } from 'react-redux';
import { createTestStore } from 'config/jest/testStore';
import InvoiceList from 'components/invoices/invoice_list';
import { GetInvoicesDocument } from 'graphql/types';

const mockInvoice = {
  id: '1',
  amount: 100.0,
  description: 'Test invoice',
  status: 'pending',
  createdAt: '2024-01-01T00:00:00Z',
};

const mocks = [
  {
    request: {
      query: GetInvoicesDocument,
      variables: { firmId: 'firm-1' },
    },
    result: {
      data: { invoices: [mockInvoice] },
    },
  },
];

const renderComponent = (props = {}) =>
  render(
    <MockedProvider mocks={mocks} addTypename={false}>
      <Provider store={createTestStore()}>
        <InvoiceList firmId="firm-1" {...props} />
      </Provider>
    </MockedProvider>,
  );

describe('InvoiceList', () => {
  it('shows loading state initially', () => {
    renderComponent();
    expect(screen.getByTestId('lm-loader')).toBeInTheDocument();
  });

  it('renders invoices after loading', async () => {
    renderComponent();
    expect(await screen.findByText('Test invoice')).toBeInTheDocument();
    expect(screen.getByText('$100.00')).toBeInTheDocument();
  });

  it('shows empty state when no invoices', async () => {
    const emptyMocks = [{ ...mocks[0], result: { data: { invoices: [] } } }];
    render(
      <MockedProvider mocks={emptyMocks} addTypename={false}>
        <Provider store={createTestStore()}>
          <InvoiceList firmId="firm-1" />
        </Provider>
      </MockedProvider>,
    );
    expect(await screen.findByText(/no invoices/i)).toBeInTheDocument();
  });
});
```

---

## Key conventions

- **Queries**: `screen.getByRole` > `screen.getByText` > `screen.getByLabelText` > `screen.getByTestId`
- **Async**: use `findBy*` queries (they wait automatically) instead of `waitFor` + `getBy*`
- **User interactions**: always use `userEvent` (not `fireEvent`) — it simulates real browser behavior
- **MockedProvider**: set `addTypename={false}` to avoid `__typename` matching issues
- **Never** assert on implementation details (class names, internal state, component internals)
- **Always** test the three states for data-fetching components: loading, loaded, error/empty

---

## Mutation testing

```tsx
it('creates an invoice on form submit', async () => {
  const user = userEvent.setup();
  const createMock = {
    request: {
      query: CreateInvoiceDocument,
      variables: { amount: 100, description: 'Test' },
    },
    result: { data: { createInvoice: mockInvoice } },
  };

  render(
    <MockedProvider mocks={[...mocks, createMock]} addTypename={false}>
      <Provider store={createTestStore()}>
        <CreateInvoiceForm firmId="firm-1" />
      </Provider>
    </MockedProvider>,
  );

  await user.type(screen.getByLabelText(/description/i), 'Test');
  await user.type(screen.getByLabelText(/amount/i), '100');
  await user.click(screen.getByRole('button', { name: /save/i }));

  await waitFor(() => {
    expect(screen.queryByText(/saving/i)).not.toBeInTheDocument();
  });
});
```

---

## Hook testing

```tsx
import { renderHook, act } from '@testing-library/react';
import { MockedProvider } from '@apollo/client/testing';
import { useInvoices } from 'hooks/useInvoices';

const wrapper = ({ children }) => (
  <MockedProvider mocks={mocks} addTypename={false}>
    {children}
  </MockedProvider>
);

it('returns invoices after fetching', async () => {
  const { result } = renderHook(() => useInvoices('firm-1'), { wrapper });

  expect(result.current.loading).toBe(true);

  await waitFor(() => {
    expect(result.current.loading).toBe(false);
  });

  expect(result.current.invoices).toHaveLength(1);
  expect(result.current.invoices[0].amount).toBe(100);
});
```

---

## Running tests

```bash
# Run all tests
npm test

# Run specific file
npm test -- src/components/invoices/invoice_list/__tests__/invoice_list.test.tsx

# Watch mode
npm run test-watch

# With coverage
npm run test-ci
```

---

## ESLint / Biome on test files

Test files follow the same lint rules with these relaxations (configured in `.eslintrc`):
- `@typescript-eslint/no-explicit-any`: off in test files
- `no-console`: off in test files

Run lint before finishing:
```bash
npx biome check src/components/<path>
# or
npx eslint src/components/<path>
```
