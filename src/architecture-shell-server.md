# Architecture: Server Shell

This document covers the structure and rules for the Server Shell layer, such as an HTTP
server exposing a REST or RPC API. The shell orchestrates the core via use cases,
implements contracts, and handles requests. It contains no business logic.

These conventions are not bound to any server framework. Examples use a framework-agnostic
`HttpRequest`, `HttpResponse`, and `HttpHandler`. Map them to the framework in use.

---

## Overview

The server shell is responsible for three things:

1. Implementing contracts defined by the core
2. Wiring contracts and use cases at initialization
3. Handling requests: parsing input, calling a use case, and mapping the result to a
   response

The shell may call core unit operators directly, including constructors and transformers.
They are pure and immutable, so calling them has no side effects. Operations that need the
outside world, such as reading or writing a database, go through use cases.

---

## Opinionated frameworks

Some server frameworks decide where files go and how the app is put together. For example,
NestJS builds every app out of its own modules, controllers, and providers. When you use
such a framework, follow it. But keep the framework's files thin, so the architecture still
holds:

- Framework files only connect the framework to our code. They contain no logic.
- The core never changes for a framework. Use cases and contracts stay plain functions,
  without framework decorators or imports.
- When the framework does a job this document describes, such as routing, building things
  at startup, or handling errors, let the framework do it. The rule behind the job still
  applies. For example, use cases are still built once, and still get their contracts
  through deps.

The rest of this document describes the setup for frameworks that do not decide these
things, including the directory structure.

---

## Directory structure

A server shell is a server app in `apps/`. Code shared with other server apps goes in a
`server` package, and code shared with client apps, such as DTOs, goes in a `shell`
package. See [architecture-repository.md](./architecture-repository.md).

```
/apps/api/src
  main.ts                    # entry point, builds config, contracts, use cases, and server
  /common                    # optional, shared across all modules
    /middlewares
    /units                   # e.g. Config, HttpResponse
  /module-name
    /handlers
      /place-order
        index.ts
        place-order-handler.ts
    /middlewares             # module-scoped middlewares
    /dtos                    # module-scoped DTO schema units
      /place-order-request-dto
        index.ts
        place-order-request-dto.ts
    /contracts               # shell implementations of core contracts
      /find-order-by-id
        index.ts
        find-order-by-id.postgres.ts
        find-order-by-id-deps.postgres.ts
    /units                   # module-scoped units
    /extensions              # module-scoped extensions, e.g. error to response mapping
      /place-order-error
        index.ts
        place-order-error.ext.ts
    module-name-contracts.ts # all contract implementations of the module
    module-name-use-cases.ts # all use cases of the module
    module-name-routes.ts    # all routes of the module
    index.ts
```

Each level (module, global) has its own scope for units, extensions, and middlewares. A
module-scoped unit is not accessible outside that module.

---

## Shell units

Units in the shell follow the rules in [architecture-unit.md](./architecture-unit.md). Unlike
core units, they may deviate from purity and immutability when needed, for example a unit
that wraps a database driver. Keep the deviation contained to the operators that need it.

Shell extensions may extend core units. The most common case is mapping a core error to a
response. See [Error mapping](#error-mapping).

Handlers and middlewares are framework units. They follow the framework's own best
practices, not the unit rules.

---

## Configuration

Configuration, such as environment variables, is read once at startup and parsed with a
schema into a `Config` unit. If it is invalid, the app fails to start instead of failing
later on a request.

Nothing outside initialization reads `process.env` or other configuration sources directly.
Whatever needs a configuration value receives it through deps.

```ts
// /common/units/config/config.ts

// --- Unit ---

export const ConfigSchema = z.object({
  port: z.coerce.number(),
  databaseUrl: z.string().url(),
})

export type Config = z.infer<typeof ConfigSchema>

// --- Constructors ---

export const parseConfig = (env: unknown): Result<Config, ConfigError> => ...
```

---

## Contract implementations

A contract implementation binds a contract defined in the core to something in the shell,
such as a database or an external API, so that use cases can use it. It is not a use case.

Contract implementations live in `/contracts` at the module level. They receive their own
dependencies via a constructor function, and are named after the technology they bind to,
as `{contract-name}.{tech}.ts`. See [naming.md](./naming.md#files-kebab-case).

```ts
// /contracts/find-order-by-id/find-order-by-id-deps.postgres.ts

type FindOrderByIdDeps = {
  database: Database
}
```

```ts
// /contracts/find-order-by-id/find-order-by-id.postgres.ts

export const createFindOrderById = (deps: FindOrderByIdDeps): FindOrderById =>
  async (request) => {
    const row = await deps.database.query(SELECT_ORDER_BY_ID, [request.id])
    if (row === null) return Option.none()
    return Option.some(parseOrderRow(row))
  }
```

Data coming back from the outside world is parsed into units before it is returned, never
cast.

---

## Module contracts and use cases

Each module centralizes its contract implementations in `module-name-contracts.ts`, and its
use cases in `module-name-use-cases.ts`. Both are units: a type listing everything the
module provides, and a constructor that builds it. This is the same as in the
[UI shell](./architecture-shell-ui.md#module-contracts-and-use-cases).

```ts
// order-contracts.ts

// --- Unit ---

export type OrderContracts = {
  findOrderById: FindOrderById
  saveOrder: SaveOrder
}

// --- Constructors ---

export const createOrderContracts = (deps: OrderContractsDeps): OrderContracts => ({
  findOrderById: createFindOrderById({ database: deps.database }),
  saveOrder: createSaveOrder({ database: deps.database }),
})
```

```ts
// order-use-cases.ts

// --- Unit ---

export type OrderUseCases = {
  placeOrder: PlaceOrder
  cancelOrder: CancelOrder
}

// --- Constructors ---

export const createOrderUseCases = (deps: OrderUseCasesDeps): OrderUseCases => ({
  placeOrder: createPlaceOrder({ saveOrder: deps.saveOrder, findUserById: deps.findUserById }),
  cancelOrder: createCancelOrder({ findOrderById: deps.findOrderById, saveOrder: deps.saveOrder }),
})
```

---

## Initialization

Configuration, infrastructure, contracts, use cases, and the server are built once, at
initialization in `main.ts`. Nothing is constructed per request except what the request
itself carries.

Initialization is also where modules are wired together. When a use case in one module
depends on a contract from another, it is passed in here.

```ts
// main.ts

const config = parseConfig(process.env).unwrap()
const database = createDatabase(config.databaseUrl)

const authContracts = createAuthContracts({ database })
const orderContracts = createOrderContracts({ database })

const authUseCases = createAuthUseCases(authContracts)
const orderUseCases = createOrderUseCases({
  ...orderContracts,
  findUserById: authContracts.findUserById,
})

const server = createServer({
  middlewares: [createAuthenticationMiddleware(authUseCases)],
  routes: [...createAuthRoutes(authUseCases), ...createOrderRoutes(orderUseCases)],
})

server.listen(config.port)
```

---

## Routes

Each module exposes a routes file that maps paths to handlers. It receives the module's use
cases and passes them to each handler's constructor.

```ts
// order-routes.ts

export const createOrderRoutes = (useCases: OrderUseCases): Route[] => [
  { method: 'POST', path: '/orders', handler: createPlaceOrderHandler(useCases) },
  { method: 'POST', path: '/orders/:id/cancel', handler: createCancelOrderHandler(useCases) },
]
```

---

## Handlers

A handler turns one request into one use case call. It is almost devoid of logic, the
server equivalent of a container component. A handler always:

1. Parses the input into a DTO, and from there into the use case request
2. Calls exactly one use case
3. Maps the result to a response

```ts
// /handlers/place-order/place-order-handler.ts

export const createPlaceOrderHandler = (useCases: OrderUseCases): HttpHandler =>
  async (request) => {
    const input = parsePlaceOrderRequestDto(request.body)
    if (input.isErr()) return badRequest(input.error)

    const result = await useCases.placeOrder(toPlaceOrderRequest(input.value, request.actor))

    return result.match({
      ok: order => created(toOrderResponseDto(order)),
      err: error => toHttpResponse(error),
    })
  }
```

A handler never calls a contract implementation or infrastructure directly, and never calls
more than one use case. When an endpoint needs several operations, they are composed in a
use case, which can depend on other use cases.

---

## DTOs

A DTO (data transfer object) describes what goes over the wire: a request body, query
parameters, or a response body. Each DTO is a schema unit. See
[architecture-unit.md](./architecture-unit.md#schema).

Every input is parsed with a DTO schema before anything else touches it. Parsing returns a
`Result`, and a failed parse becomes a `400`-style response. The DTO is then turned into the
use case request with the DTO unit's operators, calling core constructors where needed.

Output goes the other way: core units returned by a use case are turned into a response DTO
before they are sent. Core units never go over the wire as they are, so the core can change
its representation without changing the API.

DTOs live in the module's `/dtos` directory. When a client app also needs them, such as in a
fullstack app, they move into a `shell` package.

---

## Error mapping

Expected errors are values. A use case returns them as `Result`, and the shell maps them to
a response. The mapping lives in a shell extension of the core error unit, as a selector
named `toHttpResponse`. This keeps the mapping in one place per error, instead of repeated
in every handler.

```ts
// /extensions/place-order-error/place-order-error.ext.ts

// --- Selectors ---

export const toHttpResponse = (error: PlaceOrderError): HttpResponse => {
  switch (error.kind) {
    case 'userNotFound': return notFound({ code: 'userNotFound' })
    case 'outOfStock': return conflict({ code: 'outOfStock', productId: error.productId })
    case 'forbidden': return forbidden({ code: 'forbidden' })
  }
}
```

The `switch` covers every variant, so adding a variant to the core error fails to compile
until the mapping handles it.

Error responses expose a stable `code` taken from the error's `kind`, never internal details
such as stack traces or database messages.

---

## Authentication and authorization

Authentication, finding out who is making the request, belongs to the shell. Authorization,
deciding what they may do, is a business rule and belongs to the core.

- An authentication middleware reads the credentials from the request, such as a token, and
  calls an `authenticate` use case. It attaches the result to the request as an actor, such
  as a core `Actor` unit, or rejects the request when authentication fails.
- The handler passes the actor into the use case request.
- The use case decides whether the actor may perform the operation, and returns an error
  such as `{ kind: 'forbidden' }` when they may not.

Middlewares are thin, like handlers. They call use cases for anything beyond reading the
request.

---

## Unexpected errors

Throwing is reserved for truly unexpected, unrecoverable situations. Register a top-level
error handler with the framework, around every request. It reports the error, for example
to logs or error monitoring, and returns a generic `500` response without internal details.
It never swallows an error silently.

---

## Testing

The server shell follows the shell rules in [testing.md](./testing.md): at least 70%
coverage, with mocking used sparingly.

- **DTOs and error extensions** are units. Test them as pure operators: parse valid and
  invalid input, and map every error variant.
- **Handlers** are like container components, almost devoid of logic. Test each one by
  passing use cases built from plain functions, with one test per kind of result: a parse
  failure, a success, and an error.
- **Contract implementations** are tested against the real technology where practical, such
  as a test database. Mock only at the boundary with the outside world.

---

## Rules summary

| Rule | Description |
|---|---|
| Opinionated frameworks are followed thinly | Framework files only connect to our code, the core never changes |
| Unit operators can be called directly | Core operators are pure and immutable, so the shell may call them |
| Outside world goes through use cases | Reading or writing data always goes through a use case |
| Configuration is parsed at startup | Into a `Config` unit, never read directly elsewhere |
| Contracts and use cases are built at initialization | Once per module in `main.ts`, never per request |
| Contract implementations name their tech | `{contract-name}.{tech}.ts`, such as `find-order-by-id.postgres.ts` |
| Handlers are thin | Parse, call exactly one use case, map the result |
| Input is parsed with DTOs | Every input is parsed before use, a failed parse is a `400` response |
| Core units never go over the wire | Use cases' results are turned into response DTOs |
| Errors are mapped in extensions | `toHttpResponse` selector per core error unit |
| Authentication in the shell, authorization in the core | Middleware finds the actor, the use case decides what it may do |
| Unexpected errors are caught at the top | A top-level error handler reports them and returns a generic `500` |
| Testing | At least 70% coverage, mocking used sparingly, see [testing.md](./testing.md) |
