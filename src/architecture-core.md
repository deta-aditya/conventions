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
