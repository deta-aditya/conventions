# Architecture: Core

This document covers the structure and rules for the Functional Core layer.

---

## Overview

The core contains all pure domain logic. It has no side effects, no framework imports, and
no IO. Everything in the core is deterministic: same input always produces same output.

A core module is organized into five directories, ordered from lowest to highest strata.
Dependencies only flow upward. A lower stratum never depends on a higher one.

```
/module-name
  /polyfills
  /extensions
  /units
  /data-gateways
  /use-cases
```

---

## Polyfills

Polyfills fill gaps in TypeScript's type system or standard library. Examples include
validation schemas, `Option<T>`, `Result<T, E>`, and discriminated union constructors.

These are temporary by nature. Once a library covers the need (`gado` for `Option`/`Result`,
`yamin` for unions, `zod` for validation schemas), the polyfill is replaced by the library.
The rest of the codebase does not need to change since the interface stays the same.

---

## Extensions

Extensions add operators to existing language or stdlib types without modifying them. An
extension treats a language type as its unit and adds constructors, selectors, and
transformers around it.

```
/extensions
  /array
    index.ts
    array.ts    # Array<T> as the unit
```

Too many extensions on the same type is a signal that a new domain unit should be
introduced instead.

---

## Units

A unit is a single exported type and all the operators that work on it. Units are the primary
building block of the core.

### One unit per file

Each file exports exactly one unit. The filename matches the exported type name in
kebab-case. Never put multiple exported units in one file.

```
/units
  /user
    index.ts
    user.ts           # User type
    user-error.ts     # UserError type
```

### Immutability

All domain objects are immutable. No field is ever mutated after construction. Transformers
always return a new instance.

### Non-anemic domain

Functions that operate on a type live next to it, in the same file. There are no `utils/`,
`helpers/`, or floating functions. Every function belongs to a domain concept.

### Block structure

Every unit file follows this block structure:

```ts
// --- Unit ---

type User = {
  id: UserId
  email: Email
  name: string
}

// --- Private ---

const isValidName = (name: string): boolean => name.trim().length > 0

// --- Constructors ---

export const createUser = (params: CreateUserParams): Result<User, UserError> => ...

// --- Selectors ---

export const getFullName = (user: User): string => ...
export const isEligible = (user: User): boolean => ...

// --- Transformers ---

export const updateEmail = (user: User, email: Email): Result<User, UserError> => ...
```

Constructors create or produce the unit, optionally wrapped in `Result` or `Option`.

Selectors accept the unit and return a derived value. They never return the unit itself.

Transformers accept the unit and return the unit or a wrapped unit. They never return a
derived value.

This follows CQS: a function either queries or commands, never both.

### Private block

Private items are non-exported helpers that support the unit's operators. Having too many
private items is a signal that a new unit should be extracted.

Other indicators for extracting a new unit:
- An operator's implementation becomes too long
- A field inside the unit is frequently operated on independently

### Representation

Choose the representation in this order:

1. `type` is the default and should be used for most domain objects
2. Discriminated union when the domain concept has multiple variants
3. `class` only when member hiding, polymorphism, or method chaining API is needed
4. Schema (e.g. Zod) when validation at the boundary is needed and the type is inferred from it

### Make invalid state impossible

Types should encode constraints so that invalid states cannot be represented. If the type
system cannot express the constraint, a constructor function enforces it at runtime.

Use discriminated unions instead of boolean flags:

```ts
// ✅ DO
type UserStatus =
  | { kind: 'active' }
  | { kind: 'suspended'; reason: string }

// ❌ DON'T
type UserStatus = {
  isActive: boolean
  isSuspended: boolean
  suspensionReason?: string  // only meaningful when isSuspended is true
}
```

Use branded types for primitives that should not be interchangeable:

```ts
// ✅ DO
type UserId = string & { readonly __brand: 'UserId' }
type Email = string & { readonly __brand: 'Email' }

const findUser = (id: UserId): Option<User> => ...
```

Use constructor functions that return `Result` or `Option` instead of throwing:

```ts
// ✅ DO
export const createEmail = (value: string): Result<Email, EmailError> => {
  if (!EMAIL_REGEX.test(value)) return Result.err({ kind: 'invalidFormat' })
  return Result.ok(value as Email)
}

// ❌ DON'T
export const createEmail = (value: string): Email => {
  if (!EMAIL_REGEX.test(value)) throw new Error('Invalid email')
  return value as Email
}
```

### Avoid exceptions

Do not use exceptions for control flow or expected error cases. Use `Option` and `Result`
instead.

`Option<T>` is for values that may be absent. `Result<T, E>` is for operations that can
fail with a meaningful error. `Promise<Option<T>>` or `Promise<Result<T, E>>` covers the
async variants.

Throwing is reserved for truly unexpected, unrecoverable situations.

### No side effects

Unit operators must be pure functions. No IO, no mutation, no randomness, no global state.
`Promise` is allowed since it represents an async value, not a side effect itself.

### Stratified design

Units form a stratified design. Lower strata are closer to the language, such as primitive
value objects like `Email`, `UserId`, and `Money`. Higher strata compose from lower ones,
for example `User` uses `Email` and `UserId`, and `Order` uses `User` and `Money`.

A unit at a lower stratum never depends on a unit at a higher stratum.

---

## Data gateways

A data gateway is a function type that describes what data the domain needs from the outside
world, without specifying how it is fetched. The shell provides the implementation.

Data gateways depend on units but not on use cases.

```
/data-gateways
  /find-user-by-id
    index.ts
    find-user-by-id.ts           # FindUserById function type
    find-user-by-id-request.ts   # FindUserByIdRequest if needed
    find-user-by-id-response.ts  # FindUserByIdResponse if needed
```

```ts
// --- Unit ---

type FindUserById = (request: FindUserByIdRequest) => Promise<Option<FindUserByIdResponse>>
```

No constructor, no deps. Just the contract. The shell wires the actual implementation.

---

## Use cases

A use case is a function type representing a specific operation the shell wants to perform.
Use cases are the primary entry point for the shell to interact with the core. The shell never
calls unit operators directly. It always goes through a use case.

Use cases orchestrate units and data gateways to fulfill their operation. They depend on
units, data gateway types, and other use case types, but never on their implementations.
Composition of implementations is the shell's responsibility.

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

type SendWelcomeEmail = (request: SendWelcomeEmailRequest) => Promise<Result<SendWelcomeEmailResponse, SendWelcomeEmailError>>

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

1. Polyfills have no dependencies on other core concepts
2. Extensions depend on polyfills only
3. Units depend on polyfills, extensions, and other units at lower or equal strata
4. Data gateways depend on units
5. Use cases depend on units, data gateway types, and other use case types

The shell depends on all of the above. Nothing in the core depends on the shell.

---

## Summary

| Concept | Purpose | Depends On |
|---|---|---|
| Polyfills | Fill TS/stdlib gaps, replaced by libraries | Nothing |
| Extensions | Add operators to language types | Polyfills |
| Units | Domain objects and their operators | Polyfills, extensions, lower units |
| Data gateways | Contracts for fetching external data | Units |
| Use cases | Entry points for shell operations | Units, gateway types, other use case types |
