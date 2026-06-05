# Naming

Consistent naming reduces the cognitive overhead of reading code. When names follow a
predictable pattern, the reader can focus on what the code does instead of figuring out what
kind of thing they're looking at.

---

## Variables and functions: `camelCase`

Variables, functions, parameters, type fields, and class members should be in `camelCase`.

```ts
// ✅ DO

const paymentCode = 'ABCD'

// Acronyms follow camelCase rules. Treat them as words, not letter sequences.
// "newHTTPRequest" read letter-by-letter becomes "new H T T P Request."
const newHttpRequest = {}
const userId = 123

const performAction = (payload: Payload) => {
  // ...
}

// ❌ DON'T

const payment_code = 'ABCD'
const newHTTPRequest = {}
const USER_ID = 123

const PerformAction = (pay_LoAd: Payload) => {
  // ...
}
```

---

## Classes, types, schemas, decorators: `PascalCase`

Classes, types, schemas (such as Zod schema), and decorators should be in `PascalCase`.

```ts
// ✅ DO

class AsyncResult { ... }

type RepaymentRequest = { ... }

const UserRequestSchema = z.object({ ... })

// ❌ DON'T

class asyncResult { ... }

type repayment_request = { ... }

const userRequestSchema = z.object({ ... })
```

---

## Programming constants: `MACRO_CASE`

`MACRO_CASE` applies to values that are constant in the *programming* sense, which are fixed scalars
and lookup objects whose values never change at runtime.

This does **not** apply to `const` declaration of regular variables.

```ts
// ✅ DO

const DELAY_IN_SECONDS = 90

const MERCHANT = {
  ecommerce: 'ecommerce',
  bank: 'bank',
} as const

// ❌ DON'T

const delayInSeconds = 90

const Merchant = {
  ecommerce: 'ecommerce',
  bank: 'bank',
} as const
```

---

## Type parameters: single letter

Type parameters in generic and inferred types should be written as a single letter. Start from
`T`, then `U`, `V`, and so on.

Single letters are conventional and universally readable for type parameters.

```ts
// ✅ DO

class Result<T, U> { ... }

type ProcessTransaction<T extends TransactionBase> = // ...

type SnakeToCamel<T extends string> = T extends `${infer U}_${infer V}`
  ? `${U}${Capitalize<SnakeToCamel<V>>}`
  : T

// ❌ DON'T

class Result<TSuccess, TFailure> { ... }

type ProcessTransaction<t_base extends TransactionBase> = // ...

type SnakeToCamel<theString extends string> = theString extends `${infer firstString}_${infer secondString}`
  ? `${firstString}${Capitalize<SnakeToCamel<secondString>>}`
  : theString
```

---

## No single-letter prefix

Prefixes like `T`, `I`, `o`, or `i` on non-type-parameter names are redundant. TypeScript
already surfaces type information in the editor. A prefix doesn't add signal, it just adds
noise.

This convention is a holdover from dynamically typed languages where the type wasn't
otherwise visible. TypeScript is statically typed, so it doesn't apply here.

```ts
// ✅ DO

type GamificationState = { ... }
const actionPayload = { ... }

// ❌ DON'T

type TGamificationState = { ... }
const oActionPayload = { ... }
```

---

## Clear and concise names

Names should communicate intent without being cryptic or redundant. The goal is the minimum
number of words that fully conveys the meaning.

Acronyms are allowed only for well-known concepts (`http`, `sql`, `url`, `fn`) or established
domain terms. Unknown acronyms force the reader to stop and look things up. Let's avoid them.

```ts
// ✅ DO

type GetMonolithUserDataRequest = { ... }

// HTTP and params are well-known acronyms.
// Even anonymous function parameters should be named clearly.
const filteredHttpParams = httpParams.filter(params => params.includes(ALLOWED_PARAMS))

// ❌ DON'T

// What is MUD?
type GetMudRequest = { ... }

// Call what?
const call = () => { ... }

// Too long. The implementation already makes it obvious
// Also: what is "x"?
const filteredHttpParamsOnlyWithAllowedParams = httpParams.filter(x => x.includes(ALLOWED_PARAMS))
```

---

## Booleans: name as a yes/no question

Boolean names should read as a yes/no question. The real test is: does the name answer
"true or false?" naturally? Common prefixes that achieve this are `is`, `has`, `can`,
`should`, `was`, `had`, `will`, `needs`, and `allows`, but the list is not exhaustive.

```ts
// ✅ DO

const isActive = true
const hasPermission = false
const canSubmit = true
const needsRefresh = false
const wasSubmitted = true

// ❌ DON'T

const active = true        // reads as a noun, not a question
const permission = false   // same
const submitState = true   // vague, not a question
```

---

## Discriminated union variants: `camelCase`

The discriminator field of a discriminated union should always be named `kind`, and its
values should use `camelCase`.

Using `kind` avoids confusion with TypeScript's `type` keyword, and is more semantically
precise.

`camelCase` values keep variant names consistent with the rest of the codebase, 
and make it natural for tooling to derive constructor, matcher, and guard function names 
directly from the variant name.

```ts
// ✅ DO

type Section =
  | { kind: 'hero'; title: string }
  | { kind: 'momentCard'; photo: string }
  | { kind: 'countdown'; targetDate: string }

// ❌ DON'T

type Section =
  | { type: 'hero'; title: string }        // wrong discriminator field name
  | { kind: 'Hero'; title: string }        // PascalCase
  | { kind: 'moment-card'; photo: string } // kebab-case
  | { kind: 'COUNTDOWN'; targetDate: string } // MACRO_CASE
```

---

## Edge cases

These rules may be broken at the **edge of the codebase** where a third party requires it.
For example, a backend response in `snake_case`, or a library that expects `kebab-case` field
names. Once data crosses that boundary inward, the rules apply normally.

---

## Summary

| Target | Convention |
|---|---|
| Variables | `camelCase` |
| Functions | `camelCase` |
| Type fields | `camelCase` |
| Class members | `camelCase` |
| Classes | `PascalCase` |
| Types | `PascalCase` |
| Zod schemas | `PascalCase` |
| Decorators | `PascalCase` |
| Programming constants | `MACRO_CASE` |
| Type parameters | Single letter (`T`, `U`, `V`, ...) |
| Booleans | Reads as yes/no question (`is`, `has`, `can`, ...) |
| Union variant `kind` field | `camelCase` |
