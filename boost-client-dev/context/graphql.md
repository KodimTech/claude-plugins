# GraphQL Patterns (Apollo Client)

## File locations

| What | Where |
|------|-------|
| Queries | `src/graphql/queries/<domain>/` |
| Mutations | `src/graphql/mutations/<domain>/` |
| Subscriptions | `src/graphql/subscriptions/<domain>/` |
| Fragments | `src/graphql/fragments/<domain>/` |
| Generated types | `src/graphql/types.ts` |
| Apollo hooks | `src/graphql/hooks/` |
| Codegen config | `codegen.ts` |

---

## Defining operations

```graphql
# src/graphql/queries/invoices/get_invoices.graphql
query GetInvoices($firmId: ID!) {
  invoices(firmId: $firmId) {
    ...InvoiceFields
  }
}
```

```graphql
# src/graphql/mutations/invoices/create_invoice.graphql
mutation CreateInvoice($amount: Float!, $description: String) {
  createInvoice(amount: $amount, description: $description) {
    ...InvoiceFields
  }
}
```

```graphql
# src/graphql/fragments/invoices/invoice_fields.graphql
fragment InvoiceFields on Invoice {
  id
  amount
  description
  status
  createdAt
}
```

---

## Using Apollo hooks (generated)

After running codegen, use the generated typed hooks:

```tsx
import { useGetInvoicesQuery, useCreateInvoiceMutation } from 'graphql/types';

const InvoiceList: React.FC = () => {
  const { data, loading, error } = useGetInvoicesQuery({
    variables: { firmId: currentFirmId },
  });

  const [createInvoice, { loading: creating }] = useCreateInvoiceMutation({
    refetchQueries: ['GetInvoices'],
    onCompleted: () => showSuccessToast('Invoice created'),
    onError: err => showErrorToast(err.message),
  });

  if (loading) return <LMLoader />;
  if (error) return <ErrorState message={error.message} />;
  if (!data?.invoices?.length) return <EmptyState />;

  return (/* ... */);
};
```

**Rules:**
- Always handle `loading`, `error`, and empty states
- Use `refetchQueries` with query name strings, not query documents
- `onError` must show user-visible feedback — never swallow
- Use generated types (e.g., `InvoiceFieldsFragment`) — never write raw `any`

---

## Running codegen

After the backend adds new mutations/queries/types to the schema:

```bash
npm run codegen
```

This regenerates `src/graphql/types.ts` with all typed hooks and types. Commit this file.

**CI check:** If you create or modify `.graphql` files, run codegen and commit the updated `types.ts` or CI may fail.

---

## Cache updates after mutations

For mutations that add/remove items from a list, update the Apollo cache directly to avoid full refetch:

```tsx
const [createInvoice] = useCreateInvoiceMutation({
  update(cache, { data }) {
    const newInvoice = data?.createInvoice;
    if (!newInvoice) return;

    cache.modify({
      fields: {
        invoices(existing = []) {
          const ref = cache.writeFragment({
            data: newInvoice,
            fragment: InvoiceFieldsFragmentDoc,
          });
          return [...existing, ref];
        },
      },
    });
  },
});
```

Use `refetchQueries` only when cache manipulation is complex or data depends on server-side ordering/pagination.

---

## Optimistic updates

Use when mutation result is fully predictable (e.g., toggle a flag):

```tsx
const [toggleStatus] = useToggleInvoiceStatusMutation({
  optimisticResponse: {
    toggleInvoiceStatus: {
      __typename: 'Invoice',
      id: invoice.id,
      status: invoice.status === 'pending' ? 'paid' : 'pending',
    },
  },
});
```

---

## Cross-repo: verifying backend support

Before writing a query or mutation in boost-client, verify it exists in the backend schema:

```bash
# Check generated types for the operation
grep -n "createInvoice\|CreateInvoice" src/graphql/types.ts
```

If it doesn't exist, the backend (boost-api) must add it first. Document this as a **Backend Changes Required** dependency in the plan.
