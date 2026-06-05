# TypeScript

This foundation serves as a guideline for writing type-safe, consistent TypeScript code.

---

## Primitive types, not wrapper types

Use primitive types (`string`, `number`, `boolean`) instead of their wrapper object
counterparts (`String`, `Number`, `Boolean`). Wrapper types are objects and behave
unexpectedly in comparisons and type checks.

```ts
// ✅ DO

const name: string = 'Deta'
const count: number = 42
const isActive: boolean = true

// ❌ DON'T

const name: String = 'Deta'
const count: Number = 42
const isActive: Boolean = true
```

---

## `T[]` over `Array<T>`

Use the `T[]` shorthand for array types. It is more concise and reads naturally.

The exception is when the array type appears inside a generic. In that case, use `Array<T>`
to avoid the `[]` getting visually lost at the end.

```ts
// ✅ DO

const names: string[] = []
const counts: number[] = []

type MaybeNames = Option<Array<string>>
const result: Result<Array<User>, Error>

// ❌ DON'T

const names: Array<string> = []

type MaybeNames = Option<string[]>
```

---

## `type` over `interface`

There are two ways to define a custom type in TypeScript: `type` and `interface`. They are
almost identical, with one key difference: `interface` allows declaration merging.

Declaration merging has niche uses. One common case is augmenting a global interface from
a library or environment. Most of the time it is not needed. `type` is a closed type that
cannot be merged, which makes it safer, more consistent, and more predictable.

```ts
// ✅ DO

type SearchHotelListResponse = {
  // ...
}

// ❌ DON'T

interface SearchHotelListResponse {
  // ...
}
```

---

## Constant object over `enum`

Prefer constant objects with `as const` over native TypeScript `enum`. Both achieve a similar
result, but `enum` has a critical flaw: when its members are numbers, `Object.values` returns
both the keys and the values, producing unexpected results.

Constant objects make the separation between runtime value and compile-time type explicit,
which is consistent with how the rest of the codebase is written.

```ts
// ✅ DO

const STATUS_FLAGS = {
  ok: 'ok',
  created: 'created',
  clientError: 'clientError',
  serverError: 'serverError',
} as const

type StatusFlag = typeof STATUS_FLAGS[keyof typeof STATUS_FLAGS]
// Result: 'ok' | 'created' | 'clientError' | 'serverError'

// ❌ DON'T

enum StatusFlag {
  ok = 'ok',
  created = 'created',
  clientError = 'clientError',
  serverError = 'serverError',
}
```

The number enum pitfall:

```ts
enum StatusCode {
  success = 200,
  error = 400,
}

Object.values(StatusCode) // ["success", "error", 200, 400], not what you want

const STATUS_CODE = {
  success: 200,
  error: 400,
} as const

Object.values(STATUS_CODE) // [200, 400], correct
```

---

## Arrow function over traditional function

Arrow functions should be preferred over the `function` keyword. The `function` keyword
creates a `this` binding, which is archaic and a common source of confusion in modern
TypeScript code.

Traditional functions are only allowed when `this` context is explicitly needed.

```ts
// ✅ DO

const reduce = (state: State, action: Action) => {
  // ...
}

// ✅ DO  

// `this` context is explicitly needed here
descriptor.value = async function (...args: unknown[]) {
  // ...
}

// ❌ DON'T

function reduce(state: State, action: Action) {
  // ...
}
```

---

## Arguments should not exceed 3

If a function requires more than 3 arguments, use an object literal instead. This makes
call sites more readable and arguments order-independent.

```ts
// ✅ DO

type QueryPayload = {
  url: string
  headers: Record<string, string>
  queryParams: Record<string, unknown>
  body: Record<string, unknown>
}

const query = (params: QueryPayload) => {
  // ...
}

// ❌ DON'T

const query = (
  url: string,
  headers: Record<string, string>,
  queryParams: Record<string, unknown>,
  body: Record<string, unknown>,
) => {
  // ...
}
```

---

## Classes should be immutable

Resort to functions first. Classes are appropriate when method chaining produces a
significantly more readable API than nested function calls, for example, monadic containers
like `Result` or `Option`.

When using classes, always make them immutable. Class properties should be `readonly`, and
command methods should return a new instance instead of mutating the existing one.

```ts
// ✅ DO

class App implements Interactor {
  constructor(readonly config: Config) {}

  // Returns a new instance instead of mutating
  combineConfig(newConfig: Config) {
    return new App(this.config.combine(newConfig))
  }

  interact() {
    // ...
  }
}

// ❌ DON'T

class App {
  constructor(private config: Config) {}

  combineConfig(newConfig: Config) {
    this.config = this.config.combine(newConfig) // mutation
  }
}
```

---

## Prefer composition over inheritance

Avoid extending concrete classes. Inheritance creates tight coupling between parent and child,
makes behavior harder to trace, and breaks encapsulation over time.

Extending an **abstract class** is allowed. It is implementing a contract, not inheriting
behavior, and does not carry the same risks.

```ts
// ✅ DO

abstract class Interactor {
  abstract interact(): void
}

class App extends Interactor {
  interact() {
    // ...
  }
}

// ✅ DO 

// compose behavior instead of inheriting it
class App {
  constructor(
    readonly config: Config,
    readonly logger: Logger,
  ) {}
}

// ❌ DON'T 

// extending a concrete class
class SpecialApp extends App {
  // ...
}
```

---

## Prefer parsing over casting

Use `as T` only when you know more than TypeScript does, meaning the value has already
been validated. Blind casting bypasses the type system and pushes errors to runtime.

For truly unknown values, use `unknown` instead of `any`. `unknown` forces you to validate
before using the value, which aligns with the parsing approach. `any` silently disables type
checking and should be avoided.

```ts
// ✅ DO

const parseLocation = (value: string) => {
  if (LOCATIONS.map(String).includes(value)) {
    // Safe to cast here, already validated above
    return Option.some(value as Location)
  }
  return Option.none()
}

// ✅ DO

// use unknown for truly unknown values
const handleError = (error: unknown) => {
  if (error instanceof Error) {
    // Now safe to use
    console.error(error.message)
  }
}

// ❌ DON'T

const location = request.data.location as Location // no validation

// 🚫 NEVER

const location = request as unknown as Location // double cast
const handleError = (error: any) => { ... } // any disables type checking
```

Parsing can also be done with Zod, where the type is inferred directly from the schema:

```ts
// ✅ DO

const GetAllPaymentMethodsRequestSchema = z.object({
  amount: z.number(),
  bankId: z.string(),
})

type GetAllPaymentMethodsRequest = z.infer<typeof GetAllPaymentMethodsRequestSchema>

GetAllPaymentMethodsRequestSchema.safeParse(unknownData)
```

---

## Maximize type inference

Only annotate types explicitly when inference produces the wrong result. TypeScript is often
smart enough to infer types from values, and unnecessary annotations add noise and
redundancy.

The rule of thumb: check the inferred type first. If it's what you want, leave it. If not,
annotate.

```ts
// ✅ DO

// inference is correct, no annotation needed
const validPaymentMethods = paymentMethods.filter(method => method.isActive)

// ✅ DO 

// inference fails here, explicit annotation is needed
// without the annotation, TypeScript infers [] as never[]
const [checkedIds, setCheckedIds] = useState<Array<number>>([])

// ❌ DON'T 

// redundant annotation
const validPaymentMethods: Array<PaymentMethod> = paymentMethods.filter(
  (method: PaymentMethod) => method.isActive
)
```

---

## Edge cases

These rules may be broken at the **edge of the codebase** where it is absolutely necessary,

for example, a third-party library that requires inheritance from a concrete base class, or an
external API that forces the use of `any`. Only do this at the boundary. Once the code moves
inward, the rules apply normally.

---

## Summary

| Rule | Do | Don't |
|---|---|---|
| Primitive types | `string`, `number`, `boolean` | `String`, `Number`, `Boolean` |
| Array types | `T[]` in general, `Array<T>` inside generics | `Array<T>` everywhere |
| Type vs interface | Use `type` | Use `interface` |
| Constant object vs enum | Use constant object with `as const` | Use `enum` |
| Arrow vs traditional function | Use arrow function | Use `function` unless `this` is needed |
| Number of arguments | 3 or fewer, use object literal beyond that | More than 3 positional arguments |
| Classes | Immutable, `readonly` properties, return new instances | Mutable, properties reassigned |
| Inheritance | Extend abstract classes only | Extend concrete classes |
| Casting | Parse and validate first, cast only after | Blind `as T`, double cast |
| `any` vs `unknown` | Use `unknown` for unknown values | Use `any` |
| Type inference | Let TypeScript infer when correct | Annotate everything explicitly |
