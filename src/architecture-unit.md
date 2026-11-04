# Architecture: Unit

This document covers units and extensions, the building blocks used in every layer of the
codebase.

---

## Overview

A unit is a single exported type and all the operators that work on it. Constructors,
selectors, and transformers are the operators, and they all revolve around the unit's type.

Together, the type and its operators form an abstraction barrier. Code outside the unit
works with the type only through its operators, never by reaching into its internals. This
lets the unit change how it is represented without breaking the code that uses it.

Units are not specific to the core. They exist wherever there is a concept worth naming.
Core units model the domain, such as `User`, `Email`, or `Order`. Shell units model things
like page state, actions, commands, or a wrapper around a third-party API.

The rules in this document apply to every unit, wherever it lives, unless a section says
otherwise.

---

## Framework units

Framework units are the building blocks a framework defines, such as React components,
hooks, contexts, providers, and stores. They are not units in the sense of this document.
They follow the framework's own best practices, not the unit rules here. For React, see
[react.md](./react.md).

Keep framework units thin. Logic they need belongs in a unit, where it can be tested as a
pure operator.

---

## One unit per file

Each file exports exactly one unit. The filename matches the exported type name in
kebab-case. Never put multiple exported units in one file.

```
/units
  /user
    index.ts
    user.ts           # User type
    user-error.ts     # UserError type
```

---

## Exports

A unit file exports only the unit's type and its operators. When the type is inferred from
a schema, the schema may be exported too, since it is the type's runtime form. Everything
else, such as
helpers, intermediate types, and constants that are not operators, stays private to the
file.

If something else needs to be exported, it belongs to another unit and should live in its
own file.

---

## Non-anemic

Functions that operate on a type live next to it, in the same file. There are no `utils/`,
`helpers/`, or floating functions. Every function belongs to a unit, or is a framework unit.

---

## Type

The type is what every operator of the unit revolves around. It is always exported, written
in `PascalCase`, and named after its file. It lives in the `Unit` block.

Choose the form of the type in this order:

1. Custom type, the default for most units
2. Discriminated union, when the concept has multiple variants
3. Branded type, when a primitive must not be interchangeable with other values of the same
   primitive
4. Class, only when member hiding, polymorphism, or a method chaining API is needed
5. Schema, when validation at the boundary is needed and the type is inferred from it

### Custom type

A `type` alias, usually of an object. Fields use other units where they exist instead of
raw primitives.

```ts
// --- Unit ---

export type User = {
  id: UserId
  email: Email
  name: string
  status: UserStatus
}
```

### Discriminated union

A union of object types told apart by a `kind` field, with `camelCase` values. See
[naming.md](./naming.md#discriminated-union-variants-camelcase).

```ts
// --- Unit ---

export type UserStatus =
  | { kind: 'active' }
  | { kind: 'suspended'; reason: string }
```

### Branded type

A primitive intersected with a brand, so that TypeScript does not accept a plain `string`
or another branded string in its place. A branded value can only be produced by the unit's
constructor, which validates it. The `as` cast happens only inside that constructor.

```ts
// --- Unit ---

export type Email = string & { readonly __brand: 'Email' }

// --- Constructors ---

export const createEmail = (value: string): Result<Email, EmailError> => {
  if (!EMAIL_REGEX.test(value)) return Result.err({ kind: 'invalidFormat' })
  return Result.ok(value as Email)
}
```

### Class

A class whose methods are its operators. See [Class units](#class-units).

### Schema

A validation schema, such as a Zod schema, still counts as a unit. The schema is the
runtime form of the type, and the type is inferred from it. Both live in the `Unit` block,
and the schema may be exported alongside the type.

Prefer a constructor that parses with the schema and returns a `Result`, so that callers do
not depend on the schema library.

```ts
// --- Unit ---

export const SignUpRequestSchema = z.object({
  email: z.string().email(),
  password: z.string().min(8),
})

export type SignUpRequest = z.infer<typeof SignUpRequestSchema>

// --- Constructors ---

export const parseSignUpRequest = (value: unknown): Result<SignUpRequest, SignUpRequestError> => {
  const parsed = SignUpRequestSchema.safeParse(value)
  if (!parsed.success) return Result.err({ kind: 'invalid', issues: parsed.error.issues })
  return Result.ok(parsed.data)
}
```

---

## Operators

Every exported function of a unit is one of three operators.

| Operator | Parameters | Return value | Type signature | Usage |
|---|---|---|---|---|
| Constructor | Anything except the unit | The unit, optionally wrapped in a generic type | `(...params) => U` or `(...params) => W<U>` | Create or produce the unit |
| Selector | The unit first, other parameters after | A derived value, never the unit | `(unit: U, ...params) => V` or `Record<U, V>` | Query the unit |
| Transformer | The unit first, other parameters after | The unit, optionally wrapped in a generic type | `(unit: U, ...params) => U` or `(unit: U, ...params) => W<U>` | Produce a changed unit |

`U` is the unit's type, `V` is any other type, and `W` is a generic wrapper such as
`Result`, `Option`, `Promise`, or `Array`. `...params` stands for zero or more parameters.

A tuple also counts as a generic wrapper. A reducer returning `[State, Command[]]` is a
transformer, since that return type is `Tuple2<State, Array<Command>>`.

This follows CQS: an operator either queries or commands, never both.

### The unit is always the first parameter

Selectors and transformers take the unit as their first parameter. Putting the unit first
gives every operator the same shape and mirrors the order of a method call, where the
subject comes before the arguments.

### The unit parameter is never wrapped

Return values of constructors and transformers may be wrapped in a generic type. The unit
parameter of selectors and transformers may not. To work on a wrapped unit, unwrap it with
the wrapper's own operators, such as `Option.map`, instead of accepting the wrapper.

```ts
// ✅ DO
export const getFullName = (user: User): string => ...
export const updateEmail = (user: User, email: Email): Result<User, UserError> => ...

// ❌ DON'T
export const getFullName = (user: Option<User>): string => ...                         // wrapped unit parameter
export const updateEmail = (email: Email, user: User): Result<User, UserError> => ...  // unit is not first
```

### Class units

When a unit is a class, its methods are its operators. `this` is the implicit first
parameter of every method, so the unit-first rule holds by itself. Constructors are static
factory methods, and transformers return a new instance instead of mutating `this`. Methods
are grouped in the same blocks as a function unit.

```ts
// --- Unit ---

export class Money {
  private constructor(readonly amount: number, readonly currency: Currency) {}

  // --- Constructors ---

  static create(amount: number, currency: Currency): Result<Money, MoneyError> { ... }

  // --- Selectors ---

  isZero(): boolean { ... }

  // --- Transformers ---

  add(other: Money): Result<Money, MoneyError> { ... }
}
```

### `Record` as a selector

A `Record<U, V>` keyed by the unit is also a valid selector. It maps each value of the unit
to a derived value, without a function. This works when the unit can be used as a key, such
as a string literal union.

```ts
// --- Unit ---

export type UserRole = 'admin' | 'member'

// --- Selectors ---

export const USER_ROLE_LABEL: Record<UserRole, string> = {
  admin: 'Administrator',
  member: 'Member',
}
```

---

## Block structure

Every unit file follows this block structure:

```ts
// --- Unit ---

export type User = {
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

---

## Private block

Private items are non-exported helpers that support the unit's operators. Having too many
private items is a signal that a new unit should be extracted.

Other indicators for extracting a new unit:
- An operator's implementation becomes too long
- A field inside the unit is frequently operated on independently

---

## Make invalid state impossible

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

Use [branded types](#branded-type) for primitives that should not be interchangeable:

```ts
// ✅ DO
const findUser = (id: UserId): Option<User> => ...

// ❌ DON'T
const findUser = (id: string): Option<User> => ...  // any string is accepted, even an email
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

---

## Avoid exceptions

Do not use exceptions for control flow or expected error cases. Use `Option` and `Result`
instead.

`Option<T>` is for values that may be absent. `Result<T, E>` is for operations that can
fail with a meaningful error. `Promise<Option<T>>` or `Promise<Result<T, E>>` covers the
async variants.

Choosing between them is not strict. When in doubt, use `Result`. It always gives clearer
data, since it says what went wrong instead of only that something is missing.

Throwing is reserved for truly unexpected, unrecoverable situations.

---

## Purity and immutability

### In the core

Core units must be pure and immutable. This rule has no exceptions.

- No field is ever mutated after construction. Transformers always return a new instance.
- Operators are pure functions. No IO, no mutation, no randomness, no global state.
- `Promise` is allowed since it represents an async value, not a side effect itself.

### In the shell

Shell units follow the same rules by default. They may deviate when needed, for example a
unit that wraps a stateful third-party SDK, or an operator that has to call a browser API.

When a shell unit deviates, keep the deviation contained:

- Only the operators that need to be impure or mutating are. The rest stay pure.
- Domain logic does not move into the shell unit to sit next to the impure code. It stays
  in the core.

---

## Stratified design

Units form a stratified design. Lower strata are closer to the language, such as primitive
value objects like `Email`, `UserId`, and `Money`. Higher strata compose from lower ones,
for example `User` uses `Email` and `UserId`, and `Order` uses `User` and `Money`.

A unit at a lower stratum never depends on a unit at a higher stratum.

### Use cases are at the top

A use case sits at the top of the strata. It is a very specific unit: its type is a function
type for one operation, and its constructor composes units and contracts from every stratum
below it. Nothing in the core depends on a use case except another use case.

```
use cases               specific    fluid       tests are less valuable
  ^
Order
  ^
User
  ^
Email, UserId, Money
  ^
extensions, language    generic     brittle     tests are more valuable
```

### Bottom versus top

Code at the bottom is generic. It is used by many units above it, so it is brittle: a change
there ripples through everything that depends on it. Keep it stable, and change it with
care.

Code at the top is specific. Few things depend on it, so it is fluid: it can change often,
as the product changes, without affecting much else.

For the same reason, unit tests are more valuable at the bottom than at the top. A bug at the
bottom spreads to every unit that uses it, and a test there protects all of them. A test at
the top protects one operation and is more likely to change along with it. This shapes
where testing effort goes, not the coverage target. See [testing.md](./testing.md#coverage).

### Reaching too deep

When a unit or use case reaches deep into the bottom strata and uses many operators from a
much lower stratum, that may indicate a missing unit in between. The missing unit would
collect those operations under a name, so the higher stratum can use it instead.

This is a signal, not a strict rule. Reaching down to a lower stratum is allowed.

---

## Extensions

An extension adds operators to an existing unit without modifying it. The extended unit is
one the current scope does not own:

- A language or stdlib type, such as `Array<T>` or `Date`, treated as a unit
- A polyfill or library type, such as `Option<T>` or `Result<T, E>`
- A unit from `common`
- In the shell, a unit from the core

A unit the current scope owns never needs an extension. Add the operator to the unit's own
file instead.

An extension file is named `{unit-name}.ext.ts`, after the unit it extends in kebab-case.
The `.ext` suffix tells it apart from the file that defines the unit. It follows the unit
block structure, without the `Unit` block, since the type is defined elsewhere. Its
operators follow the same rules as any other operator, with the extended unit as the first
parameter.

```
/extensions
  /array
    index.ts
    array.ext.ts    # extends Array<T>
  /user
    index.ts
    user.ext.ts     # extends the core User unit (shell only)
```

Too many extensions on the same unit is a signal that a new unit should be introduced
instead.

---

## Testing

Every unit is tested through its exported operators. Core units should ideally reach 100%
coverage without mocks, and shell units at least 70%. See [testing.md](./testing.md) for
how tests are written.

---

## Summary

| Rule | Description |
|---|---|
| Abstraction barrier | Code outside the unit uses it only through its operators |
| Framework units | Components, hooks, contexts, and stores follow the framework, not these rules |
| Class units | Methods are operators, `this` is the implicit first parameter |
| One unit per file | Filename matches the type name in kebab-case |
| Exports | Only the unit's type and its operators |
| Non-anemic | Operators live next to their type, no `utils/` or `helpers/` |
| Operators | Constructors, selectors, and transformers |
| CQS | Selectors return derived values, transformers return the unit |
| Unit parameter | Always first, never wrapped |
| Return value | Constructors and transformers may wrap the unit in a generic type |
| Block structure | Unit, Private, Constructors, Selectors, Transformers |
| Type | Custom type first, then union, branded, `class`, schema |
| Invalid state impossible | Discriminated unions, branded types, validating constructors |
| Avoid exceptions | `Option` and `Result` for expected failures |
| Purity and immutability | Required in the core, default in the shell with contained deviations |
| Stratified design | Lower strata never depend on higher strata, use cases at the top |
| Bottom versus top | Generic and brittle at the bottom, specific and fluid at the top |
| Reaching too deep | Many operators used from a much lower stratum may signal a missing unit |
| Extensions | Extend a unit the current scope does not own, in `{unit-name}.ext.ts` |
| Testing | Through exported operators, see [testing.md](./testing.md) |
