# Architecture: Core

This document covers the structure and rules for the Functional Core layer.

---

## Overview

The core contains all pure domain logic. It has no side effects, no framework imports, and
no IO. Everything in the core is deterministic: same input always produces same output.

The core lives in `core` packages, organized into modules. See
[architecture-repository.md](./architecture-repository.md) for how packages are laid out.

A core module is organized into four directories, ordered from lowest to highest strata.
Dependencies only flow upward. A lower stratum never depends on a higher one.

```
/module-name
  /extensions
  /units
  /contracts
  /use-cases
```

---

## Polyfills

Polyfills such as `Option<T>` and `Result<T, E>` do not live in core modules. They live in
`polyfill` packages, which the core can import. See
[architecture-repository.md](./architecture-repository.md#polyfill).

---

## Extensions

Extensions add operators to a unit the module does not own. In the core, that is a language
or stdlib type, a polyfill or library type, or a unit from `common`. See
[architecture-unit.md](./architecture-unit.md#extensions) for the full rules.

```
/extensions
  /array
    index.ts
    array.ext.ts    # extends Array<T>
```

---

## Units

Units are the primary building block of the core. They follow all the rules in
[architecture-unit.md](./architecture-unit.md).

Core units must be pure and immutable without exception. Unlike shell units, they never
deviate from this rule.

Because of that, unit tests in the core should ideally reach 100% coverage without any
mocks. See [testing.md](./testing.md).

---

## Contracts

A contract is a function type that describes what data the domain needs from the outside
world, without specifying how it is fetched. The shell provides the implementation.

Contracts depend on units but not on use cases.

```
/contracts
  /find-user-by-id
    index.ts
    find-user-by-id.ts           # FindUserById function type
    find-user-by-id-request.ts   # FindUserByIdRequest if needed
    find-user-by-id-response.ts  # FindUserByIdResponse if needed
```

```ts
// --- Unit ---

export type FindUserById = (request: FindUserByIdRequest) => Promise<Option<FindUserByIdResponse>>
```

No constructor, no deps. Just the contract. The shell wires the actual implementation.

---

## Use cases

A use case is a function type representing a specific operation the shell wants to perform.
Use cases are the entry point for any operation that needs the outside world. The shell
may still call unit operators directly, since they are pure and immutable.

Use cases orchestrate units and contracts to fulfill their operation. They depend on units,
contract types, and other use case types, but never on their implementations. Composition
of implementations is the shell's responsibility.

```
/use-cases
  /send-welcome-email
    index.ts
    send-welcome-email.ts
    send-welcome-email-deps.ts
    send-welcome-email-request.ts
    send-welcome-email-response.ts
```

```ts
// --- Unit ---

export type SendWelcomeEmail = (request: SendWelcomeEmailRequest) => Promise<Result<SendWelcomeEmailResponse, SendWelcomeEmailError>>

// --- Constructors ---

export const createSendWelcomeEmail = (deps: SendWelcomeEmailDeps): SendWelcomeEmail =>
  async (request) => {
    // implementation
  }
```

Supporting files are created only when needed. Use cases can depend on other use case types
via the deps type, making them composable. The shell wires the actual implementations.

---

## Transactions and consistency

Sometimes several changes must happen together or not at all. For example, placing an order
saves the order and reduces the stock. If only one of them is saved, the data is wrong.

The core decides **what** must change together. The shell decides **how**, for example with
a database transaction. Use these rules, in this order.

### 1. One contract per change that must happen together

This is the default. When several changes must happen together, make them one contract
call. The use case builds everything that needs saving and hands it over in one go. The
shell implementation saves it all inside one database transaction.

```ts
// --- Unit ---

export type SavePlacedOrder = (request: SavePlacedOrderRequest) => Promise<Result<void, SaveError>>

// SavePlacedOrderRequest = { order: Order; stock: Array<Stock> }
```

The core never sees the word "transaction". It only sees a contract that saves an order
together with its stock.

### 2. Concurrent changes: a version field

Two requests may read the same data, change it, and save it at the same time. The second
save would then silently overwrite the first. To prevent this, the unit carries a `version`
number:

- The shell saves a change only if the version is still the one that was read, and raises
  the version when it does.
- If someone else saved first, the contract returns `{ kind: 'conflict' }` as an error
  value, not an exception.
- The use case decides what a conflict means: read again and retry, or return the conflict
  to the caller.

```ts
// --- Unit ---

export type Stock = {
  productId: ProductId
  available: number
  version: number
}
```

```ts
// Inside a use case: read again and retry on conflict, up to a limit
for (let attempt = 1; attempt <= MAX_ATTEMPTS; attempt++) {
  const found = await deps.findStockByProductId({ productId: request.productId })
  if (found.isNone()) return Result.err({ kind: 'productNotFound' })

  const reserved = reserve(found.value, request.quantity)
  if (reserved.isErr()) return reserved

  const saved = await deps.saveStock({ stock: reserved.value })
  if (saved.isOk()) return Result.ok(reserved.value)
}

return Result.err({ kind: 'conflict' })
```

Retrying is safe because unit operators are pure. Running `reserve` again has no side
effects to repeat. Transformers leave `version` as it is. Only the shell raises it.

This also works when a change spans more than one request, such as a user editing a form
for several minutes. The form sends back the version it loaded, so a save based on old data
returns a conflict instead of overwriting someone else's change.

### 3. A transaction contract, when the use case must control it

Use this only when rules 1 and 2 are not enough, for example when a use case must read,
decide, and write several things while nothing else changes them.

A transaction contract is a function type like any other contract. It takes the work to do,
and gives that work the contracts it may use inside the transaction.

```ts
// --- Unit ---

export type RunInOrderTransaction = (work: OrderTransactionWork) => Promise<Result<void, OrderError>>
```

```ts
// --- Unit ---

export type OrderTransactionWork = (contracts: OrderTransactionContracts) => Promise<Result<void, OrderError>>

// OrderTransactionContracts = { findStockByProductId: FindStockByProductId; saveOrder: SaveOrder; saveStock: SaveStock }
```

```ts
// Inside a use case
return deps.runInOrderTransaction(async ({ findStockByProductId, saveOrder, saveStock }) => {
  // read, decide, and write with the contracts given here
})
```

The shell implementation must roll the transaction back when the work returns an `Err`,
not only when it throws. Expected errors are values, so most rollbacks come from an `Err`.

### 4. Across modules: no shared transactions

A module is where consistency stops, the same way it is where direct imports stop. One
transaction never spans two modules.

When two modules must both change, the first module saves its change and publishes an event.
The event is saved in the same transaction as the change, so it is never lost. This is
called the outbox pattern. The second module reacts to the event and makes its own change.

### 5. Never use hidden transactions

Do not open a transaction somewhere outside the use case, such as once per request in a
middleware, and let contracts pick it up without being told. It is invisible in the code,
hard to test, and keeps transactions open longer than needed.

---

## Dependency direction

From lowest to highest:

1. Extensions depend on polyfill packages and the unit they extend
2. Units depend on polyfill packages, extensions, `common`, and other units at lower or
   equal strata
3. Contracts depend on units
4. Use cases depend on units, contract types, and other use case types

The shell depends on all of the above. Nothing in the core depends on the shell.

---

## Summary

| Concept | Purpose | Depends On |
|---|---|---|
| Extensions | Add operators to a unit the module does not own | Polyfill packages, the extended unit |
| Units | Domain objects and their operators | Polyfill packages, extensions, `common`, lower units |
| Contracts | Function types for fetching external data | Units |
| Use cases | Entry points for shell operations | Units, contract types, other use case types |
| Transactions | One contract per change that must happen together, a version field for concurrent changes, a transaction contract only when needed | Contracts |
