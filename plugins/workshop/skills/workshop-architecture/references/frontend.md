# Frontend Architecture Reference

Stack-agnostic rules for scaffolding any workshop frontend. Read before writing code. Rules are numbered; grep for `rule N` to find the canonical statement.

## 1. Data direction (rule 6)

**Rule 6: data flows in one direction only — `Routes → Components → Hooks → Queries → API client`. Never skip a layer, never reverse it.**

Routes own URL state and compose components. Components own presentation and call hooks. Hooks own orchestration and call queries (server-state library entries). Queries own caching and call the API client. The API client owns transport. Each layer depends only on the layer immediately below it.

Forbidden patterns under rule 6:

- A component calling `fetch()` directly.
- A query layer importing from a component.
- A route file containing inline data fetching outside of a hook.
- A hook reaching into the transport layer past the API client.
- The API client importing a React hook or a component.
- A utility module in `lib/` importing from `routes/` or `components/`.

Reversing the arrow (e.g. the API client importing a React hook) is an immediate red flag — it means a concern is in the wrong layer.

The point of rule 6 is replaceability. If transport changes (REST → GraphQL → tRPC), only the API client moves. If caching changes (React Query → SWR), only the query layer moves. If routing changes, only routes move. Layer-skipping welds these concerns together and makes every change a rewrite.

```
route.tsx
  └─ <FeatureView />          // component
       └─ useFeatureData()    // hook
            └─ featureQuery() // query (cache key, staleTime)
                 └─ api.feature.get() // API client
                      └─ fetch / ws / rpc transport
```

Practical consequence: when a new screen needs data, you add (in order) an API client method, a query definition, a hook that wraps the query, and a component that calls the hook. Four files, one arrow direction, reviewable in five minutes. If a scaffold skips a step ("just for now"), it never gets added back.

Common violation: a component that calls a query hook directly AND also reads from the query cache via the library's imperative API. That's two different read paths for the same data and they will drift. Under rule 6, there is exactly one read path per datum.

> Example: in `~/Projects/abertrades/abertrades-frontend` this is `routes/ → components/ → lib/tanstack-query → lib/api`.

## 2. Component tiers (rule 7)

**Rule 7: components live in exactly four tiers, ordered from most general to most specific — primitives, app shell, shared business, feature-private.**

Tier 1 `ui/` holds primitives: `Button`, `Input`, `Dialog`, `Card`. Zero business knowledge. Tier 2 `general/` holds app-shell pieces: `AppHeader`, `Sidebar`, `PageLayout`, `ErrorBoundary`. Tier 3 `shared/` holds business components reused across features: `UserAvatar`, `MoneyDisplay`, `OrderRow`. Tier 4 `routes/<feature>/-components/` holds feature-private components used by exactly one route subtree.

Ownership rule for rule 7: a higher-tier component never imports from a lower tier. Primitives never import from `general/`, `shared/`, or any feature folder. `general/` never imports from `shared/` or features. `shared/` never imports from features. Feature-private components are never re-exported through a shared barrel; the leading `-` (or equivalent convention) marks them as private.

```
src/
├── components/
│   ├── ui/           # tier 1: primitives, no business knowledge
│   ├── general/      # tier 2: app shell
│   └── shared/       # tier 3: cross-feature business components
└── routes/
    └── orders/
        ├── route.tsx
        └── -components/   # tier 4: private to /orders
            ├── OrderTable.tsx
            └── OrderFilters.tsx
```

When in doubt, start a component in the deepest tier that works (feature-private) and promote upward only when a second feature actually needs it. Premature promotion to `shared/` is the most common failure mode — it creates a "shared" component shaped entirely by its first caller, which then resists every later use.

The inverse failure is a feature folder reaching sideways into another feature folder. That is forbidden under rule 7: features are siblings, not ancestors. If two features need the same component, promote it to `shared/`; if they need the same data, the hook lives in `shared/` too. Cross-feature imports always indicate a missing tier-3 module.

A primitive in `ui/` should be importable into a brand-new app with zero changes. If it knows about your auth model, your routes, or your domain types, it isn't a primitive — move it to `shared/`.

## 3. Three state homes (rule 8)

**Rule 8: every piece of state lives in exactly one of three homes — server state, client state, or ephemeral state. Never mix them.**

The three homes under rule 8:

1. **Server state** — lives in a server-state library (React Query, SWR, TanStack Query, or equivalent). It owns:
   - caching and deduplication
   - invalidation and refetching
   - retries and backoff
   - background refresh and focus refetching
   - optimistic updates
   Anything the server is the source of truth for belongs here.

2. **Client state** — lives in a context or store. Holds cross-component shared state the client owns:
   - auth session / current user id
   - theme and locale
   - selected entity id (when multiple components read it)
   - feature flags loaded once at boot
   - in-memory command palette state

3. **Ephemeral state** — `useState` (or equivalent) for component-local UI:
   - open/closed, hovered, focused
   - draft input before submit
   - current tab, current step in a wizard
   - transient animation state

A `useState` holding data the server already owns is a bug under rule 8 — it duplicates the cache and goes stale. A global-store entry used by exactly one component is a bug under rule 8 — it leaks ephemeral state into a global namespace. A query cache entry mutated by hand instead of invalidated is a bug — it bypasses the library that owns server state.

The test: ask "who is the source of truth?" Server → query layer. Client, multi-component → store/context. Client, single component → `useState`. Three answers, three homes.

A second test for rule 8: "who else needs this value, and when does it become wrong?" If the answer to "who else" is "nobody", it's ephemeral. If the answer to "when does it become wrong" is "when the server changes", it's server state, full stop — even if the value feels small ("just the username"). Sticking server data into a context to "avoid prop drilling" is a rule 8 violation; the right fix is a query hook called from each consumer, with the cache doing the deduplication.

Form state is ephemeral until submitted. Selection state (selected row id) is client state if multiple components need it, ephemeral otherwise. Pagination cursors are usually URL state (route), not store state — they should survive a refresh.

## 4. Store discipline (when stateful) (rules 10, 16)

**Rule 10: when client state needs a store, use a single immutable store with one observer — `getState`, `setState`, `subscribe`, with `Object.is` bail-out on equal values.**

One diff site replaces scattered notifications. Every mutation goes through `setState`, which compares the next snapshot to the previous one and notifies subscribers exactly once per real change. Selectors read via `getState` and re-run on subscribe; identity comparison cuts redundant renders. Rule 10 forbids ad-hoc event emitters, multiple competing stores, and direct mutation of store internals.

**Rule 16: state shape is `Frozen<T>` deep-readonly, applied once at the store edge — not per field, not per consumer.**

`Object.freeze` (or a `DeepReadonly<T>` type) is applied to the snapshot inside `setState` before publishing it. Consumers receive a value that the type system and the runtime both refuse to mutate. Rule 16 means you write the freeze once and forget it; you do not sprinkle `as const` or `Readonly<>` through every selector.

```ts
type Store<T> = {
  getState(): Readonly<T>;
  setState(next: T): void;
  subscribe(fn: () => void): () => void;
};

function createStore<T>(initial: T): Store<T> {
  let state = deepFreeze(initial);
  const subs = new Set<() => void>();
  return {
    getState: () => state,
    setState: (next: T) => {
      if (Object.is(next, state)) return;
      state = deepFreeze(next);
      subs.forEach(fn => fn());
    },
    subscribe: (fn: () => void) => {
      subs.add(fn);
      return () => subs.delete(fn);
    },
  };
}

// selector hook with identity bail-out
function useStoreSelector<T, U>(
  store: Store<T>,
  select: (s: Readonly<T>) => U,
): U {
  const [snap, setSnap] = useState(() => select(store.getState()));
  useEffect(() => {
    return store.subscribe(() => {
      const next = select(store.getState());
      setSnap(prev => (Object.is(prev, next) ? prev : next));
    });
  }, [store, select]);
  return snap;
}
```

Selectors should be defined next to the store, not inline at call sites. A selector is a pure `(state) => slice` function; pairing it with a `useStore(selector)` hook gives every consumer the bail-out behavior for free. Inline selectors in render bodies often skip memoization and re-run on every render — usually fine, but a known footgun for object-returning selectors.

Caveat: if the app has no shared client state worth naming, skip the store entirely. Don't introduce rule 10 / rule 16 machinery for a single piece of theme state — a context provider is enough. The store earns its complexity once you have three or more pieces of cross-cutting client state and at least one of them changes often enough that prop drilling hurts.

## 5. Boundary discipline at the API client (rules 17, 18, 22)

**Rule 18 (parse, don't validate): at the API client boundary, transform every response into a fully-typed domain value once. Downstream code never re-checks shape.**

Parsing happens exactly once, at the edge. The parser returns either a domain value or a structured error; from that point on the type system carries the guarantee. Rule 18 forbids defensive `if (user && user.id)` checks in components — if the value reached the component, it is already valid.

**Rule 17 (value objects): primitives don't cross module boundaries. Wrap `UserId`, `Email`, `Money`, `Iso8601` in branded types or value objects, constructed once at the API client edge.**

Components and hooks never accept a raw `string` for an id or a raw `number` for an amount. Rule 17 makes "wrong id passed to wrong endpoint" a compile error rather than a runtime 404.

**Rule 22 (anti-corruption layer): the API client is the ONLY place that knows about transport types — raw JSON, status codes, header names, wire formats. It translates them into domain types immediately.**

Components must never see a `Response` object, a status code, or a header. rule 22 means swapping REST for GraphQL touches the API client and nothing else. A practical check: search the codebase for `Response`, `fetch(`, `status`, and `headers` outside the API client folder. Any hit is a rule 22 violation and a future migration tax.

The three rules chain: rule 22 says the API client is the only module that sees transport shapes; rule 18 says it converts those shapes to domain values exactly once; rule 17 says those domain values use branded types so the conversion sticks. Together they form a one-way gate — untyped data goes in, typed data comes out, and everything downstream can trust the type system.

```ts
// branded primitives: constructed once, trusted forever
type UserId = string & { readonly __brand: 'UserId' };
type Email  = string & { readonly __brand: 'Email' };
type Iso8601 = string & { readonly __brand: 'Iso8601' };

// domain type: components consume this, never raw JSON
type User = {
  id: UserId;
  email: Email;
  createdAt: Iso8601;
};

// parser: single entry point for untyped data
function parseUser(json: unknown): Result<User, ParseError> {
  if (!isObject(json)) return err({ kind: 'shape' });
  if (typeof json.id !== 'string') return err({ kind: 'field', field: 'id' });
  const email = parseEmail(json.email);
  if (email.isErr()) return err({ kind: 'field', field: 'email' });
  const createdAt = parseIso(json.createdAt);
  if (createdAt.isErr()) return err({ kind: 'field', field: 'createdAt' });
  return ok({
    id: json.id as UserId,
    email: email.value,
    createdAt: createdAt.value,
  });
}

// API client method: transport in, domain out
async function getUser(id: UserId): Promise<Result<User, DomainError>> {
  const res = await http.get(`/users/${id}`);
  if (!res.ok) return err(normalizeError(res));
  return parseUser(await res.json());
}
```

## 6. Functional core for view logic (rule 20)

**Rule 20: derive presentational data from props and state via pure functions. Side effects live in hooks (effects, mutations, event handlers), never in render bodies.**

Render bodies are pure: same input → same output, no I/O, no clock reads, no random, no mutations, no network. Sorting, filtering, grouping, formatting, deriving a label from a status — all of these are pure functions called from the render body or memoized via the framework's standard hook. Anything that touches the outside world (focus management, scroll, timers, navigation, mutations) is wrapped in an effect or a handler.

Concretely: a date formatter is pure, a `useEffect` that scrolls to the top on route change is not. A `groupBy(orders, 'status')` call is pure, a `localStorage.setItem` call is not. A `cn(...)` class merge is pure, a `router.push(...)` call is not. The pure half belongs in (or alongside) the render body; the impure half belongs in a hook or handler.

Rule 20 makes components trivially testable: feed inputs, assert output. It makes them safe to re-render: re-running a pure function is free of consequence. It makes the boundary between "what to show" and "when to do something" explicit.

```ts
// pure: lives outside or inside render body
function summarize(orders: Order[]): OrderSummary { /* ... */ }

function OrdersView({ orders }: { orders: Order[] }) {
  const summary = summarize(orders);          // pure
  useEffect(() => { trackView('orders'); }, []); // effect
  return <Summary data={summary} />;
}
```

## 7. Dependency wiring in hooks (rules 23, 24)

**Rule 23: hooks accept their dependencies — clients, clocks, id generators, loggers — as parameters or via a single `Providers` context. No module-level singletons reached via `import { apiClient }` inside a hook body.**

rule 23 makes hooks testable in isolation: the test supplies a fake `Providers` value, no module mocking required. It also makes multi-tenant or multi-environment apps trivial: swap the provider, get a different client. A hook that imports `apiClient` from a module is welded to that module; a hook that reads `useContext(Providers).apiClient` is portable.

Module-level singletons are fine at the app entry point (where the real providers are constructed) and inside the API client itself. They are forbidden inside hooks and components because they hide dependencies and break test isolation.

**Rule 24: `Date.now()`, `Math.random()`, `crypto.randomUUID()`, and `performance.now()` only appear inside adapter modules. Components and hooks reach them via injected `Clock`, `Rng`, `IdGenerator` interfaces.**

rule 24 kills flaky time-dependent tests and makes deterministic replay possible. A component that reads `Date.now()` directly cannot be tested without freezing global time, and a component that calls `crypto.randomUUID()` at render time produces a different id on every rerender — a subtle source of key churn in lists.

The `Providers` context should expose a single object with every injected dependency (`{ apiClient, clock, rng, idGen, logger }`). One context, one provider at the app root, one override point in tests. Multiple parallel contexts for each dependency fragment the tree and make test setup painful.

```ts
type Clock = { now(): number };
type Rng = { next(): number };
type IdGenerator = { next(): string };

type Providers = {
  apiClient: ApiClient;
  clock: Clock;
  rng: Rng;
  idGen: IdGenerator;
  logger: Logger;
};

const ProvidersContext = createContext<Providers | null>(null);

function useProviders(): Providers {
  const p = useContext(ProvidersContext);
  if (!p) throw new Error('ProvidersContext missing at app root');
  return p;
}

function useNow(intervalMs = 1000): number {
  const { clock } = useProviders();
  const [now, setNow] = useState(() => clock.now());
  useEffect(() => {
    const id = setInterval(() => setNow(clock.now()), intervalMs);
    return () => clearInterval(id);
  }, [clock, intervalMs]);
  return now;
}
```

## 8. Errors at the data-layer boundary (rules 12, 13, 19)

**Rule 19: the API client maps transport errors to domain errors once. Components never inspect status codes. Expected failures are part of the query/mutation result type; throwing is reserved for bugs and unrecoverable transport faults.**

A 404 on `getUser` becomes `NotFoundError`. A 409 on `createOrder` becomes `ConflictError`. A 401 anywhere becomes `AuthError`. The component sees `result.status === 'error'` and a tagged `error.kind`, never a number. rule 19 means error UI is driven by domain meaning, not HTTP trivia. A product manager says "if the order already exists, show a dupe warning"; the code already has `ConflictError` and renders the correct banner with zero transport knowledge.

The mapping lives in one function inside the API client — typically a `switch` on status plus a parse of the error body. That function is the single place where "status 409 means conflict" is encoded. Moving endpoints around or changing backends updates that function and nothing else.

**Rule 12: custom error classes per domain concept (`NotFoundError`, `ConflictError`, `AuthError`, `ValidationError`). Always set `this.name` in the constructor so `instanceof` survives minification and stack traces stay readable.**

**Rule 13: a single `normalizeError` function in the API client converts unknown thrown values into a domain error union. Group type-guard predicates (`isAuthError`, `isNotFound`, `isValidation`) so callers cannot forget a code path.**

```ts
class NotFoundError extends Error {
  readonly kind = 'not_found' as const;
  constructor(public resource: string) {
    super(`${resource} not found`);
    this.name = 'NotFoundError';
  }
}

class ConflictError extends Error {
  readonly kind = 'conflict' as const;
  constructor(public resource: string, public reason: string) {
    super(`${resource} conflict: ${reason}`);
    this.name = 'ConflictError';
  }
}

class AuthError extends Error {
  readonly kind = 'auth' as const;
  constructor(public reason: 'unauthenticated' | 'forbidden') {
    super(`auth: ${reason}`);
    this.name = 'AuthError';
  }
}

type DomainError = NotFoundError | ConflictError | AuthError | ValidationError;

function normalizeError(res: Response): DomainError {
  switch (res.status) {
    case 401: return new AuthError('unauthenticated');
    case 403: return new AuthError('forbidden');
    case 404: return new NotFoundError(extractResource(res));
    case 409: return new ConflictError(extractResource(res), extractReason(res));
    case 422: return new ValidationError(extractFields(res));
    default:  throw new Error(`unhandled transport status ${res.status}`);
  }
}

const isNotFound = (e: DomainError): e is NotFoundError => e.kind === 'not_found';
const isConflict = (e: DomainError): e is ConflictError => e.kind === 'conflict';
const isAuthError = (e: DomainError): e is AuthError => e.kind === 'auth';
```

rule 12 and rule 13 work together: `instanceof` for catch sites that need the class, type guards for switch sites that need exhaustiveness.

A mutation result type should look like `{ status: 'success'; data: T } | { status: 'error'; error: DomainError }`, and the `DomainError` is itself a discriminated union of the domain error classes. The component switches on `error.kind` and gets exhaustiveness from rule 14. No `error.message.includes(...)` parsing, ever.

## 9. Types and exhaustiveness (rules 11, 14, 15)

**Rule 11: query/mutation states, reducer actions, and component variants are discriminated unions, not booleans plus nullables.**

Replace `{ isLoading: boolean; error: Error | null; data: T | null }` with `{ status: 'idle' } | { status: 'loading' } | { status: 'error'; error: DomainError } | { status: 'ready'; data: T }`. The body of rule 11 makes impossible states unrepresentable — you cannot have `isLoading: true` and `data: nonNull` at the same time, because the type forbids it. The four-value boolean-and-nullable model has 2 × 2 × 2 = 8 representable states, of which only 4 are valid; the tagged union has exactly 4 representable states, all valid.

**Rule 14: no `default` branches on a discriminated union. Use `assertNever(x)` so adding a new variant breaks the build at every switch site.**

The point of rule 14 is compile-time exhaustiveness: when a new status or action variant lands, every site that handles the union lights up red until you handle it. A `default:` branch defeats this by silently swallowing new cases, usually producing a blank screen or a stale UI that the team only notices days later.

`assertNever` is one line and goes in a shared `utils` module. Every switch on a discriminated union ends with `default: return assertNever(x)`. The call is unreachable at runtime if types are honest; if a cast or an `any` sneaks a new variant in, it throws loudly instead of silently defaulting.

```ts
type QueryState<T> =
  | { status: 'idle' }
  | { status: 'loading' }
  | { status: 'error'; error: DomainError }
  | { status: 'ready'; data: T };

function assertNever(x: never): never {
  throw new Error(`unhandled variant: ${JSON.stringify(x)}`);
}

function render(s: QueryState<User>) {
  switch (s.status) {
    case 'idle':    return <Idle />;
    case 'loading': return <Spinner />;
    case 'error':   return <ErrorView e={s.error} />;
    case 'ready':   return <UserCard user={s.data} />;
    default:        return assertNever(s);
  }
}
```

**Rule 15: type-guard functions are the ONLY narrowing path. Ban `as` casts in component code.** Casts are allowed inside the API client (rule 18 parsing) and value-object constructors (rule 17), and nowhere else. rule 15 means a cast in a component is a code-review block.

Type guards are trivial to write and name well: `isOrder(x): x is Order`, `isReady(s): s is Extract<QueryState<T>, { status: 'ready' }>`. Put them next to the type they narrow. A well-placed guard removes every downstream `if (x && x.foo)` check.

Non-null assertions (`x!`) are casts under a different name and are banned by rule 15 in component code for the same reason. If you know a value is non-null, prove it to the type system with a narrowing check or an early return.

## 10. Anti-patterns

Each of these is a review-blocking violation. Grep your diff for these shapes before opening a PR:

- `fetch(` outside `lib/api/`
- `as ` in component files (excluding `as const`)
- `Date.now(` or `Math.random(` outside adapter modules
- `isLoading` alongside `data` and `error` in the same type
- `default:` inside a `switch` on a tagged status field
- `Response` as a parameter type outside the API client
- imports from `routes/*/−components/` outside that same route

- Fetching directly from a component (skip the hook + query + API client layers)
- `useState` holding data the server already owns
- A global-store entry used by exactly one component
- Status modeled as `isLoading: boolean + error: Error | null + data: T | null` instead of a tagged union
- Inline `fetch()` in a hook that bypasses the API client
- `as` casts in component code
- `Date.now()` (or `Math.random()`, `crypto.randomUUID()`) inside a render body
- A `Response` object or HTTP status code reaching a component
- Re-exporting a feature-private component from a shared barrel
- A higher-tier component (`ui/`) importing from a lower tier (`shared/` or feature folder)
- A `default:` branch on a discriminated-union switch
- Defensive shape checks downstream of the API client parser
- Module-level singleton imports inside hook bodies
- Mutating a store snapshot in place instead of producing a new one
- Catching an error and re-throwing it as a generic `Error` (loses domain kind)

> Source patterns: spec docs/superpowers/specs/2026-04-07-workshop-architecture-design.md