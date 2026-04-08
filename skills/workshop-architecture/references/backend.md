# Backend Architecture Reference

Stack-agnostic rules for scaffolding any backend in a workshop project. Read this before writing code. Rules are numbered and greppable: search `rule N` to jump.

This document is prescriptive. Apply every rule. When a rule conflicts with a framework convention, follow the rule and isolate the framework quirk in an adapter.

---

## 1. Layer map and dependency direction (rules 1, 9)

**rule 1: every backend has exactly three layers — views/controllers, services, models/entities — and this is the default split for any new module.**

**rule 9: imports flow downward only. Any upward import is a bug. Utility modules import nothing from the application.**

The layer map is a directed acyclic graph with arrows pointing from the outside in:

```
views/controllers   →   services   →   models/entities
   (HTTP edge)         (use cases)      (domain types)
```

A view may import a service. A service may import models. A model imports nothing application-specific. A util may be imported by anyone but imports only the standard library and other utils. If you find yourself wanting a service to import a view helper, the helper belongs in a util or in the model layer.

The point of rule 1 is that an AI assistant scaffolding a new feature has zero ambiguity about where files go. The point of rule 9 is that cycles and back-edges are the single largest source of architectural rot, and they are mechanically detectable. Configure the linter to fail the build on upward imports.

> Example: in `~/Projects/abertrades/abertrades-backend` this manifests as `securities/views → securities/services → securities/models`. The same triple appears for every domain module.

```
module/
  views.ts        // imports services
  services.ts     // imports models
  models.ts       // imports nothing app-specific
  serializers.ts  // imported by views only
```

---

## 2. Thin views (rule 2)

**rule 2: a view does exactly three things — validate the request, call one service method, serialize the response. Nothing else.**

A view is a translator between the transport (HTTP, gRPC, queue message) and the domain. It must contain no business logic, no database access, no orchestration of multiple services, and no transaction management. If the endpoint needs to call two services, write a third service that composes them and call that one from the view.

Forbidden inside a view:

- Direct database queries or ORM calls.
- Business rules (`if user.tier == 'gold' then ...`).
- External HTTP calls or vendor SDK invocations.
- Multi-service composition or fan-out.
- Transaction begin/commit/rollback.
- Caching decisions.
- Retry loops.

The reason is testability and substitutability. A thin view can be replaced with a different transport (CLI, queue worker, cron) without rewriting business logic. A fat view forces a rewrite every time the transport changes.

```ts
// view (pseudocode)
function handleCreateOrder(req) {
  const input = parseCreateOrderRequest(req.body); // validate
  const order = orderService.create(input);        // call service
  return serializeOrder(order);                    // serialize
}
```

---

## 3. Services as the only mutation/query boundary (rules 3, 25)

**rule 3: every write goes through a service method tagged as a mutation (transactional). Every read goes through a service method tagged as a query (no side effects).**

**rule 25: Command-Query Separation. A method either writes and returns an acknowledgement, or reads and returns data. Never both. No `getOrCreate`, no `findAndUpdate` returning the new value, no `popFromQueue`.**

Services own the transaction boundary. A mutation method opens a transaction, performs all writes, commits, and returns a minimal acknowledgement (an id, a status, or void). A query method reads consistent data and returns a domain entity. Mixing the two creates methods whose behavior depends on whether the caller cared about the return value, which is impossible to reason about and impossible to cache.

Rule 25 forbids the convenient-but-corrosive pattern of returning a freshly-created entity from `create`. If the caller needs the new entity, the caller issues a follow-up query. This sounds wasteful but it eliminates a category of bugs where the returned object disagrees with the persisted state.

```ts
class OrderService {
  @mutation
  create(input: CreateOrderInput): OrderId { /* tx, write, return id */ }

  @query
  getById(id: OrderId): Order { /* read, return entity */ }
}
```

The decorators are illustrative; the enforcement can be a naming convention (`createOrder` vs `getOrder`) plus a lint rule that bans return types other than `void | Id | Ack` from mutation methods.

---

## 4. Colocated request/response serializers (rule 4)

**rule 4: the serializer file lives next to the endpoint file. The serializer is the API contract. There is no separate schema document.**

A serializer defines the exact shape of what enters and leaves the HTTP boundary. Because it is code, it cannot drift from reality. Because it lives next to the view, an AI assistant editing the endpoint sees the contract change in the same diff.

Forbidden:

- Hand-written API documentation in markdown that duplicates serializer fields.
- OpenAPI specs maintained separately from the serializer (generate them from the serializer instead).
- "Schema" directories far away from endpoints.

Rule 4 means the serializer is the single source of truth. Generated docs are fine; parallel docs are not.

```
orders/
  views.ts
  serializers.ts   // CreateOrderRequest, OrderResponse — colocated
  services.ts
```

---

## 5. Entities, not rows (rule 5)

**rule 5: services return domain entities. Raw ORM rows, ResultSets, or database records never cross the service boundary.**

A domain entity is a typed object whose shape is dictated by the business, not by the database schema. The mapping from row to entity happens inside the service (or in a repository owned by the service). Once the entity exists, the row is discarded.

Forbidden: returning a Django model instance, a SQLAlchemy row, a Prisma client object, a Knex result, or any vendor-flavored shape from a service. The caller must not know which ORM is in use.

The rationale is twofold. First, ORM objects carry hidden behavior (lazy loading, change tracking, session affinity) that breaks when the object escapes its session. Second, the database schema is an implementation detail; entities are the domain. Coupling callers to rows means every schema change ripples through unrelated code.

```ts
// inside service
const row = await repo.findById(id);
if (!row) return null;
return toOrderEntity(row); // translate before returning
```

---

## 6. Boundary discipline: parse-don't-validate, value objects, anti-corruption layer (rules 17, 18, 22)

**rule 18 (parse don't validate): at every input edge — HTTP body, queue message, file content, environment variable — transform untrusted data into a fully-typed domain value once. Downstream code never re-checks shape.**

**rule 17 (value objects): primitives never cross module boundaries. Wrap `UserId`, `Email`, `Money`, `OrderId` in typed value objects built once at the edge by a smart constructor. An invalid instance cannot exist.**

**rule 22 (anti-corruption layer): never let a third-party schema (vendor SDK type, legacy DB row, partner API payload) leak past its adapter. Translate to a domain type immediately at the seam so external churn stops at one file.**

These three rules together implement the principle that the inside of the system speaks domain language only. Untyped strings, untrusted blobs, and vendor objects exist only in adapters. Once data is inside, its shape is guaranteed by the type system.

Rule 18 means a single parse step at the boundary returns a `Result<DomainInput, ParseError>`. There is no `if (typeof x === 'string')` deeper in the call stack. Rule 17 means a function that takes a `UserId` cannot be called with an arbitrary string — the compiler refuses. Rule 22 means a Stripe webhook payload is converted to a `PaymentEvent` in `stripe-adapter.ts` and the domain never sees the word "Stripe".

```ts
// smart constructor (pseudocode)
type UserId = string & { readonly __brand: 'UserId' };

function makeUserId(raw: string): Result<UserId, Error> {
  if (!/^usr_[a-z0-9]{24}$/.test(raw)) {
    return err(new Error('invalid user id'));
  }
  return ok(raw as UserId);
}
```

---

## 7. Functional core, imperative shell + ports & adapters (rules 20, 21)

**rule 20: push all I/O — DB, network, clock, randomness, filesystem — to the outermost layer. The core takes plain data in, returns plain data out, performs no side effects. The interesting logic becomes trivially testable with no mocks.**

**rule 21: for every external system, define a port (an interface owned by the core); concrete adapters live outside and are injected. The core never imports a vendor SDK directly.**

The functional core is a set of pure functions that compute decisions: given the current state and an input, return the next state and a list of effects to perform. The imperative shell takes those effects and executes them against the real world. Tests of the core need no database, no clock, no network — they pass plain data and assert plain data.

Rule 21 enforces dependency inversion at the I/O boundary. The core defines `interface OrderRepository { findById(id): Order | null }`. The adapter `PostgresOrderRepository implements OrderRepository`. The core imports the interface; the wiring code imports both and injects the adapter at startup. Swapping Postgres for DynamoDB touches one file.

```
+----------------+      +-----------+      +------------+
|  domain core   | ---> |  port     | <--- |  adapter   | ---> vendor SDK
|  (pure logic)  |      | (iface)   |      | (concrete) |
+----------------+      +-----------+      +------------+
```

The arrow from core to port is "depends on". The arrow from adapter to port is "implements". The vendor SDK is reachable only through the adapter.

---

## 8. Dependency wiring (rules 23, 24)

**rule 23: every dependency arrives as an explicit parameter. No globals, no service locators, no `import { db }` inside business code.**

**rule 24: `Clock`, `Rng`, and `IdGenerator` are dependencies too. Direct calls to `Date.now()`, `Math.random()`, or `uuid()` are banned outside adapters.**

Rule 23 makes the dependency graph visible in constructor signatures. Reading a class header tells you everything it can possibly do. Hidden globals defeat this and turn tests into archaeology.

Rule 24 extends the same logic to the things programmers usually forget are dependencies. A function that calls `Date.now()` is not pure — its output depends on wall-clock time. A function that calls `Math.random()` is not deterministic. Wrapping these as injectable services makes time and randomness controllable in tests and replaceable in production (e.g., a monotonic clock for benchmarks).

```ts
class OrderService {
  constructor(
    private readonly repo: OrderRepository,
    private readonly clock: Clock,
    private readonly idGen: IdGenerator,
  ) {}

  create(input: CreateOrderInput): OrderId {
    const id = this.idGen.next();
    const now = this.clock.now();
    this.repo.insert({ id, createdAt: now, ...input });
    return id;
  }
}
```

Wiring happens once, at the composition root (the entry point). Everywhere else, dependencies arrive through constructors or function parameters.

---

## 9. Errors at boundaries (rules 12, 13, 19)

**rule 19: expected failures (not found, conflict, validation error, insufficient funds) live in the return type — `Result<T, E>`, `(value, err)`, or a discriminated union. Throw only for programmer errors and infrastructure faults.**

**rule 12: custom error classes per domain. Always set `this.name = 'MyError'` in the constructor so `instanceof` survives minification and cross-realm boundaries.**

**rule 13: a single `normalizeError` function lives at the outer boundary (the controller layer). Use grouped type-guard predicates like `isFsInaccessible(e)` and `isConflict(e)` so callers cannot forget an errno code.**

Rule 19 forces the type system to enumerate failure modes. A function that returns `Result<Order, OrderError>` cannot be called without handling `OrderError`. This eliminates the silent-swallow class of bugs. Throwing is reserved for things the caller cannot reasonably handle: out-of-memory, programmer mistakes, lost database connections.

Rule 12 ensures `instanceof MyError` works in all environments. Bundlers rename classes; the explicit `this.name` survives. Rule 13 centralizes the translation from arbitrary thrown values into HTTP responses. Every controller funnels uncaught throws through `normalizeError`, which uses the grouped predicates to decide on a status code.

```ts
class OrderNotFound extends Error {
  constructor(id: string) { super(`order ${id} not found`); this.name = 'OrderNotFound'; }
}

function isConflict(e: unknown): e is ConflictError {
  return e instanceof Error && (e.name === 'Conflict' || e.name === 'VersionMismatch');
}
```

---

## 10. Types and exhaustiveness (rules 11, 14, 15)

**rule 11: state machines and lifecycle phases are discriminated unions, not booleans plus nullables. Replace `{ isLoading, error, data }` with `{ status: 'idle' | 'loading' | 'error' | 'ready', ... }`.**

**rule 14: no `default` branches on a discriminated union. Use `assertNever(x)` so adding a new variant breaks the build at every switch.**

**rule 15: type-guard functions are the only narrowing path. `as` casts are banned in business code.**

Rule 11 eliminates impossible states. With three booleans and a nullable, you can represent `isLoading: true, error: someError, data: someData` — a meaningless combination. With a tagged union, only legal states are representable. The compiler refuses to access `data` when `status === 'loading'`.

Rule 14 turns adding a new variant into a compile error in every consumer. This is the single most valuable property of discriminated unions and `default` defeats it. `assertNever` is a one-line helper that takes `never` and throws.

Rule 15 forbids the escape hatch. An `as` cast tells the compiler to stop checking; a type guard tells the compiler how to check. Business code uses guards; only adapter code at the very edge may need a cast, and that cast is reviewed.

```ts
function assertNever(x: never): never { throw new Error(`unexpected ${JSON.stringify(x)}`); }

switch (state.status) {
  case 'idle':    return renderIdle();
  case 'loading': return renderLoading();
  case 'error':   return renderError(state.error);
  case 'ready':   return renderReady(state.data);
  default:        return assertNever(state); // compile error if a variant is missed
}
```

---

## 11. Anti-patterns (short list)

Forbidden in any new code:

- Business logic in views/controllers.
- ORM row leaking past a service.
- Raw primitives crossing module boundaries.
- Catching errors deep inside business code instead of at the boundary.
- `import { db }` inside a use case.
- `Date.now()` inside a use case.
- `default:` branch on a tagged union.
- Returning the new entity from a `create*` method when CQS would split it.
- Hand-maintained API docs parallel to serializers.
- Vendor SDK types reaching the domain core.
- `as` casts inside business logic.
- Globals or singletons substituting for constructor parameters.

> Source patterns: spec docs/superpowers/specs/2026-04-07-workshop-architecture-design.md
