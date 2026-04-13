# Architecture Patterns

## Component organization (`src/components/`)

Components are organized by **domain**. Each component lives in its own folder with an `index.tsx` entry point.

```
src/components/
├── boost/              # Core UI primitives (BoostButton, BoostInput, BoostTable, ...)
├── lawmatics/          # Extended UI library (LMIcon, LMCard, LMTabs, LMLayers, ...)
├── automation/         # Automation feature components
├── calendar/           # Calendar feature
├── clients/            # Client management
├── settings/           # Settings pages
└── <domain>/           # Feature-specific components
    └── my_component/
        └── index.tsx
```

**Rule:** Always search `src/components/lawmatics/` and `src/components/boost/` before creating new UI components.

---

## Component patterns

### Functional components (always)
```tsx
// index.tsx
import React from 'react';

interface Props {
  title: string;
  onClose: () => void;
}

const MyComponent: React.FC<Props> = ({ title, onClose }) => {
  return (
    <div className="flex items-center pa3">
      <span className="fs14 boost-secondary">{title}</span>
    </div>
  );
};

export default MyComponent;
```

### Custom hooks for logic
Extract business logic from components into hooks in `src/hooks/` or co-located next to the component:
```tsx
// useMyFeature.ts
const useMyFeature = (firmId: string) => {
  const [data, setData] = useState(null);
  // ... logic
  return { data, loading };
};
```

---

## Redux (`src/actions/` + `src/reducers/`)

Redux + redux-pack for async actions (legacy pattern, still in use).

```ts
// actions/myActions.ts
export const FETCH_SOMETHING = 'FETCH_SOMETHING';

export const fetchSomething = (id: string) => ({
  type: FETCH_SOMETHING,
  payload: apolloClient.query({ query: SOME_QUERY, variables: { id } }),
});
```

```ts
// reducers/myReducer.ts
import handle from 'redux-pack';

const initialState = { items: [], loading: false, error: null };

export default (state = initialState, action: AnyAction) => {
  switch (action.type) {
    case FETCH_SOMETHING:
      return handle(state, action, {
        start:   s => ({ ...s, loading: true }),
        success: s => ({ ...s, loading: false, items: action.payload.data.items }),
        failure: s => ({ ...s, loading: false, error: action.payload }),
      });
    default:
      return state;
  }
};
```

**When to use Redux vs Apollo cache:**
- Apollo cache handles server-side data automatically via `useQuery`/`useMutation`
- Redux is for UI state, session state, or data that doesn't map to a single GraphQL entity
- Prefer Apollo cache for new features — Redux for UI-only state (modals, sidebar open/close, etc.)

---

## Apollo Client setup (`src/hooks/useCreateApolloClient.ts`)

Apollo Client is created per-session and provided at the app root. Components consume it via hooks.

---

## TypeScript path aliases (from `tsconfig.json`)

Use these instead of relative paths:

| Alias | Maps to |
|-------|---------|
| `components/*` | `src/components/*` |
| `hooks/*` | `src/hooks/*` |
| `graphql/*` | `src/graphql/*` |
| `helpers/*` | `src/helpers/*` |
| `actions/*` | `src/actions/*` |
| `reducers/*` | `src/reducers/*` |
| `types/*` | `src/types/*` |
| `lib/*` | `src/lib/*` |
| `config/*` | `src/config/*` |

```tsx
// Correct
import BoostButton from 'components/boost/boost_button';
import { useDebounce } from 'hooks/useDebounce';

// Wrong
import BoostButton from '../../../components/boost/boost_button';
```

---

## Routing & Modals (ModalRoute / LMLayers)

Slide-out panels use `ModalRoute` with `layerMode`. Structure is strict:

```tsx
<ModalRoute path="/feature/:id/edit" component={EditPanel} layerMode withHeader title="Edit" />

// Inside EditPanel:
const EditPanel = () => (
  <form onSubmit={handleSubmit} className="flex flex-column flex-auto">
    <LMLayersHeader title="Edit" />
    <LMLayersContent>
      {/* fields */}
    </LMLayersContent>
    <LMLayersFooter>
      <BoostButton title="Save" theme="success" />
    </LMLayersFooter>
  </form>
);
```

**Rule:** Wrapper of Header+Content+Footer MUST have `flex flex-column flex-auto`.

---

## Helpers (`src/helpers/`)

Always check before implementing utility logic:

| Helper | Use for |
|--------|---------|
| `DateHelper.ts` | Date formatting/parsing |
| `TimeHelper.ts` | Time operations |
| `StringHelper.js` | String manipulation |
| `ValidationHelper.js` | Form validation |
| `ArrayHelpers.ts` | Array operations |
| `NumberHelper.js` | Number formatting |
